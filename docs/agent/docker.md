# Docker：用一套固定的运行环境启动 EchoMind

## 先回答：Docker 和 README 有什么区别？

README 是给人看的项目说明，可以写安装命令、启动顺序和注意事项。Docker 是一套实际构建和运行程序的工具：Dockerfile 描述如何制作运行镜像，Docker Compose 可以按配置启动应用及其依赖服务。

| 内容 | 主要作用 | 类比 |
| --- | --- | --- |
| README | 告诉开发者项目是什么、怎样安装和启动 | 操作说明 |
| Dockerfile | 写明镜像如何制作：选基础环境、复制文件、安装依赖、设置启动命令 | 制作清单 |
| Image（镜像） | 制作完成、可以反复启动的程序运行模板 | 已准备好的应用包 |
| Container（容器） | 从镜像启动起来的一个实际进程实例 | 正在运行的一份程序 |
| Compose 文件 | 描述应用需要哪些容器、如何互相连接、端口和数据放在哪 | 多个应用包的启动与连接配置 |

README 可以告诉你“先装 Python，再装依赖，然后启动服务”。Dockerfile 则把其中一部分步骤写成机器可以执行的构建过程，生成镜像；Docker 再从镜像启动容器。README 也可以说明如何使用 Docker，两者经常一起出现。

镜像包含应用运行所需的文件、依赖和默认启动命令；它不包含完整的一台电脑，也不会自动带上生产密码和需要长期保存的用户数据。容器通常与宿主机共用操作系统内核，但拥有相对隔离的文件系统、进程和网络环境。

## 从一个小型 FastAPI 程序走一遍

一个最小的 Dockerfile 可以这样写：

~~~dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
~~~

把它逐行翻译成日常语言：

1. FROM：以一个带 Python 3.12 的基础环境开始。
2. WORKDIR：在镜像里准备 /app 作为工作目录。
3. COPY 和 RUN：复制依赖清单并安装 Python 包。
4. COPY . .：把应用代码复制进镜像。
5. CMD：容器启动时默认运行 Uvicorn。

在 Dockerfile 所在的项目目录运行：

~~~bash
docker build -t my-api:dev .
docker run --rm -p 8000:8000 my-api:dev
~~~

第一条命令读取 Dockerfile 和当前构建目录，做出名为 my-api:dev 的镜像。第二条命令从镜像启动一个容器，并把宿主机的 8000 端口连到容器的 8000 端口。这样浏览器访问宿主机的 8000 端口，就能到达容器里的 FastAPI。

这里 Dockerfile 是构建配方，镜像是构建结果，容器是运行中的程序。镜像可以启动多个容器；删除并重建容器不会修改原来的镜像。

容器写入自己的临时可写层。容器被替换后，这一层里的临时数据通常也随之消失。需要保留的 Redis、数据库或向量库数据应放到独立的数据卷。

## 多个服务怎么一起启动：Docker Compose

EchoMind 不是只有一个 FastAPI 进程，它还需要 Redis、ChromaDB、OpenJev、Prometheus 和 Nginx。Compose 用一份 YAML 说明这些服务如何启动、连接和保存数据。例如一个极简的 API + Redis 配置可以写成：

~~~yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  redis-data:
~~~

执行 docker compose up 后，Compose 会按配置创建容器、网络和数据卷。这个例子里的 API 容器通过 redis:6379 找 Redis，Redis 数据放到独立的 redis-data 卷中。实际部署还需要按服务配置密码、健康检查和启动依赖。

### 容器网络里最容易弄错的 localhost

在 API 容器中，localhost 指向 API 容器自己，并不代表 Redis 容器。如果 API 需要连接 Compose 里的 Redis，地址应使用服务名 redis 和容器端口 6379。

端口映射通常写作“宿主机端口:容器端口”，例如 8000:8000。宿主机或外部用户访问左边的端口；同一 Compose 网络中的服务通常用服务名和右边的容器端口通信。

### 数据卷解决什么问题

Redis 容器被重建时，容器自身的临时写入层会被换掉。把 Redis 数据目录挂载到 Docker volume 后，容器可以替换，数据仍保存在独立卷里。镜像负责应用程序与依赖；数据卷负责需要跨容器保存的数据。

## Docker 在 EchoMind 项目中的实际结构

### Dockerfile 做了什么

EchoMind 的 Dockerfile 使用多阶段构建：

1. base 阶段准备 Python 3.12 slim 和基础运行设置。
2. dependencies 阶段安装 requirements，并预下载 ChromaDB 使用的 ONNX embedding 模型。
3. production 阶段从依赖阶段复制安装好的包和运行代码，创建非 root 的 echomind 用户，再以 Uvicorn 启动 FastAPI。
4. development 阶段保留开发依赖，并用 --reload 启动，便于本地改代码时自动重新加载。

依赖清单先复制、应用源码后复制，可以让依赖层在代码变化时复用构建缓存。生产镜像不需要带上全部开发构建内容。

应用在容器中监听 0.0.0.0:8000，并通过 /health 提供健康检查。这里监听 0.0.0.0 的意思是接受容器网络接口上的连接；如果只监听 127.0.0.1，其他容器和端口映射可能无法连接进来。

