## 实际使用反馈与优化

### 2018-3-31, 量小的情况下过载发现太慢

对于一个小量服务, 假设一个窗口内（15s）匀速只有20次过程调用，如果远端过载, 则在已有策略上需要1succ 19err 几乎全失败才会感知

更严重的是，匀速<20次过程调用，即使全部失败也不可能达到过载条件

**优化：**

连续2个窗口内都没过载，但是这2个窗口内真实失败率都高于70%: 说明是小业务量，认为此节点过载

### 2017-11-10，某场景下节点过载发现太慢

**场景回顾：**

远程服务mod的一台节点h已经挂了，但ELB系统一直没有将h节点判定为过载

**问题分析**

业务调用方的超时时间设置的是1秒，每次调用h节点都超时，即都消耗1秒，于是1秒才向ELB agent上报一次状态

与此行为冲突的是，agent每隔15秒会重置所有正常节点的调用统计，15s内，agent仅收到15次对h节点调用错误的上报，即15个err，根本达不到h节点过载的条件，于是h节点在每个15s内都一直被ELB认为是正常的，永远无法被判定为过载

**结论**

ELB的过载判定算法没有对节点调用耗时进行考虑

假设业务设置的调用超时时间为3秒，则业务方调用h节点失败消耗了3秒，并仅向ELB agent上报一个调用错误，`err + 1`

**优化：**

当调用节点失败，向上报状态要携带本次调用时间`timecost`（ms），ELB agent在收到调用失败的汇报时，会对失败个数根据`timecost`进行放大

具体是：
>`errcnt = timecost / 100; errcnt = 1 if timecost = 0`


于是乎，业务调用方调用h节点失败并消耗了1s，上报给ELB agent后，agent会认为h节点失败了1000ms/100 = 10次，会使挂掉的h更快达到过载条件

**本优化可以更快感知调用超时情况的过载**

