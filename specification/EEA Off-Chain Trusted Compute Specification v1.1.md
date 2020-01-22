> 以太坊企业联盟链下可信计算规范v1.1 

> 原文地址 https://entethalliance.org/wp-content/uploads/2019/11/EEA_Off-Chain_Trusted_Compute_Specification_v1.1.pdf

> 翻译 github.com/WangYangA9 (Tusdao)

> 如有问题，请联系Email: a931040@gmail.com

[TOC]
# 1 简介
## 1.1背景
* 早期区块链通过大规模冗余计算实现了可信计算，但这种方法吞吐量有限并且隐私和安全性不完善。将可信的链下执行添加到区块链中可以有效提高区块链的性能。
* 在此规范中，主链维护对象的单个权威实例，强制执行策略并确保可以审计交易和结果，在链下可信计算提高吞吐量的同时，提高Work Order的完整性并保护数据的机密性。
* 图1描述了一个多个企业成员的企业以太坊区块链。每个成员有若干个Requester，一个以太坊区块链客户端和一个或多个Worker（被Worker Service支持）。Requester发布Worker Order，Worker Order在Worker上执行。Worker Order的存证能使用以太坊客户端在智能合约上记录。尽管在图1中每个企业组织有三个主要部件，这并不是必须的。比如企业1的Requester可以向企业2的Worker上发送Work Order，并且结果可能通过企业1的以太坊客户端记录。跨多个企业访问资源提升了网络弹性，能更有效的利用资源，并提供了一个比大多数单个企业所能提供的更大的总资源量。

### 1.1.1 调用模式
#### 1.1.1.1 直接模式
* 在这种调用中，Worker被JSON RPC请求调用。一个企业组织注册它的Worker到链上只能合约内，这样Dapp就能发现它们。DApp和Worker之间的后续交互是在链下完成的。可选的，交易存证可以被存储在链上。
#### 1.1.1.2 代理模式
* 在这种调用中，Worker被一个代理智能合约调用。代理调用一般是用来支持一个应用合约或Dapp无法直接调用Worker的情况。一个企业组织注册它的Worker到一个链上合约，这样Dapp或其他的只能合约就能发现Worker。可选的，交易存证能被存储在链上。
* 当前版本的规范仅支持无状态执行。Worker Order的执行状态由调用者自己管理。在后续的规范中，链下逻辑既充当Requester又充当Worker，并且是智能合约的创建者和控制者。智能合约由其创建者烙印并维护合约的状态。智能合约中的逻辑最少用于验证状态更改和本地事务的安全策略。这个版本将依赖于智能合约参与者共享的外部注册表。

# 5 Worker APIs
* 本章定义了Workers的智能合约API接口，
    * 智能合约API:Worker Registry List
    * 智能合约API:Worker Registry——用来worker链上注册
    * JSON RPC API:Off-chain Worker Registry Json RPC接口——用来worker链下注册

## 5.1 Worker Registry List(注册表)智能合约API
* 本节内的API接口是以太坊智能合约Worker Registry List的实现
* 一个明确的实现模型会被用来加强对这些API接口的权限验证
* 和在【注册一个新的worker】章节中提到的一样，现在可信计算的**type**属性对应了三种worker类型。在将来会定义更多的属性比如地理位置，对一些特殊硬件的支持或者软件特征等。Worker Registy智能合约支持Woker基于属性分组或基于组织分组。Worker Registry List智能合约的API接口允许允许多种worker注册
### 5.1.1 增加一个新的Registry
* 这个方法传递了一个Registry的状态
* 入参
    * **orgID**表明管理Registry的组织，在registryAdd方法中代表同样的意思
    * **status**定义了传递的Redistry状态，目前的定义如下：
        * 1.表明registry是可用的
        * 2.表明registry临时离线了
        * 3.表明registry被终止了
```
function registrySetStatus(bytes32 orgID, uint256 status) public
```
### 5.1.4 发起Registry查询
* 本方法检查了一个符合传入参数的Registry的id列表
* Worker必须匹配所有的传入参数才会被包含在这个列表中
* 如果这个列表太大，为了能够放入一个简单的response中(结果大小极限和实现相关)，智能合约会返回结果的第一块，并且提供一个查询tag，可以用来通过调用regisryLookUpNext接口来获取下一块数据
* 所有输入参数是可选的，并且能够以任何组合的方式提供来进行查询
* 入参
    * **appTypeId**是一个必须被workers支持的应用类型  
* 出参
    * **totalCount**是符合条件查询结果的总数，如果这个结果大于当前返回id数组的大小，调用者应该使用**lookupTag**来调用workerLookUpNext函数来获取剩余的id
    * **lookupTag**是一个非必须参数。如果返回的值不是0，意味着有更多的结果能通过registryLookUpNext方法获取
    * **ids**是一个符合入参条件的registry组织的数组
```
function registryLookUp(bytes32 appTypeId) public view returns 
    (uint256 totalCount,
    uint256 lookupTag, 
    bytes32[] ids)
```
### 5.1.5 获取附加的Registry查询结果
* 这个方法用来再次获取Registry查询的后续结果
* 入参
    * **appTypeId**是一个workers必须支持的应用类型
    * **lookupTag**是前置查询registryLoopup的返回值
* 出参
    * **totalCount**是符合条件查询结果的总数，如果这个结果大于当前返回id数组的大小，调用者应该使用**lookupTag**来调用workerLookUpNext函数来获取剩余的id
    * **newLookupTag**是一个非必须的参数，如果返回值不是0，意味着有更多符合条件的结果能通过再次调用此函数的方式获取
    * **ids**是一个符合入参条件的registry组织的数组
```
function registryLookUpNext(bytes32 applTypeId, uint256 lookUpTag) public returns(
uint256 totalCount, uint256 newLookupTag, bytes32[] ids)
```
### 5.1.6 检索注册信息
* 这个方法检索registry详细信息
* 入参
    * **id**是需要获取详情的registry的id
* 出参
    * 和对应的regisgtryAdd()函数的入参相同，额外加上registrySetSatus内定义的status
