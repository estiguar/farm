package Base.Autofarm;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.ScheduledFuture;
import java.util.function.Function;
import java.util.stream.Collectors;

import l2jorion.game.ai.CtrlEvent;
import l2jorion.game.ai.CtrlIntention;
import l2jorion.game.ai.NextAction;
import l2jorion.game.geo.GeoData;
import l2jorion.game.handler.IItemHandler;
import l2jorion.game.handler.ItemHandler;
import l2jorion.game.handler.voice.AutoFarm;
import l2jorion.game.model.L2Character;
import l2jorion.game.model.L2Object;
import l2jorion.game.model.L2ShortCut;
import l2jorion.game.model.L2Skill;
import l2jorion.game.model.L2Skill.SkillType;
import l2jorion.game.model.L2Summon;
import l2jorion.game.model.L2WorldRegion;
import l2jorion.game.model.actor.instance.L2ChestInstance;
import l2jorion.game.model.actor.instance.L2ItemInstance;
import l2jorion.game.model.actor.instance.L2MonsterInstance;
import l2jorion.game.model.actor.instance.L2PcInstance;
import l2jorion.game.model.actor.instance.L2PetInstance;
import l2jorion.game.network.SystemMessageId;
import l2jorion.game.network.serverpackets.ActionFailed;
import l2jorion.game.network.serverpackets.ExShowScreenMessage;
import l2jorion.game.network.serverpackets.ExShowScreenMessage.SMPOS;
import l2jorion.game.thread.ThreadPoolManager;
import l2jorion.game.util.Util;
import l2jorion.util.random.Rnd;

public class AutofarmPlayerRoutine
{
	private final L2PcInstance player;
	private ScheduledFuture<?> _task;
	private L2Character committedTarget = null;
	
	public AutofarmPlayerRoutine(L2PcInstance player)
	{
		this.player = player;
	}
	
	public void start()
	{
		if (_task == null)
		{
			_task = ThreadPoolManager.getInstance().scheduleAiAtFixedRate(() -> executeRoutine(), 450, 450);
			
			player.sendPacket(new ExShowScreenMessage("Auto Farming Actived...", 5 * 1000, SMPOS.TOP_CENTER, false));
			// player.sendPacket(new SystemMessage(SystemMessageId.AUTO_FARM_ACTIVATED));
			
		}
	}
	
	public void stop()
	{
		if (_task != null)
		{
			_task.cancel(false);
			_task = null;
			
			player.sendPacket(new ExShowScreenMessage("Auto Farming Deactivated...", 5 * 1000, SMPOS.TOP_CENTER, false));
			// player.sendPacket(new SystemMessage(SystemMessageId.AUTO_FARM_DESACTIVATED));
			
		}
	}
	
	public void executeRoutine()
	{
		if (player.isNoBuffProtected() && player.getAllEffects().length <= 8)
		{
			player.sendMessage("You don't have buffs to use autofarm.");
			player.broadcastUserInfo();
			stop();
			player.setAutoFarm(false);
			AutoFarm.showAutoFarm(player);
			return;
		}
		
		calculatePotions();
		checkSpoil();
		targetEligibleCreature();
		if (player.isMageClass())
		{
			useAppropriateSpell(); // Prioriza el uso de hechizos para los magos
		}
		else if (shotcutsContainAttack())
		{
			attack(); // Si es una clase no maga y tiene la acción de ataque en los shortcuts
		}
		else
		{
			useAppropriateSpell(); // Si es una clase no maga y no tiene la acción de ataque, usa hechizos
		}
		checkSpoil();
		useAppropriateSpell();
	}
	
	private void attack()
	{
		Boolean shortcutsContainAttack = shotcutsContainAttack();
		
		if (shortcutsContainAttack)
		{
			physicalAttack();
		}
	}
	
	private void useAppropriateSpell()
	{
		L2Skill chanceSkill = nextAvailableSkill(getChanceSpells(), AutofarmSpellType.Chance);
		
		if (chanceSkill != null)
		{
			useMagicSkill(chanceSkill, false);
			return;
		}
		
		L2Skill lowLifeSkill = nextAvailableSkill(getLowLifeSpells(), AutofarmSpellType.LowLife);
		
		if (lowLifeSkill != null)
		{
			useMagicSkill(lowLifeSkill, true);
			return;
		}
		
		L2Skill attackSkill = nextAvailableSkill(getAttackSpells(), AutofarmSpellType.Attack);
		
		if (attackSkill != null)
		{
			useMagicSkill(attackSkill, false);
			return;
		}
	}
	
	public L2Skill nextAvailableSkill(List<Integer> skillIds, AutofarmSpellType spellType)
	{
		for (Integer skillId : skillIds)
		{
			L2Skill skill = player.getSkill(skillId);
			
			if (skill == null)
			{
				continue;
			}
			
			if (skill.getSkillType() == SkillType.SIGNET || skill.getSkillType() == SkillType.SIGNET_CASTTIME)
			{
				continue;
			}
			
			if (isSpoil(skillId))
			{
				if (monsterIsAlreadySpoiled())
				{
					continue;
				}
				return skill;
			}
			
			if (spellType == AutofarmSpellType.Chance && getMonsterTarget() != null)
			{
				if (getMonsterTarget().getFirstEffect(skillId) == null)
				{
					return skill;
				}
				continue;
			}
			
			if (spellType == AutofarmSpellType.LowLife && getHpPercentage() > player.getHealPercent())
			{
				break;
			}
			
			return skill;
		}
		
		return null;
	}
	
	private void checkSpoil()
	{
		if (canBeSweepedByMe() && getMonsterTarget().isDead())
		{
			L2Skill sweeper = player.getSkill(42);
			if (sweeper == null)
			{
				return;
			}
			
			useMagicSkill(sweeper, false);
		}
	}
	
