# Hosted Harness Broker 环境变量与固定端口

[English](2026-10-21-hosted-harness-broker-environment.md) | [简体中文](2026-10-21-hosted-harness-broker-environment.zh-CN.md)

状态：已提出，并在同一变更中实现。

## 问题

Hosted Harness 只能通过 `--managed-runtime-broker-url` 和 `--managed-runtime-broker-token` 访问 Runtime Broker，这两个选项没有环境变量回落。因此随包的 systemd unit 把 `QWEN_HOSTED_HARNESS_BROKER_ARGS` 展开进命令行，本机任何用户都能从 `/proc/<pid>/cmdline` 读到 Broker token。unit 及其环境模板都因此带有警告：工具路径只能用于单租户主机。

Broker token 还被存了两份：Harness 通过 `harness.env` 获得它，控制面通过 `server.env` 里的 `QWEN_MANAGED_AGENT_RUNTIME_BROKER_TOKEN` 获得它，每次轮换都要改两处。

另外，请求的端口被占用时，`qwen serve` 会顺延到下一个空闲端口。对 Hosted Harness 来说，这从来不是运维想要的。控制面写死了一个 Harness URL，因此顺延后的 Harness 能正常启动，却让每个 Session 都失败，直到有人修改 `server.env`。部署文档和 systemd README 只是记录了这个陷阱，而没有阻止它。

## 目标

- Hosted Harness profile 接受来自环境的 Broker URL 和 token，随包文件不再把 token 放在命令行上。
- token 不会传给 Harness 启动的进程。
- 在 Harness 凭据文件的基础上，`harness.env` 为两个进程只保存一份 Broker token。
- Harness 端口被占用时启动失败，而不是顺延。

## 非目标

- 改变 default profile 的端口顺延。
- 改变 Broker 的认证方式或 token 格式。
- 删除现有的命令行选项；它们继续有效，并且优先。

## 设计

### 环境变量回落

在 `--profile hosted-harness` 下，`runQwenServe` 按以下顺序解析：

- `managedRuntimeBrokerUrl`：取 `--managed-runtime-broker-url`，否则取 `QWEN_HOSTED_HARNESS_BROKER_URL`；
- `managedRuntimeBrokerToken`：取 `--managed-runtime-broker-token`，否则取 `QWEN_HOSTED_HARNESS_BROKER_TOKEN`。

这与现有的 `QWEN_HOSTED_HARNESS_CAPABILITY_DIGEST` 回落方式一致。解析发生在 `validateHostedHarnessProfile` 之前，因此现有规则仍要求两个值同时存在或同时缺省。在该 profile 之外这些变量被忽略，与 digest 变量的现有行为相同。

### 子进程环境

`QWEN_HOSTED_HARNESS_BROKER_TOKEN` 加入 `SCRUBBED_CHILD_ENV_KEYS`，即 ACP bridge 在启动每个 `qwen --acp` 子进程时剔除的变量名单。该名单出于同样的理由已经包含 `QWEN_SERVER_TOKEN`：agent 通过 stdio 与 daemon 通信，从不需要这个凭据。否则，被提示注入的 agent 可以拿着 Broker token 直接调用 Broker。在 Hosted Harness profile 下，模型循环运行在 Harness 进程内，Workspace 工具运行在控制面启动的 Runtime worker 中。该 profile 不提供 shell、channel 或 WebSocket 隧道，因此目前 Harness 根本不会启动任何模型驱动的进程。这项剔除保证：一旦有路径启动 ACP agent 子进程，token 不会进入其中，与 `QWEN_SERVER_TOKEN` 现有的处理完全一致。

### 固定端口

在 `--profile hosted-harness` 下，请求的端口出现 `EADDRINUSE` 时启动失败，并给出端口号。控制面的 `QWEN_MANAGED_AGENT_HARNESS_BASE_URL` 指向这个端口，Harness 一旦顺延，每个 Session 都会失败。`--port 0` 保持其临时端口的含义。

### 共享 Broker token

Harness 凭据文件增加一个可选的第三个键。当设置了 `harness.credentials-file` 且文件中有 `QWEN_HOSTED_HARNESS_BROKER_TOKEN` 时，控制面从中取 `runtime-broker.token`。与另外两个键一样，文件设置了该键、同时又直接设置了 `QWEN_MANAGED_AGENT_RUNTIME_BROKER_TOKEN` 时，启动失败。文件中没有该键时仍由直接配置决定，因此纯文本部署不受影响。该行遵循与另外两个键相同的规则：`=` 两侧不留空白，值只用字母、数字和 `._:/+=-`。绑定是原子的：发生冲突时三个值都不会被绑定。

### 模板与文档

- bootstrap 把生成的 Broker token 以 `QWEN_HOSTED_HARNESS_BROKER_TOKEN` 写入 `harness.env`，并在旁边写上 `QWEN_HOSTED_HARNESS_BROKER_URL=http://127.0.0.1:4182`。`server.env` 不再包含该 token。
- `harness.env.example` 为工具路径列出这两个变量。
- Harness unit 的 `ExecStart` 去掉 `$QWEN_HOSTED_HARNESS_BROKER_ARGS`，并删除因它而存在的单租户警告。
- 部署文档、systemd README（中英文）和 server README 说明这两个变量和固定端口，并删去"端口顺延后更新 base URL"的说明。

## 兼容性与风险

- 通过命令行选项传参的部署不受影响；选项仍优先于对应的变量。
- 以前会顺延到空闲端口的 Hosted Harness，现在会启动失败。这个失败正是想要的信号；以前的行为会得到一个正在运行、却没有任何 Session 能访问的 Harness。
- 同时在 `harness.env` 和 `server.env` 中设置 Broker token 的部署，会在启动时失败，直到删掉其中一份。

## 验证计划

- CLI 单元测试：
  - 两个变量在该 profile 下被解析；
  - 选项覆盖对应的变量；
  - 在该 profile 之外变量被忽略；
  - 只有 URL 没有 token、或只有 token 没有 URL 时，校验仍然失败；
  - ACP 子进程环境不含 token，而父进程环境含有它。
- CLI 测试：端口被占用时 Hosted Harness profile 启动失败，而 default profile 仍顺延到下一个端口。
- Java 单元测试，针对可选的 Broker token 键：能从文件解析出来；与直接配置冲突时失败且不绑定任何值；键缺失时保留直接配置的值；该键所在行同样受凭据规则约束。
- 模板测试：`server.env` 不含 Broker token，`harness.env` 含两个变量，unit 的 `ExecStart` 不含 Broker 参数。
- 使用 bootstrap 文件的端到端测试：
  - Harness 从 `harness.env` 启动，不带 Broker 参数；
  - Harness 的 `/proc/<pid>/cmdline` 中没有 Broker token，Harness 的任何子进程也不带有它；
  - 工具路径的服务端能启动并完成一个 Turn；
  - 在同一端口上再启动一个 Harness 会启动失败。

## 验收标准

- 随包文件中没有任何一处把 Broker token 放在命令行上，且它只保存一份。
- Harness 的 ACP 子进程不继承该 token。
- Hosted Harness 端口被占用时启动停止，并给出端口号。
- 现有的命令行配置行为不变。

## 后续工作

在部署易用性系列中，除发布第一个 reference 预发布版外，没有其他计划中的后续工作。
