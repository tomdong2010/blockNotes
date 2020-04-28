# 以太坊交易处理流程
## 整体流程
* 发起交易：设定from，to，value，gas等参数生成交易

* 交易签名：使用账户私钥对交易进行签名

* 提交交易：把交易加入到交易池txpool中

* 广播交易：把交易信息广播给其他结点

* 处理交易：矿工取交易，EVM执行，改变状态

## 代码流程(全节点)
### 1.前端发起交易
### 2.internal/ethapi/api.go
PublicTransactionPoolAPI.SendTransaction,根据参数创建交易,签名,submitTransaction提交交易  

PublicTransactionPoolAPI.SendRawTransaction,解码已签名交易,submitTransaction提交交易

PrivateAccountAPI.SendTransaction,根据参数创建签名交易,submitTransaction提交交易

submitTransaction,调用后端发送交易方法Backend.SendTx

### 3.eth/api_backend.go
Backend.SendTx,调用交易池添加交易到本地EthAPIBackend.eth.txPool.AddLocal(signedTx)
### 4.core/tx_pool.go

tip：TxPool包含pending(当前可执行交易队列)和queue(当前不可执行交易队列)两个交易队列,类型为map[common.Address]*txList

* TxPool.AddLocal,调用TxPool.addTx将交易加入交易池(除AddLocal还有AddRemote,内部都调用TxPool.addTx,如果是local交易会写入本地磁盘日志中,在节点重启时加载恢复,在一些过滤操作中保留的优先级高)

* TxPool.addTx内部调用TxPool.add,add方法进行一系列验证后将交易加入queue,用于后续提升至pending中去执行,具体逻辑包括：
    ```
    通过交易哈希验证是未知交易,如果交易池已包含该交易,return
    
    TxPool.validateTx交易有效验证,验证不通过,return
    
    判断交易池已满时,判断如果交易手续费比当前交易池中手续费最低的交易还低,return,否则,移除一些交易以腾出空间 
    
    判断交易池pending中包含相同from和nonce的交易时,比较gasPrice大于旧交易gasPrice且大于threshold*(门槛),则替换并协程发送事件进行广播 go TxPool.txFeed.Send(NewTxsEvent{types.Transactions{tx}}),否则return
    
    不是以上两种替换交易池中已有交易的情况,TxPool.enqueueTx将交易插入queue
    
    判断如果是local交易,记录并将交易写入本地磁盘日志中
    ```
* add方法返回是否有替换交易(replace bool),如果为false,不是替换,仅仅是添加一个新的交易,执行参数为from的TxPool.promoteExecutables

* TxPool.promoteExecutables方法把queue中变得可执行的交易转移到pending中,并删除所有无效交易,具体逻辑包括：
    ```
    创建临时变量交易列表promoted
    判断参数accounts(类型[]common.Address)是否为空,如果为空则赋值为queue中所有地址
    遍历accounts,取出key为addr的list,即from为addr的交易列表,在循环中：
    丢弃nonce比当前账户nonce低的交易
    丢弃交易花费超过账户余额和gas使用超出当前交易池可用gas的交易
    根据TxPool.pendingState的nonce值取出所有可执行的交易,执行TxPool.promoteTx将交易插入pending,如果插入成功则把交易加入到promoted
    如果addr不是本地账户,如果列表大小超过交易池中每个账户最多不可执行交易数量(默认64),丢弃
    如果列表已经为空,删除queue中addr这个key
    判断如果promoted大小大于0,协程发送事件进行广播 go pool.txFeed.Send(NewTxsEvent{promoted})
    判断如果pending中交易数量超出限制(4096),做平衡操作：创建集合记录交易数量超出限制的非本地账户,丢弃pending中这些账户下的部分交易直到数量不超过限制,更新TxPool.pendingState
    判断如果queue中交易数量超出限制(1024),取出非本地账户交易,根据心跳排序,丢弃交易直到数量不超过限制
    ```

* 扩展,有TxPool.promoteExecutables还有TxPool.demoteUnexecutables,具体逻辑：
    ```
    遍历pending,得到addr下交易列表:
    丢弃nonce比当前账户nonce低的交易
    丢弃交易花费超过账户余额和gas使用超出当前交易池可用gas的交易,这些移除的交易最小的nonce为lowest,列表中nonce大于lowest的交易移除并加入临时列表invalids返回
    遍历invalids,调用TxPool.enqueueTx插入queue
    判断如果当前列表中找不到addr当前nonce对应的交易,从pending中移除列表中所有交易,调用TxPool.enqueueTx插入queue
    pending和beats的key该删除的删除
    调用时机：在reset中调用,reset方法在NewTxPool和TxPool.loop中收到ChainHeadEvent时调用,在矿工worker.resultLoop收到resultCh和BlockChain.InsertChain后会PostChainEvents
    ```

### 5.在有新的交易插入pending时,广播NewTxsEvent,SubscribeNewTxsEvent方法用于监听到NewTxsEvent
调用时机:

* ethstats/ethstats.go Start方法中开启loop协程,在该协程中订阅并处理,主要是通知交易事件,并在时间上过滤避免过于频繁的事件处理
* eth/filters/filter_system.go
eth/handler.go ProtocolManager.Start方法中订阅(该方法会在节点启动时调用),go pm.txBroadcastLoop()开启协程处理,当收到NewTxsEvent,调用ProtocolManager.BroadcastTxs方法广播交易到其他所有没有该交易的节点
* miner/worker.go newWorker方法中订阅,并开启了协程来处理,当收到NewTxsEvent
    ```
     判断如果节点不在挖矿,这里会立即调用worker.commitTransactions方法处理并更新快照,其中worker.commitTransactions方法核心逻辑是调用core.ApplyTransaction处理交易
     判断如果节点在挖矿,不处理
    ```
### 6.core/state_processor.go
* ApplyTransaction方法,将Transaction转化为Message,创建evm,调用core/state_transition.go ApplyMessage执行Message,执行成功后更新状态,创建交易回执并返回
### 7.core/state_transition.go
* ApplyMessage会创建一个StateTransition对象并调用其TransitionDb方法,该方法让虚拟机evm处理交易数据(创建合约或普通交易),并更新nonce,balance等状态

计算threshold(门槛),core/tx_list.go Add方法
old.gasprice(100+priceBump)/100=threshold阙值


