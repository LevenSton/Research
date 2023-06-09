# 模块化区块链笔记

## Cosmos 笔记

### AppChain architecture

Cosmos 架构

- TerdenmintCore 负责共识和 P2P 模块
- AppChain 模块通过 Cosmod SDK 构建 APPChain Layer，通过 ABCI 与 TerdenmintCore 交互，业务逻辑 APPChain 实现

![archi](./pic/AppChainArchi.png)

### TX 的生命周期和流程

![TxLifeCycle](./pic/TxLifeCycle.png)

1. TerdenmintCore 接收到用户的交易，通过 ABCI 调用 CheckTX，校验交易合法性
2. 不合法直接丢弃，合法放入 mempool，并且广播给 peer 节点
3. 由出块者从交易池根据规则收录交易打包
4. 通过 ABCI 调用 BeginBlock，标志区块开始
5. 通过 ABCI 将打包的交易发送到 Appchain layer 进行执行，失败所有状态回滚
6. 执行完毕后，通过 ABCI，调用 EndBlock 标志区块结束

### 从交易流程分析 Cosmos 源码

- CheckTx 实现了 ABCI 接口并在 CheckTx 模式下执行交易。在 CheckTx 模式下，消息不会被执行。这意味着消息仅被验证，
- 并且只有 AnteHandler 被执行。如果 AnteHandler 通过，则状态将持久化到 BaseApp 的内部 CheckTx 状态中。
- 否则，ResponseCheckTx 将包含相关的错误信息。无论交易执行结果如何，ResponseCheckTx 都将包含相关的 gas 执行上下文信息。

```plantuml
@startuml
title : Tx 时序图
entity TerdenmintCore #Red
TerdenmintCore-> abci.go: InitChain\n创世启动配置起始块高，变量等
TerdenmintCore-> abci.go: BaseApp.BeginBlock\n每个区块开始前收到RequestBeginBlock
abci.go -> abci.go: BeginBlock函数配置state，以及一些验证者信息
TerdenmintCore-> abci.go: BaseApp.CheckTx\n TerdenmintCore发送ABCI接口消息
abci.go -> abci.go: BaseApp.runTx 校验接收到的消息
abci.go -> decoder.go: BaseApp.txDecoder(bytes)解码bytes成sdk.TX
abci.go -> types.go: validateBasicTxMsgs 校验字段/gas/签名合法性
abci.go -> abci.go: anteHandler将修改状态临时存储
abci.go -> sender_nonce.go: SenderNonceMempool.Insert 添加到交易池
abci.go -> TerdenmintCore: 返回ResponseCheckTx
TerdenmintCore-> abci.go: BaseApp.DeliverTx\n执行交易
abci.go -> abci.go: runMsgs执行交易\n app.msgServiceRouter.Handler(msg)路由消息到上层
abci.go -> TerdenmintCore: 返回ResponseDeliverTx
@enduml
```

### 随笔记

![archi](./pic/cosmosnode.jpg)

```plantuml
@startuml

class SimApp
class Manager{
  +Modules map[string]AppModule
  +RegisterServices(cfg Configurator)
}
class Bank_BaseKeeper
class OtherKeeper
class BaseApp
interface module_AppModule
interface module_AppModuleGenesis
interface module_AppModuleBasic
class auth_AppModule{
  +RegisterServices(cfg module.Configurator)
}
class other_AppModule{
  +RegisterServices(cfg module.Configurator)
}
class MsgServiceRouter {
	-routes map[string]MsgServiceHandler
  +RegisterService(sd *grpc.ServiceDesc, handler interface{})
}
interface module_Configurator{
  +MsgServer() grpc.Server
	+QueryServer() grpc.Server
}
class configurator {
	-msgServer   grpc.Server
	-queryServer grpc.Server
}
class ServiceDesc {
	+ServiceName string
	+HandlerType interface{}
	+Methods     []MethodDesc
	+Streams     []StreamDesc
	+Metadata    interface{}
}
class msgServer {
	Keeper
}
class SimApp extends BaseApp
class BaseApp extends MsgServiceRouter
SimApp o-- Manager
SimApp o-- Bank_BaseKeeper
SimApp o-- OtherKeeper
interface module_AppModule extends module_AppModuleGenesis
interface module_AppModuleGenesis extends module_AppModuleBasic
Manager o-- auth_AppModule
Manager o-- other_AppModule
class auth_AppModule extends module_AppModule
class other_AppModule extends module_AppModule
class configurator extends module_Configurator
auth_AppModule *.. ServiceDesc
auth_AppModule *.. msgServer
Bank_BaseKeeper o-- msgServer
SimApp o-- configurator
MsgServiceRouter *.. ServiceDesc
MsgServiceRouter *.. Bank_BaseKeeper
@enduml
```

