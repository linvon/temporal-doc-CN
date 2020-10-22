---
id: configure-temporal-server
title: Configure Temporal
---

在 `development.yaml`可以找到 Temporal Server 的配置，其中包含以下可能的部分:

- [**global**](#global) 
- [**persistence**](#persistence) 
- [**log**](#log) 
- [**clusterMetadata**](#clustermetadata)
- [**services**](#services)
- [**kafka**](#kafka)
- [**publicClient**](#publicclient)
- archival
- dcRedirectionPolicy
- dynamicConfigClient
- namespaceDefaults

**注意：**更改`development.yaml`文件中的任何属性都需要重新启动进程才能使更改生效。

**注意：**如果您想更深入地了解我们如何实际解析此文件，请[在此处](https://github.com/temporalio/temporal/blob/master/common/service/config/config.go)查看我们的源代码。

## global

本`global`部分包含全局的配置。请参阅下面的最小配置（注释掉了可选参数）。

```yaml
global:
  membership: 
    name: temporal
    #maxJoinDuration: 30s
    #broadcastAddress: "127.0.0.1"
  #pprof:
    #port: 7936
  #tls:
    #...

```

### membership - *必填*

`membership`节控制以下成员关系层参数:

- `name` - *必填* - 用于标识 gossip 环中的其他的集群成。所有成员节点都必须相同。
- `maxJoinDuration` - 服务尝试加入 gossip 环的超时时间
- `broadcastAddress` - 用作与远程节点进行连接的地址. 
  - 当BindOnIP在多个节点相同（如0.0.0.0）且用于 nat 穿透场景时，通常使用此项。`net.ParseIP`来控制其所支持的语法。注意：仅支持IPV4。

### pprof

- `port` - 如果指定，则将在指定的端口上启动进程时初始化 pprof。

### tls

`tls`部分控制网络通信的 SSL / TLS 设置，包含两个子部分`internode`和`frontend`。`internode`部分管理角色之间的内部服务通信，`frontend`管理 SDK 客户端与 frontend 服务角色之间的通信。

这些小节中的每个小节都包含一个`server`小节和一个`client`小节。`server`包含以下参数：

- `certFile` -包含要使用的证书的PEM编码公钥文件的路径。
- `keyFile` -包含要使用的证书的PEM编码私钥文件的路径。
- `requireClientAuth`-*布尔值*-要求客户端在连接时使用证书进行身份验证，否则默认为双向 TLS。
- `clientCaFiles`-包含您希望信任的用于客户端身份验证的证书颁发机构的PEM编码公共密钥的文件的路径列表。如果`requireClientAuth`未启用，则忽略此值。

以下是在 SDK 和 frontend API 之间启用服务器 TLS（https）的示例：

```yaml
global:
  tls:
    frontend:
      server:
        certFile: /path/to/public/cert
        keyFile: /path/to/private/cert
      client:
        serverName: dnsSanInFrontendCertificate
```

注意，`client`通常是需要配置的，通过`serverName`字段指定所提供的服务器证书中包含的 DNS SubjectName；这是必需的，因为Temporal 使用 IP 到 IP 的通信。如果服务器证书包含适当的 IP 使用者备用名称，则可以忽略此选项。

此外，当客户端的主机不信任服务器使用的根CA时，需要配置`rootCaFiles`字段。下面的示例扩展了上面的示例，手动指定了 frontend 服务使用的根CA：

```yaml
global:
  tls:
    frontend:
      server:
        certFile: /path/to/public/cert
        keyFile: /path/to/private/cert
      client:
        serverName: dnsSanInFrontendCertificate
        rootCaFiles:
          - /path/to/frontend/server/ca
```

下面是一个完全安全的集群的另一个示例，它使用双向 TLS 与手动指定的 Cas 进行前端和节点间通信:

```yaml
global:
  tls:
    internode:
      server:
        certFile: /path/to/internode/publicCert
        keyFile: /path/to/internode/privCert
        requireClientAuth: true
        clientCaFiles:
          - /path/to/internode/serverCa
      client:
        serverName: dnsSanInInternodeCertificate
        rootCaFiles:
          - /path/to/internode/serverCa
    frontend:
      server:
        certFile: /path/to/frontend/publicCert
        keyFile: /path/to/frontend/privCert
        requireClientAuth: true
        clientCaFiles:
          - /path/to/internode/serverCa
          - /path/to/sdkClientPool1/ca
          - /path/to/sdkClientPool2/ca
      client:
        serverName: dnsSanInFrontendCertificate
        rootCaFiles:
          - /path/to/frontend/serverCa

```
**注意：**如果启用了客户端身份验证，则`internode.server`证书将用作服务中的客户端证书。这增加了以下要求：

- 该`internode.server`证书必须在所有角色中指定，即使是只有 frontend 配置。
- 内部节点间服务器证书必须用 **no** Extended Key Usages 或 ServerAuth and ClientAuth EKUs 生成。
- 如果您的证书颁发机构不受信任，例如上一个示例，则需要在以下位置指定节点间服务器 Ca：
  - `internode.server.clientCaFiles`
  - `internode.client.rootCaFiles`
  - `frontend.server.clientCaFiles`

## persistence
`persistence`节包含数据存储/持久层的配置。以下是受密码保护的 Cassandra 群集的最小配置示例。

```yaml
persistence:
  defaultStore: default
  visibilityStore: visibility
  numHistoryShards: 512
  datastores:
    default:
      cassandra:
        hosts: "127.0.0.1"
        keyspace: "temporal"
        user: "username"
        password: "password"
    visibility:
      cassandra:
        hosts: "127.0.0.1"
        keyspace: "temporal_visibility"
```

以下顶级配置项是必需的:

- `numHistoryShards` - *必需* - 初始化集群时要创建的历史分片数。 
  - **警告**：此值是不可变的，在第一次运行后将被忽略。请确保将该值设置得足够高，以适应该群集的最坏情况峰值负载。
- `defaultStore` - *必需* - Temporal 服务使用的数据存储定义的名称。
- `visibilityStore` - *必需* - Temporal 可视化服务使用的数据存储定义的名称。
- `datastores` - *必需* - 包含要引用的命名数据存储配置。
  - 每个定义都用一个声明名称的标题定义（即上述的：`default:`及`visibility:`），其中包含一个数据存储定义。
  - 数据存储区定义必须为`cassandra`或`sql`。

`cassandra`数据存储定义可以包含下列值:

- `hosts` - *必需* - Cassandra endpoints.
- `port` - 默认值: 9042 - 用于`gocql`客户端连接的 Cassandra 端口。
- `user` - 用于`gocql`客户端身份验证的 Cassandra 用户。
- `password` - 用于`gocql`客户端身份验证的 Cassandra 密码。
- `keyspace` - *必需* -  Cassandra 键空间.
- `datacenter` - Cassandra 的数据中心过滤器参数.
- `maxConns` - 单个TLS配置允许的到此数据存储的最大连接数。
- `tls` - 请参阅下面的TLS。

`sql`数据存储定义可以包含下列值:

- `user` - 用于身份验证的用户。
- `password` - 用于身份验证的密码。
- `pluginName` - *必需* - SQL数据库类型。
  - *有效值*: `mysql` 或 `postgres`.
- `databaseName` - *必需* - 要连接的SQL数据库的名称。
- `connectAddr` - *必需* - 数据库的远程地址。
- `connectProtocol` - *必需* - 连接远程数据库的协议
  - *有效值*: `tcp` 或 `unix`
- `connectAttributes` - 一个 K/V 映射的 Map，将作为连接`data_source_name`url 的一部分发送。
- `maxConns` - 与此数据存储区的最大连接数。
- `maxIdleConns` - 与此数据存储的最大空闲连接数。
- `maxConnLifetime` - 是连接可以存活的最长时间。
- `numShards` - 用于分片的sql数据库中的表的存储分片的数量（*默认值：* 1）。
- `tls` - 见下文。

`tls` 部分可能包含:

- `enabled` - *boolean*.
- `serverName` - 托管数据存储的服务器的名称。
- `certFile` - 证书文件的路径。
- `keyFile` - 密钥文件的路径。
- `caFile` - ca文件的路径。
- `enableHostVerification` - *boolean* -`true`验证主机名和服务器证书（就像 Cassandra 集群的通配符）。此选项基本上与`InSecureSkipVerify`相反。 见 `InSecureSkipVerify` 在 http://golang.org/pkg/crypto/tls/ 获取更多信息。

注意：`certFile`和`keyFile`是可选的，具体取决于服务器配置，但是必须同时省略两个字段，以避免使用客户端证书。

## log
`log`部分是可选的，包含以下可能的值:

- `stdout` - *boolean* - `true` 输出到标准输出
- `level` - 设置日志记录级别。
    - *有效值* - debug, info, warn, error 或 fatal.
- `outputFile` - 输出日志文件的路径。

## clusterMetadata

`clusterMetadata` 包含所有群集定义，包括那些参与跨 DC 的定义。

一个 `clusterMetadata` 小节的示例:
```yaml
clusterMetadata:
    enableGlobalNamespace: false
    failoverVersionIncrement: 10
    masterClusterName: "active"
    currentClusterName: "active"
    clusterInformation:
        active:
            enabled: true
            initialFailoverVersion: 0
            rpcAddress: "127.0.0.1:7233"
    #replicationConsumer:
      #type: kafka
```
- `currentClusterName` - *必需* - 当前群集的名称。**警告**：此值是不可变的，在第一次运行后将被忽略。
- `enableGlobalNamespace` - *默认值:* `false`.
- `replicationConsumerConfig` - 确定使用哪种方法来使用复制任务。类型可以是`kafka`或`rpc`。
- `failoverVersionIncrement` - 发生故障转移时每个集群版本的增量。
- `masterClusterName` - 主群集名称，只有主群集可以注册/更新命名空间。所有群集都可以进行命名空间故障转移。
- `clusterInformation` - 包含集群名称到`ClusterInformation`定义的映射。`ClusterInformation`部分包括：
  - `enabled` - *boolean*
  - `initialFailoverVersion` - 初始的故障转移版本
  - `rpcAddress` - 指明远程服务地址（主机：端口）。主机可以是DNS名称。使用`dns:///`前缀启用 DNS 的 IP 地址之间的轮询。

## services
`services` 节包含按服务角色类型分配的配置。当前支持四种服务角色:

- `frontend`
- `matching`
- `worker`
- `history`

以下是`services`下 `frontend` 服务定义的最小示例 :
```yaml
services:
  frontend:
    rpc:
      grpcPort: 8233
      membershipPort: 8933
      bindOnLocalHost: true
    metrics:
      statsd:
        hostPort: "127.0.0.1:8125"
        prefix: "temporal_standby"

```

每个服务标题下定义了两个部分：

### rpc - *必需*
`rpc` 包含服务与其他服务交互方式有关的设置。支持以下值：

- `grpcPort` 是gRPC侦听的端口。
- `membershipPort` - 由成员关系侦听器使用。
- `bindOnLocalHost` - 使用 `localhost` 作为侦听地址。
- `bindOnIP` - 用于在特定IP上绑定服务（例如`0.0.0.0`）- 参考`net.ParseIP`获取支持的语法，仅支持IPv4，与`BindOnLocalHost`选项互斥。
- `disableLogging` - 禁用 rpc 的所有日志记录。
- `logLevel` - 所需的日志级别。

**注意**：当前所有主机的角色类型之间的端口值均应保持一致。

### metrics
`metrics` 包含按键值进行指标监控子系统的配置。支持三种程序：

- `statsd`
- `prometheus`
- `m3`

`statsd` 部分支持以下设置:

- `hostPort` - statsd 服务的 host:port.
- `prefix` - 向`statsd上报的特定前缀 .
- `flushInterval` - 发送数据包的最大间隔。（*默认为*1秒）。
- `flushBytes` - 指定您希望发送的最大UDP数据包大小。（*默认为*1432个字节）。

此外，指标支持以下非提供的特定设置：

- `tags` - 要上报的 KV 键值对集，对每一个指标都生效。
- `prefix` - 对每一个上报的指标设置前缀

## kafka
 `kafka` 节描述了所有连接到 Kafka 集群所需的配置，并支持以下值:

- `tls` - 使用 TLS , 同 `persistence` 章节.
- `clusters` - 命名的 `ClusterConfig` 定义的映射，描述每个 Kafka 集群的配置。一个`ClusterConfig`定义包含使用`brokers`配置值的brokers 列表。 
- `topics` - 卡夫卡集群主题 map。
- `temporal-cluster-topics`- 命名的 `TopicList` 定义 map.
- `applications` - 命名的 `TopicList` 定义 map.

 `TopicList` 定义描述每个集群的主题名称，并包含以下必需值：
- `topic`
- `retryTopic`
- `dlqTopic`

以下是 `kafka` 部分示例:

```yaml
kafka:
  tls:
    enabled: false
    certFile: ""
    keyFile: ""
    caFile: ""
  clusters:
    test:
      brokers:
        - 127.0.0.1:9092
  topics:
    active:
      cluster: test
    active-dlq:
      cluster: test
    standby:
      cluster: test
    standby-dlq:
      cluster: test
    other:
      cluster: test
    other-dlq:
      cluster: test
  temporal-cluster-topics:
    active:
      topic: active
      dlq-topic: active-dlq
    standby:
      topic: standby
      dlq-topic: standby-dlq
    other:
      topic: other
      dlq-topic: other-dlq
```

## publicClient 
`publicClient`是必填部分，其中包含一个值`hostPort`，该值用于指定访问 frontend 端的 IPv4 地址或 DNS 名称和端口。

例:
```yaml
publicClient:
  hostPort: "localhost:8933"
```

使用 `dns:///` 前缀启用 DNS 的 IP 地址之间的轮询。
