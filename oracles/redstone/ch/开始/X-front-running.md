
# RedStone X


## 防止 front-running 运行的最佳保护措施

该模型采用延迟执行模式，事务分两步处理：

- 用户通过在链上记录与协议交互的意图（即打开一个永久头寸）来启动交易，而不知道执行交易的确切环境（即价格）。这就减少了任何试图通过前置运行来自交易中心的价格交付来套利协议的行为。
- 价格只在第二步推送到链上，通常发生在下一个区块。任何人（包括用户自己）都可以推送价格，因为价格的完整性是根据协议约束在链上验证的。这样的价格将用于最终结算交易。

这种模式在 GMX 等永久协议中得到了推广，并促成了新一轮的超高效 DeFi 项目，这些项目在熊市中仍在快速增长。


note: 总之，你需要做两件事：

- 调整合约，分两个阶段（请求 -> 执行）执行价格敏感型交易。
- 部署一个守护者服务，自动获取价格并触发执行。

![](https://docs.redstone.finance/assets/images/redstone-x-e9ec31fde85dd576db312d3686743644.png)



## 更新智能合约代码

### 阶段1：请求

当用户想要执行价格敏感交易时，我们需要收集一些抵押品、记录请求参数，并要求保管人提供价格数据。

为了明确这些步骤，让我们通过一个更具体的例子来了解它们。有一个简单的协议，可以将本币（如 ETH）兑换成稳定币（如 USDC）。记录交易的示例代码如下：

### 阶段2：执行

```solidity
function changeEthToUsdc() external payable {
    bytes32 requestHash = calculateHashForSwapRequest(
        msg.value,
        msg.sender,
        block.number
    );
    requestedSwaps[requestHash] = true;
    
    emit NewOracleDataRequest(msg.value, msg.sender, block.number);
}
```

在上述函数中，我们

1) 从用户处收集抵押物，并保留与交易相关的 eth。这可以防止我们向协议发送空请求。

2) 公证用户请求的所有必要参数，这些参数是验证执行步骤所必需的。在我们的示例中，让我们密封以下参数的值：

- 要交换的资金数额 msg.value
- 调用者地址 msg.sender
- 提交交易的时间（有必要提供匹配的价格） block.number（将时间戳保留为块号或哈希值更好，因为链上时间戳可能与全局时钟不完全同步，会带来恶意套利的风险）。
- 我们不需要在链上存储所有数据。记录上述值的哈希值就足够了。

3) 通过发出 NewOracleDataRequest 事件，将接收价格数据的新请求通知守护者网络。

在这一阶段，用户的请求与从守卫者网络接收到的数据一起执行。让我们以 eth -> usdc 交换为例，分析一下必要的步骤。

```solidity
function executeWithOracleData(
    uint256 ethToSwap,
    address requestedBy,
    uint256 requestedAtBlock
    ) external payable {

    // Check if the request actually exists
    bytes32 requestHash =
      calculateHashForSwapRequest(avaxToSwap, requestedBy, requestedAtBlock);
    require(requestedSwaps[requestHash],
      "Can not find swap request with the given params");
    delete requestedSwaps[requestHash];

    // We need to validate the timestamp (block.number)
    uint256 dataPackagesBlockNumber = extractTimestampsAndAssertAllAreEqual();
    require(dataPackagesBlockNumber == requestedAtBlock, "Block number mismatch in payload and request");

    // Transfer USDC to the user
    uint256 usdcAmount = getExpectedUsdAmount(ethToSwap);
    usdc.transfer(requestedBy, usdcAmount);
}
```

1) 我们需要验证保存人提供的参数是否与用户公证的参数一致。

2) 然后检查价格来源的时间戳（区块编号）是否与记录请求的时间一致。

3) 如果上述检查都没有问题，我们就可以将 usdc 的适当值发送回发出请求的用户

note: 上述示例的完整代码可在我们的[工作坊中找到](https://github.com/redstone-finance/redstone-3-models-dex-example)，其中解释了 3 种不同的模型。