```

function registryRetrieve(bytes32 workerId) public view returns 
    (string uri,
    bytes32 scAddr, 
    bytes32[] appTypeIds, 
    uint256 status)
```
## 5.2 Worker Registry 智能合约API
### 5.2.1 注册一个新worker
* 这个方法注册一个Worker，并且用于和发送以太坊地址相关联的一个Trusted Resource Service使用JSON-RPC通过二进制签名的以太坊地址调用
* 一个实现规范模型加强了这个请求的权限验证
* 入参
    * **wokerID**是一个woker id，例如一个以太坊地址或一个worker的DID的派生值
    * **wokerType**定义了woker的类型，现在有如下定义
        * 0.保留字(在查询中表示任意值)
        * 1.表示"TEE-SGC":一个Intel SGX可信执行环境
        * 2.表示"MPC":多方计算
        * 3.表示"ZK":零知识证明
    > 每种Worker类型的API接口的标准在附录A中给出
    * **organizationID**是一个非必须参数，代表管理worker的组织(比如一个财团或者匿名实体的银行)
    * **applicationTypeId**是一个非必须参数，定义了Worker支持的应用类型
    * **details**是一个worker的详细信息(JSON格式定义在附录A)。这个变量也包含了worker数据或者能通过JSON RPC API(被定义在链下Worker Registry JSON RPC API中)获取数据的registry URI
```
function workerRegister(bytes32 workerID, uint256 workerType,
    bytes32 organizationID,
    bytes32[] applicationTypeId,
    string details) public
```
### 5.2.2 更新一个Worker
* 这个方法设置了一个worker的状态，一个实现标准模型加强了这个请求的权限管理
* 入参
    * **wokerID**是一个worker的id
    * **status**:定义了worker状态，当前定义如下：
        * 1.表明worker是可用的
        * 2.表示worker暂时离线
        * 3.表示worker彻底离开了
        * 4.表明worker compromised（被破解了？？？？翻译不确定）
```
function workerSetStatus(bytes32 workerID, uint256 status) public
```
### 5.2.3 修改worker状态
* 这个方法修改worker的抓饭太，一个实现模型加强了这个方法的权限验证
* 入参
    * **wokerID**woker的id
    * **status**定义了worker的状态，目前有如下定义值：
        * 1.表明worker是活动的
        * 2.表明worker暂时离线
        * 3.表示worker彻底退出了
        * 4.表明worker compromised
```
function workerSetStatus(bytes32 workerID, uint256 status) public
```
### 5.2.4 发起worker查询
* 这个方法获取一个符合入参的worker id列表
* 结果列表的worker必须匹配所有入参条件和模式
* 如果这个列表太大(最大的内容行数与实现标准有关)，智能合约会返回结果的第一块，并提供**lookupTag**参数，用来通过**workerLoopUpNext**查询下一块数据的
* 特别的，每一个入参都能传0，代表不限制该参数
* 入参
    * **wokerType**worker的类型
    * **organizationId**是一个组织id，能用来搜索属于某个组织的worker
    * **applicationTypeId**是指worker支持的应用类型
* 出参
    * **totalCount** 是一个结果集的总数。如果这个值比ids数组的长度要打，调用者应该使用lookupTag参数取调用workerLookUpNext来获取剩余的结果
    * **lookupTag**是一个非必填参数，如果返回值不是0，代表有更多的worker的id能通过workerLoopUpNext来获取
    * **ids**是一个符合入参的worker的id数组
```
function workerLookUp(
    uint256 workerType,
    bytes32 organizationId,
    bytes32 applicationTypeId) public view
returns(
    uint256 totalCount, 
    uint256 lookupTag, 
    bytes32[] ids)
```
### 5.2.5 获取worker查询结剩余果
* 这个方法用来获取调用workerLookUp方法剩余的Worker的详情
* 入参
    * **wokerType**worker的类型
    * **organizationId**是一个组织id，能用来搜索属于某个组织的worker
    * **applicationTypeId**是指worker支持的应用类型
    * **lookupTag**是上一次该请求或workerLoopUp请求的返回值
* 出参
    * **totalCount** 是一个结果集的总数。如果这个值比ids数组的长度要打，调用者应该使用lookupTag参数取调用workerLookUpNext来获取剩余的结果
    * **newLookupTag**是一个非必填参数，如果返回值不是0，代表有更多的worker的id能通过workerLoopUpNext来获取
    * **ids**是一个符合入参的worker的id数组
```
function workerLookUpNext( uint256 workerType,
bytes32 organizationId, bytes32 applicationTypeId, uint256 lookUpTag) public view returns(
uint256 totalCount, uint256 newLookupTag, bytes32[] ids)
```
### 5.2.6 获取worker详情
* 这个方法获取worker的详细信息。它内被任何授权的以太坊地址调用。
* 入参
    * **workerId**是请求的worker的id
* 出参
    * 和worker注册的入参对比，增加了一个status
