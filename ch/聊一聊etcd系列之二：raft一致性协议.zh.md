---
title: "聊一聊etcd系列之二：raft一致性协议"
date: 2022-06-03
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/etcd/raft/FG-%E8%84%91%E8%A3%82case.png
categories:
- Go
- 后端
tags:
- raft
- etcd
- 分布式系统

metaAlignment: center
---
共识算法旨在机器集群出现异常时候能够维持一个一致认可的结论来保证对外提供服务，今天来聊一聊etcd的共识机制----raft协议。
<!--more-->

![etcd-cover-raft](https://images.pexels.com/photos/966927/pexels-photo-966927.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=1)

## 解决什么问题
在进入etcd的raft之前，我们先来聊聊分布式系统的一致性是什么，raft协议解决了什么问题。
### 一致性的不同语义
实际上，在ACID事务特性中，一致性C更像是一种能量守恒或者质量守恒，比如转账、抢购，金额物品不会凭空消失或者出现冗余。  
在分布式领域，**分布式一致性更多往往指的是共识，指的是多个节点达成一致意见（要么大家一块错，要么一块对）**，区别在如果是ACID事务语义下的一致性，一般指强一致性。

## 如何解决
etcd内部通过选举和记录同步来达到一致性，日志对每一个认可的操作进行记录和定期压缩归档。  
etcd把更新记录通知到各个节点，在半数以上节点获得ack之后，主节点提交并且通知其他节点一同提交，通过其他节点的日志数据同步，其他节点达成一致状态只是时间问题。

### 选举与状态转移
在raft协议中，每个节点有三个角色状态，分别是：领导者Leader、竞选者Candidate、追随者Follower。节点通过定期的心跳探测和记录同步来更新各个节点的状态，每个节点都有争当Leader的意识，Leader通过与Follower定期发送心跳，来告知任期未结束，当心跳超时或者出现网络分区，Follower则会参与竞选，但是要遵循一些规则：
- Leader必须由Candidate产生，且得到了大多数票数
- 在已有Leader且当前Leader的任期是最新的，Follower不参与竞选
- 在Leader心跳超时的情况，Follower参与竞选变成Candidate，并为自己记一次投票
- **如果节点的日志比Candidate候选人的日志更新时间大，就会拒绝候选人的投票请求**

三个角色的状态转换如下图[FG-角色状态转移]所示：  
![FG-角色状态转移](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/etcd/raft/FG-%E8%A7%92%E8%89%B2%E7%8A%B6%E6%80%81%E8%BD%AC%E7%A7%BB.png)

### 日志副本
etcd各个节点的数据版本是怎么同步的呢？etcd使用日志来记录数据，每个节点都有自己的本地日志，每条日志记录当时Leader的任期编号，简称为termId，这个任期编号用来区分日志文件的版本新旧。**即如果节点的日志比Candidate候选人的日志更新时间大，就会拒绝候选人的投票请求。**  

一旦新的Leader角色上任，Leader会通过向Follower同步当前自己可提交的日志下标，如果有Follower没有跟上，Leader会逐级发送上一次的日志下标给Follower，最终直到Follower追上对齐同步。
同步请求体结构如下：
```go
type Entry struct {
    // 任期id
	Term  uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"`         
	// 日志下标
	Index uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"`
	Type  EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"`
	Data  []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"`
}
// AppendEntries RPC请求：尝试同步记录给节点
func (r *raft) appendEntry(es ...pb.Entry) (accepted bool) {
	li := r.raftLog.lastIndex()
	for i := range es {
		es[i].Term = r.Term
		es[i].Index = li + 1 + uint64(i)
	}
	// Track the size of this uncommitted proposal.
	if !r.increaseUncommittedSize(es) {
	    // 当前记录日志没有跟上Leader
		r.logger.Debugf(
			"%x appending new entries to log would exceed uncommitted entry size limit; dropping proposal",
			r.id,
		)
		// 拒绝Leader的新记录，需要获得旧记录来对齐
		return false
	}
	// use latest "last" index after truncate/append
	li = r.raftLog.append(es...)
	r.prs.Progress[r.id].MaybeUpdate(li)
	// Regardless of maybeCommit's return, our caller will call bcastAppend.
	r.maybeCommit()
	return true
}
```

### 日志压缩
由于操作更新日志的增量追加，raft节点在达成一致之后，会对日志版本进行定期归档压缩，有点类似于Redis的AOF和Snapshot备份策略，一次压缩可以释放空间，减轻存储空间避免对大量操作日志进行重放，也方便落后太多日志的节点快速赶上Leader。

### 网络分区处理
我们知道网络节点故障其实是会发生的，这里我们分为两种情况来分析如何保证一致性，假设现在有一个场景：  
客户端发送请求到Leader，Leader负责分发数据到Follower并且获得多数节点认可后进行提交，正常流程如下：
![image](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/etcd/raft/FG-%E6%95%B0%E6%8D%AE%E6%8F%90%E4%BA%A4%E6%B5%81%E7%A8%8B.png)

假如客户端更新过程出现了故障，那么不同的角色会怎么处理呢？
- Follower故障：Leader会不断的通过RPC请求确认Follower记录追上
- Leader故障：  
Leader故障情况可以再次细分，在宕机时刻数据日志到达Leader之后，Follower是否提交的两种情况：
  - 数据到达Leader，且未提交到Follower，此时Leader宕机，Client未收到ACK，则会开启重试逻辑，在新的Leader恢复之后继续保证一致性。
  - 数据到达Leader，部分Follower已经提交，此时Leader宕机，新的Leader将在已提交的部分Follower产生，因为他们的记录是最新的，再次竞选成功，成为新的Leader，最后进行一致性同步。
  - Leader节点出现网络分区，在短时间内出现了“脑裂”，即网络分区无法联系对方，在2个局部网络产生了2个Leader，这时候在另一个分区中，新的Leader会选出，新的Leader term编号+1，等到网络分区恢复之后，由于新的Leader由于term编号比旧Leader大，因此最终会以TermId最大的新Leader的数据版本同步。  
![image](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/etcd/raft/FG-%E8%84%91%E8%A3%82case.png)

### 线性读
在上面的网络分区故障场景中，我们知道了有可能有一段时间存在双Leader，即数据版本可能有脏数据。其实，在etcd数据集合上有一个最大提交下标```CommitIndex```，表示当前大多数节点认可的下标，```ReadIndex```表示当前节点可以获取的最新的下标(可能还没被大多数认可)。  
etcd通过这两个记录，提供了**是否开启线性读的选项(Serializable bool类型 )**，用于阻塞读请求，在需要强一致性的情况下，开启线性读，请求则会**等到Follower版本```ReadIndex```追上一致下标```CommitIndex```再返回数据**，这个时候就能保证读到的记录是集群中一致ack的。  

相关源码如下：
```go
type RangeRequest struct {
	// key is the first key for the range. If range_end is not given, the request only looks up key.
	Key []byte `protobuf:"bytes,1,opt,name=key,proto3" json:"key,omitempty"`
	
	// ...
	// serializable sets the range request to use serializable member-local reads.
	// Range requests are linearizable by default; linearizable requests have higher
	// latency and lower throughput than serializable requests but reflect the current
	// consensus of the cluster. For better performance, in exchange for possible stale reads,
	// a serializable range request is served locally without needing to reach consensus
	// with other nodes in the cluster.
	Serializable bool
	
	// ...
}