- 在 app.go 中创建定义的状态机实例
- 使用从存储在~/.app/data 文件夹中的数据库中提取的最新已知状态初始化状态机
- 创建并启动一个新的 Tendermint 实例。节点将与其对等方进行握手，获取最新的 blockHeight，并回放块以同步到此高度（如果它大于本地 appBlockHeight）。如果 appBlockHeight 为 0，则节点从创世区块开始运行，并通过 ABCI 向应用程序发送 InitChain 消息，触发 InitChainer。

* App 实例包含：（App.go）
  - baseapp 的引用，Cosmos Sdk 中的
  - A list of store keys，一组存储 key，要操作对于的存储要使用对应的 key
  - A list of module's keepers。每个模块都定义了一个抽象的保管者，用于处理该模块存储的读写操作，要使用对应的 key 创建 Kepper
  - A reference to a module manager and a basic module manager，模块管理器是一个包含应用程序模块列表的对象。它方便与这些模块相关的操作，例如注册它们的 Msg 服务和 gRPC 查询服务，或为各种功能（如 InitChainer、BeginBlocker 和 EndBlocker）设置模块之间执行顺序。

### BeginBlock

- BeginBlock 流程
  ![archi](./pic/beginBlock.png)

### CheckTx

- CheckTx 流程
  ![archi](./pic/CheckTx.png)

### Commit

- Commit 流程
  ![archi](./pic/Commit.png)

### TxFlow 流过模块

- TxFlow 模块
  ![archi](./pic/TxFlow.png)

### App-Specific 组成

- customApp.go
  自定义应用程序,是 sdk 中 baseapp 的扩展。当 TerdenmintCore 将交易中继到应用程序时，应用程序使用 baseapp 的方法将其路由到适当的模块。baseapp 实现了大部分应用程序的核心逻辑，包括所有 ABCI 方法和路由逻辑。
- 各模块 Keeper
  每个模块定义了一个抽象 keeper，它处理该模块存储的读写操作。kepper 直接调用要互相授权，每个 keeper 有自己的 store key 去访问自己的存储区
- appCodec
  负责数据的序列号和反序列话，便于数据的存储
- module manager
  模块管理器是一个对象，它包含应用程序模块的列表。方便操作这些模块，例如注册它们的 Msg 服务和 gRPC 查询服务，或为各种功能（如 InitChainer、BeginBlocker 和 EndBlocker）设置模块之间执行顺序。

源码例子：

```go
type SimApp struct {
	*baseapp.BaseApp
	legacyAmino       *codec.LegacyAmino
	appCodec          codec.Codec
	interfaceRegistry types.InterfaceRegistry

	invCheckPeriod uint

	// keys to access the substores
	keys    map[string]*storetypes.KVStoreKey
	tkeys   map[string]*storetypes.TransientStoreKey
	memKeys map[string]*storetypes.MemoryStoreKey

	// keepers
	Bank_BaseKeeper    authkeeper.Bank_BaseKeeper
	Bank_BaseKeeper       bankkeeper.Keeper
	CapabilityKeeper *capabilitykeeper.Keeper
	StakingKeeper    stakingkeeper.Keeper
	SlashingKeeper   slashingkeeper.Keeper
	MintKeeper       mintkeeper.Keeper
	DistrKeeper      distrkeeper.Keeper
	GovKeeper        govkeeper.Keeper
	CrisisKeeper     crisiskeeper.Keeper
	UpgradeKeeper    upgradekeeper.Keeper
	ParamsKeeper     paramskeeper.Keeper
	AuthzKeeper      authzkeeper.Keeper
	EvidenceKeeper   evidencekeeper.Keeper
	FeeGrantKeeper   feegrantkeeper.Keeper
	GroupKeeper      groupkeeper.Keeper
	NFTKeeper        nftkeeper.Keeper

	// the module manager
	mm *module.Manager

	// simulation manager
	sm *module.SimulationManager

	// module configurator
	configurator module.Configurator
}
// BaseApp reflects the ABCI application implementation.
//BaseApp 反映了 ABCI 应用程序的实现
type BaseApp struct { //nolint: maligned
	// initialized on creation
	......
}
```

