# Uniswap V3 SwapRouter

## SwapRouter 是什么

在 Swap 章节，我们已经学过 pool 合约的 `swap`函数，看起来参数很少，但直接与其交互技术门槛较高：调用者要明白`amountSpecified`的正负含义，设置`sqrtPriceLimitX96`以防滑点，自己实现回调函数，并在多跳场景下自行拆分与串联多次 swap。因此，`SwapRouter` 的价值就在于把这些复杂性封装起来，为用户提供安全，易用的交易接口。

## exactInput

这个函数的的作用是：给定固定数量的输入 token，兑换成输出 token

### 参数结构

```solidity
struct ExactInputParams {
    bytes path;            // 交易路径（编码格式的多跳路径）
    address recipient;     // 最终接收输出 token 的地址
    uint256 amountIn;      // 输入 token 数量
    uint256 amountOutMinimum; // 最小可接受输出，防止滑点过大
    uint256 deadline;      // 截止时间
}
```

`path`是一个压缩过的路径，它的格式如下：

```solidity
tokenA (20 bytes) | fee (3 bytes) | tokenB (20 bytes) | fee (3 bytes) | tokenC (20 bytes)
```

### 代码解析

```solidity
function exactInput(ExactInputParams memory params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    // 1️⃣ 第一步：确定初始的付款方
    // 一开始，Router 并不持有任何 token，所有输入来自用户（msg.sender）
    address payer = msg.sender; // msg.sender pays for the first hop

    // 2️⃣ 第二步：循环处理多跳路径（可能是 A→B→C→D）
    while (true) {
        // 判断路径中是否还有多个 pool（即是否还有下一跳）
        bool hasMultiplePools = params.path.hasMultiplePools();

        // 3️⃣ 第三步：调用内部函数执行一次“单跳 swap”
        // exactInputInternal 只处理 path 的第一段 (A→B)
        // 返回值是该跳后的输出（也将作为下一跳的输入）
        params.amountIn = exactInputInternal(
            params.amountIn,                                 // 当前输入 token 数量
            hasMultiplePools ? address(this) : params.recipient, // 如果还有下一跳，先存在 Router；否则直接给用户
            0,                                                // sqrtPriceLimitX96 (设为0表示不限制价格)
            SwapCallbackData({
                path: params.path.getFirstPool(),             // 仅取 path 的第一段 (tokenIn | fee | tokenOut)
                payer: payer                                 // 谁负责支付这次输入（第一跳是用户）
            })
        );

        // 4️⃣ 第四步：判断是否继续下一跳
        if (hasMultiplePools) {
            // 当前这跳结束后，Router 已经收到了 tokenOut（它将是下一跳的 tokenIn）
            // 所以 Router 现在变成 payer
            payer = address(this);

            // 跳过当前 token，更新 path 为下一段 (B→C)
            params.path = params.path.skipToken();
        } else {
            // 如果没有更多的 pool，说明交易完成
            // 最后的 amountIn 实际上就是最终输出
            amountOut = params.amountIn;
            break;
        }
    }

    // 5️⃣ 第五步：安全检查
    // 防止因为价格波动导致收到的数量太少（滑点保护）
    require(amountOut >= params.amountOutMinimum, 'Too little received');
}

```

## exactOut

这个函数的的作用是：指定固定数量的输出 token，需要多少输入 token

### 参数结构

```solidity
struct ExactOutputParams {
    bytes path;              // 交易路径（倒序编码的路径）
    address recipient;       // 最终接收 output token 的地址
    uint256 amountOut;       // 想要拿到的 output token 的精确数量
    uint256 amountInMaximum; // 用户愿意支付的 input token 最大数量
    uint256 deadline;        // 交易有效期（时间戳）
}
```

在这里，`path`是从目标 token 开始倒推输入 token，所以是编码时是从输出到输入

### 代码解析