	private Double getHpPercentage()
	{
		return player.getCurrentHp() * 100.0f / player.getMaxHp();
	}
	
	private Double percentageMpIsLessThan()
	{
		return player.getCurrentMp() * 100.0f / player.getMaxMp();
	}
	
	private Double percentageHpIsLessThan()
	{
		return player.getCurrentHp() * 100.0f / player.getMaxHp();
	}
	
	private List<Integer> getAttackSpells()
	{
		return getSpellsInSlots(AutofarmConstants.attackSlots);
	}
	
	private List<Integer> getSpellsInSlots(List<Integer> attackSlots)
	{
		return Arrays.stream(player.getAllShortCuts()).filter(shortcut -> shortcut.getPage() == player.getPage() && shortcut.getType() == L2ShortCut.TYPE_SKILL && attackSlots.contains(shortcut.getSlot())).map(L2ShortCut::getId).collect(Collectors.toList());
	}
	
	private List<Integer> getChanceSpells()
	{
		return getSpellsInSlots(AutofarmConstants.chanceSlots);
	}
	
	private List<Integer> getLowLifeSpells()
	{
		return getSpellsInSlots(AutofarmConstants.lowLifeSlots);
	}
	
	private boolean shotcutsContainAttack()
	{
		return Arrays.stream(player.getAllShortCuts()).anyMatch(shortcut -> shortcut.getPage() == player.getPage() && shortcut.getType() == L2ShortCut.TYPE_ACTION && (shortcut.getId() == 2 || player.isSummonAttack() && shortcut.getId() == 22));
	}
	
	private boolean monsterIsAlreadySpoiled()
	{
		return getMonsterTarget() != null && getMonsterTarget().getSpoilerId() != 0;
	}
	
	private static boolean isSpoil(Integer skillId)
	{
		return skillId == 254 || skillId == 302;
	}
	
	private boolean canBeSweepedByMe()
	{
		return getMonsterTarget() != null && getMonsterTarget().isDead() && getMonsterTarget().getSpoilerId() == player.getObjectId();
	}
	
	private void castSpellWithAppropriateTarget(L2Skill skill, Boolean forceOnSelf)
	{
		if (forceOnSelf)
		{
			L2Object oldTarget = player.getTarget();
			player.setTarget(player);
			player.useMagic(skill, false, false);
			player.setTarget(oldTarget);
			return;
		}
		
		player.useMagic(skill, false, false);
	}
	
	private void physicalAttack()
	{
		if (!(player.getTarget() instanceof L2MonsterInstance))
		{
			return;
		}
		
		L2MonsterInstance target = (L2MonsterInstance) player.getTarget();
		
		if (!player.isMageClass())
		{
			if (target.isAutoAttackable(player) && GeoData.getInstance().canSeeTarget(player, target))
			{
				if (GeoData.getInstance().canSeeTarget(player, target))
				{
					player.getAI().setIntention(CtrlIntention.AI_INTENTION_ATTACK, target);
					player.onActionRequest();
					
					if (player.isSummonAttack() && player.getPet() != null)
					{
						// Siege Golem's
						if (player.getPet().getNpcId() >= 14702 && player.getPet().getNpcId() <= 14798 || player.getPet().getNpcId() >= 14839 && player.getPet().getNpcId() <= 14869)
						{
							return;
						}
						
						L2Summon activeSummon = player.getPet();
						activeSummon.setTarget(target);
						activeSummon.getAI().setIntention(CtrlIntention.AI_INTENTION_ATTACK, target);
						
						int[] summonAttackSkills =
						{
							4261,
							4068,
							4137,
							4260,
							4708,
							4709,
							4710,
							4712,
							5135,
							5138,
							5141,
							5442,
							5444,
							6095,
							6096,
							6041,
							6044
						};
						if (Rnd.get(100) < player.getSummonSkillPercent())
						{
							for (int skillId : summonAttackSkills)
							{
								useMagicSkillBySummon(skillId, target);
							}
						}
					}
				}
			}
			else
			{
				if (target.isAutoAttackable(player) && GeoData.getInstance().canSeeTarget(player, target))
				{
					if (GeoData.getInstance().canSeeTarget(player, target))
					{
						player.getAI().setIntention(CtrlIntention.AI_INTENTION_FOLLOW, target);
					}
				}
			}
		}
		else
		{
			if (player.isSummonAttack() && player.getPet() != null)
			{
				// Siege Golem's
				if (player.getPet().getNpcId() >= 14702 && player.getPet().getNpcId() <= 14798 || player.getPet().getNpcId() >= 14839 && player.getPet().getNpcId() <= 14869)
				{
					return;
				}
				
				L2Summon activeSummon = player.getPet();
				activeSummon.setTarget(target);
				activeSummon.getAI().setIntention(CtrlIntention.AI_INTENTION_ATTACK, target);
				
				int[] summonAttackSkills =
				{
					4261,
					4068,
					4137,
					4260,
					4708,
					4709,
					4710,
					4712,
					5135,
					5138,
					5141,
					5442,
					5444,
					6095,
					6096,
					6041,
					6044
				};
				if (Rnd.get(100) < player.getSummonSkillPercent())
				{
					for (int skillId : summonAttackSkills)
					{
						useMagicSkillBySummon(skillId, target);
					}
				}
			}
		}
	}
	
