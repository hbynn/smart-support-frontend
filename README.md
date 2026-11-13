# 智能客服工单平台 · Web 前端

由 [hbynn](https://github.com/hbynn) 维护的智能客服工单平台 Web 前端，使用 Vue 构建，提供客服对话、知识库管理和运行状态查看。支持连接 Python 后端及兼容的 Java 后端，退款工单能力由 Python 后端提供。

## 功能

- 在页面中切换 Java / Python 后端。
- 统一适配 `/chat` 响应字段：
  - Python：`conv_id`、`agent_type`、`latency_ms`
  - Java：`conversation_id`、`agent_type`、`latency_ms`
- 支持聊天调试、健康检查、监控摘要、知识库检索、知识库文档导入、文件上传。
- 支持 Docker + Nginx 部署。

## 默认后端地址

| 后端 | 默认地址 |
|------|----------|
| Python | `http://localhost:8000` |
| Java | `http://localhost:8080` |

开发模式下，Vite 会代理：

| 前端路径 | 代理到 |
|----------|--------|
| `/api/python` | `http://localhost:8000` |
| `/api/java` | `http://localhost:8080` |

Docker 模式下，Nginx 会通过运行时注入的地址访问后端。默认仍指向宿主机上的 Python / Java 服务。

## 本地运行

安装依赖：

```bash
npm install
```

启动：

```bash
npm run dev
```

访问：

```text
http://localhost:5173
```

如果后端端口不是默认值，可以启动时覆盖：

```bash
VITE_PYTHON_API_URL=http://localhost:8000 \
VITE_JAVA_API_URL=http://localhost:8080 \
npm run dev
```

## Docker 部署

联合部署前，检查 [docker-compose.yml](docker-compose.yml) 中各服务的构建目录、环境文件路径和数据卷。路径必须与本机实际目录一致。配置完成后运行：

```bash
docker compose up -d --build
```

## 后端启动参考

Python 版默认：

```text
http://localhost:8000
```

Java 版默认：

```text
http://localhost:8080
```

两个后端不需要同时启动。前端页面里选择当前要调试的后端即可。


后端 Compose 服务通过 `env_file` 自动读取各自项目的 `.env`，普通启动不再需要额外 `--env-file`。
共享 Redis 的密码和前端端口仍由前端目录的 `.env`、shell 或显式 `--env-file` 提供；
Compose 中的 Redis 连接配置会覆盖后端文件内的 Redis 设置，保证两个后端连接同一个服务。
Python 文件中的 `EVAL_JUDGE_MODEL`、`EVAL_JUDGE_MAX_TOKENS` 等配置也会完整传入容器。
手动选择的后端不会因健康检查失败而改变；切换后端后，旧请求不能覆盖新页面的状态或回复。

## 退款工单链路

退款工单由 [Python 后端](https://github.com/hbynn/smart-support-backend#refund-workflow) 实现。体验退款业务时选择 Python 后端；当前前端提供聊天入口，尚无独立的工单创建和审批页面，可通过后端 Swagger（`http://localhost:8000/docs`）操作。

流程为：显式创建工单 → 补齐订单信息 → 查询订单并检查退款资格 → 按金额判断是否需要审批 → 幂等执行 → 查询退款结果 → 完成并记录审计。

- `POST /tickets` 创建工单，`GET /tickets/{ticket_id}` 查看进度。
- 金额达到默认阈值 100 时等待审批，通过 `/approve` 或 `/reject` 接口处理。
- 缺少订单信息时，通过 `/details` 接口补充。
- 结果未知时，通过 `/retry` 先核验再恢复执行，避免重复退款。
- `/audit` 接口查看审批、执行及核验记录；以上操作路径均以 `/tickets/{ticket_id}` 为前缀。

聊天中的退款咨询不等于执行退款。当前退款是 SQLite 本地业务演示，未接入真实支付渠道；完整接口、配置与状态说明见后端 README。