```solidity
function exactOutput(ExactOutputParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    /**
     * Step 1: 调用 exactOutputInternal()
     *
     * 这个函数才是真正执行 swap 的地方，它会：
     *  - 从路径的最后一跳（目标 token 池）开始，倒序执行 swap。
     *  - 每个 swap 都会触发 pool 的回调，Router 在回调中负责支付输入 token。
     *  - 最终得出总共需要多少输入 token。
     *
     * 注意：这里固定了 payer = msg.sender，
     * 因为只有“最后一跳”的 swap（也就是第一个被执行的 swap）需要用户实际付款，
     * 其余中间 hop 的支付由 Router 在回调中完成（嵌套调用）。
     */

    exactOutputInternal(
        params.amountOut, // 想拿到的最终输出 token 数量（精确值）
        params.recipient, // 最终接收 token 的地址
        0,                // 初始 amountIn = 0，由内部递归计算得出
        SwapCallbackData({
            path: params.path,       // 交易路径，例如 tokenA → tokenB → tokenC
            payer: msg.sender        // 用户是最终付款方
        })
    );

    /**
     * Step 2: 从缓存中取回最终计算出的输入数量
     *
     * exactOutputInternal 执行完后会把实际的花费 amountIn
     * 暂存在 amountInCached 变量中（因为是内部递归结构，不方便直接返回）
     */
    amountIn = amountInCached;

    /**
     * Step 3: 检查花费是否超出用户允许的最大值
     *
     * 用户调用 exactOutput() 时会传入一个 params.amountInMaximum，
     * 这是他愿意花的最多的输入 token。
     * 如果实际花费更高，则 revert。
     */
    require(amountIn <= params.amountInMaximum, 'Too much requested');

    /**
     * 🧹 Step 4: 清理缓存
     *
     * 避免后续交易意外复用之前的缓存值。
     */
    amountInCached = DEFAULT_AMOUNT_IN_CACHED;
}

```

## uniswapV3SwapCallback

```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata _data
) external override {
    // 必须至少有一个 delta > 0，否则 swap 没有意义
    require(amount0Delta > 0 || amount1Delta > 0);

    // 解码 Router 在 swap 时 encode 的 SwapCallbackData
    SwapCallbackData memory data = abi.decode(_data, (SwapCallbackData));

    // 验证调用者是合法池子，防止恶意调用
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
    CallbackValidation.verifyCallback(factory, tokenIn, tokenOut, fee);

    // 判断当前池子要求支付的代币和数量
    // amount0Delta > 0 表示池子要求支付 token0，amount1Delta > 0 表示要求支付 token1
    (bool isExactInput, uint256 amountToPay) =
        amount0Delta > 0
            ? (tokenIn < tokenOut, uint256(amount0Delta))
            : (tokenOut < tokenIn, uint256(amount1Delta));

    if (isExactInput) {
        // -------------------------------
        // exactInput 直接支付 tokenIn 给当前池子
        // -------------------------------
        pay(tokenIn, data.payer, msg.sender, amountToPay);
        // data.payer 通常是用户钱包
        // msg.sender 是池子地址
    } else {
        // -------------------------------
        // exactOutput 情况：可能是多跳 swap
        // -------------------------------
        if (data.path.hasMultiplePools()) {
            // 还有后续池子
            // 1. 跳过当前 token，进入下一跳
            data.path = data.path.skipToken();

            // 2. 递归调用 exactOutputInternal
            //    计算上一跳需要多少 tokenIn，逐层反算
            // 3. 最顶层 token 资金来源是用户钱包
            exactOutputInternal(amountToPay, msg.sender, 0, data);
        } else {
            // 单跳终点：
            // 直接从用户钱包支付给池子
            amountInCached = amountToPay;
            tokenIn = tokenOut; // swap in/out 因为 exactOutput 是倒序计算
            pay(tokenIn, data.payer, msg.sender, amountToPay);
        }
    }
}
```

从这里结合上面两个函数，`Router`扮演了资金调度的角色：

1. exactInput：

- 用户授权 Router 支付第一跳的 tokenIn，然后每次 pool 回调 `uniswapV3SwapCallback` 时，Router 将 token 转给当前池子（pay(tokenIn, payer, pool, amount)）。

- 中间跳的输出 token 暂时保存在 Router 内部（recipient 设置为 Router 地址），

- 最后一跳的 recipient 设置为用户，用户最终收到 tokenOut。

2. exactOutput：

- 多跳：Router 在第一跳中，pool 回调 `uniswapV3SwapCallback` 时递归调用 `exactOutputInternal` 计算上一跳需要多少 tokenIn，逐层反算。

- 终点跳：Router 从用户钱包支付最顶层 tokenIn 给池子（pay(tokenIn, payer, pool, amount)）。

- 每一跳 swap 的资金流都是通过 Router 调度的，最终保证用户只需提供最顶层的 tokenIn，就能完成整个多跳 exactOutput swap。