- 构造函数

```go
func NewSimApp(
  logger log.Logger,
  db dbm.DB,
  traceStore io.Writer,
  loadLatest bool,
  appOpts servertypes.AppOptions,
  baseAppOptions ...func(*baseapp.BaseApp),
) *SimApp
```

构造函数执行的主要操作如下：

- 使用基本管理器实例化一个新编解码器并初始化应用程序模块的每个编解码器。
- 使用引用到 baseapp 实例、编解码器和所有适当的存储键来实例化一个新应用程序。
- 使用每个应用程序模块的 NewKeeper 函数定义，实例化应用程序类型中定义的所有 keeper 对象。请注意，必须按正确顺序实例化 keepers，因为一个模块的 NewKeeper 可能需要对另一个模块的 keeper 进行引用。
- 使用每个应用程序模块的 AppModule 对象来实例化应用程序模块管理器。
- 通过该模块管理器，初始化应用程序消息服务、gRPC 查询服务、传统消息路由和传统查询路由。当 CometBFT 通过 ABCI 将交易中继到该应用时，它会使用此处定义的路由将其路由到相应模块的 Msg 服务。同样地，在收到 gRPC 查询请求时，它会使用此处定义的 gRPC 路由将其路由到相应模块的 gRPC 查询服务上。Cosmos SDK 仍支持传统 Msgs 和传统 CometBFT 查询，并分别使用传统 Msg 路由和传统查询路由进行路由。
- 通过该模块管理器，在变量不变式注册表中注册该应用程序各自所属于各自不变式。不变量是在每个区块结束时计算（例如代币总供给） 的变量。

## Celestia 笔记

### DA Layer Modular Blockchain

Celestia DA 层由一个 PoS 区块链组成。 Celestia 将这个区块链称为 celestia-app，它是一个提供交易以促进 DA 层的应用程序，并使用 Cosmos SDK 构建。

![archi](./pic/celestiaA.jpg)

celestia-app 是建立在 celestia-core 之上的，后者是 Tendermint 共识算法的修改版本。改动：

1. 启用块数据的抹除编码（使用二维 Reed-Solomon 编码方案）。
2. 将 Tendermint 用于存储块数据的常规 Merkle 树替换为 Namespaced Merkle 树，从而使上述层（即执行和结算）仅下载所需数据（有关详细信息，请参见下面描述用例的部分）。

### The lifecycle of a celestia-app transaction

类似于 Cosmos SDK：
![archi](./pic/life.jpg)

- 用户通过 PayForData 交易将数据交给 Celestia DA 层处理
- Celestia 通过类似 Cosmos ABCI++ABCI++将消息传给 STATE MACHINE 处理，将消息解析为 2 部分：

  ```go
  Msg{
  data(message)
  namespace ID
  }
  e-TX{
  sender
  data commitment
  data size
  namespace ID
  signature
  }
  ```

* 区块生产者将区块数据的承诺添加到区块头中。

### Data availability

![archi](./pic/network.jpg)
为了增强连接性，celestia-node 通过一个单独的 libp2p 网络（即所谓的 DA 网络）来扩展 celestia-app，以处理 DAS 请求。

- 轻节点监听区块头，可选的对数据进行 DAS 验证
- 轻节点可以发送 PayForData 请求

### 节点类型

- Consensus Network

  - Validator Node
    验证者节点，通过生成和投票区块来参与共识。
  - Consensus Full Node
    全节点，A Celestia-App Full Node 同步区块链历史数据

- Data Availability Network
  - Bridge Node
    桥节点，在 DA 层和共识层桥接区块
  - Full Storage Node
    存储所有数据，但是不连接共识网络
  - Light Node
    轻节点，在 DA 网络通过数据采样验证数据可用性

### 仓库

- https://github.com/celestiaorg/celestia-app
  celestia-app 是应用程序和权益证明逻辑运行的状态机。 celestia-app 基于 Cosmos SDK 构建，还包括 celestia-core，celestia-core 是状态交互、共识和块生产层。 celestia-core 基于 Tendermint Core 构建。