	public void targetEligibleCreature()
	{
		if (player.getTarget() == null)
		{
			selectNewTarget(); // Llamada a selectNewTarget si el jugador no tiene un objetivo
			return;
		}
		
		if (committedTarget != null)
		{
			if (!isSameInstance(player, committedTarget) || (committedTarget.isDead() && GeoData.getInstance().canSeeTarget(player, committedTarget)))
			{
				committedTarget = null;
				selectNewTarget(); // Llamada a selectNewTarget después de que el objetivo actual muere o es de otra instancia
				return;
			}
			else if (!committedTarget.isDead() && GeoData.getInstance().canSeeTarget(player, committedTarget))
			{
				attack();
				return;
			}
			else if (!GeoData.getInstance().canSeeTarget(player, committedTarget))
			{
				committedTarget = null;
				selectNewTarget(); // Buscar otro objetivo si el jugador no puede ver al objetivo actual
				return;
			}
			player.getAI().setIntention(CtrlIntention.AI_INTENTION_FOLLOW, committedTarget);
			committedTarget = null;
			player.setTarget(null);
		}
		
		if (committedTarget instanceof L2Summon)
		{
			return;
		}
		
		List<L2MonsterInstance> targets = getKnownMonstersInRadius(player, player.getRadius(), creature -> GeoData.getInstance().canSeeTarget(player.getX(), player.getY(), player.getZ(), creature.getX(), creature.getY(), creature.getZ()) && !player.ignoredMonsterContain(creature.getNpcId())
			&& !creature.isRaidMinion() && !creature.isRaid() && !creature.isDead() && !(creature instanceof L2ChestInstance) && !(player.isAntiKsProtected() && creature.getTarget() != null && creature.getTarget() != player && creature.getTarget() != player.getPet())
			&& isSameInstance(player, creature));
		
		if (targets.isEmpty())
		{
			return;
		}
		
		L2MonsterInstance closestTarget = targets.stream().min((o1, o2) -> Integer.compare((int) Math.sqrt(player.getDistanceSq(o1)), (int) Math.sqrt(player.getDistanceSq(o2)))).get();
		
		committedTarget = closestTarget;
		player.setTarget(closestTarget);
	}
	
	// Función para verificar si dos objetos pertenecen a la misma instancia
	private static boolean isSameInstance(L2Object obj1, L2Object obj2)
	{
		return obj1.getInstanceId() == obj2.getInstanceId();
	}
	
	// Función para seleccionar un nuevo objetivo
	private void selectNewTarget()
	{
		List<L2MonsterInstance> targets = getKnownMonstersInRadius(player, player.getRadius(), creature -> GeoData.getInstance().canSeeTarget(player.getX(), player.getY(), player.getZ(), creature.getX(), creature.getY(), creature.getZ()) && !player.ignoredMonsterContain(creature.getNpcId())
			&& !creature.isRaidMinion() && !creature.isRaid() && !creature.isDead() && !(creature instanceof L2ChestInstance) && !(player.isAntiKsProtected() && creature.getTarget() != null && creature.getTarget() != player && creature.getTarget() != player.getPet())
			&& isSameInstance(player, creature));
		
		if (targets.isEmpty())
		{
			return;
		}
		
		L2MonsterInstance closestTarget = targets.stream().min((o1, o2) -> Integer.compare((int) Math.sqrt(player.getDistanceSq(o1)), (int) Math.sqrt(player.getDistanceSq(o2)))).get();
		
		committedTarget = closestTarget;
		player.setTarget(closestTarget);
	}
	
	public final static List<L2MonsterInstance> getKnownMonstersInRadius(L2PcInstance player, int radius, Function<L2MonsterInstance, Boolean> condition)
	{
		final L2WorldRegion region = player.getWorldRegion();
		if (region == null)
		{
			return Collections.emptyList();
		}
		
		final List<L2MonsterInstance> result = new ArrayList<>();
		
		for (L2WorldRegion reg : region.getSurroundingRegions())
		{
			for (L2Object obj : reg.getVisibleObjects().values())
			{
				if (!(obj instanceof L2MonsterInstance) || !Util.checkIfInRange(radius, player, obj, true) || !condition.apply((L2MonsterInstance) obj))
				{
					continue;
				}
				
				result.add((L2MonsterInstance) obj);
			}
		}
		
		return result;
	}
	
	public L2MonsterInstance getMonsterTarget()
	{
		if (!(player.getTarget() instanceof L2MonsterInstance))
		{
			return null;
		}
		
		return (L2MonsterInstance) player.getTarget();
	}
	
	private void useMagicSkill(L2Skill skill, Boolean forceOnSelf)
	{
		if (skill.getSkillType() == SkillType.RECALL && player.getKarma() > 0)
		{
			player.sendPacket(ActionFailed.STATIC_PACKET);
			return;
		}
		
		if (skill.isToggle() && player.isMounted())
		{
			player.sendPacket(ActionFailed.STATIC_PACKET);
			return;
		}
		
		if (player.isOutOfControl())
		{
			player.sendPacket(ActionFailed.STATIC_PACKET);
			return;
		}
		
		if (player.isAttackingNow())
		{
			player.getAI().setNextAction(new NextAction(CtrlEvent.EVT_READY_TO_ACT, CtrlIntention.AI_INTENTION_CAST, () -> castSpellWithAppropriateTarget(skill, forceOnSelf)));
		}
		else
		{
			castSpellWithAppropriateTarget(skill, forceOnSelf);
		}
	}
	
	private void useMagicSkillBySummon(int skillId, L2Object target)
	{
		// No owner, or owner in shop mode.
		if (player == null || player.isInStoreMode())
		{
			return;
		}
		
		final L2Summon activeSummon = player.getPet();
		if (activeSummon == null)
		{
			return;
		}
		
		// Pet which is 20 levels higher than owner.
		if (activeSummon instanceof L2PetInstance && activeSummon.getLevel() - player.getLevel() > 20)
		{
			player.sendPacket(SystemMessageId.PET_TOO_HIGH_TO_CONTROL);
			return;
		}
		
		// Out of control pet.
		if (activeSummon.isOutOfControl())
		{
			player.sendPacket(SystemMessageId.PET_REFUSING_ORDER);
			return;
		}
		
		// Verify if the launched skill is mastered by the summon.
		final L2Skill skill = activeSummon.getSkill(skillId);
		if (skill == null)
		{
			return;
		}
		
		// Can't launch offensive skills on owner.
		if (skill.isOffensive() && player == target)
		{
			return;
		}
		
		activeSummon.setTarget(target);
		return;
	}
	
