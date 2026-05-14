# oblogproxy Binlog 服务

> 适用版本：oblogproxy V4.x / OceanBase V4.2+
> 兼容模式：MySQL Mode

---

## 概述

oblogproxy 是 OceanBase 的增量日志代理服务，将 OceanBase 的事务日志（Redo Log）转换为 MySQL Binlog 格式，供下游消费者（如 Kafka、Canal、Flink CDC）消费。

| 功能 | 说明 |
|------|------|
| Binlog 协议 | 兼容 MySQL Binlog 协议 |
| 实时同步 | 毫秒级增量数据推送 |
| DDL 同步 | 支持 DDL 变更同步 |
| 多消费端 | 支持多个下游同时消费 |
| 断点续传 | 记录消费位点，支持断点恢复 |

---

## 1. 架构

```
OceanBase (Redo Log)
     ↓
oblogproxy (日志抽取 + Binlog 格式转换)
     ↓
下游消费端:
  ├── Canal → MySQL / 其他数据库
  ├── Kafka → 消息队列
  ├── Flink CDC → 实时计算
  └── Debezium → CDC 平台
```

---

## 2. 部署

### 2.1 安装

```bash
# 下载
wget https://mirrors.aliyun.com/oceanbase/OBLOGPROXY/current/oblogproxy-linux-x86_64.tar.gz

# 解压
tar -xzf oblogproxy-linux-x86_64.tar.gz -C /opt/oblogproxy

# 配置
cd /opt/oblogproxy
vi conf/oblogproxy.conf.toml
```

### 2.2 配置文件

```toml
# oblogproxy.conf.toml

[log]
level = "info"
dir = "/opt/oblogproxy/log"

[server]
listen_port = 2983    # oblogproxy 端口

[oceanbase]
rpc_port = 2882       # OBServer RPC 端口
hosts = [
  {ip = "10.0.0.1", rpc_port = 2882},
  {ip = "10.0.0.2", rpc_port = 2882},
  {ip = "10.0.0.3", rpc_port = 2882}
]

[checkpoint]
storage = "file"
dir = "/opt/oblogproxy/checkpoint"
```

### 2.3 启动

```bash
# 启动
cd /opt/oblogproxy
bash bin/oblogproxy.sh start

# 查看日志
tail -f log/oblogproxy.log

# 验证
telnet localhost 2983
```

---

## 3. 下游对接

### 3.1 Canal 对接

```properties
# canal.properties
canal.serverMode = kafka
canal.mq.servers = kafka-broker:9092

canal.instance.master.address = oblogproxy-host:2983
canal.instance.dbUsername = root
canal.instance.dbPassword = ******
canal.instance.connectionCharset = UTF-8
canal.instance.filter.regex = app_db\\..*
```

### 3.2 Flink CDC 对接

```java
// Flink CDC 消费 OceanBase
MySqlSource<String> source = MySqlSource.<String>builder()
    .hostname("oblogproxy-host")
    .port(2983)
    .databaseList("app_db")
    .tableList("app_db.orders")
    .username("root")
    .password("******")
    .deserializer(new JsonDebeziumDeserializationSchema())
    .build();

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.fromSource(source, WatermarkStrategy.noWatermarks(), "OceanBase CDC")
   .print();
env.execute("OB CDC Job");
```

### 3.3 Debezium 对接

```json
{
  "name": "ob-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "oblogproxy-host",
    "database.port": "2983",
    "database.user": "root",
    "database.password": "******",
    "database.server.id": "184054",
    "database.server.name": "ob_cluster",
    "database.include.list": "app_db",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes"
  }
}
```

### 3.4 MySQL Slave 对接

```bash
# 将 OceanBase 作为 MySQL 主库的 Slave
mysql -h oblogproxy-host -P 2983 -u root -p

CHANGE MASTER TO
  MASTER_HOST='oblogproxy-host',
  MASTER_PORT=2983,
  MASTER_USER='root',
  MASTER_PASSWORD='******',
  MASTER_LOG_FILE='ob-binlog.000001',
  MASTER_LOG_POS=4;

START SLAVE;
SHOW SLAVE STATUS\G
```

---

## 4. 配置选项

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `listen_port` | oblogproxy 监听端口 | 2983 |
| `rpc_port` | OBServer RPC 端口 | 2882 |
| `log.level` | 日志级别 | info |
| `checkpoint.storage` | Checkpoint 存储 | file |
| `checkpoint.dir` | Checkpoint 目录 | ./checkpoint |
| `worker.threads` | 处理线程数 | 4 |
| `batch.size` | 批量发送大小 | 500 |
| `flush.interval` | 刷间隔(ms) | 100 |

---

## 5. 监控

```bash
# oblogproxy 提供 Prometheus 指标
# 默认端口: 2984

curl http://oblogproxy-host:2984/metrics

# 关键指标：
#   oblogproxy_lag_seconds        -- 同步延迟
#   oblogproxy_events_total       -- 事件总数
#   oblogproxy_errors_total       -- 错误数
#   oblogproxy_connections_active -- 活跃连接数
```

---

## 6. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 无法连接 oblogproxy | 端口未开放 | 检查防火墙 |
| 同步延迟大 | 消费端处理慢 | 增加消费端并发 |
| Checkpoint 损坏 | 异常退出 | 清除 checkpoint 重新消费 |
| DDL 未同步 | 过滤了 DDL | 检查消费端 DDL 配置 |
| OOM | 消息堆积 | 增大内存或增加消费者 |

---

## 参考文档

- oblogproxy 文档：https://www.oceanbase.com/docs/oceanbase-database-cn/V420/oblogproxy
- Canal 对接：https://github.com/alibaba/canal
- Flink CDC：https://nightlies.apache.org/flink/flink-cdc-docs-stable/
- Debezium MySQL Connector：https://debezium.io/documentation/reference/stable/connectors/mysql.html
