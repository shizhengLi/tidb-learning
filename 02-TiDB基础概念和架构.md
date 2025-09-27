# TiDB 基础概念和架构详解

## 1. TiDB 核心概念

### 1.1 分布式数据库基础

#### 什么是分布式数据库？

分布式数据库是将数据分散存储在多个物理节点上的数据库系统，通过网络进行协调和管理。与传统单机数据库相比，分布式数据库具有以下特点：

- **水平扩展**：通过添加节点来扩展容量和性能
- **高可用性**：单点故障不影响整体服务
- **数据分布**：数据自动分布到多个节点
- **透明访问**：用户像使用单机数据库一样使用

#### CAP 理论在 TiDB 中的应用

CAP 理论指出分布式系统不可能同时满足：
- **一致性 (Consistency)**：所有节点在同一时间看到相同的数据
- **可用性 (Availability)**：每个请求都能收到响应
- **分区容错性 (Partition Tolerance)**：网络分区时系统仍能运行

TiDB 选择 **CP**（一致性 + 分区容错性），确保数据强一致性。

### 1.2 TiDB 的核心特性

#### 1.2.1 分布式事务

TiDB 实现了完整的 ACID 事务：

```sql
-- 完整的 ACID 事务示例
BEGIN;

-- 转账操作
UPDATE accounts SET balance = balance - 100
WHERE user_id = 1 AND balance >= 100;

UPDATE accounts SET balance = balance + 100
WHERE user_id = 2;

-- 检查约束
SELECT balance FROM accounts WHERE user_id = 1;
SELECT balance FROM accounts WHERE user_id = 2;

COMMIT;
```

**两阶段提交 (2PC) 协议**：
1. **准备阶段**：所有参与者预执行并锁定资源
2. **提交阶段**：协调者通知所有参与者提交或回滚

#### 1.2.2 水平扩展能力

TiDB 的水平扩展体现在：

- **存储层扩展**：TiKV 节点可以动态添加
- **计算层扩展**：TiDB 节点可以独立扩容
- **负载均衡**：自动将请求分发到不同节点

```sql
-- 查看数据分布情况
SHOW TABLE regions FROM your_table;
SHOW TABLE DIGESTS FROM your_database;
```

#### 1.2.3 MySQL 兼容性

TiDB 高度兼容 MySQL 协议和语法：

```sql
-- MySQL 应用无需修改即可迁移
-- 支持常见的 MySQL 数据类型和函数
SELECT
    id,
    name,
    email,
    DATE_FORMAT(created_at, '%Y-%m-%d') as formatted_date,
    MD5(name) as name_hash
FROM users
WHERE created_at > '2024-01-01';
```

## 2. TiDB 架构详解

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Application Layer                       │
│  (MySQL Client, JDBC, ORM, etc.)                                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        TiDB Cluster                             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   TiDB      │  │   TiDB      │  │   TiDB      │  SQL Layer   │
│  │   Server    │  │   Server    │  │   Server    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│         │              │              │                       │
│         └──────────────┼──────────────┘                       │
│                        │                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │     PD       │  │     PD       │  │     PD       │ Placement  │
│  │   Server     │  │   Server     │  │   Server     │ Driver     │
│  └─────────────┘  └─────────────┘  └─────────────┘ Scheduler  │
│         │              │              │                       │
│         └──────────────┼──────────────┘                       │
│                        │                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │    TiKV     │  │    TiKV     │  │    TiKV     │ Storage     │
│  │   Server    │  │   Server    │  │   Server    │ Layer       │
│  │(Row Store)  │  │(Row Store)  │  │(Row Store)  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   TiFlash   │  │   TiFlash   │  │   TiFlash   │ Analysis    │
│  │   Server    │  │   Server    │  │   Server    │ Engine      │
│  │(Column Store)│  │(Column Store)│  │(Column Store)│              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件详解

#### 2.2.1 TiDB Server (SQL 层)

**职责**：
- SQL 解析和优化
- 事务管理和协调
- 元数据缓存
- 客户端连接管理

**主要模块**：
- **Parser**：SQL 解析器
- **Optimizer**：查询优化器
- **Executor**：执行引擎
- **Transaction Manager**：事务管理器

```sql
-- 查看当前连接的 TiDB Server 信息
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM information_schema.GLOBAL_VARIABLES
WHERE VARIABLE_NAME LIKE 'tidb_%';
```

#### 2.2.2 TiKV Server (存储层)

**核心特性**：
- 分布式事务键值存储
- Raft 一致性协议
- MVCC 多版本并发控制
- 自动数据分片

**数据模型**：
```
Key:   table_id + index_id + row_values
Value: encoded_row_data
```

**Raft 组**：
- 每个 Region (默认 96MB) 形成一个 Raft 组
- Leader 负责读写，Follower 同步数据
- 自动选举和故障转移

```sql
-- 查看 Region 信息
SHOW TABLE regions FROM your_table;
SHOW REGIONS WHERE db_name = 'your_database';
```

#### 2.2.3 PD Server (调度器)

**主要功能**：
- 集群元数据管理
- Region 调度和负载均衡
- 时间戳分配器 (TSO)
- 全局唯一 ID 分配