	private void calculatePotions()
	{
		if (percentageHpIsLessThan() < player.getHpPotionPercentage())
		{
			forceUseItem(1539);
		}
		
		if (percentageMpIsLessThan() < player.getMpPotionPercentage())
		{
			forceUseItem(728);
		}
	}
	
	private void forceUseItem(int itemId)
	{
		final L2ItemInstance potion = player.getInventory().getItemByItemId(itemId);
		if (potion == null)
		{
			return;
		}
		
		final IItemHandler handler = ItemHandler.getInstance().getItemHandler(potion.getItemId());
		if (handler != null)
		{
			handler.useItem(player, potion);
		}
	}
}
\ No newline at end of file
diff --git src/Base/Autofarm/AutofarmSpellType.java src/Base/Autofarm/AutofarmSpellType.java
new file mode 100644
index 0000000..e24f40c
--- /dev/null
+++ src/Base/Autofarm/AutofarmSpellType.java
@@ -0,0 +1,8 @@
+package Base.Autofarm;
+
+public enum AutofarmSpellType
+{
+    Attack,
+    Chance,
+    LowLife    
+}
\ No newline at end of file
diff --git src/l2jorion/game/handler/VoicedCommandHandler.java src/l2jorion/game/handler/VoicedCommandHandler.java
index 490d47c..634649f 100644
--- src/l2jorion/game/handler/VoicedCommandHandler.java
+++ src/l2jorion/game/handler/VoicedCommandHandler.java
@@ -6,6 +6,7 @@
 import l2jorion.Config;
 import l2jorion.game.autofarm.AutofarmCommandHandler;
 import l2jorion.game.handler.custom.CustomBypassHandler;
+import l2jorion.game.handler.voice.AutoFarm;
 import l2jorion.game.handler.voice.DressMe;
 import l2jorion.game.handler.voice.Event_CTF;
 import l2jorion.game.handler.voice.Event_DM;
@@ -90,6 +91,7 @@
 		}
 		
 		registerVoicedCommandHandler(new ExpireItems());
+		registerVoicedCommandHandler(new AutoFarm());
 		CustomBypassHandler.getInstance().registerCustomBypassHandler(new ExpireItems());
 		
 		LOG.info("VoicedCommandHandler: Loaded " + _datatable.size() + " handlers");
