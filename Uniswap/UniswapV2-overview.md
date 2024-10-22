# Uniswap V2

### Constant Product Automated Market Maker

**$x * y = k$**

- x和y代表了一个交易对中各自token的供应量。k是一个不变的常量，k的大小反应了流动性池的大小。资产由LP（流动性提供者）提供给池子，LP会收到LP Token来代表他们在池子中的份额。流动性提供者余额的跟踪方式类似于`ERC4626`的方式。

- 在AMM中，价格发现是自动的。它由池中资产比例决定，无需像订单簿模型中需要等待合适的“出价”和“要价”，只有有流动性，它永远存在。但是这种现货价格并不是实际交易中的价格，实际交易中的价格往往会比较低。

- 在AMM中，只需要记录两个token，并按照一定的规则转移它们，所以与需要大量簿记的订单簿相比，更加节省gas。 
  
### AMM的缺点

- **价格总是变动，滑点频繁**
  在AMM中，价格发现是由池中资产比例决定的，所以每一笔交易都会影响价格。买入或卖出通常会遇到更多的滑点，受流动性深度，大额交易，价格波动，交易顺序等因素影响。攻击者可能会利用滑点发动三明治攻击。

- **LP会遭受无常损失**
  无常损失在AMM中是不可避免的，在其他章节有对于无常损失的计算过程。

### Uniswap V2的架构

Uniswap遵循`core-periphery`的设计模式，最核心的逻辑位于core中，可选逻辑位于periphery。这样做的目的是让core包含尽可能少的代码，减少核心业务逻辑中出现错误的可能性。用户可以选择通过periphery中的合约与core中的合约进行交互，也可以自己定制逻辑通过自己的合约直接与core中的合约交互。

# core

- **UniswapV2Factory**
  1. 负责创建UniswapV2Pair，通过`create2`的方式创建。并在内部保存了所有UniswapV2Pair的地址。
  2. 保存了feeTo这个可以收取治理费用的地址，以及拥有设置并修改feeTo权限的feeToSetter地址。
   
- **UniswapV2Pair**
  1. 该合约持有两个ERC20代币的交易对，每个交易对都有一个UniswapV2Pair合约。如果所需的交易对不存在，则无需许可从UniswapV2Factory创建一个新的合约。
  2. 交易者可以交换，LP可以为其提供流动性。UniswapV2Pair合约本身也是ERC20代币，该代币为LP Token，用于跟踪用户提供的流动性，类似于ERC4626的工作方式。
   
# periphery
  
  提供了2个router合约，这些合约提供一些面向用户的机制，在uniswap的基本逻辑上做了增强，使得用户与uniswap交互更加安全。