```
function workerRetrieve(bytes32 workerId) public view returns (
    uint256 status,
    uint256 workerType,
    bytes32 organizationId, 
    bytes32[] applicationTypeId, 
    string details)
```
## 5.3 JSON RPC API:链下Worker注册
* 本章是worker registry智能合约api的JSON-RPC版本。所有的消息遵循一种请求-返回格式并且在同一session中同步完成
* Errors和状态使用的标准JSON RPC error
### 5.3.1 worker register JSON请求
* 这个消息注册一个worker。它并没有一个具体的返回实例，替代的是一个通用的错误返回
```
{
  "jsonrpc": "2.0",
  "method": "WorkerRegister",
  "id": <integer>,
  "params": {
    "workerId": <hexstringorDID>,
    "workerType": <uint>,
    "organizationId": <hexstring>,
    "applicationTypeId": [
      <oneormorehexstrings>
    ],
    "details": {
      <workertypespecificdata>
    }
  }
```
* **method**必须是WorkerRegister
* **params**是一组请求参数，和5.2.1相同
### 5.3.2 worker更新 JSON请求
* 这条消息更新一个worker。它并没有一个具体的返回实例，替代的是一个通用的错误返回
```
{
  "jsonrpc": "2.0",
  "method": "WorkerUpdate",
  "id": <integer>,
  "params": {
    "workerId": <hexstringorDID>,
    "details": {
      <workertypespecificdata>
    }
  }
}
```
* **method**必须是WorkerUpdate
* **params**是一组请求参数，和5.2.2相同
### 5.3.3 worker修改状态 JSON请求
* 这条消息修改worker状态。它并没有一个具体的返回实例，替代的是一个通用的错误返回
```
{
  "jsonrpc": "2.0",
  "method": "WorkerSetStatus",
  "id": <integer>,
  "params": {
    "workerId": <hexstringorDID>,
    "status": <number>
  }
}
```
* **params**是一组请求参数，和5.2.3相同
### 5.3.4 worker查询 JSON请求
* 这条消息查询注册的worker
```
{
  "jsonrpc": "2.0",
  "method": "WorkerLookUp",
  "id": <integer>,
  "params": {
    "workerType": <uint>,
    "organizationId": <hexstring>,
    "applicationTypeId": [
      <oneormorehexstrings>
    ]
  }
}
```
* **method**必须是WorkerLoopUp
* **params**是一组请求参数，和5.2.4相同
### 5.3.5 worker查询下一页 JSON请求
* 这条消息继续查询在worker查询中未全部返回的worker
```
{
  "jsonrpc": "2.0",
  "method": "WorkerLookUpNext",
  "id": <integer>,
  "params": {
    "workerType": <uint>,
    "organizationId": <hexstring>,
    "applicationTypeId": [
      <oneormorehexstrings>
    ]"lookUpTag": <string>
  }
}
```
* **method**必须是WorkerLoopUpNext
* **params**是一组请求参数，和5.2.5相同
### 5.3.6 worker查询 JSON返回
* 5.3.4和5.3.4 json请求的返回值
```
{
  "jsonrpc": "2.0",
  "id": <integer>,
  "result": {
    "totalCount": <integer,
    "lookupTag": <string,
    ids: [
      <oneormorehexstrings>
    ]
  }
}
```
* result返回值见5.2.4出参
### 5.3.7 worker详情查询 JSON请求
* 这条消息通过workerid查询它的详情
```
{
  "jsonrpc": "2.0",
  "method": "WorkerRetrieve""id": <integer>,
  "params": {
    "workerId": <hexstringorDID>
  }
}
```
* **method**必须是WorkerRetrieve
* **params**是一组请求参数，和5.2.6相同
### 5.3.8 worker详情查询 JSON返回
* 5.3.7请求对应的返回
```
{
  "jsonrpc": "2.0",
  "id": <integer>,
  "result": {
    "workerType": <uint>,
    "organizationId": <hexstring>,
    "applicationTypeId": [
      <oneormorehexstrings>
    ],
    "details": {
      <workertypespecificdata>
    },
    "status": <number>
  }
}
```
* **method**必须是WorkerRetrieve
* **result**见5.2.6相同
# 6 工作指令
* 直接模式：JSON-RPC工作指令通过网络调用
* 代理模式：工作指令使用以太坊（代理）智能合约调用
## 6.1 直接模式调用
* 这个章节定义了一种在链下通过网络JSON RPC API执行工作指令的机制
* API能在下面几种模式下使用：
    * 同步请求模式。工作指令的请求和完成在同一个HTTP会话中。同步模式适用于不需要太长执行时间的工作指令
    * 结果拉取模式。在此模式下，DAPP在提交工作请求后断开连接，然后定期的拉取工作指令结果
    * 异步模式。在此模式下，DAPP在工作指令请求中提供一个用于接收结果的URI。DAPP在发送工作指令后断开连接。在工作指令结束时，Worker提交结果到提供的URI
    * 通知模式。在此模式下，DAPP提供一个URI来接收工作时令完成的通知。DAPP在发送指令后断开连接。当指令完成后，Worker给URI发送一个事件。在接收到事件后，客户端自己去获取结果。
### 6.1.1 工作指令请求
* 首先，请求者通通过下面定义的JSON-RPC基础格式发送工作指令请求
```
{
  "jsonrpc": "2.0",
  "method": "WorkOrderSubmit",//必须填写
  "id": <integer>,
  "params": {
    "responseTimeoutMSecs": <integer>,
    "payloadFormat": <string>"resultUri": <string>,
    "notifyUri": <string>,
    "workOrderId": <hexstring>,
    "workerId": <hexstringorDID>,
    "workloadId": <hexstring>,
    "requesterId": <hexstring>,
    "workerEncryptionKey": <hexstring>,
    "dataEncryptionAlgorithm": <string>,
    "encryptedSessionKey": <hexstring>,
    "sessionKeyIv": <hexstring>,
    "requesterNonce": <hexstring>,
    "encryptedRequestHash": <hexstring>,
    "requesterSignature": <BASE64string>,
    "inData": [
      <object>
    ],
    "outData": [
      <object>
    ]
  }
}
```
* **responseTimeoutMsecs**是一个毫秒单位的调用者等待超时时间。设置超时时间为0意味着工作指令通过异步模式(如果有resultUri)，通知模式(如果有notifyUri)，拉取模式（resultUri和notifyUri都不存在）提交。在例子中，TCS应该调度这个请求执行并且马上返回一个error response，错误码为**scheduled**。如果超时时间不等于0，代表工作指令是同步模式。TCS会在给参与者返回结果之前等待工作指令的完成。如果请求无法在分配的时间内完成，工作指令会被取消并且一个相应的错误会被返回给参与者
* **payloadFormat**定义了在工作指令请求和相应的返回中如何格式化签名和数据项。它的值在附录A中给出了定义
* **resultUri**是一个非必须参数。如果它被给出了，WorkerService会提交工作指令到这个URI。具体见6.1.5
* **notifyUri**是一个非必须参数。如果它被给出了，WorkerService会发送一个工作指令完成事件到URI。见6.1.6
* **workOrderId**是一个通过调用者指定给工作指令的id，可以只用工作指令收据API注册
* **workerId**是一个处理工作请求的Worker id。(以太坊地址或他的DID)
* **workloadId**是能够被worker执行的workload的id。如果worker只有一个workload时，这个参数是非必须的
* **requesterId**是请求者的以太坊地址或DID
* **workerEncryptionKey**是一个非必须参数，包含了worker用于此项工作指令的加密密钥。如果一个Worker经常更新它的加密密钥并且允许允许偶尔利用多个密钥叠加时会非常的有用。我们再次假定worker注册期间递交的“详细信息”包含一个或多个与该worker关联的公钥
* **dataEncryptionAlgorithm**是一个非必须参数，定义了一个在工作指令中加密数据的算法。默认值是Worker（由workerId定义）相应参数中的第一个值。见8.1
* **encryptedSessionKey**是一个由参与者提交工作指令生成的一次性的密钥。它使用worker的公钥加密后发送。
* **sessionKeyIv**，如果被数据加密算法需要的话，这是一个初始化向量。默认是全0
* **requesterNonce**是一个参与者生成的随机的字符串。它被用来计算工作指令请求的哈希
* **encryptedRequestHash**是一个由encryptedSessionKey提供的key加密后的工作指令请求的哈希
* **requesterSignature**是一个非必须参数。见6.1.8
* **inData**包含指定数据的JWT或一个或者多个工作请求输入。比如状态、包含输入变量的消息。见6.1.7
* **outData**包含工作指令执行结果如何返回的信息
* 在一个工作指令请求被收到后，WorkerService能通过下面三种方式之一响应：
    * 完成工作指令并返回结果
    * 如果工作指令被拒绝或者执行失败返回一个错误
    * 调度一个工作指令延迟执行返回一个响应的状态