func (s *EtcdServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
	trace := traceutil.New("range",
		s.Logger(),
		traceutil.Field{Key: "range_begin", Value: string(r.Key)},
		traceutil.Field{Key: "range_end", Value: string(r.RangeEnd)},
	)
	ctx = context.WithValue(ctx, traceutil.TraceKey, trace)

	var resp *pb.RangeResponse
	var err error
	//...

	if !r.Serializable {
		// 线性读, 阻塞等待
		err = s.linearizableReadNotify(ctx)
		trace.Step("agreement among raft nodes before linearized reading")
		if err != nil {
			return nil, err
		}
	}
	chk := func(ai *auth.AuthInfo) error {
		return s.authStore.IsRangePermitted(ai, r.Key, r.RangeEnd)
	}

	get := func() { resp, err = txn.Range(ctx, s.Logger(), s.KV(), nil, r) }
	if serr := s.doSerialize(ctx, chk, get); serr != nil {
		err = serr
		return nil, err
	}
	return resp, err
}
```

### 总结
这是这几天通过对etcd内部共识机制的一些理解，主要涉及选举状态转移、日志压缩等概念。本系列的下一篇章将聊一聊etcd节点是怎么通信的。


## 参考链接
- raft消息类型  
https://pkg.go.dev/go.etcd.io/etcd/raft/v3#hdr-MessageType
- etc技术内幕 - 百里燊
- 云原生分布式存储基石etcd深入解析 - 华为云