# Uniswap V2 mint/burn

## 量化流动性

自动化做市商模型，不仅提供了自动价格发现的算法。也需要自动的计算 LP 的收益，由于协议是无需中介，无需许可的。这意味着任何人任何时间都可能提供流动性和撤出流动性，所以应该有一种算法来保证 LP 的收益是公平的，要做到这一点首先要量化池子中的流动性。

| x    | y    | k     |
| ---- | ---- | ----- |
| 1    | 1    | 1     |
| 10   | 10   | 100   |
| 100  | 100  | 10000 |
| 1000 | 1000 | 1e6   |

可以看到，随着资产数量的增长，k 值是成平方级别的增长，而对 K 开方后$\sqrt{k}$是成线性增长的。使用$\sqrt{k}$来表示流动性可以确保新加入的 LP 可以获得与其提供的流动性相对应的份额。

## 追踪 LP 的流动性份额

在量化流动性后，需要追踪每个 LP 的流动性份额，也就是池子中资产的股份。协议使用了类似 ERC4626 的方式来实现。它的工作原理是，存入资产并获得另外一个 token（称为 share）。这个 share 代币就代表 LP 所拥有池子中资产的份额。实际上，池子合约本身也是一个 ERC20，也就是这个 share 代币。

- 当有人添加流动性的时候，增发 share 代币给他，提升他的份额，稀释他人的份额，相当于股份增持。
- 当有人退出流动性的时候，销毁他的 share 代币，降低他的份额，并发放给他对应份额的资产，相当于减持套现。

这样的好处是显而易见的，就是不必时刻去更新所有人的份额，只需要在有人在添加和退出流动性的时候，增发或销毁对应份额的 share 代币。

### 如何计算增发/销毁的 share 代币？

定义：

池子中原本的的 share 代币为 s ，增发或销毁的 share 代币为 $\Delta s$。

量化的流动性$\sqrt{k}$，则池子中原本的流动性为$\sqrt{xy}$,新增或退出的流动性为$\sqrt{\Delta x \Delta y}$

share 代币和流动性份额成正比，则有：

$$
\frac{\sqrt{\Delta x \Delta y}}{\sqrt{xy}} = \frac{\Delta s}{s}
$$

$$
\frac{\Delta x \Delta y}{xy} = \frac{\Delta s}{s}
$$

协议默认添加的流动性应该满足$\frac{\Delta x}{x} = \frac{\Delta y}{y}$，如果有人企图通过提供不平衡的 token 比例来操纵池子的价格,协议总是选择比例小的那个 token 来确定流动性份额，即：$\frac{\Delta x}{x} > \frac{\Delta y}{y}$，$\frac{\Delta s}{s} = \frac{\Delta y}{y}$）。

结合上面的式子:

$$
\Delta s = \frac{\Delta x s }{x} = \frac{\Delta y s }{y}
$$

## mint 代码解析

![UniswapV2-mint](images/UniswapV2-mint.jpg)

### 初始化流动性问题

```solidity
if (_totalSupply == 0) {
    liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
    _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens;
}
```

和任何 vault 类型的合约一样，UniswapV2 也需要防御“通货膨胀攻击”。UniswapV2 的防御措施是在第一笔流动性被添加的时候，将`MINIMUM_LIQUIDITY`数量流动性对应的 share 代币发送到零地址，相当于锁死了这部分流动性。以确保没有人可以拥有池中所有流动性份额，从而发动通货膨胀攻击。

### 流动性比例检查

```solidity
liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
```

用户将会获得提供的两种 token 中计算出来那个最小值的流动性份额，激励用户添加与池中资产比例相等的流动性，否则将会遭受损失。这是为了保护其他 LP 的资产。我们可以看下下面这个例子。

- 假设池子当前有 100 个 token0 和 1 个 token1，LP Token 的总供应量为 10
- 假设 token0 的总价值为 100 美元，token1 的总价值为 100 美元，池子的总资产为 200 美元
- 如果有人添加了 10token0（10 美元）和 1 个 token1（100 美元），总成本 110 美元
- amount0 _ totalSupply / reserve0 = 10 _ 10 / 100 = 1
- amount1 _ totalSupply / reserve1 = 1 _ 10 / 1 = 10
- 如果取最大值，他将获得 10 个 LP token，意味着他拥有了 LP token 总供应量的 50%
- 但是他的流动性资产只占到了池中所有资产的：110 / 310 = 35.4%，相当于他偷走了其他 LP 的资产

## burn 代码解析

![UniswapV2-burn](images/UniswapV2-burn.jpg)

### 移除的流动性是通过池合约收到的 LP token 数量来衡量的

```solidity
uint liquidity = balanceOf[address(this)];
```

直接与`burn`函数交互，需要在调用前向池合约转入 LP token。这两次调用需要在一个交易内，否则其他人可以燃烧掉你的 LP token 并偷走你的流动性。

### 计算移除的流动性资产

```solidity
uint _totalSupply = totalSupply;
amount0 = liquidity.mul(balance0) / _totalSupply;
amount1 = liquidity.mul(balance1) / _totalSupply;
require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
```

这个公式在上面已经推导过。