### 6.1.2 工作指令结果
* 如果一个提交的工作指令被完成，它的结果会以如下格式返回
```
{
  "jsonrpc": "2.0",
  "id": <integer>,
  "result": {
    "workOrderId": <hexstring>,
    "workloadId": <hexstring>,
    "workerId": <hexstring>,
    "requesterId": <hexstring>,
    "workerNonce": <string>,
    "workerSignature": <BASE64string>,
    "outData": [
      <object>
    ]
  }
}
```
* **result**是每个JSON-RPC规范的一组参数。这些参数的定义如下
    * **workOrderId**适合请求中相对应的工作指令id
    * **workloadId**是非必须的。如果存在，它一定会和请求中对应的值一致
    * **workerId**是非必须的。如果存在，它一定会和请求中对应的值一致
    * **requesterId**是非必须的。如果存在，它一定会和请求中对应的值一致
    * **workNonce**是一个由worker生成的随机字符串。它被用来计算本次工作指令返回的哈希
    * **dorkerSignature**是工作指令response的签名。见6.1.8
    * **outData**包含工作指令执行结果。见6.1.7
### 6.1.3 工作指令状态
* 如果请求失败、被拒绝、被调度为延迟执行或它的执行需要很长的时间，工作指令Error response会以如下格式返回，具体见4.1
```
{
  "jsonrpc": "2.0",
  "id": <integer>,
  "error": {
    "code": "integer",
    "message": <string>,
    "data": {
      "workOrderId": <hexstring>
    }
  }
}
```
* **code**是一个用来表示一种错误或者工作指令状态的整数。支持的值有：
    * -32768--32000的值被预留为JSON-RPC的标准
    * 5表示工作指令是待定状态——被调度了或者还没有开始
    * 6表示工作指令正在处理——已经开始执行，但是没有完成
    * 7-999被预留了
    * 其他所有的值可以被worker service(又名具体实现)使用
* **data**是error包含的额外的细节
* **workOrderId**是16进制表示的工作指令id
### 6.1.4工作指令拉取
* 如果一个请求者在它的工作指令状态时"被调度"或"执行中"需要一个相应说明，他需要稍后向Worker Service拉取结果。请求者有两种拉取选项：
    *  定时的想worker servce拉取结果，直到返回已完成或者一个error
    *  等待工作指令收据完成时间，然后检索最终结果。见7
* 在任一种情况下，工作指令拉取请求必须符合下列各式
```
{
  "jsonrpc": "2.0",
  "method": "WorkOrderGetResult",
  "id": <integer>,
  "params": {
    "workOrderId": <hexstring>
  }
}
```
* **method**必须是WorkOrderGetResult
* **params**是每个JSON-RPC规范的一组参数。这些参数的定义如下
    * **workOrderId**是和WorkOrderSubmit请求中对应的工作指令id
### 6.1.5 工作指令异步结果
* 如果客户端在工作指令请求中提供了notifyUri，Worker会在工作指令完成后给请求者发送一个时间，告知工作指令的执行最终状态是成功或失败
* 这个事件格式如下
```
{
  "jsonrpc": "2.0",
  "id": <integer>,
  "result": {
    "workOrderId": <hexstring>
  }
}
```
* **result**是JSON-RPC参数的集合，如下所示
    * **workOrderId**是在工作指令请求中发送的工作指令id
* 在收到这个事件后，客户端会拉取雑 6.1.4中定义的工作指令结果
### 6.1.7 工作指令数据格式
* 这个章节定义了工作指令请求和返回中**inData**和**outData**数据项的格式。这个格式依赖于工作数据请求中的**payloadFormat**的值
* **inData**和**outData**以下列JSON格式对象的数组形式发送
```
{
  "index": <number>,
  "dataHash": <hexstring>,
  "data": <BASE64string>,
  "encryptedDataEncryptionKey": <hexstring>,
  "iv": <hexstring>
}
```
* 如果**payloadFormat**被设置为自定义的值，它就是一个应用特有的格式
* 下面是关于上的数JSON数据体的描述
    * **index**是决定用于哈希生成的数据项顺序的索引。它还能被worker用来定义不同的输入和输出
    * **dataHash**是一个非必须的数据哈希值。它仅仅适用于请求中的**indata**和返回中的**outData**
    * **data**包含JSON文档中的内联数据或对该数据的引用(比如URI)。由Worker来决定如何解释数据内容。该参数适用于：
        * 请求中的**inData**
        * 请求中的**outData**（如果有这个字段的话）
        * 返回中的**outData**
* **encryptedDataEncryptionKey**定义了**data**是否被加密并且key用来做什么。他仅仅作为下列选项被包含在工作指令请求中
    * 如果这个值没有被设置成"null"或者""，请求中的**data**是使用**encryptedSessionKey**加密的
    * 如果这个值被设置为"-"，数据项并不会被加密，用明文的方式发送
    * 除此之外，数据项通过一个拥有该数据项的第三方的使用一次性的密钥加密发送(可能不是数据指令的发送者)。**encryptedDataEncryptionKey**包含双重加密的密钥
        * 第一，它被worker的公钥加密(由拥有数据的第三方加密，因此请求者无法看到数据)
        * 然后，第一步的加密结果由请求者使用**encryptedSessionKey**加密，来加强工作指令的完整性
    * **iv**如果加密算法需要的话，iv是一个初始化向量。默认值是0。如果同样的密钥被用了加密工作指令请求中超过一个数据项或哈希值，对于 每个加密操作来说iv必须是一个唯一的随机数字。它只在工作指令请求中存在
