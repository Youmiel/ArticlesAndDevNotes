# 快速回血信标时序分析

快速回血信标的设计者是 Zomie101，TMA Discord 消息链接：https://discord.com/channels/594920796867133446/594937596786900993/894035096246575214

1. 使用了 32gt 的时钟遮挡信标，信标每 80gt 刷新一次药水效果，两者的公倍数是 160，也相当于把信标工作周期延长了一倍，也就是玩家的生命恢复1 效果每 160gt 才会刷新；

2. 生命恢复1 的回血方式是“每当药水效果剩余时间是 50gt 的整数倍时，恢复一点血量”，而满级信标给予玩家 340gt 的生命恢复，结合 80gt 的刷新周期不难发现，玩家只有在生命恢复1 效果剩余 300gt 时回血一次，实际上每 80gt 恢复 1 血，不及正常回血的 50gt 恢复 1 血；

3. 将生命恢复1 效果刷新周期延长到 160gt 后，玩家可以在生命恢复1 效果剩余 300gt、250gt、200gt 时回血，共计三次，也就是每 160gt 恢复 3 血，平均每 80gt 恢复 1.5 血，回血速度是常规信标的 1.5 倍。

## 文件内容

整个 Jupyter Notebook 中都是我用来 Matplotlib 绘制时序图的代码