- https://github.com/celestiaorg/celestia-node
  celestia-node 通过单独的 libp2p 网络增强了上述功能，该网络提供数据可用性采样请求。

Celestia-app 仓库 属于 Consensus Network，是运行 Validator Node 和 Consensus Full Node 必须跑的程序，celestia-node 仓库是 Bridge/Light/Full-Storage 节点跑的程序。

## Cosmos SDK 与 Rollkit 差异

### Cosmos SDK

- Cosmos SDK 应用链架构

![archi](./pic/cosmosnode.jpg)

- 共识
  采用 Tendermint 共识，每个节点对等权利，BFT 类算法，能抵抗 1/3 恶意节点
- 应用链基于 Cosmos SDK 搭建，模块化组件，方便用户搭建 application-specific blockchain
- Tendermint Core 通过 ABCI 接口协议于应用链通信

Tx 流程是：

1. Tendermint Core 收到 Tx，通过 ABCI 发送给 App-Specific
2. App-Specific 通过 CheckTx 校验合法性
3. 通过后将交易加入 Mempool，同时广播给其他节点
4. 出块时间间隔达到后，由出块节点从 mempool 选取交易打包成 Block，发起 proposal
5. 各节点收到 block proposal 通过 ABCI 调用 App-Specific BeginBlock 函数
6. 通过 ABCI 调用 App-Specific DeleverTx 函数，块中每笔交易都执行，修改 deliverState 状态
7. 通过 ABCI 调用 App-Specific EndBlock 代表区块结束
8. 各节点间进行 BFT 共识，共识达成后，通过 ABCI 调用 App-Specific Commit 函数持久化状态修改

---

### Rollkit

- rollkit Tx Flow
  ![archi](./pic/rollupStr.png)
- rollkit node
  ![archi](./pic/rollkitnode.png)

---

Rollkit Tx 流程：

1. 用户将交易发送给 Rollkit node, FullNode 收到交易，像 Cosmos SDK 一样通过 ABCI 调用 CheckTx 校验合法性
2. 通过校验后，将交易放入 mempool 中
3. 由 Sequencer Node 负责对交易打包区块，从 Mempool 中收集交易
4. 通过 ABCI 调用 App-Rollup BeginBlock 函数
5. 通过 ABCI 调用 App-Rollup DeleverTx 函数，块中每笔交易都执行，修改 deliverState 状态
6. 通过 ABCI 调用 App-Rollup EndBlock 代表区块结束
7. 其他 Node 同步区块
8. Sequencer Node epoch 定时批量将 Blocks push 到 DA Layer 中
9. FullNode 可以从 DA Layer 拉取数据校验正确性，发现作恶可以发起挑战

目前 rollkit 还不支持 fraud-proofs，使用的是悲观模式，并且轻节点也没实现，DA 层现在只支持 Celestia

## MEP 模块化 Rollup

- MEP SDK Componentant
  ![mepArchi](./pic/mepsdk.jpeg)

### Modular Rollup Flow

- Choose VM from VM-Adaptor
  MEP Modular Rollup SDK support Move VM/Evm/Wasm.
- Choose DA Layer from DA-Layer Adaptor
  MEP Modular Rollup SDK support Eth/Celestia/Polygon/Bitcoin
- Build the config.(Include genesis/Sequencer ...)
  config the genesis/sequencer/Da/... infomation
- Luanch Rollup Node

### MEP SDK（like celestia's rollkit）

MEP Modular SDK 可以帮助项目轻松搭建 rollup，项目方只需专注于业务逻辑的编写
SDK 为项目方提供多种选择，VM 支持 Evm/Wasm/Move VM，支持了更加安全以及高性能的 Move VM，并且可以讲 DA 层托管给其他方，例如：Celestia/Ethereum/BTC/......，前期采用 OP rollup 的解决方案
使用 MEP SDK 可以让项目轻松搭建 Rollup，实现 Rollup 的冷启动，不用自己组织 Validator，不要自己设计共识，治理等组件

为了实现 Rollup 直接的通信，MEP 采用基于 Cosmos IBC 协议，并且内置在 MEP SDK 中，默认所有新建的 rollup 都与 MEP Roolup 相连，实现以 MEP Roolup 为中心，其他所有 Roolup 通过 MEP Roolup 进行消息传递

### MEP Architechture Diagram

- MEP Rollup Architecture
  ![mepArchi](./pic/MEP-Rollup.png)
