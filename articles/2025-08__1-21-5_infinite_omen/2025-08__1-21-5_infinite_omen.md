# 1.21.5+ 不祥之兆无限转化袭击之兆代码分析 —— 并发修改异常及其利用

据网友反馈，有一部分袭击农场的设计在 1.21.5 及以上使用时，不祥之兆转化为袭击之兆后并没有消失，在效果持续的 100 分钟内，每 30 秒即可免费制造一次袭击之兆。本文将对上述特性进行代码解释，并讨论相关原理的拓展应用。

在此特别感谢 Lemon_Iced, Nickid2018, s_yh 等人在研究过程中提供的帮助，排名不分先后。


## TL;DR

如果你对 Java 编程比较熟悉，以下是对该特性的简短解释：

在集合迭代时，不祥之兆的代码逻辑中修改了集合结构（添加袭击之兆），使得 hashmap 的迭代器在 `iterator.remove()` 的时候抛出 `ConcurrentModificationException` (并发修改异常)，没有成功移除不祥之兆效果。直到下一游戏刻，不祥之兆再次转换袭击之兆时才成功移除。

*注：`ConcurrentModificationException` 并不一定需要在多线程环境下触发，这个问题中的异常就完全是单线程下制造的。*

## 目录
- [TL;DR](#tldr)
- [〇、环境准备](#〇环境准备)
- [一、前置知识：Minecraft 1.21.5 状态效果系统的总体介绍](#一前置知识minecraft-1215-状态效果系统的总体介绍)
- [二、何为“并发修改异常”](#二何为并发修改异常)
- [三、不祥之兆无限转化袭击之兆：并发修改异常最直接的利用](#三不祥之兆无限转化袭击之兆并发修改异常最直接的利用)
- [四、猜测：为何该特性没有第一时间被发现，后续可能会如何修复](#四猜测为何该特性没有第一时间被发现后续可能会如何修复)
- [五、1.21.5 前后 CME 触发情况差异](#五1215-前后-cme-触发情况差异)
- [六、CME 跳过状态效果处理](#六cme-跳过状态效果处理)


## 〇、环境准备

如果你还不知道从何处获取源码，但是希望对照本文自行理解的话，可以按照 [1.21.x 袭击者在 \[96, 112\) 区间内特殊表现的代码分析](../2025-04__1-21_captain_replace/2025-04-09__1-21_captain_replace.md) 中的步骤反编译游戏源码。

本文的讲解基于 Minecraft 1.21 和 1.21.5 版本，使用 Yarn 反混淆表。


## 一、前置知识：Minecraft 1.21.5 状态效果系统的总体介绍

生物状态效果的计算在 `LivingEntity::baseTick()` 的末尾，名为 `LivingEntity::tickStatusEffects()`。以下是该方法的主要代码，作为参考。

```java
public abstract class LivingEntity extends Entity
implements Attackable, ServerWaypoint {
    /* ... */
    private final Map<RegistryEntry<StatusEffect>, StatusEffectInstance> activeStatusEffects = Maps.newHashMap();
    /* ... */

    protected void tickStatusEffects() {
        World world = this.getWorld();
        if (world instanceof ServerWorld) {
            ServerWorld lv = (ServerWorld)world;
            Iterator<Object> iterator = this.activeStatusEffects.keySet().iterator();
            // activeStatusEffects 是一个 hashmap, 将状态效果注册表项和生物带有的状态效果实例联系起来
            // 瞬时的状态效果不会加入，例如 瞬间伤害 和 瞬间治疗
            try {
                while (iterator.hasNext()) {
                    RegistryEntry lv2 = (RegistryEntry)iterator.next();
                    StatusEffectInstance lv3 = this.activeStatusEffects.get(lv2);
                    if (!lv3.update(lv, this, () -> this.onStatusEffectUpgraded(lv3, true, null))) {
                        // ↑ 逐个更新状态效果实例
                        iterator.remove();         // update 方法返回 false 的情况下，调用迭代器的 remove() 方法移除这个效果
                        this.onStatusEffectsRemoved(List.of(lv3));
                        continue;
                    }
                    if (lv3.getDuration() % 600 != 0) continue;
                    this.onStatusEffectUpgraded(lv3, false, null);
                }
            } catch (ConcurrentModificationException lv2) {
                // 捕获了 ConcurrentModificationException，但是没做任何处理
                // 这个异常是本文的主角 
            }
            /* ... */
            // 其他代码，不是重点，略
        } else {
            /* ... */
            // 客户端侧的逻辑，不是重点，略
        }
    }
}
```

对于状态效果实例，以上代码中调用到了 `StatusEffectInstance::update()`，它的内容如下：

```java
public class StatusEffectInstance implements Comparable<StatusEffectInstance> {
    public boolean update(ServerWorld world, LivingEntity entity, Runnable hiddenEffectCallback) {
        int i;
        if (!this.isActive()) {  // 判断本状态效果是否为无限长或者剩余时长大于零，否，则不处理后续逻辑
            return false;
        }
        int n = i = this.isInfinite() ? entity.age : this.duration;
        if (this.type.value().canApplyUpdateEffect(i, this.amplifier) && 
                !this.type.value().applyUpdateEffect(world, entity, this.amplifier)) {
            // ↑ 检查当前游戏刻本状态效果实例是否需要处理
            // 是，则调用 applyUpdateEffect() 处理状态效果，大多数状态效果的实现都写在这一方法中
            return false;
            // 若本实例在当前游戏刻需要处理，且 applyUpdateEffect() 返回 false，则返回 false
            // 到 tickStatusEffects() 中，会执行移除当前状态效果实例的代码
        }
        this.updateDuration();   // 将状态效果的剩余时长减 1，也就是先计算，再减少计时
        if (this.tickHiddenEffect()) {
            hiddenEffectCallback.run();  
            // 处理隐藏的状态效果，例如同时具有 “长时间、低等级” 和 “短时间、高等级” 的同 ID 状态效果
        }
        return this.isActive();   // 同前述，即通过剩余时长控制是否清除当前效果
    }
}
```


## 二、何为“并发修改异常”

ConcurrentModificationException (并发修改异常，以下简称 CME) 是 Java 中的一个运行时异常，通常在对集合进行迭代的过程中修改了集合结构时（例如，增删元素）抛出。如果希望全面了解 CME 的触发原理和场景，请阅读 [java - Why is a ConcurrentModificationException thrown and how to debug it - Stack Overflow](https://stackoverflow.com/questions/602636/why-is-a-concurrentmodificationexception-thrown-and-how-to-debug-it)，本文只在此章节列出必要的几项。

### 为什么需要“并发修改异常”

假如没有这个异常，在集合迭代时修改集合很容易造成难以排查的问题。例如以下 JavaScript 代码：

```javascript
let arr = [1, 2, 3];
for (let i of arr) {
    arr.push(4); // 每次循环，向数组末尾加入元素“4”。不抛异常，但可能无限循环或逻辑错误
}
```

在 NodeJS 环境下，执行该代码会导致无限循环，而程序员不一定注意到这个循环编写得有问题。

### 触发场景

`livingEntity.activeStatusEffect` 是一个 HashMap，对于 HashMap，有以下两种场景会触发 CME：

1. **在增强型 for 循环中 / Iterator 遍历中直接用集合方法修改集合** <br>
增强型 for 循环（如 `for (T item : map.keySet())`）本质上还是使用 Iterator 遍历集合，如果在遍历过程中使用 `map.put(key, value)` 或 `map.remove(key)` 增删了元素，会在下次调用 `iterator.next()` 或 `iterator.remove()` 时触发异常。

1. **在多线程环境中使用 fail-fast 的非线程安全集合（如 HashMap、ArrayList）进行并发修改** <br>
因为 Minecraft 的主要游戏逻辑是单线程处理的，状态效果逻辑并不涉及并发修改的问题。


## 三、不祥之兆无限转化袭击之兆：并发修改异常最直接的利用

### 原理

且看不祥之兆的实现代码：

```java
class BadOmenStatusEffect extends StatusEffect {
    protected BadOmenStatusEffect(StatusEffectCategory arg, int i) {
        super(arg, i);
    }

    @Override
    public boolean canApplyUpdateEffect(int duration, int amplifier) {
        return true;    // 每个游戏刻都需要执行 applyUpdateEffect()
    }

    @Override
    public boolean applyUpdateEffect(ServerWorld world, LivingEntity entity, int amplifier) {
        Raid lv2;
        ServerPlayerEntity lv;
        if (entity instanceof ServerPlayerEntity 
                && !(lv = (ServerPlayerEntity) entity).isSpectator()
                && world.getDifficulty() != Difficulty.PEACEFUL 
                && world.isNearOccupiedPointOfInterest(lv.getBlockPos())
                && ((lv2 = world.getRaidAt(lv.getBlockPos())) == null
                        || lv2.getBadOmenLevel() < lv2.getMaxAcceptableBadOmenLevel())) {
            // ↑ 添加袭击之兆的条件判断
            lv.addStatusEffect(new StatusEffectInstance(StatusEffects.RAID_OMEN, 600, amplifier));
            // ↑ 此处添加了袭击之兆
            lv.setStartRaidPos(lv.getBlockPos());
            return false;    // 此处返回了 false
        }
        return true;
    }
}
```

`BadOmenStatusEffect::applyUpdateEffect()` 在满足了袭击之兆添加条件后，会在添加袭击之兆之后返回 `false`。结合前面的代码描述，`StatusEffectInstance::update()` 也返回 `false`，到 `LivingEntity::tickStatusEffects()` 会调用 `iterator.remove()`。因为前面添加袭击之兆后 `activeStatusEffect` 的元素数量改变，后续 `iterator.remove()` 抛出 CME，且未能移除不祥之兆。

假如下一游戏刻，袭击之兆的添加条件仍然满足，那么代码执行情况同上，但是 `activeStatusEffect` 中已有袭击之兆，再次添加不会改变元素数量，后续 `iterator.remove()` 正常移除不祥之兆，不抛出 CME。

以上就是不祥之兆无限转化袭击之兆的原理，对应到袭击农场的设计上，就是让玩家每次获取袭击之兆时只满足袭击之兆添加条件 1 游戏刻。Lemon_Iced 的袭击农场使用了可控 POI 认领技术，并且恰好只让村庄区段存在了 1 游戏刻，所以出现无限转化袭击之兆的现象；如果村庄区段保持存在，让玩家只在村庄区段中停留 1 游戏刻同样能达到效果。需要注意的是，在获取袭击之兆后、袭击之兆未转化成袭击前，玩家不能再次进入村庄区段，否则不祥之兆也会被移除。

### 时序影响

为表述清晰，以下使用“常规触发”指代常见的、玩家获得袭击之兆时处于村庄区段内的时长大于 1 游戏刻的效果获取方式；用“无限触发”指代前述玩家仅处于村庄区段内 1 游戏刻的袭击之兆效果无限获取方式。

对于常规触发的设计，最小触发周期不确定，存在两种结果，确切地来说，取决于不祥之兆和袭击之兆在同一游戏刻内的运算顺序，而这个运算顺序不是绝对确定的。阅读前面的代码，可以发现 `livingEntity.activeStatusEffect` 是 `HashMap<RegistryEntry<StatusEffect>, StatusEffectInstance>` 类型，并且 Mojang 没有为 `RegistryEntry.Reference<T>` (注册表项的引用，`RegistryEntry<T>` 接口的实现) 实现 `hashCode()` 方法。这意味着每次程序启动，同样的一组状态效果会有不同的遍历顺序，而集合扩容（触发 rehash）也会进一步影响它们的遍历顺序。

#### 常规触发，如果不祥之兆先于袭击之兆

剩余时间指 `StatusEffect::canApplyUpdateEffect()` 被调用时的效果剩余时间，后面同理。

- GT 0: 不详之兆添加袭击之兆，触发 CME，中断遍历；袭击之兆不计算
- GT 1: 不详之兆再次添加袭击之兆，移除自身；袭击之兆计算，剩余时间 600 gt
- GT 2: 袭击之兆剩余时间 599 gt
- ... 
- ... (某个时间点，玩家获得不祥之兆)
- ... 
- GT 600: 袭击之兆剩余时间 1 gt，开始袭击事件，移除自身
- GT 601: 不祥之兆可以再次添加袭击之兆

最小袭击生成周期 601 gt。

#### 常规触发，如果袭击之兆先于不祥之兆

- GT 0: 不详之兆添加袭击之兆，触发 CME，中断遍历；袭击之兆不计算
- GT 1: 袭击之兆计算，剩余时间 600 gt（计算结束后减为 599 gt）；不详之兆再次添加袭击之兆，移除自身，将袭击之兆剩余时间刷新为 600 gt
- GT 2: 袭击之兆剩余时间 600 gt
- GT 3: 袭击之兆剩余时间 599 gt
- ... 
- ... (某个时间点，玩家获得不祥之兆)
- ... 
- GT 601: 袭击之兆剩余时间 1 gt，开始袭击事件，移除自身
- GT 602: 不祥之兆可以再次添加袭击之兆

最小袭击生成周期 602 gt。

#### 无限触发

- GT 0: 不详之兆添加袭击之兆，触发 CME，中断遍历；袭击之兆不计算
- GT 1: 袭击之兆剩余时间 600 gt
- GT 2: 袭击之兆剩余时间 599 gt
- ... 
- ... 
- GT 600: 袭击之兆剩余时间 1 gt，开始袭击事件，移除自身
- GT 601: 不祥之兆可以再次添加袭击之兆

最小袭击生成周期 601 gt。

#### 真玩家、假玩家的区别

[堆叠袭击塔对真人适用程度分类](../2024-02__raid_explaination/2024-02-02__categories.md) 中对真假玩家状态效果计算阶段差异的描述仍然有效。


## 四、猜测：为何该特性没有第一时间被发现，后续可能会如何修复

### 为何 Mojang 和玩家社区之前都没有发现这个问题

#### 1. 触发条件苛刻

无限转化袭击之兆要求玩家仅在村庄区段停留 1 游戏刻，除非特意构造（使用红石电路控制、`/tick rate 1` 下精细操作、开发环境下断点再移动玩家等），人工操作很难做到停留 1 游戏刻，即使是 `/tick freeze` 也不能阻止玩家的状态效果继续计算。常规的触发方式也只是有概率让袭击之兆的结束时间延后 1 游戏刻，仅凭肉眼观察很难注意到。 

#### 2. Mojang 程序开发时纪律性不足

回想一下，有很多莫名其妙的问题都来源于 Mojang 开发时不讲章法，例如：
1. [依赖于维度的随机的红石](https://www.bilibili.com/video/BV1Gv411t7sb?p=1)：原因是 Mojang 使用了 hashmap 来储存三个维度，却没给维度实现 `hashCode` 方法
2. [Minecraft 1.18.2+ 中地狱堡垒的地狱砖刷怪游走问题](https://blog.fallenbreath.me/zh-CN/2024/fortress-nether-bricks-pack-spawning-issue-1182/#more)：原因是 Mojang 没有给 `SpawnEntry` 实现 `equals` 方法，这一问题遗留了近三年才被发现并意外修复。

这一次，Mojang 犯的低级错误至少有：
- 在集合迭代时直接增加元素数量
- 捕获异常之后什么都不做
- 没有给 `RegistryEntry.Reference<T>` 实现 `hashCode()` 方法
- 使用 `HashMap` 这种遍历顺序不稳定的数据结构，导致问题有概率被微时序掩盖

### Mojang 可能会如何修复

除了补全未实现的方法、更换更合适的数据结构以外，最核心的问题是在状态效果集合迭代时添加了新的状态效果，而现有状态效果系统并没有考虑到这样特殊的用法。解决办法是避免在遍历集合时添加新的状态效果（我看把不祥之兆改成直接生成袭击就挺好的，对吧 `:P`），或者干脆为了这个需求重构一下状态效果系统。

至于何时修复，要看这个特性什么时候传到 Mojang，或者 Mojang 哪天心血来潮随机重构一下状态效果的代码然后就修复了。


## 五、1.21.5 前后 CME 触发情况差异

其实 25w02a 时 Mojang 已经有意识地在避免药水效果代码引发 CME，并修改了一部分状态效果的处理逻辑。接下来我将说明 1.21 ~ 1.21.4 版本的状态效果逻辑与 1.21.5 后的版本有什么区别，以及如何触发 CME。

### 1.21 状态效果代码

首先是 `LivingEntity::tickStatusEffect()`，它的相关代码基本没有变化。

```java
protected void tickStatusEffects() {
    List<ParticleEffect> list;
    Iterator<RegistryEntry<StatusEffect>> iterator = this.activeStatusEffects.keySet().iterator();
    try {
        while (iterator.hasNext()) {
            RegistryEntry<StatusEffect> lv = iterator.next();
            StatusEffectInstance lv2 = this.activeStatusEffects.get(lv);
            if (!lv2.update(this, () -> this.onStatusEffectUpgraded(lv2, true, null))) {
                // 遍历方式没变
                if (this.getWorld().isClient) continue;
                iterator.remove();   // 移除方式没变 
                this.onStatusEffectRemoved(lv2);
                continue;
            }
            if (lv2.getDuration() % 600 != 0) continue;
            this.onStatusEffectUpgraded(lv2, false, null);
        }
    } catch (ConcurrentModificationException lv) {
        // 捕获了 ConcurrentModificationException，但是没做任何处理
    }
    /* ... */
    // 其他代码，不是重点，略
}
```

然后是 `StatusEffectInstance::update()`

```java
public boolean update(LivingEntity entity, Runnable overwriteCallback) {
    if (this.isActive()) {
        int i;
        int n = i = this.isInfinite() ? entity.age : this.duration;
        if (this.type.value().canApplyUpdateEffect(i, this.amplifier) 
                && !this.type.value().applyUpdateEffect(entity, this.amplifier)) {
            entity.removeStatusEffect(this.type);
            // applyUpdateEffect() 返回 false 后，不像 1.21.5 那样继续向外返回 false，而是当场移除效果
            // 内部实现使用的是 map.remove(item)，不会抛出 CME
            // 但是会使得后续调用 iterator.next() 等方法抛出 CME
            // 由于没有直接 return，后续逻辑仍然执行（持续时间减一、处理隐藏效果等）
        }
        this.updateDuration();    // 先计算，再倒计时，没变
        if (this.duration == 0 && this.hiddenEffect != null) {
            // 处理隐藏的状态效果
            this.copyFrom(this.hiddenEffect);
            this.hiddenEffect = this.hiddenEffect.hiddenEffect;
            overwriteCallback.run();
        }
    }
    this.fading.update(this);
    return this.isActive();    // 相比 1.21.5，只有一处返回位置
}
```

### 1.21 ~ 1.21.4 制造 CME 方法分析

可以发现，1.21 的状态效果处理逻辑不会在添加袭击之兆的时候调用 `iterator.remove()` 而抛出 CME，导致不祥之兆移除失败。不过我们依然可以尝试在 `iterator.next()` 或 `iterator.remove()` 调用前增删状态效果从而抛出 CME。

增删方式总共有两种：
1. 不祥之兆中符合触发条件添加袭击之兆
2. 尝试让 `applyUpdateEffect()` 返回 `false`，从而移除自身，目前只有不祥之兆添加袭击之兆时、袭击之兆生成袭击时、伤害吸收耗尽黄心时会返回 `false`

可以抛出 CME 的位置有两处：
1. `iterator.next()`：如果发生增删的效果不是最后一个，在迭代到下一个效果时调用
2. `iterator.remove()`：只有在状态效果持续时间结束时调用

排列组合可以得到如下几种制造 CME 的方式：
1. 袭击之兆生成袭击时固定触发。触发位置 `iterator.remove()`
2. 不祥之兆添加袭击之兆或伤害吸收耗尽黄心，且还有状态效果未迭代时。触发位置 `iterator.next()`
3. 伤害吸收耗尽黄心，同时持续时间结束。触发位置 `iterator.remove()`

由于[【三】](#时序影响)中所述状态效果迭代顺序的不确定性，`2.`提及的 CME 触发方式不稳定。


## 六、CME 跳过状态效果处理

CME 触发时，本游戏刻处理该玩家状态效果的循环会直接中断，未遍历到的状态效果不仅持续时间没有减少，该有的效果也没有产生，也就是跳过了一个游戏刻的处理。

对于不祥之兆在 1.21.5 及以上版本的表现，因为添加袭击之兆后，该分支只是 `return false`，并不会让持续时间减少，所以利用无限触发特性，还能一定程度上延长不祥之兆的持续时间。

### 1.21 ~ 1.21.4 CME 对袭击农场的时序影响

#### 不祥之兆先于袭击之兆计算

- GT 0: 不祥之兆符合转化条件，添加袭击之兆，移除不祥之兆，触发 CME 与否不影响
- GT 1: 袭击之兆剩余时间 600 gt
- GT 2: 袭击之兆剩余时间 599 gt
- ...
- ...(某个时间点，玩家获得不祥之兆)
- ...
- GT 600: 袭击之兆剩余时间 1 gt，开始袭击事件，移除自身
- GT 601: 不祥之兆符合转化条件，添加袭击之兆...

最小袭击生成周期 601 gt。

#### 袭击之兆先于不祥之兆计算

- GT 0: 不祥之兆符合转化条件，添加袭击之兆，移除不祥之兆，触发 CME 与否不影响
- GT 1: 袭击之兆剩余时间 600 gt
- GT 2: 袭击之兆剩余时间 599 gt
- ...
- ...(某个时间点，玩家获得不祥之兆)
- ...
- GT 600: 袭击之兆剩余时间 1 gt，开始袭击事件，移除自身，触发 CME，不处理不祥之兆
- GT 601: 不祥之兆符合转化条件，添加袭击之兆...

最小袭击生成周期 601 gt。


## 七、总结

综上，可见 1.21.5 不祥之兆无限转化袭击之兆属于 Mojang 重构代码，（可能）尝试解决并发修改异常时，无意中制造的新特性。利用并发修改异常，除了可以免费获取袭击之兆以外，还可以跳过一部分状态效果计算，虽然暂不清楚是否能有意义地利用它，但是理解它的作用也多少有助于 Minecraft 生存技术的探索。


<br>
<br>
<br>
<br>

"1.21.5+ 不祥之兆无限转化袭击之兆代码分析 —— 并发修改异常及其利用" © 2025 作者: Youmiel 采用 CC BY-NC-SA 4.0 许可。如需查看该许可证的副本，请访问 http://creativecommons.org/licenses/by-nc-sa/4.0/。