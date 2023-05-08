# Optimism 分析

1.  OP 上的 native token 到底是 ETH 还是别的。 ETH 在 L2 上是原生的，还是合约
2.  在 L2 上提款的时候，sendCrossDomainMessage 是怎么发给 L1，如果有监听服务的话，是否做到了去中心化
3.  在 L2 消息路由到 L1 的 Messenger 中的时候，如何去附带一个 proof，让 L1 信服
4.  在 L1 上提款的时候，finalizeETHWithdrawal 是谁调用的，符合什么条件，比如是否过了挑战期，对应 L2 上的提款 tx 是否在 L1 上已经 finalized
5.  如果挑战者是成功的，SCC 是怎么 revert 的，以及更加详细的惩罚措施

## L1 -> L2 Deposit

- 时序图 1

```plantuml
@startuml
actor user
Box "L1" #lightblue
user -> L1StandardBridge.sol: depositETH/depositETHTo\ndepositERC20/depositERC20To
L1StandardBridge.sol -> L1StandardBridge.sol: _initiateETHDeposit/_initiateERC20Deposit\nbridge合约接收Eth或者ERC20 Token
L1StandardBridge.sol -> CrossDomainEnabled.sol: sendCrossDomainMessage
CrossDomainEnabled.sol -> L1CrossDomainMessenger.sol: sendMessage\nemit SentMessage
L1CrossDomainMessenger.sol -> L1CrossDomainMessenger.sol: enqueue\nemit TransactionEnqueued
end box
Box "L2-data-transport-layer" #orange
l1service.ts -> l1service.ts: _init() 加载配置
l1service.ts -> l1service.ts: _start() 监听事件
note left
监听3个事件TransactionEnqueued/SequencerBatchAppended/StateBatchAppended
end note
l1service.ts -> l1service.ts: _syncEvents()
note left
将监听到的事件根据事件类型通过storeEvent存储到数据库
end note
end box
@enduml
```

- **L1 合约解析**

  - depositETH/depositERC20 只允许 EOA 地址调用，不能指定接收地址，想指定地址调用 depositETHTo/depositERC20To
  - sendCrossDomainMessage(l2TokenBridge, l2Gas, message)前，组织 message 字段，调用 l2TokenBridge 合约的 finalizeDeposit 接口

  ```shell
  bytes memory message = abi.encodeWithSelector(
            IL2ERC20Bridge.finalizeDeposit.selector,
            address(0),
            Lib_PredeployAddresses.OVM_ETH,
            _from,
            _to,
            msg.value,
            _data
        );
  ```

  - enqueue(address target, uint256 gasLimit,bytes memory data)前，组织 data 字段，调用 L2CrossDomainMessenger 合约的 relayMessage 接口

  ```shell
  function encodeXDomainCalldata(
        address _target,
        address _sender,
        bytes memory _message,
        uint256 _messageNonce
    ) internal pure returns (bytes memory) {
        return
            abi.encodeWithSignature(
                "relayMessage(address,address,bytes,uint256)",
                _target,
                _sender,
                _message,
                _messageNonce
            );
    }
  ```

  - 合约流 User -> L1StandardBridge -> L1CrossDomainMessenger -> L2CrossDomainMessenger -> L2StandardBridge
  - data 的 size 不能超过 50000
  - 100000 < gaslimit < 15000000
  - 关于 L2 gaslimit, 由于 L1->L2 交易是免 L2 gas 费的，为了防止设置过大的 gaslimit 来阻塞 L2 的运行，CTC 合约做了限制，当 gaslimit 大于 1920000 时，必须要支付额外燃烧的 gas 费

  ```shell
  if (_gasLimit > enqueueL2GasPrepaid) {
        uint256 gasToConsume = (_gasLimit - enqueueL2GasPrepaid) / l2GasDiscountDivisor;
        uint256 startingGas = gasleft();
        require(startingGas > gasToConsume, "Insufficient gas for L2 rate limiting burn.");
        uint256 i;
        while (startingGas - gasleft() < gasToConsume) {
            i++;
        }
    }
  ```

  - queueElements 记录所有的 L1->L2 交易的简介，使用 queueIndex 作为本交易的在 L2 对于交易的 nonce 值

---

- **L2 Data-transport-layer 要点解析**

  - data-transport-layer 组件是由 ts 写的服务，负责监听 L1 合约发出的事件，将其放入数据库
  - 配置要监听的起始 L1 块高，获取当前 L1 最新的确认块高(减去 12,12 个延时最终确认)
  - 遍历两者之间的块高，过滤 3 个事件
    - L1 CTC 合约的 TransactionEnqueued 事件
    - L1 CTC 合约的 SequencerBatchAppended 事件
    - L1 SCC 合约的 StateBatchAppended 事件
  - 解码事件成结构体，调用对应文件的 storeEvent 将数据存入数据库，并且更新最新同步的块高

---

- **时序图 2**
  syncQueueTransactionRange