### 6.1.8 工作指令签名
* 工作中指令请求和返回使用工作指令请求中**payloadFormat**的值来签名
    * JSON-RPC的签名机制见
    * JSON-RPC-JWT签名机制被定义在
    * 自定义的值签名机制是应用特有的，并且它不被定义在此标准中
* 说明一下，这里有两种机制来确保工作指令请求的完整性
    * **encryptedRequestHash**包含使用**encryptedSessionKey**加密的工作指令请求的哈希值。即使对于一个匿名的请求者来说，这个机制确保了也能请求的完整性。这种情况下，这个哈希值是Worker负责验证的
    * **requesterSignature**是一个可选的签名，它使用相同的计算得出的哈希值，并使用请求着的私钥对其进行签名。相应公钥的分发超出了本规范的范围。这个签名由service或worker验证都可以。这个签名的使用是特定于应用程序的，并且它对于请求完整性验证是非必须的。它能用于请求者的权限验证和工作指令内部或外部的权限验证
* 一个工作指令请求的签名一直由执行该请求的Worker生成
#### 6.1.8.1 request哈希的计算和加密
* 这个小节定义了工作指令请求的哈希计算和加密的步骤。所有数据使用BASE64或16进制格式传输的数据需要被解码，以进行哈希计算
* 哈希算法被定义在工作指令的参数**hashingAlgorith**中，具体见8.1
* 首先，请求者计算本次工作指令请求哈希：
    * 生成一个随机数字并且存储它的哈希到**requesdterNonce**中
    * 将下列字段连在一起，计算哈希值：**requesterNonce**, **workOrderId**, **workerId**, **workloadId**, and **requesterId**.
    * 将**inData**数组中的的每一个对象内的字段**dataHash, data, encryptedDataEncryptionKey, iv**连在一起计算哈希。数组项根据**index**字段排序。如果数据是加密的，则对加密数据计算哈希值。如果**encryptedDataEncryptionKey**包含一个加密密钥，则对加密数据进行哈希计算。
    * 将**outData**数组内的每个对象内的字段**dataHash, data, encryptedDataEncryptionKey, iv**连在一起计算哈希值。数组项根据**index**字段排序。如果**encryptedDataEncryptionKey**包含一个加密密钥，则对加密数据进行哈希计算。
    * 将上面所有计算的所有哈希按照生成顺序组合到一个数组中。计算该数组的另一个哈希。
* 然后请求者必须加密计算的哈希值，并将值放到共走请求中：
    * 通过**encryptedSessionKey**的key加密最后的组合哈希
    * 将加密哈希转换为十六进制的值
    * 把它放入工作指令请求的**encryptedRequestHash**字段
* 额外可选的，计算的哈希值可以使用6.1.8.3和6.1.8.4两个小节提到的格式之一进行签名。
#### 6.1.8.2 response哈希计算
* 这个小结定义了工作指令响应的哈希计算。所有数据使用BASE64或16进制格式传输的数据需要被解码，以进行哈希计算
* 哈希算法被定义在工作指令的参数**hashingAlgorith**中，具体见8.1
* Worker执行以下步骤：
    * 生成一个随机数字并且存储它的哈希到**requesdterNonce**中
    * 将下列字段** workerNonce, workOrderId, workerId, workloadId, and requesterId**连在一起并计算哈希值
    * 对于**outData**数组中每个数据项，将每个数据项内的**dataHash and data**连在一起计算哈希。数组内的数据项使用**index**字段排序。如果**data**是被加密的，根据加密后的数据计算哈希值
    * 组合上面所有计算的哈希值，按照生成的放入到数组中，计算一个新的哈希值
#### 6.1.8.3 给JSON-RPC格式签名
* 这个小节定义了JSON-RPC格式的工作指令签名机制。。所有数据使用BASE64或16进制格式传输的数据需要被解码，以进行哈希计算
* 哈希算法在Worker的**hashingAlgorithm**变量中被定义。见8.1
* 对于一个工作指令请求，签名是非必须的。如果提供了签名，它是使用以下步骤生成的：
    * 签名算法被定义在了Worker的**signingAlgorithm**参数中，见8.1
    * 工作指令相应的被定义在6.1.8.2的哈希是使用Worker的私钥签名的
    * 签名被格式化为BASE64字符串
    * 返回的字符串被放在了工作指令响应中的**workerSignature**字段
#### 6.1.8.4 给JSON-RPC-JWT格式签名
* 本章节定义了JSON-RPC—JWT格式工作指令签名机制
* 工作指令请求**requesterSignature**和响应**workerSignatures**是JWT格式字符串
* **JWT**的header第一要包含哈希和签名算法，分别放在**workerSignatures**和**signingAlgorithm**变量中。具体见8.1
* JWT payload的格式如下
```
{
    "hash": <HEX string>
}
```
* 工作指令请求**hash**参数包含了一个6.1.8.1定义在中的值
* 工作指令响应**hash**参数包含了一个6.1.8.2定义在中的值
### 6.1.9 获取加密密钥请求
* 本节定义了一个requester获取Worker密钥的JSON-RPC请求。如果Worker除此之外还支持请求者特定的加密密钥则使用它而不是使用附录A：Worker特有的详细数据中定义的加密密钥
* 如果这个请求失败了，会返回一个工作指令状态Payload。下面是**code**的定义
    * 1-通用的错误
    * 2-不支持的操作
    * 3-非法的参数
    * 4-不允许访问
    * 5-没有准备好，稍后再试。如果请求者首次提出密钥请求，或者请求者检索密钥的速度比Worker生成的快，则这是一个可能发生的可恢复的错误，请求者应稍后重试。
