---
layout: post
title: "分布式ID生成系统"
date: 2025-4-11
tags: [微服务]
comments: true
author: mazezen
---

代码地址: <a href="https://github.com/mazezen/mid" target="_blank">github mid</a>

## 简介

分布式 ID 生成系统是一个高性能、可靠的 ID 生成服务，支持两种模式：Snowflake（基于时间戳的内存生成）和 Segment（基于 MySQL 的号段分配）。系统采用双 Buffer 策略优化性能，集成 Prometheus 监控和 Zap 结构化日志，确保高可用性和可观测性。通过 gRPC 提供服务接口，支持高并发场景下的唯一 ID 生成，适用于分布式系统中的订单号、用户 ID 等场景。

设计灵感来源于<a href="https://tech.meituan.com/2019/03/07/open-source-project-leaf.html">美团 Leaf 系统</a>，结合现代技术栈（gRPC、Prometheus、Zap），优化了性能和运维体验。水平有限,仅供参考~

### BenchMark 性能测试结果汇总

Snowflake 和 Segment 模式的性能指标：

| 模式      | 并发级别 | QPS (请求/秒) | 平均延迟 (ms) | P99 延迟 (ms) | 错误次数 |
| --------- | -------- | ------------- | ------------- | ------------- | -------- |
| Snowflake | 10       | 53609.48      | 0.18          | 0.50          | 0        |
| Snowflake | 50       | 75156.37      | 0.66          | 1.57          | 0        |
| Snowflake | 100      | 76381.77      | 1.30          | 2.61          | 0        |
| Snowflake | 200      | 83512.68      | 2.38          | 3.71          | 0        |
| Segment   | 10       | 60197.30      | 0.16          | 0.49          | 0        |
| Segment   | 50       | 73595.61      | 0.67          | 1.54          | 0        |
| Segment   | 100      | 73389.83      | 1.35          | 2.38          | 0        |
| Segment   | 200      | 82509.48      | 2.41          | 3.56          | 0        |

### 说明

- **数据来源**：每个并发级别的结果取自测试输出的最后一次运行（如 bench_test.go:140 和 bench_test.go:156 的最后一行），以反映稳定的性能指标。
- **QPS**：每秒请求数，反映吞吐量。Snowflake 模式在并发 200 时 QPS 最高（83512.68），Segment 模式在并发 10 时 QPS 较高（60197.30），但在高并发下略低于 Snowflake。
- **平均延迟**：平均每次请求的延迟（毫秒）。Segment 在低并发（10）下延迟略低（0.16ms vs. 0.18ms），但高并发下两者接近（~2.4ms @ 200）。
- **P99 延迟**：99% 请求的延迟，反映尾部延迟。Segment 在并发 200 时 P99 延迟略低（3.56ms vs. 3.71ms），但在某些运行中出现异常高峰（例如 16.17ms）。
- **错误次数**：所有测试均无错误（Errors=0），表明系统稳定性良好。
- **运行环境**：测试在 macOS（Darwin）、AMD64 架构、VirtualApple @ 2.50GHz CPU 上运行，Buffer 大小为 10000，请求数为 10 次/协程。

###

## 设计目的

本系统旨在解决分布式系统中唯一 ID 生成的以下问题：

1. **唯一性**：保证全局唯一的 64 位整数 ID，无冲突。
2. **高性能**：支持高并发请求，低延迟（平均延迟 < 1ms），高吞吐量（QPS > 100,000）。
3. **高可用**：通过双 Buffer 和 MySQL 重试机制，减少服务中断。
4. **可扩展**：支持多种生成模式（Snowflake 和 Segment），便于业务扩展。
5. **可观测**：集成 Prometheus 监控 ID 生成速率、Buffer 使用率、MySQL 延迟和 NTP 偏移，使用 Zap 日志记录关键操作，便于调试和优化。

## 技术栈

- **编程语言**：Go 1.21+
- **服务框架**：gRPC（高性能 RPC 框架）
- **ID 生成**：
  - Snowflake：基于时间戳、数据中心 ID、机器 ID 和序列号，集成 `github.com/beevik/ntp` 进行时钟同步。
  - Segment：基于 MySQL 号段分配，集成 `github.com/cenkalti/backoff/v4` 实现指数退避重试。