```plantuml
@startuml
Box "l2geth module" #grey
main.go -> worker.go: makeFullNode/.../newWorker
note left
启动worker进程，rollupCh订阅了NewTxsEvent事件，并且像eth一样启动了4个进程
end note
worker.go -> worker.go: mainLoop
main.go -> main.go: startNode
main.go -> sync_service.go: Start()
alt isVerifyNode
sync_service.go -> sync_service.go: VerifierLoop()
loop
sync_service.go -> sync_service.go: verify()
end
else isSequencer
loop
sync_service.go -> sync_service.go: SequencerLoop/sequence/syncQueueToTip
sync_service.go -> sync_service.go: syncQueue/syncQueueTransactionRange
loop start<end
sync_service.go -> client.go: GetEnqueue/enqueueToTransaction
note left
比较当前rollup最新index跟数据库最新的index，从数据库(由L2-data-transport-layer
模块插入的数据)查询并且转变成Transaction结构
end note
sync_service.go -> sync_service.go: applyTransaction/applyTransactionToTip
sync_service.go -> feed.go: Send 发出NewTxsEvent到channel
end
end
end
feed.go -> worker.go: rollupCh收到消息
worker.go -> worker.go: commitNewTx
worker.go -> state_processor: ApplyTransaction
worker.go -> worker.go: commit() 执行交易，确认修改，->w.taskCh
note left
激活了taskCh，taskLoop获取密封任务并将其推送到共识引擎
end note
end box
@enduml
```

- **l2geth module 解析**
  - Sequencer 默认 15 秒从数据库拉取 L1->L2 的数据
  - Sequencer 从 data-transport-layer 查询 TransactionEnqueued 事件，转为交易并执行，挖出对应区块
  - L1 过来的交易转化为 L2 交易时，to, gasLimit, data 设置为 L1 调用的对应字段，value, gasPrice 设置为 0，nonce 设置为 queueIndex，v, r, s 未设置
  - 因此 L1->L2 交易 From 的 nonce 可能不连续，跳过 nonce 检查，rsv 未设置，From 设置为 L1MessageSender
  - 因为 L2 免 gas 费，gaslimit 可以随便设置，为了防止攻击，所以在 L1 上对 gaslimit 过大做了 gas 燃烧的机制

---

## L2 -> L1 Withdraw

- 时序图

```plantuml
@startuml
actor user
Box "L1" #lightblue
user -> L2StandardBridge.sol: withdraw/withdrawTo
L2StandardBridge.sol -> L2StandardBridge.sol: _initiateWithdrawal/burn token(eth/erc20一样逻辑)
L2StandardBridge.sol -> CrossDomainEnabled.sol: sendCrossDomainMessage\nemit WithdrawalInitiated
CrossDomainEnabled.sol -> L2CrossDomainMessenger.sol: sendMessage\nemit SentMessage
L2CrossDomainMessenger.sol -> OVM_L2ToL1MessagePasser.sol: record sentMessages hash
end box
Box "L2-batch-submitter" #orange
main.go -> batch_submitter.go: Main()
batch_submitter.go -> batch_submitter.go: 初始化配置
note left
创建L1/L2 Client，TxBatchSubmitter，StateBatchSubmitter
end note
loop all service
batch_submitter.go -> service.go: eventLoop()
service.go -> seq_driver.go: GetBatchBlockRange获取要push到CTC合约的start/end块高
service.go -> seq_driver.go: CraftBatchTx遍历块高，把所有交易打包成一笔交易发给CTC合约
pro_driver.go -> CTC.sol: RawTransact
service.go -> pro_driver.go: GetBatchBlockRange获取要push到SCC合约的start/end块高
service.go -> pro_driver.go: CraftBatchTx遍历块高，把所有交易的roothash打包成一笔交易发给SCC合约
pro_driver.go -> SCC.sol: appendStateBatch
SCC.sol -> SCC.sol: _appendBatch
end
end box
@enduml
```

- **l2geth module 解析**

  - L2 withdraw 到 L1 与 L1->L2 逻辑大体一致，data 为调用 L1StandardBridge.sol 的 finalizeETHWithdrawal 或者 finalizeERC20Withdrawal 函数
  - 打包 root 到 SCC 合约有最小数量限制 MinStateRootElements
  - 打包 tx 到 CTC 合约有配置交易的限制 MinTxSize MaxTxSize
  - 打包的 sender 必须在 BondManager isCollateralized 有质押的，目前合约写的必须 OVM_Proposer 地址
  - appendBatch 将多笔 stateRoot 组织成一棵 Merkle Tree，得到根哈希 batchRoot，组织 meta 信息，得到 batchHeader 存储 batchHeader 到数组 (可以通过 batchIndex 进行索引)

---

- 时序图 relayer

```plantuml
@startuml
[->cross_chain_message.ts: getMessageProof(message)
note left
通过sdk为消息生产证明，返回一系列参数用于调用replayMessage函数
end note
[->L1CrossDomainMessage.sol:relayMessage用上面返回的值作为参数
L1CrossDomainMessage.sol -> L1CrossDomainMessage.sol:_verifyXDomainMessage
L1CrossDomainMessage.sol -> SCC.sol: verifyStateCommitment
note left
_verifyStateRootProof校验是否过挑战期
verifyStateCommitment校验证明的有效性等
end note
L1CrossDomainMessage.sol -> L1CrossDomainMessage.sol: _target.call(_message)调用data具体方法\n修改状态变量等
@enduml
```

relayer 用于测试，生产需要自己用 sdk 嵌入到产品，由用户或者项目方来支付 L1 的 gas 费用

---