* 如果请求成功了，响应被定义在6.1.10
* payload格式如下：
```
{
	"jsonrpc": "2.0",
	"method": "EncryptionKeyGet",
	"id": < integer > ,
	"request": {
		"workerId": < hex string > ,
		"lastUsedKeyNonce": < hex string > ,
		"tag": < hex string > ,
		"requesterId": < hex string > ,
		"signatureNonce": < hex string > ,
		"signature": < BASE64 string >
	}
}
```
* **method**必须是EncryptionKeyGet
* **workerId**是被请求加密密钥的worker id
* **lastUsedKeyNonce**是一个和上一次检索的密钥相关的可选的随机数。如果提供，收到的密钥应该比此密钥新。否则任何密钥都能被收到。
* **tag**是应该与返回的键，例如requester id想关联的标签。这是一个非必须参数。如果提供，**requesterId**被用作一个密钥
* **requesterId**是计划使用返回的密钥来提交一个或者多个工作请求的请求者id
* **signatureNonce**是一个非必须参数并且只能和**signature**一同存在
* **signature**是**workerId ,  lastUsedKeyNonce ,  tag , and
signatureNonce**的一个非必须的签名。Worker的哈希和签名算法被定义在**hashingAlgorithm and encryptionAlgorithm**。见8.1
### 6.1.10 获取密钥响应payload
* 本节定义了一个Worker Service返回的成功获取密钥请求的payload
```
{
	"jsonrpc": "2.0",
	"id": < integer > ,
	"request": {
		"workerId": < hex string > ,
		"encryptionKey": < hex string > ,
		"encryptionKeyNonce": < hex string > ,
		"tag": < hex string > ,
		"signature": < BASE64 string >
	}
}
```
* **workerId**是创建密钥的worker id
* **encryptionKey**是一个加密密钥
* **encryptionKeyNonce**是一个和密钥相关的随机数
* **tag**是一个和密钥相关的标签
* **signature**是Worker商城的一个签名。Worker的哈希和签名算法被定义在**hashingAlgorithm and encryptionAlgorithm**，见8.1。签名使用如下步骤计算出来：
    * 一个连接**workerId ,  encryptionKey ,encryptionKeyNonce , and  tag**计算的哈希值
    * 哈希值由与附录A中定义的的**verificationKey**对应的Worker的签名密钥
    * hash被格式化为Base64字符串
### 6.1.11 设置加密密钥请求
* 本节定义了一个由Worker或WorkerService来接收Worker的密钥的JSON-RPC请求
* 该请求的响应见6.1.3.错误值被定义在了6.1.9
* 请求格式乤
```
{
  "jsonrpc": "2.0",
  "method": "EncryptionKeySet",
  "id": <integer>,
  "request": {
    "workerId": <hexstring>,
    "encryptionKey": <hexstring>,
    "encryptionKeyNonce": <hexstring>,
    "tag": <hexstring>,
    "signatureNonce": <hexstring>,
    "signature": <BASE64string>
  }
}
```
* **method**必须是**EncryptionKeySet**
* 其他参数定义见6.1.10
# 6.2 代理模式调用
### 6.2.1 工作指令提交
* 这个方法创建了一个新的工作指令。它被一个DAPP或者企业应用智能合约从请求者的地址中调用。代理智能合约被DAPP或企业应用智能合约从请求者的以太坊地址用调用。
* 副作用是，此功能可能会为工作指令创建一个工作指令收据——见第八章
* 这个方法发送一个**workOrderSubmitted**事件
* 这个方法使用包含在时间**workOrderNew**中并由**workOrderGetRequest**返回隐式参数的**transaction sender**
* 入参：
    * **WorkOrderId**应该匹配**workOrderRequest**中响应的字段
    * **workerId**应该匹配**workOrderRequest**中响应的字段
    * **requestId**是一个requester id，它必须匹配**workOderRequest**中的字段
    * **workOrderRequest**是下面几种格式的工作指令请求数据之一：
        * JSON payload，在6.1直接模型调用中**workOrderRequest**的详细信息中所定义
        * 指向和上一条小童的JSON payload的HTTP(S) URI。通常使用URI来最小化在区块链上存储工作指令的空间
* 出参
    * **errorCode**是一个错误码，0代表成功，其他代表失败
```
function workOrderSubmit(bytes32workOrderId,
    bytes32workerId,
    bytes32requesterId,
    stringworkOrderRequest)
public returns(uint256errorCode)
```
### 6.2.2 工作指令提交事件
* 这个事件被**workerSubmit**方法发送；**wokrOrderId**是被提交的工作指令id
* 这个事件适用于应该执行工作指令的Worker
* 参数
    * **workOrderId, workerId, requesterId, workOrderRequest, and errorCode**被定义在章节 《提交一个新的工作指令》
    * **senderAddress**是一个以太坊地，从改地址进行了响应的workerOrderSubmit()调用
    * **version**是API的版本号，比如0x01020381对应版本号"1.2.3.129"
```
event workOrderSubmitted (bytes32 indexed workOrderId, 
    bytes32 indexed workerId,
    bytes32 indexed requesterId, 
    string workOrderRequest, 
    uint256 errorCode,
    address senderAddress, 
    bytes4 version)
```
### 6.2.3 完成一个工作指令
* 被Worker Service调用此方法来成功或错误的结束工作指令
* 可以UI从Worker或Worker Servive地址执行此方法
* 这个方法会发送**workOrderCompleted**事件
* 入参
    * **workerOrderId**是一个工作指令在调用**workOrderSUbmit**时提供的id
    * **workOrderResponse**是下列各式之一的工作指令响应数据
        * JSON payload，在6.1直接模型调用中**workOrderRequest**的详细信息中所定义
        * 指向和上一条小童的JSON payload的HTTP(S) URI。通常使用URI来最小化在区块链上存储工作指令的空间
* 出参
* **errorCode**是一个错误码，0代表成功，其他大表失败
```
function workOrderComplete(bytes32 workOrderId,string workOrderResponse) 
    public returns (uint256 errorCode)
```
### 6.2.4 工作指令完成事件
### 6.2.5 收取工作指令响应信息
### 6.2.6 设置加密密钥
### 6.2.7 加密密钥设置事件
### 6.2.8 获取加密密钥
### 6.2.9 开始加密密钥生成
### 6.2.10 加密密钥开始事件
# 7 工作指令Receipt
* 本章节定义了两种工作指令收据的模式
    * 由以太坊工作指令Receipt智能合约管理收据时的**智能合约模式**
    * 由链下管理Receipt并通过JSON RPC API访问的**直连模式**