- **数据库**：MySQL 8.0+（存储 Segment 模式的 ID 段）
- **监控**：Prometheus（`github.com/prometheus/client_golang`）暴露指标，Grafana 可选。
- **日志**：Zap（`go.uber.org/zap`）提供结构化、高性能日志。
- **依赖管理**：Go Modules

## 系统架构

- **核心组件**：
  - **Snowflake 模式**：内存生成 ID，依赖 NTP 同步时钟，适合高性能场景。
  - **Segment 模式**：通过 MySQL 分配 ID 段，适合需要持久化和严格递增的场景。
  - **双 Buffer**：每个模式维护两个 Buffer（buffer1 服务，buffer2 异步填充），减少阻塞。
- **服务接口**：gRPC 服务，提供 `GenerateID` 方法，通过 `mode` 参数选择生成模式（`snowflake` 或 `segment`）。
- **监控与日志**：
  - Prometheus 暴露 `/metrics` 端点，监控 ID 生成速率、Buffer 使用率、MySQL 延迟和 NTP 偏移。
  - Zap 记录 Buffer 填充、MySQL 查询、NTP 同步等关键事件。

## 快速开始

### 环境要求

- Go 1.21+
- MySQL 8.0+
- protoc（Protocol Buffers 编译器）
- Prometheus（可选，用于监控）

### 安装依赖

```bash
go mod init idgenerator
go get google.golang.org/grpc
go get github.com/go-sql-driver/mysql
go get github.com/cenkalti/backoff/v4
go get github.com/prometheus/client_golang/prometheus
go get go.uber.org/zap
```

### 配置 MySQL

1. 创建数据库和表：

```sql
CREATE DATABASE mid;
USE mid;
CREATE TABLE id_segments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    biz_tag VARCHAR(50) NOT NULL UNIQUE,
    max_id BIGINT NOT NULL DEFAULT 0,
    step INT NOT NULL DEFAULT 10000,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
INSERT INTO id_segments (biz_tag, max_id, step) VALUES ('default', 0, 10000);
```

2. 更新 server.go 中的 MySQL 数据源（DSN）：

```base
dsn := "user:password@tcp(localhost:3306)/mid"
```

### 编译 gRPC 服务

```base
protoc --go_out=. --go-grpc_out=. id_maker.proto
```

### 运行服务端

```bash
go run server.go
```

- 服务监听 localhost:50051（gRPC）和 localhost:9190（Prometheus 指标）。

- 确认 Zap 日志：

```log
2025-05-17 14:29:11	INFO	mid/main.go:24	Prometheus metrics server starting on :9190
2025-05-17 14:29:11	INFO	mid/server.go:100	Buffer filled	{"mode": "snowflake", "size": 1000}
2025-05-17 14:29:11	INFO	mid/server.go:100	Buffer filled	{"mode": "snowflake", "size": 1000}
2025-05-17 14:29:11	INFO	mid/segment.go:101	Fetched new segment	{"new_max": 11000, "duration_seconds": 0.020109417}
2025-05-17 14:29:11	INFO	mid/server.go:100	Buffer filled	{"mode": "segment", "size": 1000}
2025-05-17 14:29:11	INFO	mid/segment.go:101	Fetched new segment	{"new_max": 12000, "duration_seconds": 0.0031245}
2025-05-17 14:29:11	INFO	mid/server.go:100	Buffer filled	{"mode": "segment", "size": 1000}
2025-05-17 14:29:11	INFO	mid/main.go:80	gRPC server running on :50051
```

### 运行客户端

```
go run client.go
```

- 输出示例：

  ```text
  Snowflake ID: 578779521390612487
  Segment ID: 1001
  ```

### 配置 Prometheus

1. 创建 prometheus.yml：

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "mid"
    static_configs:
      - targets: ["localhost:9190"]
```

1. 运行 Prometheus：

```
prometheus --config.file=prometheus.yml
```

1. 访问 http://localhost:9190 查看指标：

- id_generate_total：ID 生成次数。
- buffer_usage：Buffer 剩余 ID 数量。
- mysql_query_duration_seconds：MySQL 查询延迟。
