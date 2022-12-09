# 背书验证策略服务


## 基本概念





## 源码分析
背书验证所涉及的函数都在`core/endorser/endorser.go`中。背书验证服务的函数是文件中的`ProcessProposal`函数，`endorser.go`中的其他函数都是配合实现`ProcessProposal`函数的。

接下来从`ProcessProposal`函数开始分析：

首先看函数签名：
```go
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error)
```
这里就涉及到了许多结构体了，包括`Endorser`、`pb.SignedProposal`和`pb.ProposalResponse`：
```go
type Endorser struct {
	ChannelFetcher         ChannelFetcher
	LocalMSP               msp.IdentityDeserializer
	PrivateDataDistributor PrivateDataDistributor
	Support                Support
	PvtRWSetAssembler      PvtRWSetAssembler
	Metrics                *Metrics
}
type SignedProposal struct {
	// The bytes of Proposal
	ProposalBytes []byte `protobuf:"bytes,1,opt,name=proposal_bytes,json=proposalBytes,proto3" json:"proposal_bytes,omitempty"`
	// Signaure over proposalBytes; this signature is to be verified against
	// the creator identity contained in the header of the Proposal message
	// marshaled as proposalBytes
	Signature            []byte   `protobuf:"bytes,2,opt,name=signature,proto3" json:"signature,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}

