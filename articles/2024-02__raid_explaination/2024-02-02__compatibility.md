# 1.18+ 堆叠袭击农场真人可用性测试结果

关于真人可用性分类请看另一篇专栏。

本次测试仅考察这些袭击塔在何种情况下能可靠地堆叠袭击，不排查是否会刷恼鬼，因为这项要求需要太多时间，除非在测试过程中出现了恼鬼。

## 列表：作品名 + 作者名 + 视频链接

| **作品** | **作者** | **视频链接** | **可用性** | **备注** |
|---|---|---|---|---|
| Tape袭击塔V2 | anew_tape | https://www.bilibili.com/video/BV1N94y1z7Pc | 3. 允许玩家操作因为网络延迟存在一定的不精确性 | 加上盔甲架补充和横扫检测防信号反转就完美了 |
| 次世代袭击农场Gen3 | CCS_Covenant | https://www.bilibili.com/video/BV1xB4y117sf | 3. 允许玩家操作因为网络延迟存在一定的不精确性 | 加上盔甲架补充就完美了 |
| 1.18+袭击农场231k绿宝石/h | DaveRooney | https://www.bilibili.com/video/BV1h84y1Q7zX | 1. 依赖假人在实体更新阶段的攻击 |  |
| 弱加载抢夺堆叠袭击农场 | GaRLic BrEd | https://www.bilibili.com/video/BV1Bu41127N7 | 不好确定 | 应该会受到真人与假人触发袭击时间不同这个问题的影响 |
| Chronos袭击塔v3 | GaRLic BrEd | https://www.bilibili.com/video/BV1pF411P7nP | 1. 依赖假人在实体更新阶段的攻击 |  |
| 终极堆叠袭击农场 | GaRLic BrEd | https://www.bilibili.com/video/BV1zX4y117zk | 不好确定 | 应该会受到真人与假人触发袭击时间不同这个问题的影响，活塞移动玩家在方块实体阶段，晚于实体更新阶段 |
| “原汁原味的村庄袭击”632k10gt堆叠袭击塔 | MU_mushroom | https://www.bilibili.com/video/BV18C4y1r7Sx | 无存档，目测属于“1. 依赖假人在实体更新阶段的攻击” |  |
| 丐版袭击塔 | Nash, ianxofour | https://www.bilibili.com/video/BV1Su411b7Nz | 2. 需要稳定的玩家行为 | 假人挂机的袭击生成速率大约比真人多500次/小时，原因为假人在实体更新阶段攻击绕过了袭击招募 |
| 20gt 446K堆叠袭击农场 | Nash/NashCasty | https://www.bilibili.com/video/BV1fN411W7Cj | 1. 依赖假人在实体更新阶段的攻击 |  |
| NuggTech袭击塔 | Nash/NashCasty | https://www.bilibili.com/video/BV1oX4y1x7yX | 1. 依赖假人在实体更新阶段的攻击 |  |
| NuggTech袭击塔v2 | Nash/NashCasty | https://www.bilibili.com/video/BV1394y187zK | 37村民版本：1. 依赖假人在实体更新阶段的攻击 <br>17村民版本：1. 机器时序依赖假人的袭击触发时序 | 设计者可能没有意识到静态袭击生成能够解决真玩家挂机问题 |
| 通用袭击塔 | Nash/NashCasty | https://www.youtube.com/watch?v=yKnLcEiLci0 | 2. 需要稳定的玩家行为 | 假人挂机的袭击生成速率大约比真人多150次/小时，原因为假人在实体更新阶段攻击绕过了袭击招募 |
| 460K袭击塔 | qwert_JANG | https://www.bilibili.com/video/BV1tj411B7bi | 1. 依赖假人在实体更新阶段的攻击 | 不是，哥们，怎么你的怪物归中还会卡怪啊？ |
| 高速袭击塔，空岛可用【41.3万/小时】 | Scorpio天蝎君 | https://www.bilibili.com/video/BV1Ja411G7Ag | 1. 依赖假人在实体更新阶段的攻击 |  |
| 410k烟花袭击农场 | Youmiel | https://www.bilibili.com/video/BV1Hb4y1M7aQ | 3. 允许玩家操作因为网络延迟存在一定的不精确性 |  |
| 220k安心挂机袭击农场 | Youmiel | https://www.bilibili.com/video/BV1sN4y1s7DW | 3. 允许玩家操作因为网络延迟存在一定的不精确性 |  |
| 835k绿宝石农场 | 何为氕氘氚 | https://www.bilibili.com/video/BV1cR4y1H7No | 1. 依赖假人在实体更新阶段的攻击 |  |
| 960k~967k绿宝石农场  | 何为氕氘氚 | https://www.bilibili.com/video/BV1ha411J7b8 | 1. 依赖假人在实体更新阶段的攻击 |  |
| 堆叠袭击农场V6 | 何为氕氘氚 | https://www.bilibili.com/video/BV1TY4y1M7zt | 1. 依赖假人在实体更新阶段的攻击 |  |
| 堆叠袭击农场V7 | 何为氕氘氚 | https://www.bilibili.com/video/BV1bz4y1M7zM | 1. 机器时序依赖假人的袭击触发时序 |  |
| 5gt堆叠袭击农场 | 何为氕氘氚 | https://www.bilibili.com/video/BV1QT411q7UG | 1. 机器时序依赖假人的袭击触发时序 |  |
| 绿宝石印钞机2.0气泡柱版本 | 黑山大叔 | https://www.bilibili.com/video/BV1f44y1x7vY | 不好确定，但是真人能堆叠 | 能用的原因是时钟很慢，两波怪之间的距离大于招募距离 |
| 高版本袭击塔修改版（3.0） | 黑山大叔 | https://www.bilibili.com/video/BV1mP411J744 | 不好确定，但是真人能堆叠 | 和2.0大差不差 |
| 袭击塔生成架牵引修改版(3.5) | 黑山大叔 | https://www.bilibili.com/video/BV1RW4y1u7Mj | 不好确定，但是真人能堆叠 | 如前面所述，使用28gt时钟其实是因为怪摔落的时间在28gt左右，刚好能拉开足够的距离 |
| 袭击塔4.0 | 黑山大叔 | https://www.bilibili.com/video/BV1WN411A7dM | 2. 需要稳定的玩家行为 | 能用的原因是时钟很慢，两波怪之间的距离大于招募距离 |
| 30万印钞机，改自小也睡醒了 | 黑山大叔 | https://www.bilibili.com/video/BV1bg4y127Ve | 1. 依赖假人在实体更新阶段的攻击 |  |
| 阿尔法修改版40w袭击塔 | 黑山大叔, alpha-hhh | https://www.bilibili.com/video/BV16y4y1d7r2 | 1. 依赖假人在实体更新阶段的攻击 |  |
| 1.18+340k四村民无恼鬼上迁印钞机 | 沫幽忧 | https://www.bilibili.com/video/BV1Gu411G79Y | 无存档 |  |
| 无恼无雪4村民420k印钞机V2 | 沫幽忧 | https://www.bilibili.com/video/BV1bu411G7tz | 不好评价 | 时序设计不良，按照作者给的使用方式都很容易刷出恼鬼。 |
| 欠时代堆叠袭击农场[430k/h] | 沫幽忧 | https://www.bilibili.com/video/BV1VX4y1x77z | 无存档 |  |
| 欠欠欠时代袭击农场[430k/h] | 沫幽忧 | https://www.bilibili.com/video/BV11m4y1p7ws | 2. 需要稳定的玩家行为 | 不用侦测器观察绊线或许能到 3. |
| 袭畸塔 | 沫幽忧 | https://www.bilibili.com/video/BV1RG411f77y | 无存档 |  |
| 欠时代袭击扁塔[260k/h] | 沫幽忧 | https://www.bilibili.com/video/BV1Su4y1e7g2 | 1. 机器时序依赖假人的特殊性质 | 开关线路上的活塞被QC了 |
| 250k可实装欠扁袭击塔 | 沫幽忧 | https://www.bilibili.com/video/BV1Vj41117fn | 1. 机器时序依赖假人的特殊性质 | 侦测器观察绊线，之后的T触发器会因为恼鬼误触绊线而反转信号 |
| 350k欠扁袭击塔 | 沫幽忧 | https://www.bilibili.com/video/BV1o34y1T7pL | 无存档 |  |
| 340k牛牛喷射塔 | 沫幽忧 | https://www.bilibili.com/video/BV1V14y1C72X | 无存档 |  |
| 袭击农场ProMin[200k/h] | 沫幽忧 | https://www.bilibili.com/video/BV1Yp421Z7X6 | 不好评价 |  |
| 极其优雅的袭击塔 | 丨半世丶浮殇丨 | https://www.bilibili.com/video/BV1Cw411s7ZV | 1. 依赖假人在实体更新阶段的攻击 |  |
| 极其优雅的袭击塔V2 | 丨半世丶浮殇丨 | https://www.bilibili.com/video/BV1Fw411W7nS | 2. 需要稳定的玩家行为 | 需要输入端防T触发反转、自动盔甲架补充和不会阻挡生成的平台 |