## 7.1 代理模式Receipt处理
### 7.1.1 创建工作指令Receipt
### 7.1.2 更新一个工作指令Receipt
### 7.1.3 检索一个工作指令Receipt
### 7.1.4 检索一个工作指令Receipt更新
### 7.1.5 工作指令Receipt查询
### 7.1.6 工作指令Receipt查询下一页
### 7.1.7 工作和指令Reiceipt更新事件
### 7.1.8 工作指令Receipt创建事件
## 7.2 直接模式Receipt处理
* 请求者发送该请求来穿件一个新的工作指令Receipt。这个请求没有一个规范的返回Payload，因此错误码和状态码payload被作为返回值
### 7.2.1 错误码和状态码Payload结构
### 7.2.2 Receipt创建请求payload
### 7.2.3 Receipt更新请求payload
### 7.2.4 Receipt检索请求Payload
### 7.2.5 Receipt检索响应Payload
### 7.2.6 Receipt更新检索请求Payload
### 7.2.7 Receipt更新检索R响应Payload
### 7.2.8 Receipt查询请求Payload
### 7.2.9 Receipt查询响应Payload
### 7.2.10 Receipt查询下一页请求Payload
# 8 附录A
* 本附录定义了在章节5.2.1中定义的参数**detail**中包含的数据。里面包含了所有worker类型和每种worker类型数据规范的常用数据
## 8.1 所有工作类型的通用数据
```
{
  "workOrderSyncUri": <hexstring>,
  "workOrderAsyncUri": <hexstring>,
  "workOrderPullUri": <hexstring>,
  "workOrderNotifyUri": <hexstring>,
  "receiptInvocationUri": <hexstring>,
  "workOrderInvocationAddress": <hexstring>,
  "receiptInvocationAddress": <hexstring>,
  "fromAddress": <hexstring>,
  "hashingAlgorithm": <string>,
  "signingAlgorithm": <string>,
  "keyEncryptionAlgorithm": <string>,
  "dataEncryptionAlgorithm": <string>,
  "workOrderPayloadFormats": [<hexstring>],
  "workerTypeData": {...}
}
```
* **workOrderSyncUri, workOrderAsyncUri, workOrderPullUri, and workOrderNotifyUri**是用来在直接模式内提交工作指令的URI，相应的在同步，异步，拉取和通知模式下使用。多个属于同一家组织的Worker能共享同样的URI
* **receiptInvocationURI**是直接模式下的Worker用作管理工作指令Receipt处理的URI。多个属于同一家组织的Worker能共享同样的URI
* **workOrderInvocationAddress**是在代理模式用被用来提交工作指令的一个工作指令智能合约的地址。多个属于同一家组织的Worker能共享同样的URI。
* **receiptInvocationAddress**是在代理模式用被用来提交工作指令Receipt的一个工作指令Receipt智能合约的地址。多个属于同一家组织的Worker能共享同样的URI。
* **fromAddress**是一个以太坊地址，由该Worker或代表此Worker来提交交易。他可以与可信计算服务的ID相同
* **hashingAlgorithm**是非必须的参数，包含了支持的哈希算法列表，使用逗号分隔。定义的值有
    * SHA-256
    * KECCAK-256
* **signingAlgorithm**是一个非必填的字符串，包含了一个支持的前面算法，使用逗号分隔。默认值是SECP256K1。定义的值有
    * SECP256K1
    * RSA-OAEP-3072
* **keyEncryptionAlgorithm**是一个包含一个非对称加密算法的字符串，被worker用来加密对称的数据密钥，比如RSA-OAEP-3072
* **dataEncryptionAlgorithm**是一个非必须的字符串，包含了被支持的数据加密算法，使用逗号分隔。例如AES-GCM-256。一个AES GSM加密信息以标签开头，后面跟密文。
* **workOrderPayloadFormats**定义了工作指令请求和相应的格式。Worker可能支持多种格式。当前规范现在支持下面定义的格式(之后可能会增加)：
    * JSON-RPC：paylaod通过不使用JWT签名的JSON-RPC格式来提供，见第4章
    * JSON-RPC-JWT：payload通过JWT签名的JSON-RPC格式来提供。见第4章
    * 自定义的工作指令payload格式应该使用“~”开始
* **workerTypeData**包含下面定义的特定于Worker类型的详细信息
## 8.2 TEE(可信执行环境) Worker数据
* 本小节定义了“TEE-SGX”worker类型的数据。请参阅5.2.1中的参数**workType**
### 8.2.1 Intel SGX Worker类型数据
* 识别和证明的payload
```
{
    "workerTypeData": {
        "verificationKey": <hex string>,
        "extendedMeasurements": [<string>],
        "proofDataType": <hex string>,
        "proofData": {...},
        "encryptionKey": <optional hex string>,
        "encryptionKeyNonce": <optional hex string>,
        "encryptionKeySignature": <optional hex string>,
        "enclaveCertificate": <optional string>
    } 
}
```
* **workerType**被定义在小节5.1中定义
* **verificationKey**是代表ECDSA/SECP256K1公钥的16进制字符串，用于验证Enclave创建的签名。该字段必须包含在**proofData**中
* **extendedMeasurements**是实现特定于应用程序的逻辑的字符串。比如，在包含多种类型enclave(每种有一个不同的MRENVLAVE值的)enclave池中，这个参数可能包含一个逗号分隔的16进制字符串，每个对应一种MRENCLAVE。具体见8.2.2
* 除非区块链客户端包含一个预编译合约，在链上确认在Enclave的注册时通过了验证，请求者被期望在链下去验证Enclave。有关的匹配机制，请参见https://software.intel.com/en-us/sgx
* **proofDataType**是一种“TEE-”前缀的数据类型。现在仅支持Intel SGX 安全数据类型。更多的类型会在之后添加：
    * TEE-SGX-IAS表示intel认证服务器（IAS）为Worker发布了一个验证报告
