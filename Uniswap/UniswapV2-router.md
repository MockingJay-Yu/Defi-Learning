# UniswapV2 router
>`periphery`中提供了两个重要的合约`UniswapV2Router01`和`UniswapV2Router02`。其中后者继承了前者，并增加了对交易token收费的支持。

## router合约是面向用户的合约，用于
- 安全的添加和移除流动性
- 安全的swap
- 添加了`core`合约中省略的与滑点相关的安全检查
- 通过与`WETH`合约集成，增加交换以太币的能力
- 增加了对`Fee on Transfer Tokens`的支持

## swapExactTokensForTokens和swapTokensForExactTokens

![UniswapV2-router](images/UniswapV2-router.jpg)

### 参数
- `swapExactTokensForTokens`：意味者正在交换的输入token的数量是固定的。用户需要准确指定将要存入的token的数量`amountIn`。接受的输出token的最低数量`amountOutMin`。
- `swapTokensForExactTokens`：意味者接收的输出token的数量是固定的。用户需要准确指定想要接收的token的数量`amountOut`。需要存入的token的最大数量`amountInMax`。
- `path`：用户可以指定需要交换的token路径，可以跨越多个池子。
- `to`：接收地址
- `deadline`:最晚时间
  
### 使用哪个函数？
大多数EOA用户可能会选择使用精确输入的函数，因为他们需要有一个批准交易的步骤，如果他们输入的超过他们批准的数量，交易就会失败，通过获得准确的输入，他们可以批准确切的金额。

与Uniswap集成的智能合约可能具有更复杂的要求，因此Uniswap提供了两种选择。

### 代码解析
```solidity
amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
```
当调用`swapExactTokensForTokens`时，首选会根据入参计算单个swap或跨越多个池子的swap预期的输出，如果预期的输出低于用户指定的最低输出，函数将revert。对于`swapTokensForExactTokens`，计算所需的输入，如果高于用户指定的最大输入数量，则revert。

```solidity
TransferHelper.safeTransferFrom(
    path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
);
_swap(amounts, path, to);
```
- 然后这两个函数都会将用户的token转入到`path`中第一个池子中去。在`UniswapV2Pair`中的`swap`函数要求用户要先把token转账到池子合约中去。
- 最后他们都调用了`_swap`函数。

![UniswapV2-router2](images/UniswapV2-router2.jpg)

## _addLiquidity

在mintAndburn章节，我们提到过流动性安全检查。具体来说，我们希望确保存入的两种token数量与池子中的token余额比率是相同的，否则，我们获得的LP Token数量是存入的数量和池子余额两个比率中最小的那个。但是在LP发起添加流动性交易和交易被确认之间，池子中资产会发生变化。router中提供了`_addLiquidity`函数可以为LP提供必要的流动性安全检查。

![UniswapV2-router3](images/UniswapV2-router3.jpg)

## removeLiquidity

移除流动性会燃烧LP Token，如果token比率在LP发起移除流动性交易和交易被确认之间发生剧烈变化，那么LP将无法取回他们预期的token数量。router中提供了`removeLiquidity`函数可以为LP提供必要的安全检查。

![UniswapV2-router4](images/UniswapV2-router4.jpg)

## 对`Fee on Transfer Tokens`的支持

在UniswapV2中，处理`Fee on Transfer Tokens`时需要进行特殊处理，因为这些token在每次转账的时候会扣除一部分费用，不能直接对用户输入的token数量等参数进行计算。在swap和removeLiquidity时会涉及到这种情况，addLiquidity不受影响，用户实际转入多少，就记多少。