type ProposalResponse struct {
	// Version indicates message protocol version
	Version int32 `protobuf:"varint,1,opt,name=version,proto3" json:"version,omitempty"`
	// Timestamp is the time that the message
	// was created as  defined by the sender
	Timestamp *timestamp.Timestamp `protobuf:"bytes,2,opt,name=timestamp,proto3" json:"timestamp,omitempty"`
	// A response message indicating whether the
	// endorsement of the action was successful
	Response *Response `protobuf:"bytes,4,opt,name=response,proto3" json:"response,omitempty"`
	// The payload of response. It is the bytes of ProposalResponsePayload
	Payload []byte `protobuf:"bytes,5,opt,name=payload,proto3" json:"payload,omitempty"`
	// The endorsement of the proposal, basically
	// the endorser's signature over the payload
	Endorsement *Endorsement `protobuf:"bytes,6,opt,name=endorsement,proto3" json:"endorsement,omitempty"`
	// The chaincode interest derived from simulating the proposal.
	Interest             *ChaincodeInterest `protobuf:"bytes,7,opt,name=interest,proto3" json:"interest,omitempty"`
	XXX_NoUnkeyedLiteral struct{}           `json:"-"`
	XXX_unrecognized     []byte             `json:"-"`
	XXX_sizecache        int32              `json:"-"`
}
```

然后看该函数的具体实现。
```go
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error) {
	// 获取处理Proposal的开始时间
	startTime := time.Now()
	// Peer节点接收到的Proposal数 + 1
	e.Metrics.ProposalsReceived.Add(1)
	// 从上下文中提取发出Proposal的地址
	addr := util.ExtractRemoteAddress(ctx)
	// 调试信息输出
	endorserLogger.Debug("request from", addr)

	// variables to capture proposal duration metric
	success := false
	// 从signedProp取出Proposal
	up, err := UnpackProposal(signedProp)
	// 错误处理
	if err != nil {
		e.Metrics.ProposalValidationFailed.Add(1)
		endorserLogger.Warnw("Failed to unpack proposal", "error", err.Error())
		return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
	}
	// 设置通道
	var channel *Channel
	// 如果Proposal中指定了通道，那就是它
	if up.ChannelID() != "" {
		channel = e.ChannelFetcher.Channel(up.ChannelID())
		if channel == nil {
			return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: fmt.Sprintf("channel '%s' not found", up.ChannelHeader.ChannelId)}}, nil
		}
	} else {
		channel = &Channel{
			IdentityDeserializer: e.LocalMSP,
		}
	}

	// 对已签名的Proposal进行预处理，就是检查提案合法性
	err = e.preProcess(up, channel)
	if err != nil {
		endorserLogger.Warnw("Failed to preProcess proposal", "error", err.Error())
		return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
	}
	// 会在该方法结束后被调用
	defer func() {
		meterLabels := []string{
			"channel", up.ChannelHeader.ChannelId,
			"chaincode", up.ChaincodeName,
			"success", strconv.FormatBool(success),
		}
		e.Metrics.ProposalDuration.With(meterLabels...).Observe(time.Since(startTime).Seconds())
	}()
	// 通过验证后，模拟交易，完成背书，返回pResp
	pResp, err := e.ProcessProposalSuccessfullyOrError(up)
	if err != nil {
		endorserLogger.Warnw("Failed to invoke chaincode", "channel", up.ChannelHeader.ChannelId, "chaincode", up.ChaincodeName, "error", err.Error())
		// Return a nil error since clients are expected to look at the ProposalResponse response status code (500) and message.
		return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, nil
	}
	// 错误处理
	if pResp.Endorsement != nil || up.ChannelHeader.ChannelId == "" {
		// We mark the tx as successful only if it was successfully endorsed, or
		// if it was a system chaincode on a channel-less channel and therefore
		// cannot be endorsed.
		success = true

		// total failed proposals = ProposalsReceived-SuccessfulProposals
		e.Metrics.SuccessfulProposals.Add(1)
	}
	return pResp, nil
}
```

函数中也有很多结构体，如：
```go
// UnpackedProposal contains the interesting artifacts from inside the proposal.
type UnpackedProposal struct {
	ChaincodeName   string
	ChannelHeader   *common.ChannelHeader
	Input           *peer.ChaincodeInput
	Proposal        *peer.Proposal
	SignatureHeader *common.SignatureHeader
	SignedProposal  *peer.SignedProposal
	ProposalHash    []byte
}
```








## Fabric整体流程

- fabric节点类别：
	- 客户端：客户端提交实际交易调用倒背书者，广播交易请求到排序服务节点
		- 代表最终用户实体
		- 它必须连接到一个Peer节点以便与区块链交付。
		- 客户端可以选择连接任何Peer节点，创建并调用交易
	- Peer节点：提交交易、维持状态和账本拷贝。此外，Peer节点可以有一个特殊的背书角色
		- Peer节点以块的形式从排序服务节点接受有序状态更新，维护状态和账本
		- Peer节点能够附加一个特殊的**背书节点**角色。
		- 背书节点的特殊功能是关于特殊链码，存在于提交之前会背书一个交易。
		- 每个链码可以指定一个背书策略，引用一组背书节点。
		- 策略定义一个有效交易背书的必要和充分条件
		- 在部署交易的特殊情况下，安装链码（部署）背书策略是由系统链码的背书策略指定
	- 排序服务节点：运行通信服务实现交付保证，像原子或全序广播。
		- 排序者产生排序服务，即一个提供交付保证的通信架构

- 交易背书的基本工作流程
	- 客户端创建交易并且将其发送给它选择的背书Peer节点
	- 背书Peer节点模拟交易和产生背书签名
	- 提交客户端收集的交易背书并通过排序服务广播它
	- 排序服务向Peer节点提交交易

- 背书策略
	- 是背书一个交易的条件
	- 区块链Peer节点有一组预先确定的背书策略，它们是被安装特定链码的部署交易引用
	- 背书策略能参数化，这些参数能被部署交易指定



- Fabric交易流程
	- 客户端发起交易
	- 背书节点验证签名并执行交易
	- 审查提案反馈
	- 客户组合交易背书
	- 交易验证和提交
	- 账本更新



网络拓扑：
- 客户端
- Peer（Anchor（锚节点，与order进行连接）、Endorse（背书节点，为交易做担保）、Committer（记账节点，验证区块的有效性和交易的有效性））
- Orderer
	- 接受交易
	- 打包区块
	- 分发到Peer Anchor
	- Solo
	- KFK
- CA
	- 证书颁发



### 2 fabric共识

fabric共识模式采用的 Endorse+Kafka+Commit 的模式，这里我们简称EKC共识。此共识包含以下几个步骤：

1.  请求背书：  
    客户端用自己的私钥对交易进行签名后，按照指定格式将交易和签名信息进行打包，然后将打包后的数据发给背书节点请求背书。
    
2.  验证背书：  
    背书节点收到背书请求后，验证交易的签名是否正确并调用智能合约验证交易内容是否合法。验证通过的话，背书节点用自己的私钥对背书结果进行签名并按照指定格式打包，然后将打包后的数据发给客户端。
    
3.  提交交易：  
    客户端收到背书结果后，验证背书结果的签名是否正确。验证通过后，对交易请求和背书结果签名并打包。然后，把打包后的数据发送给orderer节点提交交易。
    
4.  排序广播：  
    orderer节点收到交易后，验证数据的客户端签名[[bJ1]](#_msocom_1) 是否正确。验证通过后，将交易发给kafka集群对应的topic。由于orderer中的对于每个通道都在kafka上监听对应的消息，因此，kafka将消息存放到对应topic上之后，会将消息广播给通道上的所有orderer。因为各个orderer的消息都是由kafka按照相同顺序发送的，因此，这个过程也实现了消息的排序。
    
5.  打包出块：  
    orderer节点接收到从kafka推送的消息（kafka节点见同步消息不需要验证），当满足出块策略[[bJ2]](#_msocom_2) ：缓存交易个数达到区块最大交易数或者时间达到出快时间，则将交易进行打包、对数据签名，然后出块，并将区块分发给peer节点。
    
6.  验证记账：  
    peer节点接收到区块后，验证交易是否有效即验证区块的交易是否满足背书策略以及区块中交易的读写集版本是否正确[[bJ3]](#_msocom_3) 。验证通过的话，执行此交易的内容更改状态数据库。验证失败的话，对此条交易不做任何处理。当区块中的交易全部处理完成后，将区块记录在本地数据库。
    

 







