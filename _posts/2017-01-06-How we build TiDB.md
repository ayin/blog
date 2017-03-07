---
title: How we build TiDB
categories:
  - Technology
tags:
  - 数据库
---

今天公司请黄东旭([dongxu’s blog](http://0xffff.me/)) 进行分析《如何打造开源分布式关系数据库 TiDB》，自己对这方面的了解很少，听到很多东西，想完全弄明白他所讲的内容估计还很遥远，下面主要记录一下一些以前不熟悉或者不知道的名词：

* Raft: 分布式一致性算法，类似于Paxos
* 分布式存储的核心：sharding策略；元信息存储
* Sharding 策略：Range 和 Hash
* CoreOS: CoreOS是一个基于Docker的轻量级容器化Linux发行版，专为大型数据中心而设计，旨在通过轻量的系统架构和灵活的应用程序部署能力简化数据中心的维护成本和复杂度
* etcd： etcd 是一个应用在分布式环境下的 key/value 存储服务
* Rust: 类C++语言， [如何看待 Rust 的应用前景？ - Rust（编程语言） - 知乎](https://www.zhihu.com/question/30407715)
* NewSQL：支持分布式存储，同时支持关系型SQL语句查询
* Codis：https://github.com/CodisLabs/codis，类似美团的medis
* 写放大和读放大
* LevelDB: https://github.com/google/leveldb
* RocksDB: [GitHub - facebook/rocksdb: A library that provides an embeddable, persistent key-value store for fast storage.](https://github.com/facebook/rocksdb)
* F1: [F1: A Distributed SQL Database That Scales](https://research.google.com/pubs/pub41344.html)
* Spanner: [从Google Spanner漫谈分布式存储与数据库技术](http://history.programmer.com.cn/14015/)
* TiDB：类似于F1， [GitHub - pingcap/tidb: TiDB is a distributed NewSQL database compatible with MySQL protocol](https://github.com/pingcap/tidb)
* TiKV：类似于Spanner， https://github.com/pingcap/tikv