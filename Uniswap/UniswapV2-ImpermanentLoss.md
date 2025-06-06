# Uniswap V2 Impermanent Loss

### 什么是无常损失？

无常损失，是指当池子资产价格发生波动时，AMM 会强制重新配比你的资产，这会导致“机会成本”上的损失。也就是 LP 的代币总价值会小于只是单独持有这些代币时的总价值。

### 举例：

在一个 ETH/USDC 的交易对中，池子当时的价格为 100 USDC/ETH ，LP 添加了 1 ETH 和 100 USDC 进入流动性池

这部分流动性为： $\sqrt{1 \cdot 100} = 10$

此时 LP 拥有的价值为：200 USDC

- **ETH 增值**

  ETH 增值到 156.25 USDC/ETH

  由于 LP 的流动性不变， $\sqrt{ 0.8 * 125} = 10$

  此时 LP 拥有代币的数量为 ：$0.8 ETH + 125 USDC$

  LP 拥有的价值： $0.8 \cdot 156.25 + 125 = 250 U$

  假设 LP 最初没有添加流动性，而是持有这些代币，那么此时这些代币的总价值为：$1 \cdot 156.25 + 100 = 256.25 U$
  **添加流动性比不添加流动性少赚了 ：256.25 - 250 = 6.25 U**

- **ETH 贬值**

  ETH 贬值到：64 USDC/ETH

  由于 LP 流动性不变， $\sqrt{1.25 * 80} = 10$

  此时 LP 拥有的代币数量为： $1.25 ETH + 80 USDC$

  LP 拥有的价值： $1.25 \codt 64 + 80 = 160 U$

  假设 LP 最初没有添加流动性，而是持有这些代币，那么此时这些代币的总价值为：$1 \codt 64 + 100 = 164 U$
  **添加流动性比不添加流动性多亏了： 164 - 160 = 4 U**

💡 从上面的例子可以看出来，无论价格是涨还是跌，AMM 都会重新配比 LP 的资产，无常损失也是无法避免的。所以 协议会用手续费来补偿这部分亏损。