--------------------------------

## 1.18+ 堆叠袭击农场设计之怪现象：假人依赖 

1.18+ 的堆叠袭击塔，去掉 carpet 少一半，要求效率不会明显降低再少一半，剩下能用的屈指可数。只要看到一座袭击塔是<u>最简单的向下迁移链 + 刷怪平台没有下弹加速或者附近没有村民 + 玩家始终在村庄区段中</u>，基本可以认定这是一座仅供 carpet 假人使用的袭击塔了，除非这座塔的时钟足够慢。

试问现在玩“生电”的群体中，有多少人默认假人等价于理想网络条件下的真玩家？但是事实正如之前专栏所述，假人的表现和真人完全不同，即使是 `/player` 指令控制的真人也不是，正是这种默认让很多设计者根本没有想过真玩家的可用性是需要验证的。那些所谓的“简易袭击塔”，多数只是把真玩家挂机需要，而假人挂机不需要的部分剔除，就标榜自己省材了。然而他们无意中隐藏了最贵重的部分：不支持真玩家，必须要搭配 carpet mod 才能使用。

我相信很多人曾在别人面前反复强调过“Paper 不原版，Forge 不原版，我们应该用 Fabric”，也知道玩原版红石的出发点是“我的设计，别人哪怕不装任何 mod 也能使用”。重新审视那些仅供 carpet 假人的袭击塔，它们还对得起这些话么？如果我说得不够明白的话也可以看看 [过长的动态 EP003：mod VS 科技](https://www.bilibili.com/read/cv16970398)

## 结语

如果您读到这里，还是认为堆叠袭击塔只能使用假人是理所应当的，那么请允许我如此断定：您只是想要投影和假人带来的便利性，而并不关心获得便利的方式是什么。那么我建议您尝试以下两种选择：使用更为便利的指令，例如 `/give`；或加入更为有趣和便利的模组，例如机械动力、工业、应用能源等。原版的内容过于匮乏，生产方式过于落后，并不适合您游玩。

<br>
<br>
<br>

1.18+ 堆叠袭击农场真人可用性测试结果 © 2024 作者: Youmiel 采用 CC BY-NC-SA 4.0 许可。如需查看该许可证的副本，请访问 http://creativecommons.org/licenses/by-nc-sa/4.0/。