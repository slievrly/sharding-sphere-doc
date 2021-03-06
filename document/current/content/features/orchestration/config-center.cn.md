+++
pre = "<b>3.3.1. </b>"
toc = true
title = "配置中心"
weight = 1
+++

## 实现动机

- 配置集中化：越来越多的运行时实例，使得散落的配置难于管理，配置不同步导致的问题十分严重。将配置集中于配置中心，可以更加有效进行管理。

- 配置动态化：配置修改后的分发，是配置中心可以提供的另一个重要能力。它可支持数据源、表与分片及读写分离策略的动态切换。

## 配置中心数据结构

配置中心在定义的命名空间的config下，以YAML格式存储，包括数据源，数据分片，读写分离、ConfigMap及Properties配置，可通过修改节点来实现对于配置的动态管理。

```
config
    ├──authentication                            # Sharding-Proxy权限配置
    ├──configMap                                 # 数据分片ConfigMap配置，以K/V形式存储，如：{"key1":"value1"}
    ├──props                                     # 属性配置
    ├──schema                                    # Schema配置
    ├      ├──sharding_db                        # SchemaName配置
    ├      ├      ├──datasource                  # 数据源配置
    ├      ├      ├──rule                        # 数据分片规则配置
    ├      ├──masterslave_db                     # SchemaName配置
    ├      ├      ├──datasource                  # 数据源配置
    ├      ├      ├──rule                        # 读写分离规则
```

### config/authentication

```yaml
password: root
username: root
```

### config/configmap

读写分离ConfigMap配置，以K/V形式存储。

```yaml
key2: value2
```

### config/sharding/props

相对于sharding-sphere配置里面的Sharding Properties。

```yaml
executor.size: 20
sql.show: true
```

### config/schema/schemeName/datasource

多个数据库连接池的集合，不同数据库连接池属性自适配（例如：DBCP，C3P0，Druid, HikariCP）。

```yaml
ds_0: !!org.apache.shardingsphere.orchestration.yaml.YamlDataSourceConfiguration
  dataSourceClassName: com.zaxxer.hikari.HikariDataSource
  properties:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_0?serverTimezone=UTC&useSSL=false
    password: null
    maxPoolSize: 50
    maintenanceIntervalMilliseconds: 30000
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    minPoolSize: 1
    username: root
    maxLifetimeMilliseconds: 1800000
ds_1: !!org.apache.shardingsphere.orchestration.yaml.YamlDataSourceConfiguration
  dataSourceClassName: com.zaxxer.hikari.HikariDataSource
  properties:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_1?serverTimezone=UTC&useSSL=false
    password: null
    maxPoolSize: 50
    maintenanceIntervalMilliseconds: 30000
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    minPoolSize: 1
    username: root
    maxLifetimeMilliseconds: 1800000
```

### config/schema/sharding_db/rule

数据分片配置，包括数据分片 + 读写分离配置。

```yaml
tables:
  t_order:
    actualDataNodes: ds_$->{0..1}.t_order_$->{0..1}
    keyGenerator:
      column: order_id
    logicTable: t_order
    tableStrategy:
      inline:
        algorithmExpression: t_order_$->{order_id % 2}
        shardingColumn: order_id
  t_order_item:
    actualDataNodes: ds_$->{0..1}.t_order_item_$->{0..1}
    keyGenerator:
      column: order_item_id
    logicTable: t_order_item
    tableStrategy:
      inline:
        algorithmExpression: t_order_item_$->{order_id % 2}
        shardingColumn: order_id
bindingTables:
  - t_order,t_order_item
broadcastTables:
  - t_config
  
defaultDataSourceName: ds_0
defaultDatabaseStrategy:
  inline:
    algorithmExpression: ds_$->{user_id % 2}
    shardingColumn: user_id
    
masterSlaveRules: {}
```

### config/schema/masterslave/rule

读写分离独立使用时使用该配置。

```yaml
name: ds_ms
masterDataSourceName: ds_master 
slaveDataSourceNames:
  - ds_slave0
  - ds_slave1
loadBalanceAlgorithmType: ROUND_ROBIN
```

## 动态生效
在注册中心上修改、删除、新增相关配置，会动态推送到生产环境并立即生效。