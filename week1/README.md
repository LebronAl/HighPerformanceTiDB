<!-- TOC -->

- [实验要求](#实验要求)
- [实验环境](#实验环境)
- [实验过程](#实验过程)
    - [环境配置](#环境配置)
    - [编译部署](#编译部署)
        - [TiDB 部署](#tidb-部署)
        - [PD 部署](#pd-部署)
        - [TiKV 部署](#tikv-部署)
        - [客户端连接](#客户端连接)
        - [Dashboard 体验](#dashboard-体验)
    - [日志打印](#日志打印)
        - [添加过程](#添加过程)
        - [添加结果](#添加结果)
- [实验资料](#实验资料)

<!-- /TOC -->

## 实验要求
本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：
* 1 TiDB
* 1 PD
* 3 TiKV

改写后
* 使得 TiDB 启动事务时，会打一个 “hello transaction” 的日志

## 实验环境
| 配置 | 描述 |
|  ----  | ----  |
| 操作系统 | macOS 10.13.6 |
| 处理器 | Intel Core i5 5257U (2.7 GHz)|
| 内存 | 8G |
| 磁盘 | 256G SSD |

## 实验过程
### 环境配置
* **go 环境配置**：可参照此篇[博客](https://www.jianshu.com/p/ad57228c6e6a)安装并配置 go 环境。

* **rust 环境配置**：可参照此篇[博客](https://www.jianshu.com/p/5efdd9ce8565)安装并配置 rust 环境。

* **make 和 cmake 环境配置**：由于三个项目都用了 makefile 来自动化编译，因此需要安装 make 命令，macOS 下只要安装了 xcode 就会自动带有 make 命令。此外在编译 TiKV 时发现其有依赖包需要 cmake 来编译，因此也需要安装 cmake 命令，macOS 下在 terminal 下输入 `brew install cmake` 即可安装。

* **mysql 客户端环境配置**：由于 TiDB 兼容 mysql 协议，且在搭建集群后需要客户端来测试，因此也需要安装 mysql 客户端。macOS 下在 terminal 下输入 `brew install mysql` 即可安装。

* **命令行翻墙工具**：三个项目在编译时都会拉取一些远程 github 上的包，因此如果有此工具可以加快拉取速度，否则会特别慢。这个自行解决吧。

* **操作系统打开文件的最大句柄数**：启动 TiKV 时系统打开文件的最大句柄数需要大于等于 82920。macOS 下在 terminal 下输入 `ulimit -n 82920` 即可在此 shell 进程中修改。

### 编译部署
由于作业要求对 TiDB 代码改动后重新编译，因此需要对三个组件分别编译（由于只需要在 TiDB 中加日志，所以实际只编译 TiDB，其他两个组件用官方打的包也可以）。

启动时可以 nohup & 后台启动，也可以直接启动。为了查看日志和调试方便，我选择了后者。

#### TiDB 部署
克隆并编译即可，之后更改代码后只用 make 即可更新可执行文件。

启动时需要指定存储引擎为真实的 tikv 而不是基于内存实现的 mocktikv，在指定存储引擎 tikv 后也需要指定 pd 的地址。默认服务端口是 4000。
```
git clone https://github.com/pingcap/tidb
cd tidb
make server

./bin/tidb-server --store=tikv --path="127.0.0.1:2379"
```

#### PD 部署
克隆并编译即可，也可直接用官方包。

直接启动即可，默认服务端口是 2379。
```
git clone https://github.com/pingcap/pd
cd pd
make pd-server

./bin/pd-server
```

#### TiKV 部署
克隆并编译即可，也可直接用官方包。（编译巨慢，我本机需要半个小时）

启动 TiKV 时需要指定 PD 的地址。此外由于题目要求启动三个 TiKV 节点，所以如果使用一个可执行文件启动的话不仅要利用 addr 参数来指定不同的端口，还要利用 data-dir 来指定不同的文件夹来存数据。默认服务端口是 20160。
```
git clone https://github.com/pingcap/tikv
cd tikv
make

./target/release/tikv-server --pd-endpoints="127.0.0.1:2379"  --addr="127.0.0.1:20160" --data-dir=tikv
./target/release/tikv-server --pd-endpoints="127.0.0.1:2379"  --addr="127.0.0.1:20161" --data-dir=tikv1
./target/release/tikv-server --pd-endpoints="127.0.0.1:2379"  --addr="127.0.0.1:20162" --data-dir=tikv2
```

#### 客户端连接
使用 mysql 客户端连接 TiDB 集群即可感受 TiDB 了。
```
mysql -h 127.0.0.1 -P 4000 -u root -D test
```

#### Dashboard 体验
打开 [Dashboard](http://127.0.0.1:2379/dashboard/#/overview) 来查看集群状态。

### 日志打印

#### 添加过程
经过阅读相关文档和代码，发现 `NewTxn` 函数会在一个新事务开启时执行，所以追踪其在 session.go 中的实现。
```
// Context is an interface for transaction and executive args environment.
type Context interface {
	// NewTxn creates a new transaction for further execution.
	// If old transaction is valid, it is committed first.
	// It's used in BEGIN statement and DDL statements to commit old transaction.
	NewTxn(context.Context) error

    ...
}
```
session 结构体对 context 接口进行了实现。其中在 NewTxn 函数的实现中，最开始的几行应该是在判断此 context 是否已有事务，若有就提交它。因此在其后打印日志就满足了题意：TiDB 启动事务时，会打一个 “hello transaction” 的日志。
```
func (s *session) NewTxn(ctx context.Context) error {
	if s.txn.Valid() {
		txnID := s.txn.StartTS()
		err := s.CommitTxn(ctx)
		if err != nil {
			return err
		}
		vars := s.GetSessionVars()
		logutil.Logger(ctx).Info("NewTxn() inside a transaction auto commit",
			zap.Int64("schemaVersion", vars.TxnCtx.SchemaVersion),
			zap.Uint64("txnStartTS", txnID))
	}

	logutil.Logger(ctx).Info("start transcation")

	txn, err := s.store.Begin()
	if err != nil {
		return err
	}
	txn.SetVars(s.sessionVars.KVVars)
	if s.GetSessionVars().GetReplicaRead().IsFollowerRead() {
		txn.SetOption(kv.ReplicaRead, kv.ReplicaReadFollower)
	}
	s.txn.changeInvalidToValid(txn)
	is := domain.GetDomain(s).InfoSchema()
	s.sessionVars.TxnCtx = &variable.TransactionContext{
		InfoSchema:    is,
		SchemaVersion: is.SchemaMetaVersion(),
		CreateTime:    time.Now(),
		StartTS:       txn.StartTS(),
		ShardStep:     int(s.sessionVars.ShardAllocateStep),
	}
	return nil
}
```

#### 添加结果
客户端 cli 输入
```
start transaction;
```

服务端打印日志：
```
[2020/08/16 22:22:55.494 +08:00] [INFO] [session.go:1502] ["NewTxn() inside a transaction auto commit"] [conn=2] [schemaVersion=23] [txnStartTS=418798039482499073]
[2020/08/16 22:22:55.495 +08:00] [INFO] [session.go:1506] ["start transcation"] [conn=2]
```

## 实验资料
[【High Performance TiDB】Lesson 01：TiDB 整体架构](https://www.bilibili.com/video/BV17K411T7Kd?from=search&seid=14120828942041644681)

[【High Performance TiDB】知识扩展：TiDB 开发者社区现状](https://www.bilibili.com/video/BV1ak4y117mr?from=search&seid=14120828942041644681)

[TiDB 源码阅读系列文章（一）序](https://pingcap.com/blog-cn/tidb-source-code-reading-1/)

[TiDB 源码阅读系列文章（二）初识 TiDB 源码](https://pingcap.com/blog-cn/tidb-source-code-reading-2/)

[TiDB 源码阅读系列文章（三）SQL 的一生](https://pingcap.com/blog-cn/tidb-source-code-reading-3/)

[Percolator 和 TiDB 事务算法](https://pingcap.com/blog-cn/percolator-and-txn/)