* **proofData**是一个worker验证数据。它的格式定义在下面的**Proof Data Fromat**中
* **encryptionKey**是用来加密发送给Enclave的16进制REA公钥。这个字段必须按照**encryptionKeySignature**描述的方式进行签名
* **encryptionKeyNonce**是一个十六进制字符串。每次生成新的**encryptionKey**时，它要么是随机数要么是单调的计数器
* **encryptionKeySignature**是十六进制字符串，代表使用Worker的私钥签名的**encryptionKey**签名
* **enclaveCertificate**是一个代表使用Worker私钥签名的enclave证书的字符串，并且必须包含**encryptionKey and encryptionKeyNonce**。enclave证书应该符合X.509格式
* 请注意，这两个选项值守有一个是请求者验证所必须的：
    * 在**workerTypeData**中提供提供**encryptionKey and encryptionKeySignature**
    * 在**workerTypeData**中提供**enclaveCertificate**
#### Proof Data  Format
* 对于**TEE_SGX-IAS**类型，**proofData**格式如下。
```
{
    “X-IASReport-Signature”:<required, base64>,
    ”X-IASReport-Signing-Certificate”: <required, string>,
    “Advisory-URL”: <optional, string>,
    “Advisory-IDs”: <optional, string>,
    “Verification-report”:<required string>
}
```
* 参数描述具体参见[IAS API](https://api.trustedservices.intel.com/documents/sgx-attestation-api-spec.pdf)
* **Verification-report:isvEnclaveQuoteBody:REPORTBODY:REPORTDATA**包含解码的**verificationKey**和（UTF-8）extendedMeasurements值的连接后的SHA256哈希值(前32字节)。最后312字节设置为0
### 8.2.2 TEE-SGX负载均衡和Enclave池
* 本规范嘉定Envlace可以被分组，已实现更好的负载均衡和管理。在这种情况下，任何池中的Enclave都是用相同的验证和加密密钥来调用工作负载
    * 如果池中包含超过一种enclave，**extendedMeasurements**包含一个逗号分隔的十六进制字符串列表，每一个对应一个MRENCLAVE。这个列表作为host service的输入被提供给enclave
    * master Enclave生成证书和密钥
    * master Enclave计算一个SHA256哈希，然后把它放入**Verification- report:isvEnclaveQuoteBody:REPORTBODY:REPORTDATA**，见8.2.1
    * master enclave被注册到worker注册表
    * master enclave仅通过步骤1提供的列表中的enclave，才使用验证和加密密钥转发和安全地执行工作负载
* 明确的Enclave池的供应机制不在本规范的范围内。被请求者可以使用包含在**proofData:Verification-report**的MRENCLAVE值来确定用于操作Enclave pool的机制
## 8.3 MPC(多方计算)Worker数据
* 本章定义了MPCWorker类型的数据。参见**workType**参数在5.2.1
```
{
    "workerTypeData": {
        "workerType":<hex string>,     //"MPC-" prefix
        "verificationKey": <hex string>,
        "encryptionKey": <hex string>,
        "proofDataType": <hex string>,
        "proofData": <hex string>,
    }
}
```
* **workerType**定义在5.1节中
* **verifycationKey**是一个十六进制字符串，代表用于验证MPCWorker提供的的数据的公钥。该字段必须包含在**proofData**中
* **encryptionKey**是一个十六进制字符串，代表用于加密发送给MPC Workder的数据的非对称公约。该字段必须包含在**proofData**中
* **proofType**是一种在将来会被定义的“MPC-”为前缀的数据类型
* **proofData**是和**proofDataType**相对应的可信数据。参见第9章
## 8.4 ZK(零知识证明)Worker数据
* 本章定义了“ZK”Worker类型的数据。参见5.2.1中的**workType**字段
* 鉴定和证明的payload
```
{
    "workerTypeData": {
        "workerType":<hex string>,     //"ZK-" prefix
        "verificationKey": <hex string>,
        "encryptionKey": <hex string>,
        "proofDataType": <hex string>,
        "proofData": <hex string>,
    } 
}
```
* **workerType**定义在5.1节中
* **verificationKey**是一个十六进制字符串，代表用于验证Zk Worker生成的证明的公共密钥。该字段必须包含在proofData中
* **encryptionKey**是一个十六进制字符串，代表用于加密发送给ZK Worker的数据的非对称公钥
* **proofDataType**是一种“ZK-”为前缀的数据，会在未来被定义
* **proofData**是和**proofDataType**对应的可信数据。见第9章
# 9 实施建议
## 9.1 建议1：Receipts
## 9.2 建议2：Worker Service
## 9.3 建议3：Proof Data
## 9.4 建议4：安全考虑
# A 附加信息
## A.1 术语
* **requester**是一个通过DAPP或应用智能合约发出工作指令的实体
* **Worker**是一个执行工作指令的计算资源。一个Worke可能用以太坊地址或一个DID做标识
* **Trusted Compute**是一个执行工作指令的可信计算资源。它可以保护数据的机密性，执行的完整性并强制数据访问策略。所有符合规范的Worker都是可信计算。可信计算可能一各种方式实现这些保证。例如，可信计算可以基于软件的密码安全保证，服务的信誉，虚拟化或或基于硬件的可信执行环境(如Intel SGX)来建立信任
* **Trusted Execution Environment(TEE可信执行环境)是一个基于硬件技术的，仅执行经过验证的任务，产生经过证明的结果，提供对恶意主机软件的保护，并确保共享加密数据的机密性
* **Enclave**是基于硬件的TEE中的Trusted Compute实例。某些硬件继续TEE设备(包含Intel SGX)，允许Enclaves的多个实例同时执行。为了简化，在本说明书中术语TEE和Enclave是等价的
* **Worker Service**是依赖于实现的中间件实体，充当以太坊区块链和Worker之间的通信桥梁。Worker Service可能属于企业，云服务提供商或共享其计算能力的个人资源(视供应而定)
* **worker Order**工作指令是请求和提交给Worker执行的工作单位。工作指令可能包括一个或多个输入(比如消息，输入参数，状态，数据集等) 和一个或多个输出。工作指令输入和输出能作为请求或响应体的一部分发送或者链接到一个远程存储地址。工作指令输入和输出通常是被加密发送的。
* **Direct Model**直接模式是一种请求者(DAPP)直接调用JSON-RPC网络接口来执行工作指令的执行模型
* **Proxy Model**是一种企业应用智能合约通过工作指令调用代理智能合约来执行工作指令的模型
* **Attested Oracle**经过验证的预言机是使用可信计算来验证一些数据(例如，环境特征、财务水平。库存水平)的设备
# B 参考文献
## B.1 规范性参考
## B.2 信息参考