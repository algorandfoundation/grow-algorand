# 示例：授权有状态投票智能合约应用程序

## 概览
在线投票应用程序可以非常有用，但大多数都存在缺乏透明度等问题，对于使他们成为一个理想的候选区块链造成一定影响。特别是当你用它们来对客户进行随意的调查，或者在某个特定的话题上征求公众意见，这种情况尤其明显。因此，投票的利害关系更加重要，如何解决这种应对关系，这对区块链应用程序构建者来说是一个问题。公共区块链应用程序有一些基本的匿名性，与此同时，一些挑战需要解决。最突出的问题是双重投票。如何防止一个人创建多个账户并且投票不止一次？我们需要的是一个获准的投票应用程序。允许投票的应用程序应该只允许一个人投票一次。在Algorand区块链平台上，这个演练说明了一个方法。

该应用程序将涉及使用许多Algorand技术，包括无状态智能合约、原子转移、标准资产和资产交易。本文将从设计概述开始，然后详细介绍如何在Algorand上构建应用程序。

## 设计概述
为了在Algorand实现一个获得许可的投票应用程序，需要一个中央机构来为用户提供投票权。在这个例子中，这是由一个[Algorand标准资产处理](https://developer.algorand.org/docs/features/asa/)的。中央当局创建一个投票代币，然后给予已经注册的选民一个投票代币。然后选民通过投票智能合约在一个整数范围内注册，选择进入合约。然后，选民通过将两种交易分组投票。第一个是调用一个智能合约去给候选人a或候选人b投票，第二个是转移投票代币回到中央当局。投票只允许在投票范围内进行。

要在Algorand上创建这种类型的应用程序，必须支持以下四个步骤。
1. [创造资产](https://developer.algorand.org/solutions/example-permissioned-voting-stateful-smart-contract-application/#asset-creation-step-1) - 中央当局需要使用Algorand ASAs创建一个投票代币。选民需要选择进入该资产，资产ID需要存储在有状态智能合约中，以验证用户何时投票，并确认他们正在使用投票代币。中央政府需要向选民发送一个投票代币；
2. [创建投票智能合约](https://developer.algorand.org/solutions/example-permissioned-voting-stateful-smart-contract-application/#voting-smart-contract-creation-step-2) - 中央当局需要在Algorand区块链上创建投票智能合约，并通过轮数迭代进行注册和投票。创建者地址被传递给创建方法。这仅用于允许创建者删除有投票权的智能合约；
3. [登记投票](https://developer.algorand.org/solutions/example-permissioned-voting-stateful-smart-contract-application/#voters-opt-into-voting-smart-contract-step-3) - 选民需要通过选择进入合约的方式以在智能投票合约中注册。注册投票发生在合约创立过程中设定的一组回合之间；
4. [投票](https://developer.algorand.org/solutions/example-permissioned-voting-stateful-smart-contract-application/#users-vote-step-4) - 选民通过原子式地将两个交易分组并提交给区块链进行投票。第一个交易是调用智能合约，为候选人a或候选人b投票。第二个交易是从选民到中央当局的资产转移，以使用他们的投票权标记。

![image](https://github.com/GinVenXi/test/blob/master/figure%201.png)

![image](https://github.com/GinVenXi/test/blob/master/figure%202.png)

这个架构中的每个步骤将在下面的小节中解释。此解决方案仅使用goal来使应用程序调用有状态应用程序。SDKs也提供了同样的功能，可以用来代替goal。

## 创造资产-第一步
中央机构首先需要创建一个投票代币。这可以通过[SDKs](https://developer.algorand.org/docs/features/asa/#creating-an-asset)，或者goal [命令行工具](https://developer.algorand.org/docs/reference/cli/goal/asset/create/)，或使用像[algodesk.io](https://algodesk.io/)这样的资产创建IDE来实现。

![image](https://github.com/GinVenXi/test/blob/master/figure%203.png)

下面是一个使用goal创建投票代币的示例。

```
$ goal asset create --creator {ACCOUNT} --total 1000 --unitname votetkn --decimals 0 -d ~/node/data
```

在这个示例中，创建了1000个投票代币。

```
$ goal asset send -a 0 -f {VOTER_ACCOUNT} -t {VOTER_ACCOUNT}  --creator {CENTRAL_ACCOUNT} --assetid {ASSETID} -d ~/node/data
```

在创建投票代币之后，从区块链返回的 assetID 必须被传递到上面的调用中。然后，中央当局应向在中央当局注册的选民发送选票代币。中央机关是如何处理注册的，这篇文章没有解释。

```
$ goal asset send -a 1 -f {CENTRAL_ACCOUNT} -t {VOTER_ACCOUNT}  --creator {CENTRAL_ACCOUNT} --assetid {ASSETID} -d ~/node
```

投票代币的assetID应该硬编码到投票智能合约中。也可以将其作为参数传入。assetID在[步骤4](https://developer.algorand.org/solutions/example-permissioned-voting-stateful-smart-contract-application/#users-vote-step-4)中有更详细的讨论。要了解有关使用Assets的更多信息，请参阅[开发人员文档](https://developer.algorand.org/docs/features/asa/)。

## 投票智能合约创建-第二步

Goal命令行工具提供了一组用于操作应用程序且与应用程序相关的命令。goal app create命令用于创建应用程序。这是一个针对区块链的特定应用程序交易，类似于Algorand Assets的工作方式，它将返回一个应用程序ID。将几个参数传递给创建方法。这些参数主要围绕应用程序使用了多少存储空间。在有状态智能合约中，您将存储指定为全局或本地存储。全局存储表示应用程序本身可用的空间量，本地存储表示每个账户的余额记录中应用程序每个账户将使用的空间量。

![image](https://github.com/GinVenXi/test/blob/master/figure%204.png)

在投票应用程序中使用了七个全局变量(一个字节片和六个整数)和一个本地存储变量(字节)。全局字节切片用于构建创建者地址，六个全局整数表示注册和投票的整数范围。本地字节切片用于存储特定账户(即候选人a或候选人b)的投票。

```
$ goal app create --creator {CENTRAL_ACCOUNT}   --approval-prog ./p_vote.teal --global-byteslices 1 --global-ints 6 --local-byteslices 1 --local-ints 0 --app-arg "int:1" --app-arg "int:20" --app-arg "int:20" --app-arg "int:100" --clear-prog ./p_vote_opt_out.teal
```

在这个示例中，还将几个应用程序参数传递给create方法。这些是有状态智能合约应用程序参数，代表注册和投票的轮数范围。这个投票应用程序使用这些范围而不是时间戳。如果需要，可以将应用程序通过[时间戳](https://developer.algorand.org/docs/features/asc1/stateful/#global-values-in-smart-contracts)进行修改。审批和清除程序也被传递给创建方法。有关[参数传递](https://developer.algorand.org/docs/features/asc1/stateful/#passing-arguments-to-stateful-smart-contracts)和有状态智能[合约创建](https://developer.algorand.org/docs/features/asc1/stateful/#creating-the-smart-contract)的更多信息，请参见开发人员文档。

有状态智能合约的TEAL代码执行以下操作。

* 检查是否没有设置应用程序ID，表示这是一个创建调用
* 将创建者地址存储到全局状态
* 存储注册和投票轮数范围到全局状态

```
// Approval Program 审批程序
#pragma version 2

// check if the app is being created 检查是否app被创建
// if so save creator 如果是，保存创建者
int 0
txn ApplicationID
==
bz not_creation

byte "Creator"
txn Sender
app_global_put

// 4 args must be used on creation
txn NumAppArgs 创建txn NumAppArgs必须使用4个参数
int 4
==
bz failed

// set round ranges 设定轮数范围
byte "RegBegin"
txna ApplicationArgs 0
btoi
app_global_put

byte "RegEnd"
txna ApplicationArgs 1
btoi
app_global_put

byte "VoteBegin"
txna ApplicationArgs 2
btoi
app_global_put

byte "VoteEnd"
txna ApplicationArgs 3
btoi
app_global_put

int 1
return

not_creation:
```

## 选民选择签订投票智能合约-第三步
如果合约使用本地存储，用户必须选择有状态智能合约。此应用程序将选民的选择存储在本地存储中，并要求用户选择加入。

![image](https://github.com/GinVenXi/test/blob/master/figure%205.png)

使用goal或sdk可以选择加入。

```
$ goal app optin  --app-id {APPID} --from {ACCOUNT} --app-arg "str:register" -d ~/node/data
```

这个投票应用程序使用一个应用程序参数，其值为“register” ，以执行opt in操作。智能合约执行以下操作。

* 检查智能合约的第一个参数是否为“register”
* 确认该轮目前处于开始注册和结束注册轮之间
* 验证该账户是否已选择加入

```
// register 注册

txna ApplicationArgs 0
byte "register"
==
bnz register

.
.
.

register:
global Round
byte "RegBegin"
app_global_get
>=

global Round
byte "RegEnd"
app_global_get
<=
&&

int OptIn
txn OnCompletion
==
&&

bz failed
int 1
return
```

有关选择有状态智能合约的更多信息，请参见[开发人员文档](https://developer.algorand.org/docs/features/asc1/stateful/#optioning-into-the-smart-contract)。

## 用户投票-第四步
一旦注册了智能合约，用户就可以投票了。这需要在两个交易中使用原子传输。第一个应该是调用有状态智能合约投票，第二个应该是从投票者到中央机构的资产转移(投票代币)。

![image](https://github.com/GinVenXi/test/blob/master/figure%206.png)

本例中的原子传输是使用goal命令行工具完成的。

```
$ goal app call --app-id {APPID} --app-arg "str:vote" --app-arg "str:candidatea" --from {ACCOUNT}  --out=unsignedtransaction1.tx
$ goal asset send --from={ACCOUNT} --to={CENTRAL_ACCOUNT} --creator {CENTRAL_ACCOUNT} --assetid {VOTE_TOKEN_ID} --fee=1000 --amount=1 --out=unsignedtransaction2.tx

$ cat unsignedtransaction1.tx unsignedtransaction2.tx > combinedtransactions.tx
$ goal clerk group -i combinedtransactions.tx -o groupedtransactions.tx 
$ goal clerk sign -i groupedtransactions.tx -o signout.tx
$ goal clerk rawsend -f signout.tx
```

智能合约通过以下操作处理此请求。
* 验证包含字符串“vote”的第一个应用程序参数
* 验证投票调用是在投票回合的开始和结束之间
* 检查投票人是否选择了智能合约
* 检查投票者的账户，以确认他们至少有一个投票代币(硬编码，应该更改)
* 验证组中是否有两个交易
* 检查第二个交易是否为资产转移，转移的代币是投票代币
* 检查第二个交易的接收方是否是应用程序的创建者
* 检查账户是否已经投票，如果已经投票，则只返回true，不更改全局状态
* 验证用户是否投票给候选人a或b
* 从全局状态读取候选人的当前总数并增加值
* 将候选选项存储到用户的本地状态

```
// vote 投票
txna ApplicationArgs 0
byte "vote" 
==
bnz vote
.
.
.
vote:

// verify in voting rounds 在投票轮中进行验证

global Round
byte "VoteBegin"
app_global_get
>=

global Round
byte "VoteEnd"
app_global_get
<=
&&
bz failed

// Check that the account has opted in 检查账户是否选择
// account offset (0 == sender, 
// 1 == txn.accounts[0], 2 == txn.accounts[1], etc..) 检查账户余额

int 0 
txn ApplicationID
app_opted_in
bz failed

// check if they have the vote token 检查是否有投票代币
// assuming assetid 2. This should
// be changed to appropriate asset id
// sender
int 0

// hard-coded assetid 硬编码
int 2
// returns frozen asset balance 返回冻结资产余额
// pop frozen
asset_holding_get AssetBalance
pop

// does voter have at least 1 vote token 判断投票者至少有一个投票代币
int 1
>=
bz failed

// two transactions 两个交易
global GroupSize
int 2
==
bz failed

// second tx is an asset xfer 第二个交易是资产转移
gtxn 1 TypeEnum
int 4
==
bz failed

// creator receiving the vote token 创建者接收投票代币
byte "Creator"
app_global_get
gtxn 1 AssetReceiver
==
bz failed

// verify the proper token spent
gtxn 1 XferAsset 验证投票代币消费
// hard coded and should be changed
int 2
==
bz failed

// spent 1 vote token 消费一个投票代币
gtxn 1 AssetAmount
int 1
==
bz failed

//check local to see if they have voted 检查本地，确认是否进行投票
int 0 // sender
txn ApplicationID
byte "voted"
app_local_get_ex 

// if voted skip incrementing count 如果投票，跳过
bnz voted
pop

// can only vote for candidate a
// or candidate b 仅能为候选人a或b投票
txna ApplicationArgs 1
byte "candidatea" 
==
txna ApplicationArgs 1
byte "candidateb" 
==
||
bz failed

// read existing vote candidate
// in global state and increment vote 读取存在的候选人
int 0
txna ApplicationArgs 1
app_global_get_ex
bnz increment_existing
pop
int 0
increment_existing:
int 1
+
store 1
txna ApplicationArgs 1
load 1
app_global_put

// store the voters choice in local state 在本地状态中存储投票者选择
int 0 //sender
byte "voted"
txna ApplicationArgs 1
app_local_put

int 1
return

voted:
pop
int 1
return
```

有关[原子传输](https://developer.algorand.org/docs/features/atomic_transfers/)、[有状态智能合约](https://developer.algorand.org/docs/features/atomic_transfers/)或[资产](https://developer.algorand.org/docs/features/atomic_transfers/)的更多信息，请参见开发人员文档。

## 总结
这个授权投票应用说明了如何使用Algorand的几个第一层特性来实现一个功能性的智能合约。应用程序的完整源代码可以在[Github](https://github.com/algorand/smart-contracts/tree/master/devrel/permissioned-voting)上找到。