diff --git src/l2jorion/game/handler/voice/AutoFarm.java src/l2jorion/game/handler/voice/AutoFarm.java
new file mode 100644
index 0000000..c5a9ca1
--- /dev/null
+++ src/l2jorion/game/handler/voice/AutoFarm.java
@@ -0,0 +1,333 @@
+package l2jorion.game.handler.voice;
+
+import java.util.StringTokenizer;
+
+import Base.Autofarm.AutofarmPlayerRoutine;
+import l2jorion.game.handler.IVoicedCommandHandler;
+import l2jorion.game.model.L2Object;
+import l2jorion.game.model.actor.instance.L2MonsterInstance;
+import l2jorion.game.model.actor.instance.L2PcInstance;
+import l2jorion.game.network.serverpackets.ExShowScreenMessage;
+import l2jorion.game.network.serverpackets.ExShowScreenMessage.SMPOS;
+import l2jorion.game.network.serverpackets.NpcHtmlMessage;
+import l2jorion.util.StringUtil;
+
+public class AutoFarm implements IVoicedCommandHandler
+{
+	private final String[] VOICED_COMMANDS =
+	{
+		"autofarm",
+		"enableAutoFarm",
+		"radiusAutoFarm",
+		"pageAutoFarm",
+		"enableBuffProtect",
+		"healAutoFarm",
+		"hpAutoFarm",
+		"mpAutoFarm",
+		"enableAntiKs",
+		"enableSummonAttack",
+		"summonSkillAutoFarm",
+		"ignoreMonster",
+		"activeMonster"
+	};
+	
+	@Override
+	public boolean useVoicedCommand(final String command, final L2PcInstance activeChar, final String args)
+	{
+		final AutofarmPlayerRoutine bot = activeChar.getBot();
+		
+		if (command.startsWith("autofarm"))
+		{
+			showAutoFarm(activeChar);
+		}
+		
+		if (command.startsWith("radiusAutoFarm"))
+		{
+			StringTokenizer st = new StringTokenizer(command, " ");
+			st.nextToken();
+			try
+			{
+				String param = st.nextToken();
+				
+				if (param.startsWith("inc_radius"))
+				{
+					activeChar.setRadius(activeChar.getRadius() + 200);
+					showAutoFarm(activeChar);
+				}
+				else if (param.startsWith("dec_radius"))
+				{
+					activeChar.setRadius(activeChar.getRadius() - 200);
+					showAutoFarm(activeChar);
+				}
+				activeChar.saveAutoFarmSettings();
+			}
+			catch (Exception e)
+			{
+				e.printStackTrace();
+			}
+		}
+		
+		if (command.startsWith("pageAutoFarm"))
+		{
+			StringTokenizer st = new StringTokenizer(command, " ");
+			st.nextToken();
+			try
+			{
+				String param = st.nextToken();
+				
+				if (param.startsWith("inc_page"))
+				{
+					activeChar.setPage(activeChar.getPage() + 1);
+					showAutoFarm(activeChar);
+				}
+				else if (param.startsWith("dec_page"))
+				{
+					activeChar.setPage(activeChar.getPage() - 1);
+					showAutoFarm(activeChar);
+				}
+				activeChar.saveAutoFarmSettings();
+			}
+			catch (Exception e)
+			{
+				e.printStackTrace();
+			}
+		}
+		
+		if (command.startsWith("healAutoFarm"))
+		{
+			StringTokenizer st = new StringTokenizer(command, " ");
+			st.nextToken();
+			try
+			{
+				String param = st.nextToken();
+				
+				if (param.startsWith("inc_heal"))
+				{
+					activeChar.setHealPercent(activeChar.getHealPercent() + 10);
+					showAutoFarm(activeChar);
+				}
+				else if (param.startsWith("dec_heal"))
+				{
+					activeChar.setHealPercent(activeChar.getHealPercent() - 10);
+					showAutoFarm(activeChar);
+				}
+				activeChar.saveAutoFarmSettings();
+			}
+			catch (Exception e)
+			{
+				e.printStackTrace();
+			}
+		}
+		
+		if (command.startsWith("hpAutoFarm"))
+		{
+			StringTokenizer st = new StringTokenizer(command, " ");
+			st.nextToken();
+			try
+			{
+				String param = st.nextToken();
+				
+				if (param.contains("inc_hp_pot"))
+				{
+					activeChar.setHpPotionPercentage(activeChar.getHpPotionPercentage() + 5);
+					showAutoFarm(activeChar);
+				}
+				else if (param.contains("dec_hp_pot"))
+				{
+					activeChar.setHpPotionPercentage(activeChar.getHpPotionPercentage() - 5);
+					showAutoFarm(activeChar);
+				}
+				activeChar.saveAutoFarmSettings();
+			}
+			catch (Exception e)
+			{
+				e.printStackTrace();
+			}
+		}
+		
+		if (command.startsWith("mpAutoFarm"))
+		{
+			StringTokenizer st = new StringTokenizer(command, " ");
+			st.nextToken();
+			try
+			{
+				String param = st.nextToken();
+				
+				if (param.contains("inc_mp_pot"))
+				{
+					activeChar.setMpPotionPercentage(activeChar.getMpPotionPercentage() + 5);
+					showAutoFarm(activeChar);
+				}
+				else if (param.contains("dec_mp_pot"))
+				{
+					activeChar.setMpPotionPercentage(activeChar.getMpPotionPercentage() - 5);
+					showAutoFarm(activeChar);
+				}
+				activeChar.saveAutoFarmSettings();
+			}
+			catch (Exception e)
+			{
+				e.printStackTrace();
+			}
+		}
+		
+		if (command.startsWith("enableAutoFarm"))
+		{
+			if (activeChar.isAutoFarm())
+			{
+				bot.stop();
+				activeChar.sendMessage("AutoFarm Desativado.");
+				activeChar.setAutoFarm(false);
+			}
+			else
+			{
+				bot.start();
+				activeChar.sendMessage("AutoFarm Ativo.");
+				activeChar.setAutoFarm(true);
+			}
+			
+			showAutoFarm(activeChar);
+		}
+		
+		if (command.startsWith("enableBuffProtect"))
+		{
+			activeChar.setNoBuffProtection(!activeChar.isNoBuffProtected());
+			showAutoFarm(activeChar);
+			activeChar.saveAutoFarmSettings();
+		}
+		
+		if (command.startsWith("enableAntiKs"))
+		{
+			activeChar.setAntiKsProtection(!activeChar.isAntiKsProtected());
+			
+			if (activeChar.isAntiKsProtected())
+			{
+				// activeChar.sendPacket(new SystemMessage(SystemMessageId.ACTIVATE_RESPECT_HUNT));
+				activeChar.sendMessage("Respect Hunt On.");
+				activeChar.sendPacket(new ExShowScreenMessage("Respct Hunt On", 3 * 1000, SMPOS.TOP_CENTER, false));
+			}
+			else
+			{
+				// activeChar.sendPacket(new SystemMessage(SystemMessageId.DESACTIVATE_RESPECT_HUNT));
+				activeChar.sendMessage("Respect Hunt Of.");
+				activeChar.sendPacket(new ExShowScreenMessage("Respct Hunt Off", 3 * 1000, SMPOS.TOP_CENTER, false));
+			}
+			
+			activeChar.saveAutoFarmSettings();
+			showAutoFarm(activeChar);
+		}
+		
+		if (command.startsWith("enableSummonAttack"))
+		{
+			activeChar.setSummonAttack(!activeChar.isSummonAttack());
+			if (activeChar.isSummonAttack())
+			{
+				
+				activeChar.sendPacket(new ExShowScreenMessage("Auto Farm Summon Attack On", 3 * 1000, SMPOS.TOP_CENTER, false));
+			}
+			else
+			{
+				
+				activeChar.sendPacket(new ExShowScreenMessage("Auto Farm Summon Attack Off", 3 * 1000, SMPOS.TOP_CENTER, false));
+			}
+			activeChar.saveAutoFarmSettings();
+			showAutoFarm(activeChar);
+		}
+		
+		if (command.startsWith("summonSkillAutoFarm"))
+		{
+			StringTokenizer st = new StringTokenizer(command, " ");
+			st.nextToken();
+			try
+			{
+				String param = st.nextToken();
+				
+				if (param.startsWith("inc_summonSkill"))
+				{
+					activeChar.setSummonSkillPercent(activeChar.getSummonSkillPercent() + 10);
+					showAutoFarm(activeChar);
+				}
+				else if (param.startsWith("dec_summonSkill"))
+				{
+					activeChar.setSummonSkillPercent(activeChar.getSummonSkillPercent() - 10);
+					showAutoFarm(activeChar);
+				}
+				activeChar.saveAutoFarmSettings();
+			}
+			catch (Exception e)
+			{
+				e.printStackTrace();
+			}
+		}
+		
+		if (command.startsWith("ignoreMonster"))
+		{
+			int monsterId = 0;
+			L2Object target = activeChar.getTarget();
+			if (target instanceof L2MonsterInstance)
+			{
+				monsterId = ((L2MonsterInstance) target).getNpcId();
+			}
+			
+			if (target == null)
+			{
+				activeChar.sendMessage("You dont have a target");
+				return false;
+			}
+			
+			activeChar.sendMessage(target.getName() + " has been added to the ignore list.");
+			activeChar.ignoredMonster(monsterId);
+		}
+		
+		if (command.startsWith("activeMonster"))
+		{
+			int monsterId = 0;
+			L2Object target = activeChar.getTarget();
+			if (target instanceof L2MonsterInstance)
+			{
+				monsterId = ((L2MonsterInstance) target).getNpcId();
+			}
+			
+			if (target == null)
+			{
+				activeChar.sendMessage("You dont have a target");
+				return false;
+			}
+			
+			activeChar.sendMessage(target.getName() + " has been removed from the ignore list.");
+			activeChar.activeMonster(monsterId);
+		}
+		
+		return false;
+	}
+	
+	private static final String ACTIVED = "<font color=00FF00>STARTED</font>";
+	private static final String DESATIVED = "<font color=FF0000>STOPPED</font>";
+	private static final String STOP = "STOP";
+	private static final String START = "START";
+	
+	public static void showAutoFarm(L2PcInstance activeChar)
+	{
+		NpcHtmlMessage html = new NpcHtmlMessage(0);
+		html.setFile("data/html/mods/menu/AutoFarm.htm");
+		html.replace("%player%", activeChar.getName());
+		html.replace("%page%", StringUtil.formatNumber(activeChar.getPage() + 1));
+		html.replace("%heal%", StringUtil.formatNumber(activeChar.getHealPercent()));
+		html.replace("%radius%", StringUtil.formatNumber(activeChar.getRadius()));
+		html.replace("%summonSkill%", StringUtil.formatNumber(activeChar.getSummonSkillPercent()));
+		html.replace("%hpPotion%", StringUtil.formatNumber(activeChar.getHpPotionPercentage()));
+		html.replace("%mpPotion%", StringUtil.formatNumber(activeChar.getMpPotionPercentage()));
+		html.replace("%noBuff%", activeChar.isNoBuffProtected() ? "back=L2UI.CheckBox_checked fore=L2UI.CheckBox_checked" : "back=L2UI.CheckBox fore=L2UI.CheckBox");
+		html.replace("%summonAtk%", activeChar.isSummonAttack() ? "back=L2UI.CheckBox_checked fore=L2UI.CheckBox_checked" : "back=L2UI.CheckBox fore=L2UI.CheckBox");
+		html.replace("%antiKs%", activeChar.isAntiKsProtected() ? "back=L2UI.CheckBox_checked fore=L2UI.CheckBox_checked" : "back=L2UI.CheckBox fore=L2UI.CheckBox");
+		html.replace("%autofarm%", activeChar.isAutoFarm() ? ACTIVED : DESATIVED);
+		html.replace("%button%", activeChar.isAutoFarm() ? STOP : START);
+		activeChar.sendPacket(html);
+	}
+	
+	@Override
+	public String[] getVoicedCommandList()
+	{
+		return VOICED_COMMANDS;
+	}
+}
\ No newline at end of file
diff --git src/l2jorion/game/model/L2Character.java src/l2jorion/game/model/L2Character.java
index 7d9ef22..17b7a6e 100644
--- src/l2jorion/game/model/L2Character.java
+++ src/l2jorion/game/model/L2Character.java
@@ -1997,7 +1997,7 @@
 	}
 	
 	// Only for monsters
