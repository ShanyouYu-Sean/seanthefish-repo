---
layout: post
title: raft算法
date: 2020-11-29 10:00:00
tags: 
- raft
categories:
- raft
---

本来想在k8s章节里在说raft算法的，但看redis选主的时候发现用的也是raft算法，故提前把raft算法拿出来说一下。

raft算法的动画演示：http://thesecretlivesofdata.com/raft/

不上图了，图都在动画中。

raft是一个共识算法（consensus algorithm），所谓共识，就是多个节点对某个事情达成一致的看法。

## 什么是分布式共识？

假设我们有一个单节点服务器和一个客户端，客户端可以很轻松的向单个节点发送存储数据的请求，从而达成一致。

但如果我们的服务器是多节点的呢？客户端没办法发送一个储存请求，使多节点达成一致，这就是分布式架构中的共识问题。

## raft是如何解决问题的

首先，一个节点可以存在三种状态：

- follower state（跟随者状态）
- candidate state（候选人状态）
- leader state（领导者状态）

初始化状态时，节点都是Follower状态

### leader选举（选主）

所有的节点在最开始，都始于 follower state。

- 如果一个follower得不到leader的响应，它就会成为一个candidate
- candidate会向其他的节点发送申请投票成为leader的请求，其他节点会响应此次投票
- 如果candidate得到了大多数节点的票数，他就会成为leader。

### 日志复制

当leader选出后，所有针对集群的改动，都会先经过leader节点。

- leader节点收到的每个改变的请求时，都会在节点的log中加入该请求的条目，但此时，这个请求并未被提交，所以并不会对节点上的值做实质性的改变。
- 要想提交这次请求，leader节点必须首先复制log中的条目，并发送给它的follower，然后leader会等待，直到大多数的follower响应并写入log。此时，leader节点方可提交这次请求，并把leader的值进行改变。
- 然后leader会通知它的follower这次请求已被提交，follower方可作出值的改变。至此，集群中的所有节点就都达成了共识。

## leader选举 详解

在raft算法中，用两个超时设置来控制选举：

- 选举超时（election timeout）（其实是不是叫等待选举更好？）
  选举超时是指节点从Follower状态成为Candidate状态需要等待的时间。在这个时间内，节点必须等待，不能成为Candidate状态。（通常选举超时会被随机设为150ms到300ms之间）
  - 当选举超时时间过去后，节点会从Follower成为Candidate，并发起一次新的选举任期，并为自己投票
  - 然后会发送投票请求到其他的节点，收到请求的节点如果没有在此次任期中投过票，就会投给当前这个Candidate，并重置自己的选举超时时间。
  - 当Candidate获得大多数节点的投票后，他就会成为leader。
- 心跳超时（heartbeat timeout）
  leader会根据固定间隔时间向Follower发送消息，这个固定间隔时间被称为心跳超时
  - 当节点成为leader后，会向它的Follower发送添加log条目（心跳）的请求，这个请求会根据心跳超时的时间来循环发送。
  - Follower节点收到每一条添加log条目（心跳）的请求，都需要向Leader节点响应这条日志复制的结果，并重置其选举超时时间。
  - 这一个选举任期不会有节点状态改变，除非，一个Follower节点，因为没有在它的选举超时时间内，得到复制日志的请求（心跳），而变成Candidate，并重新发起一个选举任期。

要求会的大多数其票数，意味着，在一个任期内，只会有一个leader被选出。

如果两个节点在同时变成了Candidate，那么分票的情况便可能发生，因为一个Follower在一选举任期内只能响应一个Candidate的投票请求，那么这时有可能两个Candidate都得不到大多数的票数，此时，在超过选举超时事件后，Candidate会重置选举超时时间，并进行重新开始一个选举任期，直到有一方胜出。

如何判断选举任期？ 选举任期只有Candidate才能发起，当发起一个选举任期后，Candidate在申请投票的请求中包含任期信息，而follower会在响应投票请求记录当前选举任期信息，这样就能保证follower不能在一个选举任期内响应多个投票请求。

## 日志复制 详解

当我们的集群中有了leader后，我们便需要向其他follower同步日志。

- 首先，客户端会向leader发送改变值的请求，这次请求会添加到leader的log中
- 然后leader会带着这次log的改动，向follower发送下一次心跳
- 但大多数follower响应了这次log改动（心跳）请求后，leader上这次log改动的条目就会被提交
- 最后值会被改变，leader会返回给客户端response

## raft算法在网络分区中保持一致

当出现网络故障，节点可能会出现网络分区，假设我们有两个节点在一个网络中，三个节点在另一个网络中。

此时便会出现多个leader，但两个节点的所在的分区，由于无法得到大多数节点的响应，因此不能将作出改变的日志进行提交。

而当网络分区修复时，为提交日志记录的节点在收到新的同步日志请求时，就会回滚之前的未提交记录，而去同步新主的日志，从而修复节点的不一致性。

## 如何确定节点下线（redis的实现）

follower在心跳超时后，不会立即晋升为candidate，只标记成leader主管下线，在等待超过半数的节点都认为leader主观下线后，才会确认客观下线，同时根据自己log和主节点的偏移量去sleep相应的时间后再晋升为candidate去发起投票。

## raft 和 redis 的异同

https://zhuanlan.zhihu.com/p/112651338

