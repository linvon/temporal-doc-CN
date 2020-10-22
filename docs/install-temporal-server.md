---
id: install-temporal-server
title: Install Temporal
---

## 概述

本指南将向您展示如何使用`docker-compose`来快速安装和运行Temporal 。

## 准备工作

1. [Install Docker](https://docs.docker.com/engine/install)
2. [Install docker-compose](https://docs.docker.com/compose/install)

## 安装 Temporal

在终端中，`cd`进入要安装和运行Temporal的目录。

运行以下命令以下载临时docker-compose文件：

```bash
curl -L https://github.com/temporalio/temporal/releases/latest/download/docker.tar.gz | tar -xz --strip-components 1 docker/docker-compose.yml
```

完成后，您应该在工作目录中看到该`docker-compose.yml`文件。

:::note

您可以通过更改URL中的发行版本来安装特定版本的 Temporal：

`https://github.com/temporalio/temporal/releases/download/<release>/docker.tar.gz`

:::

## 运行 Temporal

运行以下命令以启动 Temporal 服务:

```bash
docker-compose up
```

您将看到类似于以下内容的输出:

```
Creating network "quick_start_default" with the default driver
Pulling temporal (temporalio/temporal-auto-setup:0.29.0)...
...
...
temporal_1   | Description: Default namespace for Temporal Server
temporal_1   | OwnerEmail:
temporal_1   | NamespaceData: map[string]string(nil)
temporal_1   | Status: NamespaceStatusRegistered
temporal_1   | RetentionInDays: 1
temporal_1   | EmitMetrics: false
temporal_1   | ActiveClusterName: active
temporal_1   | Clusters: active
temporal_1   | HistoryArchivalStatus: Enabled
temporal_1   | HistoryArchivalURI: file:///tmp/temporal_archival/development
temporal_1   | VisibilityArchivalStatus: Disabled
temporal_1   | Bad binaries to reset:
temporal_1   | +-----------------+----------+------------+--------+
temporal_1   | | BINARY CHECKSUM | OPERATOR | START TIME | REASON |
temporal_1   | +-----------------+----------+------------+--------+
temporal_1   | +-----------------+----------+------------+--------+
temporal_1   | + echo 'Default namespace registration complete.'
temporal_1   | Default namespace registration complete.
```

Temporal Server 现在正在运行!

您可以在[localhost：8088上](http://localhost:8088/)查看 Temporal Web 界面。

:::note

如果您希望将Temporal部署到Kubernetes集群，请遵循[helm-charts指南](https://github.com/temporalio/helm-charts)。

:::

## 运行工作流

现在，您可以通过 Temporal 运行工作流。

通过运行 [Go示例](https://github.com/temporalio/go-samples) 或 [Java示例 ](https://github.com/temporalio/java-samples)快速入门，或者使用 [Go SDK](https://docs.temporal.io/docs/go-quick-start/) 或 [Java SDK ](https://docs.temporal.io/docs/java-quick-start/)编写自己的[程序](https://github.com/temporalio/java-samples)。