-	protected void useMagic(L2Skill skill)
+	public void useMagic(L2Skill skill)
 	{
 		if (skill == null || isDead() || isAllSkillsDisabled() || skill.isPassive() || isCastingNow() || isAlikeDead() || skill.isChance())
 		{
diff --git src/l2jorion/game/model/actor/instance/L2PcInstance.java src/l2jorion/game/model/actor/instance/L2PcInstance.java
index fd67a67..66e8716 100644
--- src/l2jorion/game/model/actor/instance/L2PcInstance.java
+++ src/l2jorion/game/model/actor/instance/L2PcInstance.java
@@ -22,6 +22,7 @@
 import java.util.concurrent.TimeUnit;
 import java.util.concurrent.locks.ReentrantLock;
 
+import Base.Autofarm.AutofarmPlayerRoutine;
 import javolution.text.TextBuilder;
 import javolution.util.FastMap;
 import javolution.util.FastSet;
@@ -15965,6 +15966,7 @@
 	public synchronized void deleteMe()
 	{
 		AutofarmManager.INSTANCE.onPlayerLogout(this);
+		_bot.stop();
 		
 		if (inObserverMode())
 		{
@@ -20224,4 +20226,234 @@
 	{
 		return true;
 	}
+	
+	// ------------
+	// Autofarm
+	// ------------
+	
+	private boolean _autoFarm;
+	
+	public void setAutoFarm(boolean comm)
+	{
+		_autoFarm = comm;
+	}
+	
+	public boolean isAutoFarm()
+	{
+		return _autoFarm;
+	}
+	
+	private int autoFarmRadius = 1200;
+	
+	public void setRadius(int value)
+	{
+		autoFarmRadius = Util.limit(value, 200, 3000);
+	}
+	
+	public int getRadius()
+	{
+		return autoFarmRadius;
+	}
+	
+	private int autoFarmShortCut = 9;
+	
+	public void setPage(int value)
+	{
+		autoFarmShortCut = Util.limit(value, 0, 9);
+	}
+	
+	public int getPage()
+	{
+		return autoFarmShortCut;
+	}
+	
+	private int autoFarmHealPercente = 30;
+	
+	public void setHealPercent(int value)
+	{
+		autoFarmHealPercente = Util.limit(value, 20, 90);
+	}
+	
+	public int getHealPercent()
+	{
+		return autoFarmHealPercente;
+	}
+	
+	private boolean autoFarmBuffProtection = false;
+	
+	public void setNoBuffProtection(boolean val)
+	{
+		autoFarmBuffProtection = val;
+	}
+	
+	public boolean isNoBuffProtected()
+	{
+		return autoFarmBuffProtection;
+	}
+	
+	private boolean autoAntiKsProtection = false;
+	
+	public void setAntiKsProtection(boolean val)
+	{
+		autoAntiKsProtection = val;
+	}
+	
+	public boolean isAntiKsProtected()
+	{
+		return autoAntiKsProtection;
+	}
+	
+	private boolean autoFarmSummonAttack = false;
+	
+	public void setSummonAttack(boolean val)
+	{
+		autoFarmSummonAttack = val;
+	}
+	
+	public boolean isSummonAttack()
+	{
+		return autoFarmSummonAttack;
+	}
+	
+	private int autoFarmSummonSkillPercente = 0;
+	
+	public void setSummonSkillPercent(int value)
+	{
+		autoFarmSummonSkillPercente = Util.limit(value, 0, 90);
+	}
+	
+	public int getSummonSkillPercent()
+	{
+		return autoFarmSummonSkillPercente;
+	}
+	
+	private int hpPotionPercent = 60;
+	private int mpPotionPercent = 60;
+	
+	public void setHpPotionPercentage(int value)
+	{
+		hpPotionPercent = Util.limit(value, 0, 100);
+	}
+	
+	public int getHpPotionPercentage()
+	{
+		return hpPotionPercent;
+	}
+	
+	public void setMpPotionPercentage(int value)
+	{
+		mpPotionPercent = Util.limit(value, 0, 100);
+	}
+	
+	public int getMpPotionPercentage()
+	{
+		return mpPotionPercent;
+	}
+	
+	private List<Integer> _ignoredMonster = new ArrayList<>();
+	
+	public void ignoredMonster(Integer npcId)
+	{
+		_ignoredMonster.add(npcId);
+	}
+	
+	public void activeMonster(Integer npcId)
+	{
+		if (_ignoredMonster.contains(npcId))
+		{
+			_ignoredMonster.remove(npcId);
+		}
+	}
+	
+	public boolean ignoredMonsterContain(int npcId)
+	{
+		return _ignoredMonster.contains(npcId);
+	}
+	
+	private AutofarmPlayerRoutine _bot = new AutofarmPlayerRoutine(this);
+	
+	public AutofarmPlayerRoutine getBot()
+	{
+		if (_bot == null)
+		{
+			_bot = new AutofarmPlayerRoutine(this);
+		}
+		
+		return _bot;
+	}
+	
+	// Función para guardar los valores del autofarm en la base de datos
+	public void saveAutoFarmSettings()
+	{
+		try (Connection con = L2DatabaseFactory.getInstance().getConnection())
+		{
+			String updateSql = "REPLACE INTO character_autofarm (char_id, char_name,  radius, short_cut, heal_percent, buff_protection, anti_ks_protection, summon_attack, summon_skill_percent, hp_potion_percent, mp_potion_percent) VALUES (?, ?, ?,  ?, ?, ?, ?, ?, ?, ?, ?)";
+			try (PreparedStatement updateStatement = con.prepareStatement(updateSql))
+			{
+				updateStatement.setInt(1, getObjectId()); // char_id
+				updateStatement.setString(2, getName()); // char_name
+				
+				updateStatement.setInt(3, autoFarmRadius);
+				updateStatement.setInt(4, autoFarmShortCut);
+				updateStatement.setInt(5, autoFarmHealPercente);
+				updateStatement.setBoolean(6, autoFarmBuffProtection);
+				updateStatement.setBoolean(7, autoAntiKsProtection);
+				updateStatement.setBoolean(8, autoFarmSummonAttack);
+				updateStatement.setInt(9, autoFarmSummonSkillPercente);
+				updateStatement.setInt(10, hpPotionPercent);
+				updateStatement.setInt(11, mpPotionPercent);
+				updateStatement.executeUpdate();
+			}
+		}
+		catch (SQLException e)
+		{
+			e.printStackTrace();
+		}
+	}
+	
+	public void loadAutoFarmSettings()
+	{
+		try (Connection con = L2DatabaseFactory.getInstance().getConnection())
+		{
+			String selectSql = "SELECT * FROM character_autofarm WHERE char_id = ?";
+			try (PreparedStatement selectStatement = con.prepareStatement(selectSql))
+			{
+				selectStatement.setInt(1, getObjectId()); // char_id
+				try (ResultSet resultSet = selectStatement.executeQuery())
+				{
+					if (resultSet.next())
+					{
+						
+						autoFarmRadius = resultSet.getInt("radius");
+						autoFarmShortCut = resultSet.getInt("short_cut");
+						autoFarmHealPercente = resultSet.getInt("heal_percent");
+						autoFarmBuffProtection = resultSet.getBoolean("buff_protection");
+						autoAntiKsProtection = resultSet.getBoolean("anti_ks_protection");
+						autoFarmSummonAttack = resultSet.getBoolean("summon_attack");
+						autoFarmSummonSkillPercente = resultSet.getInt("summon_skill_percent");
+						hpPotionPercent = resultSet.getInt("hp_potion_percent");
+						mpPotionPercent = resultSet.getInt("mp_potion_percent");
+					}
+					else
+					{
+						// Si no se encontraron registros, cargar valores predeterminados
+						
+						autoFarmRadius = 1200;
+						autoFarmShortCut = 9;
+						autoFarmHealPercente = 30;
+						autoFarmBuffProtection = false;
+						autoAntiKsProtection = false;
+						autoFarmSummonAttack = false;
+						autoFarmSummonSkillPercente = 0;
+						hpPotionPercent = 60;
+						mpPotionPercent = 60;
+					}
+				}
+			}
+		}
+		catch (SQLException e)
+		{
+			e.printStackTrace();
+		}
+	}
 }
\ No newline at end of file
diff --git src/l2jorion/game/network/clientpackets/EnterWorld.java src/l2jorion/game/network/clientpackets/EnterWorld.java
index cbd2c16..30214e5 100644
--- src/l2jorion/game/network/clientpackets/EnterWorld.java
+++ src/l2jorion/game/network/clientpackets/EnterWorld.java
@@ -282,6 +282,7 @@
 		notifySponsorOrApprentice(activeChar);
 		
 		activeChar.onPlayerEnter();
+		activeChar.loadAutoFarmSettings();
 		
 		if (Config.PCB_ENABLE)
 		{
diff --git src/l2jorion/game/network/clientpackets/RequestBypassToServer.java src/l2jorion/game/network/clientpackets/RequestBypassToServer.java
index 4a02d48..9223bcd 100644
--- src/l2jorion/game/network/clientpackets/RequestBypassToServer.java
+++ src/l2jorion/game/network/clientpackets/RequestBypassToServer.java
@@ -34,6 +34,8 @@
 import l2jorion.game.datatables.xml.DressMeData;
 import l2jorion.game.handler.AdminCommandHandler;
 import l2jorion.game.handler.IAdminCommandHandler;
+import l2jorion.game.handler.IVoicedCommandHandler;
+import l2jorion.game.handler.VoicedCommandHandler;
 import l2jorion.game.handler.custom.CustomBypassHandler;
 import l2jorion.game.handler.voice.Vote;
 import l2jorion.game.handler.vote.Brasil;
@@ -116,6 +118,23 @@
 		
 		try
 		{
+			
+			if (bp.bypass.startsWith("voiced_"))
+			{
+				String command = bp.bypass.split(" ")[0];
+				
+				IVoicedCommandHandler ach = VoicedCommandHandler.getInstance().getVoicedCommandHandler(bp.bypass.substring(7));
+				
+				if (ach == null)
+				{
+					activeChar.sendMessage("The command " + command.substring(7) + " does not exist!");
+					LOG.warn("No handler registered for command '" + bp.bypass + "'");
+					return;
+				}
+				
+				ach.useVoicedCommand(bp.bypass.substring(7), activeChar, null);
+			}
+			
 			if (bp.bypass.startsWith("admin_"))
 			{
 				String command;
diff --git src/l2jorion/game/network/serverpackets/Die.java src/l2jorion/game/network/serverpackets/Die.java
index cb67140..d0d7855 100644
--- src/l2jorion/game/network/serverpackets/Die.java
+++ src/l2jorion/game/network/serverpackets/Die.java
@@ -18,6 +18,7 @@
  */
 package l2jorion.game.network.serverpackets;
 
+import Base.Autofarm.AutofarmPlayerRoutine;
 import l2jorion.game.managers.CastleManager;
 import l2jorion.game.managers.FortManager;
 import l2jorion.game.model.L2Attackable;
@@ -51,9 +52,18 @@
 		if (cha instanceof L2PcInstance)
 		{
 			L2PcInstance player = (L2PcInstance) cha;
+			final AutofarmPlayerRoutine bot = player.getBot();
 			_allowFixedRes = (player.getAccessLevel().allowFixedRes() || player.isInsideZone(ZoneId.ZONE_RANDOM));
 			_clan = player.getClan();
 			_canTeleport = !((TvT.is_started() && player._inEventTvT) || (DM.is_started() && player._inEventDM) || (CTF.is_started() && player._inEventCTF) || player.isInFunEvent() || player.isInArenaEvent() || player.isPendingRevive());
+			
+			if (player.isAutoFarm())
+			{
+				bot.stop();
+				player.sendMessage("Morto, Auto Farm Desativado.");
+				player.setAutoFarm(false);
+			}
+			
 		}
 		
 		_charObjId = cha.getObjectId();
diff --git src/l2jorion/game/util/Util.java src/l2jorion/game/util/Util.java
index 4906a9b..495d9f0 100644
--- src/l2jorion/game/util/Util.java
+++ src/l2jorion/game/util/Util.java
@@ -534,4 +534,15 @@
 		}
 		return true;
 	}
+	
+	/**
+	 * @param numToTest : The number to test.
+	 * @param min : The minimum limit.
+	 * @param max : The maximum limit.
+	 * @return the number or one of the limit (mininum / maximum).
+	 */
+	public static int limit(int numToTest, int min, int max)
+	{
+		return (numToTest > max) ? max : ((numToTest < min) ? min : numToTest);
+	}
 }
diff --git src/l2jorion/util/StringUtil.java src/l2jorion/util/StringUtil.java
index c00d609..bfc575f 100644
--- src/l2jorion/util/StringUtil.java
+++ src/l2jorion/util/StringUtil.java
@@ -21,6 +21,9 @@
  */
 package l2jorion.util;
 
+import java.text.NumberFormat;
+import java.util.Locale;
+
 import javolution.text.TextBuilder;
 
 public final class StringUtil
@@ -125,4 +128,13 @@
 		}
 		return null;
 	}
+	
+	/**
+	 * @param value : the number to format.
+	 * @return a number formatted with "," delimiter.
+	 */
+	public static String formatNumber(long value)
+	{
+		return NumberFormat.getInstance(Locale.ENGLISH).format(value);
+	}
 }