.dockerignore 用来排除不该发给镜像构建过程的本机内容，例如虚拟环境、Git、日志、.env、Chroma 本地数据和大型 GGUF 模型。它可以减少构建上下文，也能避免把密钥或本机数据打包进镜像。应用仍需要的配置和模型由环境变量或运行时挂载提供。

### Compose 拉起了哪些东西

EchoMind 的 docker-compose.yml 描述了：

- EchoMind FastAPI 应用。
- Redis：保存 24 小时工作记忆，并使用 AOF 配置和命名卷。
- ChromaDB：保存情景记忆与用户画像的向量数据，并挂载持久化卷。
- OpenJev：独立决策服务，通过 Compose 网络提供给 EchoMind。
- Prometheus：定时读取服务指标并保存时间序列。
- Nginx：作为反向代理，把用户流量转到健康的 EchoMind 应用。
- 固定样本 Benchmark 和 E2E 场景使用的评测服务。

Compose 网络里，EchoMind 用服务名访问 redis:6379、chromadb:8000 和 openjev:3001。Redis、ChromaDB 和 Prometheus 的宿主机端口只绑定在 127.0.0.1；EchoMind 主端口和 Nginx 对外发布。这样内部依赖可以留在服务网络里，用户入口则由应用或代理发布。

Compose 配置了健康检查和 service_healthy 启动依赖，让主应用在 Redis 等依赖通过健康检查后再启动。它解决的是启动阶段的依赖顺序；应用运行过程中仍需要处理服务断连、请求超时和重试。

### 哪些东西留在镜像外面

- Redis AOF 数据、ChromaDB 数据和 Prometheus 历史数据分别放在命名卷中。
- 本地密钥放在被 Git 忽略的 .env 或 .env.openjev；仓库保留不含真实密钥的示例模板。
- GGUF 模型通过只读挂载提供给应用，不放进生产镜像。
- Compose 要求提供 REDIS_PASSWORD；缺失时配置解析会报错。
- Prometheus 镜像当前使用 latest 标签。若希望部署版本更可复现，应固定镜像版本并在升级时验证。

这也说明“用 Docker 启动项目”仍然需要正确配置密钥、模型文件和持久化数据。镜像提供应用运行环境，Compose 连接服务，配置与数据由部署环境提供。

## 常见问题：按现象理解

| 现象 | 直白解释 | EchoMind 里检查什么 |
| --- | --- | --- |
| 容器访问 localhost:6379 失败 | 应用容器正在找自己，而不是 Redis 容器 | 使用服务名 redis:6379 |
| 本机访问不到容器里的服务 | 容器端口没有发布到宿主机，或监听地址不对 | 看 ports 映射和 0.0.0.0 监听 |
| 改了源码，运行结果没变化 | 当前容器可能还在用旧镜像 | 重新构建并重建对应服务 |
| 重建 Redis 容器后数据还在 | 数据写在独立 volume 中 | 查看 Compose 的卷挂载 |
| 容器启动后连不上 ChromaDB | 依赖仍在启动、服务名或端口配置错误 | 查看健康检查、Compose 配置和容器日志 |
| 构建时找不到某个文件 | 构建目录不对，或文件被 .dockerignore 排除 | 检查 build context、COPY 路径和忽略规则 |
| 容器出现权限错误 | 容器用的 echomind 非 root 用户无法访问挂载路径 | 检查挂载目录权限和目标路径 |

常用排查命令：

~~~bash
docker compose config
docker compose ps
docker compose logs --tail=100 echomind
docker compose build echomind
docker compose up -d echomind
~~~

config 可以检查 Compose 最终配置；ps 查看服务状态和健康情况；logs 看启动报错；build 重新构建镜像；up -d 按新配置启动或替换服务。

## 面试回答

### Docker 在项目里解决什么问题？

> Dockerfile 把 EchoMind 应用和 Python 依赖构建成镜像，Compose 再按配置启动 FastAPI、Redis、ChromaDB、OpenJev、Prometheus 和 Nginx。Compose 网络让服务通过名字互相访问，健康检查管理启动依赖，命名卷保存 Redis、ChromaDB 和 Prometheus 数据。这样开发和部署可以使用一致的应用依赖与服务拓扑，密钥和持久数据由运行环境单独提供。

### README 和 Docker 有什么区别？

> README 是给人看的安装和使用说明；Dockerfile 是可执行的镜像构建配方，Compose 是多服务的启动和连接配置。README 可以指导开发者使用 Docker，但本身不会构建镜像或隔离应用运行环境。

### 为什么 Redis 数据放 volume，应用代码放镜像？

> 镜像适合版本化、可替换的应用代码和依赖；Redis 和 ChromaDB 的数据需要跨容器重建保存，所以放在独立卷里。升级应用镜像时保留数据卷，可以替换应用进程，同时继续读取原来的会话和向量数据。

## 参考资料

- [Dockerfile：构建指令和 Python 示例](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile)
- [Compose 网络：服务名发现和容器通信](https://docs.docker.com/compose/how-tos/networking/)
- [Docker Volumes：持久化容器数据](https://docs.docker.com/engine/storage/volumes/)
- [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