**调度策略**：
- Region 均衡分布
- Leader 均衡分布
- 热点 Region 调度
- 副本健康检查

```sql
-- 查看集群状态信息
SHOW TIDB_CLUSTER_CONFIG;
SHOW PLACEMENT LABELS;
```

#### 2.2.4 TiFlash Server (分析引擎)

**列式存储特性**：
- 高压缩率
- 向量化执行
- 实时数据同步
- HTAP 查询优化

**数据同步机制**：
- 通过 Raft Learner 协议同步
- 异步构建列式存储
- 支持实时和批量同步

```sql
-- 设置 TiFlash 副本
ALTER TABLE your_table SET TIFLASH REPLICA 1;

-- 查看 TiFlash 副本状态
SHOW TABLE tiflash_replica FROM your_database;
```

### 2.3 关键技术原理

#### 2.3.1 MVCC (多版本并发控制)

TiDB 使用 MVCC 实现事务隔离：

```sql
-- MVCC 数据结构
-- Key: table_id_index_id_rowid
-- Value: commit_timestamp + row_data

-- 查看数据版本信息
BEGIN;
SET TRANSACTION READ ONLY;
START TRANSACTION WITH CONSISTENT SNAPSHOT;

-- 执行查询，看到特定时间点的数据
SELECT * FROM users WHERE id = 1;

COMMIT;
```

**事务隔离级别**：
- **RC (Read Committed)**：默认级别
- **RR (Repeatable Read)**：支持快照隔离

#### 2.3.2 分布式事务实现

**全局时间戳**：
- PD 为每个事务分配唯一时间戳
- start_ts：事务开始时间戳
- commit_ts：事务提交时间戳

**两阶段提交流程**：
1. **Prewrite 阶段**：在所有相关节点锁住数据
2. **Commit 阶段**：根据协调器决策提交或回滚

```sql
-- 查看事务信息
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- 查看活跃事务
SELECT * FROM information_schema.CLUSTER_TRANSACTIONS;
```

#### 2.3.3 数据分片策略

**自动分片**：
- 表数据按主键自动分片
- 每个Region 默认大小 96MB
- 支持预分片减少热点

**分片键选择**：
- 主键或唯一索引
- 哈希分片避免热点
- 支持自定义分片策略

```sql
-- 预分片设置
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id INT,
    order_date DATE
) SHARD_ROW_ID_BITS = 4 PRE_SPLIT_REGIONS = 3;

-- 查看分片信息
SHOW TABLE regions FROM orders;
```

### 2.4 高可用机制

#### 2.4.1 Raft 一致性协议

**Raft 组特性**：
- Leader 选举
- 日志复制
- 成员变更
- 快照压缩

**故障恢复流程**：
1. 检测 Leader 故障
2. 发起新选举
3. 选举新 Leader
4. 恢复服务

#### 2.4.2 多副本策略

**副本配置**：
- 默认 3 副本
- 支持跨机房部署
- 可配置副本数量

**数据一致性保证**：
- 强一致性写入
- 读一致性可配置
- 自动副本修复

```sql
-- 配置副本策略
CREATE PLACEMENT POLICY my_policy
PRIMARY_REGION="beijing"
REGIONS="beijing,shanghai";

CREATE TABLE my_table (
    id INT PRIMARY KEY
) PLACEMENT POLICY my_policy;
```

## 3. 架构优势和适用场景

### 3.1 架构优势

#### 3.1.1 弹性扩展
- **存储扩展**：TiKV 节点动态添加
- **计算扩展**：TiDB 节点独立扩容
- **读写分离**：TiFlash 专门处理分析查询

#### 3.1.2 高可用性
- **无单点故障**：每个组件都有多个副本
- **自动故障转移**：秒级故障检测和恢复
- **数据安全**：多副本保证数据不丢失

#### 3.1.3 一致性保证
- **强一致性**：分布式事务保证 ACID
- **线性一致性**：全局时间戳保证
- **数据正确性**：MVCC 避免脏读幻读

### 3.2 适用场景

#### 3.2.1 适合场景

1. **高并发 OLTP 系统**
   - 电商订单系统
   - 金融交易系统
   - 用户管理系统

2. **实时分析系统**
   - 业务报表系统
   - 用户行为分析
   - 实时监控分析

3. **数据中台和仓库**
   - 统一数据平台
   - 实时数仓
   - 跨库查询

#### 3.2.2 不适合场景

1. **强依赖存储过程的系统**
   - TiDB 对存储过程支持有限

2. **超高写入吞吐量的特殊场景**
   - 如 IoT 时序数据（建议用专用时序数据库）

3. **对 MySQL 特有功能强依赖**
   - 部分高级 MySQL 特性暂不支持

## 4. 总结

TiDB 的分布式架构设计实现了：

- **水平扩展**：通过添加节点无限扩展
- **高可用**：多副本 + 自动故障转移
- **强一致性**：分布式事务 + MVCC
- **HTAP 能力**：同时支持事务和分析
- **云原生**：容器化部署和管理

理解这些基础概念和架构原理，有助于更好地使用 TiDB 进行应用开发和运维管理。

---

**下一步学习**：继续学习 TiDB 的安装部署、基本操作和高级特性。