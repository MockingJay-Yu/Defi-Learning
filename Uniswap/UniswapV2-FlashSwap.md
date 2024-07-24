# Uniswap V2 Flash Swap

### Flash Swap

**Flash Swap（闪电交换的概念）**

Flash Swap是一种在DeFi领域中使用的代币交换机制。它指的是在一个原子交易（atomic transaction）中，快速、低成本的代币交换机制。

**关于Flash Swap一些关键点**

- 原子性：Flash Swap强调代币的交换必须是在一个原子交易中完成，如果其中一个步骤出错，出错，所有操作全部回滚。
- 低成本：Flash Swap允许用户在短时间内借入资金，实现杠杆作用，从而实现快速交换和套利，而只需要支付交易手续费。
- 安全性：使用智能合约来确保整个交易的安全性和正确性。
- 去中心化：交易不依赖于任何中心化机构。

<aside>
💡 Flash Swap是一个广泛承认的加杠杆的方法，它本身不存在危害。但是可以把某些项目的漏洞无限放大，有些项目需要大量资金才能攻击的漏洞，有了Flash Swap就没有资金门槛了。如果没有Flash Swap那就是谁资金多谁赚的多。

</aside>

### Uniswap V2中的Flash Swap

在Uniswap V2中，Flash Swap是通过swap这个函数实现的。

```solidity
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock 
        if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
        if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
        if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
    }
```

```solidity
interface IUniswapV2Callee {
    function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external;
}
```

swap函数中，calldata这个参数如果不为空的，会调用IUniswapV2Callee().uniswapV2Call()，这个方法需要用户自己实现，用来做一些自定义的操作。从而实现快速交换和套利。

借入的资金来源是LP提供的流动性，偿还的时候可以选择交易对中任意一种token，全部过程只需要支付30个基点交易手续费，就像是完成了一次普通交易一样。