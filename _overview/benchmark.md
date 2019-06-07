---
title: Benchmark
layout: page
show_sidebar: false
menubar: overview_menu
---

## 测试环境

机器配置：

* CPU：Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz （24 cores）
* 内存​：128GB
* 存储：480G SSD *8
* 网卡：10Gb

集群配置：

* 节点数：5个replica server节点
* 测试表的Partition数：64个

## v1.10.0 测试结果

* 单条数据大小：320字节
* 测试时间：2018/07/27
* 测试工具：[YCSB](https://github.com/xiaomi/pegasus-ycsb) (使用Pegasus Java Client)
* 读写请求的数据分布特征：zipfian，可以理解为遵守80/20原则的数据分布，即80%的访问都集中在20%的内容上。

| 测试Case                      | 读写比 | 运行时长 | 读QPS    | 读Avg延迟 | 读P99延迟 | 写QPS | 写Avg延迟 | 写P99延迟 |
| ----------------------------- | ------ | -------- | -------- | --------- | --------- | ----- | --------- | --------- |
| (1)数据加载: 3客户端*10线程   | 0:1    | 1.89     | -        | -         | -         | 44039 | 679       | 3346      |
| (2)​读写同时: 3客户端*15线程  | ​1:3   | 1.24     | 16690    | 311       | 892       | 50076 | 791       | 4396      |
| ​(3)读写同时: 3客户端*30线程  | ​30:1  | 1.04     | 311633   | 264       | 511       | 10388 | 619       | 2468      |
| (4)数据只读: 6客户端*100线程  | 1:0​   | ​0.17    | ​978884  | 623       | ​1671     | -     | -         | -         |
| (5)数据只读: 12客户端*100线程 | 1:0​   | ​0.28    | ​1194394 | 1003      | ​2933     | -     | -         | -         |

注：

* 运行时长单位：小时。
* QPS单位：条/秒。
* 延迟单位：微秒。