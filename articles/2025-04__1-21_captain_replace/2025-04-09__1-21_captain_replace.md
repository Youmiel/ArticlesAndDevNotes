# 1.21.x 袭击者在 \[96, 112\) 区间内特殊表现的代码分析

本文是对 [BV1HHZhYGE7U](https://www.bilibili.com/video/BV1HHZhYGE7U) 的代码分析。视频作者发现，当袭击者距离袭击 96 ~ 112 格时，掠夺者队长可以掉落不祥之瓶，且袭击小队成员可以捡起不详旗帜。


## 〇、环境准备

如果你还不知道从何处获取源码，但是希望对照本文自行理解的话，以下内容可能会有所帮助。为了规避潜在的授权风险，本文使用的是 Yarn 反混淆表，由 FabricMC 维护，你可以使用以下两种方式获得基于 Yarn 反编译的游戏源码：

1. 使用 [FabricMC/yarn](https://github.com/FabricMC/yarn)，打包下载或者 `git clone` 对应分支的源代码皆可。然后确保在系统环境变量中有合适版本的 Java 环境后，在项目根目录执行命令 `./gradlew decompileCFR` 即可，反编译结果在 `build/namedSrc/` 中。

2. (需要 Git 和 Java 集成开发环境，例如 IntelliJ IDEA) 使用 Git 复制 [FabricMC/fabric-example-mod](https://github.com/FabricMC/fabric-example-mod) 到本地，使用 IDE 打开项目。切换到你需要的游戏版本所在的 commit，然后按照 [设置开发环境 | Fabric 文档](https://docs.fabricmc.net/zh_cn/develop/getting-started/setting-up-a-development-environment) 中的指引配置好项目，你可以在依赖库中找到 Minecraft，就是反编译得到的游戏代码。

游戏中有一部分逻辑不是以代码的形式存在的，而是内置的资源包和数据包，你可以用压缩文件管理器打开游戏目录中 `versions/[游戏版本]/[游戏版本].jar` 找到，`assets/` 为资源包目录，`data/` 为数据包目录。

本文的讲解基于 Minecraft 1.21 版本。


## 一、前置知识 - 袭击者“在袭击中”的代码含义

我们常说袭击者“在袭击中”，事实上代码中袭击和袭击者对象中各自保存了对方的引用。“在袭击中”根据语境的不同，可能会指“袭击中包含袭击者的引用”或者“袭击者包含所属袭击的引用”。

```java
public class Raid {
    // 袭击包含每一波袭击者和袭击队长的映射
    private final Map<Integer, RaiderEntity> waveToCaptain;
    private final Map<Integer, Set<RaiderEntity>> waveToRaiders;
    /* ... */
}
```

```java
public abstract class RaiderEntity extends PatrolEntity {
    @Nullable
    protected Raid raid;  // 袭击者所属袭击
    /* ... */
}
```

在 `RaiderEntity` 中，有以下一些方法检查袭击存在性。

```java
// 获取所属袭击的实例
@Nullable
public Raid getRaid() {
    return this.raid;
}

public boolean hasRaid() {
    World world = this.getWorld();
    if (!(world instanceof ServerWorld)) {
        return false;
    }
    ServerWorld lv = (ServerWorld)world;
    return this.getRaid() != null || lv.getRaidAt(this.getBlockPos()) != null;
    // 袭击者在袭击中的判据：袭击引用不为空或者当前位置 96 格内不存在袭击
    // lv.getRaidAt() - 找到维度中指定坐标 96 格内某一个袭击（取决于 HashMap 的遍历顺序）
}

public boolean hasActiveRaid() {
    return this.getRaid() != null && this.getRaid().isActive();
    // 检查袭击者是否属于一个活动的袭击
    // this.getRaid().isActive() - 袭击处于 ONGOING 状态并且袭击中心位置被加载
}

@Override
public boolean hasNoRaid() {
    return !this.hasActiveRaid();
    // 检查袭击者是否不在袭击中（上一个判断的反逻辑）
}
```

“袭击者包含所属袭击的引用”另一种说法是“袭击者具有 RaidId”，因为在实体数据序列化成 NBT 的代码中，键 `RaidId` 展示的是袭击的数字 ID。这是可以使用 `/data` 命令查看到的，在实践中，是一种比较方便的检查手段。

```java
@Override
public void writeCustomDataToNbt(NbtCompound nbt) {
    super.writeCustomDataToNbt(nbt);
    nbt.putInt("Wave", this.wave);
    nbt.putBoolean("CanJoinRaid", this.ableToJoinRaid);
    if (this.raid != null) {
        nbt.putInt("RaidId", this.raid.getRaidId());
    }
}
```


## 二、前置知识 - 袭击者如何才算“队长”

`RaiderEntity` 继承自 `PatrolEntity`，两者皆有类似“判断是否是队长”的方法：

```java
public abstract class PatrolEntity extends HostileEntity {
    /* ... */
    private boolean patrolLeader;
    /* ... */
    // 是否是“巡逻队队长”
    public boolean isPatrolLeader() {
        return this.patrolLeader;
    }
    /* ... */

    // this.patrolLeader 在 NBT 中由键 PatrolLeader 保存
    public void writeCustomDataToNbt(NbtCompound nbt) {
        super.writeCustomDataToNbt(nbt);
        if (this.patrolTarget != null) {
            nbt.put("patrol_target", NbtHelper.fromBlockPos(this.patrolTarget));
        }
        nbt.putBoolean("PatrolLeader", this.patrolLeader);
        nbt.putBoolean("Patrolling", this.patrolling);
    }
}
```

```java
public abstract class RaiderEntity extends PatrolEntity {
    // 是否是袭击队长
    public boolean isCaptain() {
        // 检查头部是否有不祥旗帜
        ItemStack lv = this.getEquippedStack(EquipmentSlot.HEAD);
        boolean bl = !lv.isEmpty() && ItemStack.areEqual(lv, Raid.getOminousBanner(this.getRegistryManager().getWrapperOrThrow(RegistryKeys.BANNER_PATTERN)));
        // 检查是否是“巡逻队队长”
        boolean bl2 = this.isPatrolLeader();
        return bl && bl2;
    }
}
```

值得注意的是，`RaiderEntity::isCaptain(DamageSource)` 方法，于 24w13a 加入，仅用于战利品表谓词 `RaiderPredicate`，原袭击机制中所有判断队长的逻辑都是借助 `PatrolEntity::PatrolLeader()` 完成的。`RaiderPredicate` 目前只用于掠夺者的战利品表，判断是否应该掉落不详之瓶。

```java
public record RaiderPredicate(boolean hasRaid, boolean isCaptain) implements EntitySubPredicate
{
    /* ... */
    // 掉落不详之瓶的要求：hasRaid = false, isCaptain = true
    public static final RaiderPredicate CAPTAIN_WITHOUT_RAID = new RaiderPredicate(false, true);

    /* ... */
    @Override
    public boolean test(Entity entity, ServerWorld world, @Nullable Vec3d pos) {
        if (entity instanceof RaiderEntity) {
            RaiderEntity lv = (RaiderEntity)entity;
            // 谓词的实现逻辑，是一个简单的与逻辑
            return lv.hasRaid() == this.hasRaid && lv.isCaptain() == this.isCaptain;
        }
        // 非袭击者永远返回 false
        return false;
    }
}
```

另外，Yarn 映射表下 `PillagerEntity::isRaidCaptain(ItemStack)` 实际上是判断掠夺者“是否想捡起这个物品”的方法，在 Mojang 映射表中称为 `wantsItem`，并不是判断队长。


## 三、前置知识 - 袭击者何时会捡起旗帜

袭击者捡起旗帜的行为由 `PickupBannerAsLeaderGoal` 控制（注意，此处的“行为”与 Minecraft 的“记忆行为” AI 系统完全无关，并且袭击者使用的 AI 系统是“意向”系统，关于 MC 中生物 AI 系统的运行原理请参考 [Minecraft Wiki - 生物AI](https://zh.minecraft.wiki/w/%E7%94%9F%E7%89%A9AI)），这类 AI 的基本框架请参考 `net.minecraft.entity.ai.goal.Goal` 类。

```java
public class PickupBannerAsLeaderGoal<T extends RaiderEntity> extends Goal {
    /* ... */

    // canStart() 参与决定的这个 AI 意向是否可以运行
    @Override
    public boolean canStart() {
        List<ItemEntity> list;
        Raid lv = ((RaiderEntity)this.actor).getRaid();
        if (!((RaiderEntity)this.actor).hasActiveRaid() 
            || ((RaiderEntity)this.actor).getRaid().isFinished() 
            || !((PatrolEntity)this.actor).canLead() 
            || ItemStack.areEqual(
                ((MobEntity)this.actor).getEquippedStack(EquipmentSlot.HEAD), 
                Raid.getOminousBanner(((Entity)this.actor).getRegistryManager().getWrapperOrThrow(RegistryKeys.BANNER_PATTERN)))) {
            // 不符合启动要求的时候提前返回
            // 条件：执行者不属于一个活动的袭击，
            //      或属于一个已经结束的袭击（胜利或失败），
            //      或该生物不可以成为队长（取决于 canLead() 方法的重写情况，女巫、劫掠兽为 false，其余皆为true），
            //      或该生物头盔栏已经有不祥旗帜
            return false;
        }
        RaiderEntity lv2 = lv.getCaptain(((RaiderEntity)this.actor).getWave());
        if (!(lv2 != null 
              && lv2.isAlive() 
              || (list = ((Entity)this.actor).getWorld().getEntitiesByClass(
                  ItemEntity.class, 
                  ((Entity)this.actor).getBoundingBox().expand(16.0, 8.0, 16.0), 
                  OBTAINABLE_OMINOUS_BANNER_PREDICATE)
                  ).isEmpty())) {
            // 更进一步的条件：
            // 执行者所属袭击和波次中不存在队长，或队长已死亡，且附近存在不祥旗帜掉落物（具体检测范围见代码）
            return ((MobEntity)this.actor).getNavigation().startMovingTo(list.get(0), 1.15f);
        }
        return false;
    }
    /* ... */
}

```


## 四、阶段性总结

有了前置知识，我们现在可以总结一下问题的讨论环境。

首先，袭击者距离袭击 96~112 格内，并且袭击正在进行，区块也是强加载，此时袭击中 `waveToCaptain`、`waveToRaiders` 应该都存在映射，袭击者的 `raid` 字段也正常引用了所属的袭击。那么 `RaiderEntity` 中四个判断袭击存在性的方法返回值应该为：

- `getRaid()`: 当前袭击
- `hasRaid()`:
  - `this.getRaid() != null`: `raid` 字段不为空，所以是 `true`
  - `lv.getRaidAt(this.getBlockPos()) != null`: 当前位置已在袭击 96 格外，获取不到，所以是 `false`
  - 以上两者作逻辑或运算，结果是 `true`
- `hasActiveRaid()`: 袭击正在进行，区块强加载，所以是 `true`
- `hasNoRaid()`: 上一个方法取反，为 `false`


袭击队长的判断方法中，无论是刷出的袭击队长还是袭击者捡起旗帜成为的队长，都属于正常途径制造的队长，`patrolLeader` 字段为 `true`，那么：

- `PatrolEntity::isPatrolLeader()`: 直接和字段关联，所以结果是 `true`
- `RaiderEntity::isCaptain()`: 头盔栏有不祥旗帜，且 `isPatrolLeader()` 返回 `true`，所以结果是 `true`


非队长袭击者捡起旗帜的意向，假设此时队长已经死亡：

- 执行者属于一个活动的袭击：是
- 执行者属于一个未结束的袭击：是
- 该类生物可以成为队长：非女巫、劫掠兽，所以是
- 执行者头盔栏没有不祥旗帜：是
- 执行者所属袭击和波次中不存在队长，或队长已死亡：是
- 执行者附近存在不祥旗帜掉落物：队长会掉落旗帜，所以是

由此可得，袭击者在距离袭击 96~112 格内可以正常触发捡起旗帜的 AI 意向。


再看掠夺者掉落不详之瓶使用的谓词 `RaiderPredicate`：

- `lv.hasRaid() == false`: `true == false`，结果是 `false`
- `lv.isCaptain() == true`: `true == true`，结果是 `true`
- 以上两者作逻辑与，结果是 `false`

按照这样推断，掠夺者队长不应该掉落不详之瓶，与事实有出入，一定有某处代码出现了问题。


## 五、袭击者的死亡逻辑

既然是掉落物不符合预期，说明战利品表的逻辑出现了异常，现在关注袭击者的 `onDeath()` 方法。`PillagerEntity` 的继承链是 `Entity` -> `LivingEntity` -> `MobEntity` -> `PathAwareEntity` -> `HostileEntity` -> `PatrolEntity` -> `RaiderEntity` -> `PillagerEntity`，其中只有 `RaiderEntity` 重写了 `LivingEntity` 中的 `onDeath(DamageSource)` 方法。

```java
public abstract class RaiderEntity extends PatrolEntity {
    /* ... */
    @Override
    public void onDeath(DamageSource damageSource) {
        if (this.getWorld() instanceof ServerWorld) {
            Entity lv = damageSource.getAttacker();
            Raid lv2 = this.getRaid();
            if (lv2 != null) {
                if (this.isPatrolLeader()) {
                    // 如果是队长，就从袭击的 waveToCaptain 表里移除自己
                    lv2.removeLeader(this.getWave());
                }
                if (lv != null && lv.getType() == EntityType.PLAYER) {
                    lv2.addHero(lv);
                }
                lv2.removeFromWave(this, false);    // 将自己从袭击中移除
            }
        }
        super.onDeath(damageSource);    // 执行基类同名方法
    }
    /* ... */
}
```

```java
// 接上块，将袭击者移出袭击的逻辑中，会将此袭击者的 raid 属性置空
// 这样双方都不会保留对另一方的引用了
public class Raid {
    public void removeFromWave(RaiderEntity entity, boolean countHealth) {
        boolean bl2;
        Set<RaiderEntity> set = this.waveToRaiders.get(entity.getWave());
        if (set != null && (bl2 = set.remove(entity))) {    // 将袭击者从 waveToRaiders 表里移除
            if (countHealth) {
                this.totalHealth -= entity.getHealth();
            }
            entity.setRaid(null);    // 将此袭击者的 raid 属性置空
            this.updateBar();
            this.markDirty();
        }
    }
}
```

```java
public abstract class LivingEntity extends Entity implements Attackable {
    /* ... */
    public void onDeath(DamageSource damageSource) {
        if (this.isRemoved() || this.dead) {
            return;
        }
        Entity lv = damageSource.getAttacker();
        LivingEntity lv2 = this.getPrimeAdversary();
        if (this.scoreAmount >= 0 && lv2 != null) {
            lv2.updateKilledAdvancementCriterion(this, this.scoreAmount, damageSource);
        }
        if (this.isSleeping()) {
            this.wakeUp();
        }
        if (!this.getWorld().isClient && this.hasCustomName()) {
            LOGGER.info("Named entity {} died: {}", (Object)this, (Object)this.getDamageTracker().getDeathMessage().getString());
        }
        this.dead = true;
        this.getDamageTracker().update();
        World world = this.getWorld();
        if (world instanceof ServerWorld) {
            ServerWorld lv3 = (ServerWorld)world;
            if (lv == null || lv.onKilledOther(lv3, this)) {
                this.emitGameEvent(GameEvent.ENTITY_DIE);
                this.drop(lv3, damageSource);    // 处理掉落物
                this.onKilledBy(lv2);
            }
            this.getWorld().sendEntityStatus(this, EntityStatuses.PLAY_DEATH_SOUND_OR_ADD_PROJECTILE_HIT_PARTICLES);
        }
        this.setPose(EntityPose.DYING);
    }
    /* ... */

    protected void drop(ServerWorld world, DamageSource damageSource) {
        boolean bl;
        boolean bl2 = bl = this.playerHitTimer > 0;
        if (this.shouldDropLoot() && world.getGameRules().getBoolean(GameRules.DO_MOB_LOOT)) {
            this.dropLoot(damageSource, bl);                // 处理战利品表，掉落战利品
            this.dropEquipment(world, damageSource, bl);    // 掉落装备
        }
        this.dropInventory();                       // 掉落物品栏内容
        this.dropXp(damageSource.getAttacker());    // 掉落经验
    }
    /* ... */
}
```

因为袭击者的 `onDeath()` 方法中先执行了本类的处理逻辑，再调用基类的 `onDeath()` 方法，导致实际处理战利品表的时候，被击杀的掠夺者队长已经被移出了袭击，那么 `RaiderEntity::hasRaid()` 的判断结果就出现了变化：

- `hasRaid()`:
  - `this.getRaid() != null`: 袭击者已经被移出袭击， `raid` 字段为空，所以是 `false`
  - `lv.getRaidAt(this.getBlockPos()) != null`: 没有变化，仍然是 `false`
  - 以上两者作逻辑或运算，结果是 `false`

相应地，谓词 `RaiderPredicate` 判断结果也出现了变化：

- `lv.hasRaid() == false`: `false == false`，结果是 `true`
- `lv.isCaptain() == true`: `true == true`，结果是 `true`
- 以上两者作逻辑与，结果是 `true`

因此，掠夺者 `RaiderPredicate` 判断通过，可以掉落不详之瓶。

从上述分析中我们也可以得知，在袭击者死亡时，`RaiderEntity::hasRaid()` 中 `this.getRaid() != null` 部分的结果永远为 `false`，返回值完全依赖 `lv.getRaidAt(this.getBlockPos()) != null` 的结果。那么掠夺者掉落不详之瓶的条件就可以简化为 `raider.getRaidAt(raider.getBlockPos()) != null && raider.isCaptain() == true`，即它的死亡位置不在任意袭击 96 格范围内，且是袭击队长。

## 六、总结

综上所述，正是因为袭击者移出袭击的距离（112格）大于掠夺者队长掉落不详之瓶的距离（96格），才使得 \[96, 112\) 这一特殊区间内掠夺者队长可以掉落不祥之瓶，且袭击小队成员可以捡起不详旗帜。


<br>
<br>
<br>
<br>

"1.21.x 袭击者在 \[96, 112\) 区间内特殊表现的代码分析" © 2025 作者: Youmiel 采用 CC BY-NC-SA 4.0 许可。如需查看该许可证的副本，请访问 http://creativecommons.org/licenses/by-nc-sa/4.0/。
