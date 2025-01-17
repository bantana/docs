# 拜占庭容错
## 概念
PBFT是Practical Byzantine Fault Tolerance的缩写，这个算法在保证活性和安全性（liveness & safety）的前提下提供了(n-1)/3的容错性。

## 拜占庭网络模型
* 消息可能会丢失、损坏、延迟、重复发送，并且接收的顺序与发送的顺序不一致
* 节点可以随时加入、退出网络，可以丢弃消息、伪造消息、停止工作等，
还可能发生各种人为或非人为的故障

## PBFT属性
* 节点总数为n = 3f + 1
* 允许最多f个拜占庭(恶意)节点
* 允许最多f个非恶意节点因为宕机、网络延迟等原因没有响应
* 至少等待2f + 1个响应（防止网络分区）
* 至少等待f + 1个相同结果的响应（防止消息伪造）

## 为什么是3f + 1
### 可用性（非拜占庭网络）
* 当n = f + 1时，可以保证至少有1个节点可用（Primary/Backup）

### 一致性（非拜占庭网络）
* 当n = 2f + 1时，可以保证一致性（Paxos, Raft）

### 安全性（拜占庭网络）
* 允许f个节点没有响应，则至少收到n - f个响应
* n - f个响应中，允许f个恶意节点，则至少收到n - 2f个非恶意节点的响应和f个恶意节点的响应
* 为了保证最终采纳非恶意节点的响应，遵循少数服从多数的原则，则n - 2f > f，即n > 3f