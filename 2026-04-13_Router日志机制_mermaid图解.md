# Router 日志机制 Mermaid 图解（2026-04-13）

> 目标：帮助快速理解本仓库中“文件日志（router/api/relay/error）”与“数据库日志（event_logs）”两套体系，以及 error 级别日志进入 `error.log` 的完整链路。

## 1) 泳道图（总览）

```mermaid
flowchart LR
  %% Swimlane-like lanes via subgraph
  subgraph L1[配置与启动]
    A1[common.Init\n读取 --log-dir / config.yaml]
    A2[设置 logger.LogDir]
    A3[app.Run -> logger.SetupLogger]
  end

  subgraph L2[Logger 初始化]
    B1[创建 router.log writer]
    B2[创建 error.log writer]
    B3[创建 api.log writer]
    B4[创建 relay.log writer]
    B5[gin.DefaultWriter -> stdout + api.log]
    B6[gin.DefaultErrorWriter -> stderr + router.log + error.log]
  end

  subgraph L3[请求与业务]
    C1[普通 HTTP 请求\ngin.LoggerWithFormatter]
    C2[Relay 请求\nRelayLogger + controller.Relay]
    C3[系统/业务代码\nSysLog/SysError/Error/RelayError]
    C4[业务消费日志\nmodel.RecordConsumeLog]
  end

  subgraph L4[落盘与存储]
    D1[api.log\n访问日志]
    D2[router.log\n系统日志主文件]
    D3[relay.log\nrelay 专项日志]
    D4[error.log\nERROR/FATAL 聚合]
    D5[event_logs 表\n业务日志]
  end

  A1 --> A2 --> A3
  A3 --> B1
  A3 --> B2
  A3 --> B3
  A3 --> B4
  B3 --> B5
  B1 --> B6
  B2 --> B6

  C1 --> D1
  C2 --> D3
  C3 --> D2
  C3 -.ERROR/FATAL duplicate.-> D4
  B6 --> D4
  C4 --> D5
```

## 2) 序列图（error.log 关键链路）

```mermaid
sequenceDiagram
  autonumber
  participant U as Client
  participant G as Gin Router
  participant M as Middleware\n(TraceID/RelayLogger)
  participant R as Business Controller
  participant L as common/logger
  participant RL as relay.log
  participant ROT as router.log
  participant ERR as error.log

  U->>G: HTTP/Relay 请求
  G->>M: 进入中间件
  M->>R: 执行业务

  alt 正常
    R->>L: Info/SysLog/RelayInfof
    L->>ROT: 写 INFO/WARN 到 router.log
    L->>RL: (relay info) 写 relay.log
    G-->>U: 2xx
  else 业务错误/上游错误
    R->>L: Error/SysError/RelayErrorf/Fatal
    L->>ROT: 写 ERROR/FATAL 到 router.log
    L->>RL: (relay error) 写 relay.log
    L->>ERR: 额外写入 ERROR/FATAL（聚合）
    G-->>U: 4xx/5xx
  else Gin panic/recovery 输出
    G->>ERR: 通过 gin.DefaultErrorWriter
    G->>ROT: 同时写 stderr+router.log
    G-->>U: 500
  end
```

## 3) 状态图（Relay 请求与日志级别）

```mermaid
stateDiagram-v2
  [*] --> Received: 请求进入 /v1 或 /api/v1/public
  Received --> RelayStart: RelayLogger START

  RelayStart --> UpstreamCall: 调用上游

  UpstreamCall --> Success: status < 400 且无 relay_error
  Success --> LogInfo: RelayInfof(END)
  LogInfo --> [*]

  UpstreamCall --> ClientErr: 400-499
  ClientErr --> LogWarn: RelayWarnf(END)
  LogWarn --> [*]

  UpstreamCall --> ServerErr: status >= 500 或 relay_error 非空
  ServerErr --> Retrying: shouldRetry=true 且有候选渠道
  Retrying --> UpstreamCall: 切换渠道重试

  ServerErr --> FinalFail: 不可重试/重试耗尽
  FinalFail --> LogError: RelayErrorf(FAIL)
  LogError --> ErrorFile: ERROR/FATAL 进入 error.log
  ErrorFile --> [*]
```

## 4) 序列图（event_logs 数据库日志链路）

```mermaid
sequenceDiagram
  autonumber
  participant C as Relay/Billing/User Controller
  participant M as internal/admin/model.Log API
  participant Repo as internal/admin/repository/log
  participant DB as LOG_DB.event_logs
  participant FL as File Logger

  C->>M: RecordConsumeLog / RecordTopupLog / RecordLog
  M->>Repo: mustLogRepo().RecordXxx(...)
  Repo->>Repo: 补全字段(id, created_at, trace_id, type)
  Repo->>DB: INSERT event_logs

  alt INSERT 成功
    Repo->>FL: logger.Infof("record log: ...")
  else INSERT 失败
    Repo->>FL: logger.Error("failed to record log: ...")
    FL->>FL: ERROR 同时写 router.log + error.log
  end
```

## 结论速记

- `error.log` 不是离线筛选，而是运行时在 `ERROR/FATAL` 级别“同步额外写入”。
- `api.log` 主要承载 HTTP 访问日志；`relay.log` 承载 relay 结构化事件；`router.log` 承载通用系统日志。
- `event_logs` 是业务统计/账务维度日志（前端日志页主要读这里），与文件日志并行存在。
