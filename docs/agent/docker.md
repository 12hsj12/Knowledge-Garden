# Docker：镜像、容器与 EchoMind 部署

## Docker 到底打包了什么

- **Dockerfile**：镜像构建步骤。
- **Image（镜像）**：可复用的只读应用运行模板，包含基础文件系统、依赖和默认启动命令。
- **Container（容器）**：镜像启动后的隔离进程和可写运行层。
- **Volume（数据卷）**：独立于容器生命周期的数据存储。
- **Network（网络）**：容器之间按服务名通信的网络。
- **Docker Compose**：用一份 YAML 描述多个容器、网络、健康检查、端口和卷，并以同一套命令启动。

容器化让应用代码和依赖以一致方式运行；Redis、ChromaDB、Prometheus 等状态服务仍然作为独立容器运行，持久数据放在卷或挂载目录中。镜像不是把整套业务数据永久封进一个程序包。

## EchoMind 的 Dockerfile

EchoMind 的 `Dockerfile` 使用多阶段构建：

1. `base`：Python 3.12 slim、工作目录和运行环境变量。
2. `dependencies`：安装 requirements，预下载 ChromaDB 使用的 ONNX embedding 模型。
3. `production`：从依赖阶段复制已安装包，只复制应用运行需要的文件，创建 `echomind` 非 root 用户并以 Uvicorn 启动 FastAPI。
4. `development`：保留开发依赖和 `--reload` 启动方式。

分阶段构建让生产镜像不必携带全部构建环境；依赖清单先于源码复制，也让依赖层能在代码改动时复用缓存。生产容器以 `0.0.0.0:8000` 监听容器端口，并通过 `/health` 做健康检查。

`.dockerignore` 排除了本机虚拟环境、Git、日志、`.env`、Chroma 本地数据和大型 GGUF 模型；这些内容分别通过安全配置或运行时挂载处理。源码提交时应同步检查 Docker build context 与 ignore 规则，确保镜像能拿到必需文件，也不把本机数据和密钥带入构建上下文。

## Compose 如何连接 EchoMind 的服务

当前 `docker-compose.yml` 编排：

- EchoMind FastAPI 主应用。
- Redis：24 小时工作记忆，使用 AOF 配置和命名卷。
- ChromaDB：情景记忆与用户画像的向量存储，使用持久化目录卷。
- OpenJev：独立决策服务，只通过 Compose 网络暴露给 EchoMind。
- Prometheus：采集服务指标并保存时间序列。
- Nginx：反向代理到健康的 EchoMind 应用。
- 显式启用的评测容器：执行固定样本 Benchmark 或 E2E 场景。

Compose 为服务创建 bridge 网络，应用使用服务名访问依赖：`redis:6379`、`chromadb:8000`、`openjev:3001`。容器中的 `localhost` 指当前容器自身；从 EchoMind 容器访问 Redis 时，连接目标是 Redis 服务名而不是宿主机 localhost。宿主机端口映射只用于宿主机或外部访问，服务间通信使用容器端口。

Redis、ChromaDB 和 Prometheus 的宿主机发布绑定到 `127.0.0.1`；EchoMind 主端口和 Nginx 对外发布。这种差异体现了内部依赖和用户入口的网络边界。

Compose 使用 healthcheck 和 `depends_on: condition: service_healthy` 控制启动依赖。例如 Redis 健康后再启动主应用；ChromaDB 与 OpenJev 也先通过健康检查。它处理服务启动顺序，应用仍要对运行期间断连、超时和重试定义处理方式。

## 数据、配置与安全边界

- Redis 的 AOF 数据放入 `redis-data` 命名卷，ChromaDB 使用 `chromadb-data`，Prometheus 使用 `prometheus-data`。
- `docker compose up` 重建容器时，独立命名卷继续保存数据；只有明确删除卷才会清掉对应状态。
- 本地密钥保存在被 Git 忽略的 `.env` 或 `.env.openjev`，仓库提交 `.env.example` 这类无密钥模板。
- `REDIS_PASSWORD` 是 Compose 必需配置，缺失时服务定义会在解析阶段报错。
- 模型 GGUF 通过只读挂载提供给应用，不打进生产镜像；OpenJev 与模型凭证按其独立服务配置传入。
- Compose 的 Prometheus 镜像使用可变的 `latest` 标签。追求可复现部署时，固定版本标签并在升级时成组验证。

## 常见 Docker 问题与排查路径

| 现象 | 常见原因 | 在 EchoMind 中检查 |
| --- | --- | --- |
| `COPY failed` 或找不到文件 | 构建上下文目录不对，或文件被 `.dockerignore` 排除 | 确认从 Python 项目根目录执行 build；检查 COPY 源路径和 ignore 规则 |
| 容器里访问 `localhost:6379` 失败 | localhost 指向应用容器本身 | 使用 Compose 服务名 `redis:6379` |
| Python 依赖改了但镜像没更新 | 构建缓存继续复用旧依赖层 | 确认 requirements 先复制；需要时重新构建镜像 |
| 容器启动后马上报 Redis/Chroma 错误 | 依赖未健康、密码未注入或服务地址错误 | 查看 `docker compose ps`、健康检查与容器日志 |
| 本地运行正常，容器里权限错误 | 非 root 用户、挂载目录属主或路径权限不同 | 检查 `echomind` 用户可写路径和 volume 权限 |
| 重建容器后数据仍在 | 数据存放在命名卷，而不在容器可写层 | 查 Compose 顶层 volumes 与挂载目标 |
| 源码改了但线上行为没变 | 运行的是旧镜像或旧容器 | 构建新镜像并重建对应服务，确认运行的 image ID |
| 容器外无法访问 | 容器监听地址、host/container 端口映射或绑定地址错误 | 确认 Uvicorn 监听 `0.0.0.0`，映射顺序为 host:container |

有用的日常命令：

```bash
docker compose config
docker compose ps
docker compose logs --tail=100 echomind
docker compose exec echomind python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8000/health').status)"
docker compose build echomind
docker compose up -d echomind
```

## 面试回答

### Docker 在项目里解决什么问题？

> 我把 EchoMind 应用及 Python 依赖构建成镜像，再用 Compose 编排 FastAPI、Redis、ChromaDB、OpenJev、Prometheus 和 Nginx。Compose 网络让服务按名字通信，健康检查控制启动依赖，命名卷保存 Redis、ChromaDB 和 Prometheus 数据。这样本地运行和部署环境能使用一致的服务拓扑与依赖版本。

### 为什么 Redis 数据放 volume，应用代码放镜像？

> 镜像适合版本化、可替换的应用代码和依赖；Redis、ChromaDB 的数据要跨容器重建保存，所以放独立 volume。应用镜像更新时保留数据卷，就可以升级服务而不把会话状态绑在单个容器的可写层里。

### Docker 常见的“代码能跑，容器不能跑”问题是什么？

> 最常见的是服务地址和运行环境不一致：容器内部应该通过 Compose 服务名访问 Redis/ChromaDB；必须确认 build context、依赖层、环境变量和非 root 用户的目录权限；再核对健康检查和端口映射。我的排查顺序是先 `docker compose config`，再 `ps` 和 `logs`，最后从应用容器内部访问依赖或健康接口。

## 参考资料

- [Dockerfile overview](https://docs.docker.com/build/concepts/dockerfile/)
- [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Compose networking](https://docs.docker.com/compose/how-tos/networking/)
- [Compose volumes](https://docs.docker.com/reference/compose-file/volumes/)
- [Docker Compose quickstart: health checks](https://docs.docker.com/compose/gettingstarted/)
