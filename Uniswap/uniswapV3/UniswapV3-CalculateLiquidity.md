# Uniswap V3 Calculate Liquidity

## uniswapV2 和 V3 的区别

**uniswapV2: 全区间，统一池，被动参与**

- LP 的添加的流动性都被平均分布在整个价格区间$(0,\infty)$。无论当前市场价格处于那个位置，LP 的资金总是会参与报价。
- 所有资金都放在一个共享的大池子中，添加新的流动性只需要按照当前市场价格比例，等值提供两种 token。

因此，V2 中的流动性可以直接根据 LP 添加的 token 数量来计算。

**uniswapV3: 指定价格区间，多池，主动控制**

- LP 不再被迫提供全区间流动性，可以指定在特定价格区间提供流动性，每个价格区间都有自己的价格曲线，流动性计算也是独立的。只有当市场价格在该价格区间运行时，LP 的资金才会参与报价。
- 当在非活跃价格区间提供流动性，只需提供其中一种 token，也就是可以提供单边流动性。

因此在 V3 中流动性计算不单单决定于 token 数量，而是由 token 数量和 LP 指定的价格区间和当前市场价格共同决定的。

**Uniswap V2 的流动性分布是全局的，被动的，而 V3 的流动性是主动部署的局部资产，其计算逻辑也随之从“比例确定”转变为“价格区间，token 数量，当前市场价格”的函数关系。**

## 数学模型

<img src="images/UniswapV3-01.jpg" alt="uniswapV3 计算流动性" width="50%" height="50%">

在上一章中，我们计算了 uniswapV3 中某个价格区间的虚拟流动性，这节将重点分析 LP 实际投入的真实流动性，假设存在交易对为$x,y$，价格区间为$[p_a,p_b]$, 当前市场价格为$p$, 在$p$点的真实流动性为$\Delta x$,$\Delta y$。然后我们分析以下三种情况：

1. $p_a<p<p_b$，当前市场价格位于这个价格区间内时，我们可以发现从$p$到$p_a$这段价格曲线的流动性实际上是由资产 $y$ 支撑的，因为价格从$p$到$p_a$消耗的是池子中资产$y$。$p$到$p_b$的流动性是由资产$x$支撑的，价格从$p$到$p_b$消耗的事池子中资产$x$。然后计算$\Delta x$ 和 $\Delta y$：

   $$
   \begin{aligned}
   \Delta x = x_p - x_b = \frac{L}{\sqrt{p}} - \frac{L}{\sqrt{p_b}} = L(\frac{1}{\sqrt{p}} - \frac{1}{\sqrt{p_b}})\\
   \Delta y = y_p - y_a = L \cdot \sqrt{p} - L \cdot \sqrt{p_a} = L(\sqrt{p} - \sqrt{p_a})
   \end{aligned}
   $$

   反推 L：

   $$
   \begin{aligned}
   L = \frac{\Delta x \cdot \sqrt{p} \cdot \sqrt{p_b}}{\sqrt{p} - \sqrt{p_b}} \\
   L = \frac{\Delta y}{\sqrt{p} - \sqrt{p_a}}
   \end{aligned}
   $$

2. $p>= p_b$，当前市场价格大于等于这个价格区间的上边界时，此时在这个区间添加的流动性处于非活跃状态，不会参与当前市场的撮合。只有当前市场价格持续下跌至$[p_a,p_b]$区间时才会被激活，从$p_b$到$p_a$这段价格曲线的流动性是由资产$y$支撑的，计算价格从$p_b$下跌到$p_a$消耗池子中资产$y$的数量$\Delta y$：

   $$
   \begin{aligned}
   \Delta y = y_b - y_a = L \cdot \sqrt{p_b} - L \cdot \sqrt{p_a} = L(\sqrt{p_b} - \sqrt{p_a})
   \end{aligned}
   $$

   反推 L：

   $$
   \begin{aligned}
   L = \frac{\Delta y}{\sqrt{p_b} - \sqrt{p_a}}
   \end{aligned}
   $$

3. $p<= p_a$,当前市场价格小于等于这个价格区间的下边界时，同理，只有市场价格上涨至$[p_a,p_b]$区间内才会被激活，而从$p_a$到$p_b$这段价格曲线的流动性是由资产$x$支撑的，计算价格从$p_a$上涨到$p_b$消耗池子中资产$x$的数量$\Delta x$:

   $$
   \begin{aligned}
   \Delta x = x_a - x_b = \frac{L}{\sqrt{p_a}} - \frac{L}{\sqrt{p_b}} = L(\frac{1}{\sqrt{p_a}} - \frac{1}{\sqrt{p_b}})
   \end{aligned}
   $$

   反推 L：

   $$
   \begin{aligned}
   L = \frac{\Delta x \cdot \sqrt{p_a} \cdot \sqrt{p_b}}{\sqrt{p_a} - \sqrt{p_b}}
   \end{aligned}
   $$

利用上面的公式，我们就可以计算 LP 添加的 token 的数量对应的流动性了，因为当前市场价格$p$，LP 设定要添加流动性的价格区间$[p_a,p_b]$，要添加的资产数量$\Delta x$,$\Delta y$都是已知的。

- 先看第一种情况，当添加的价格区间是当前市场价格活跃的区间时，我们可以看到添加两种 token 会计算出两个 L，那么实际上 uniswapV3 要使用哪个呢？答案是小的那个，因为在用户添加了流动性后，我们要构造一个新的价格曲线，如果两边的 L 不同，那么就不是一条连续的价格曲线。如果取大的那个，就会导致小的那个 L 那边支撑的那种资产不足以支撑其流动性。而取小的那个，可以反过来推算出支撑另一边的流动性需要的资产数量，然后将多余的 token 退还给用户即可。
- 第二和第三种情况本质上是一样的，当添加的价格区间是当前市场价格非活跃的区间时，我们可以看到 LP 只需要添加其中一种 token，也就是所谓的单边流动性，只有当市场价格进入此区间内，该流动性才会被使用到。
