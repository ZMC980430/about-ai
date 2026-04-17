## 一、先搞清楚：什么场景该用Docker？什么场景不该用？
- ### 1. 强烈推荐使用Docker的核心场景（项目开发首选）
  Docker的核心价值是**「一次构建，到处运行」**，解决环境不一致、依赖冲突、部署效率低的核心痛点，以下场景收益最高：
	- **微服务/模块化项目开发**：每个业务模块、中间件独立封装，避免依赖冲突，支持独立扩缩容与迭代
	- **多环境一致性保障**：开发、测试、预发、生产环境使用同一套镜像，彻底解决「我本地能跑，线上跑不了」的问题
	- **CI/CD 自动化流水线**：与Jenkins、GitLab CI等工具无缝集成，实现代码提交→镜像构建→自动部署的全流程自动化
	- **多版本/多实例隔离**：同一中间件（如MySQL、Redis）的不同版本可在同一台机器并行运行，互不干扰
	- **跨平台部署**：一套镜像可在x86/ARM架构的Windows、Mac、Linux服务器上无差别运行
	- **快速弹性扩缩容**：容器秒级启动，适合流量波动大的业务，可快速批量复制实例
- ### 2. 不建议/不适合使用Docker的场景
	- **极致性能要求的底层应用**：如数据库集群（核心交易库）、高性能计算、内核级开发，Docker的轻量级虚拟化仍有极微的性能损耗
	- **单进程极简应用**：仅一个静态二进制文件、无任何依赖的程序，直接部署即可，Docker反而增加冗余
	- **强安全隔离的敏感业务**：容器共享宿主机内核，相比虚拟机隔离性更弱，涉密、高安全等级业务需额外加固
	- **大量状态数据持久化的重型应用**：如分布式存储、大数据集群，容器化会增加数据管理复杂度，非必要不容器化
- ---
- ## 二、Docker 核心概念&环境准备
- ### 1. 5个核心概念（1分钟搞懂）
  | 概念 | 通俗解释 | 核心作用 |
  |------|----------|----------|
  | 镜像（Image） | 项目的「只读安装包」，包含代码、运行环境、依赖、配置文件 | 容器的模板，一个镜像可创建无数个容器 |
  | 容器（Container） | 镜像运行起来的「独立实例」，有自己的文件系统、网络、进程空间 | 项目的实际运行载体，轻量级虚拟机 |
  | Dockerfile | 镜像的「构建脚本」，定义了镜像的制作步骤、依赖、启动命令 | 实现镜像构建的标准化、可复现 |
  | Docker Compose | 多容器编排工具，通过yml文件一键定义、启动、管理多个关联容器 | 解决多容器依赖、网络、启停顺序的管理问题 |
  | 镜像仓库（Registry） | 镜像的「代码仓库」，用于存储、分发镜像 | 实现镜像的跨环境迁移、团队共享 |
- ### 2. 全平台环境快速安装
  开发环境优先安装 Docker Desktop（内置Docker Engine、Compose、可视化界面），生产环境安装Docker Engine。
	- **Windows/Mac**：直接从[Docker官网](sslocal://flow/file_open?url=https%3A%2F%2Fwww.docker.com%2Fproducts%2Fdocker-desktop%2F&flow_extra=eyJsaW5rX3R5cGUiOiJjb2RlX2ludGVycHJldGVyIn0=)下载安装包，一键安装，启动即可使用
	- **Linux（CentOS/Ubuntu）**：执行官方一键安装脚本
	  ```bash
	  # 官方一键安装脚本
	  curl -fsSL https://get.docker.com | bash
	  # 启动Docker并设置开机自启
	  systemctl start docker
	  systemctl enable docker
	  # 验证安装成功
	  docker --version
	  docker compose version
	  ```
	  
	  ---
- ## 三、项目Docker化核心流程：从创建镜像到运行容器
  以Java SpringBoot项目（贴合后端开发场景）为例，完整演示从镜像创建到容器运行的全流程，其他语言项目（Node.js、Python、Go）逻辑完全一致，仅基础镜像和构建步骤不同。
- ### 1. 第一步：编写Dockerfile（镜像构建的核心）
  Dockerfile是镜像构建的唯一依据，必须放在项目根目录，以下是**生产级多阶段构建Dockerfile示例**，兼顾构建效率、镜像体积与安全性。
	- #### 核心指令速览
	  | 指令 | 作用 | 核心注意事项 |
	  |------|------|--------------|
	  | FROM | 指定基础镜像 | 优先用官方精简镜像（alpine/distroless），固定版本号，禁用latest |
	  | COPY | 复制本地文件到镜像 | 区分构建上下文，避免复制冗余文件（如node_modules、target） |
	  | RUN | 镜像构建时执行的命令 | 多条命令合并为一条，减少镜像层数，构建完成后清理缓存 |
	  | WORKDIR | 设置镜像内的工作目录 | 避免频繁cd切换目录，所有相对路径均基于此目录 |
	  | EXPOSE | 声明容器监听的端口 | 仅做声明，不做端口映射，提升Dockerfile可读性 |
	  | ENV | 设置环境变量 | 用于配置注入，避免硬编码配置到镜像中 |
	  | CMD/ENTRYPOINT | 容器启动时执行的命令 | CMD可被启动命令覆盖，ENTRYPOINT固定启动入口，生产环境推荐组合使用 |
	- #### 生产级Dockerfile示例（SpringBoot 3.x）
	  ```dockerfile
	  # 多阶段构建：第一阶段 构建环境（仅用于编译打包，不进入最终镜像）
	  FROM maven:3.9.6-eclipse-temurin-21-alpine AS builder
	  # 设置工作目录
	  WORKDIR /app
	  # 优先复制pom.xml，缓存maven依赖（代码不变时，无需重新下载依赖）
	  COPY pom.xml .
	  RUN mvn dependency:go-offline -B
	  # 复制源代码并打包
	  COPY src ./src
	  RUN mvn clean package -DskipTests -Dmaven.test.skip=true
	  
	  # 多阶段构建：第二阶段 运行环境（最终镜像，仅包含jre和编译好的jar包）
	  FROM eclipse-temurin:21-jre-alpine
	  # 安全加固：创建非root用户运行容器，禁止root权限
	  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
	  # 设置工作目录
	  WORKDIR /app
	  # 从构建阶段复制编译好的jar包到运行镜像
	  COPY --from=builder /app/target/*.jar app.jar
	  # 声明服务端口
	  EXPOSE 8080
	  # 切换到非root用户
	  USER appuser
	  # 容器启动命令
	  ENTRYPOINT ["java", "-jar", "app.jar"]
	  # 可覆盖的JVM参数默认值
	  CMD ["-Xms512m", "-Xmx1g"]
	  ```
	- #### 配套优化文件：.dockerignore
	  作用：排除不需要复制到镜像中的文件，减小镜像体积，避免构建缓存失效，放在项目根目录，示例如下：
	  ```
	  # 编译产物
	  target/
	  # 本地依赖
	  .mvn/
	  # 版本控制
	  .git/
	  .gitignore
	  # 本地配置
	  application-local.yml
	  # 日志、临时文件
	  logs/
	  temp/
	  *.log
	  # IDE文件
	  .idea/
	  .vscode/
	  ```
- ### 2. 第二步：构建镜像
  在Dockerfile所在目录（项目根目录）执行构建命令，生成可复用的镜像：
	- ```bash
	  # 基础构建命令
	  # -t：给镜像打标签，格式为 仓库地址/项目名/镜像名:版本号，版本号必须固定，禁用latest
	  docker build -t my-springboot-app:v1.0.0 .
	  
	  # 进阶构建：指定构建参数、不使用缓存
	  docker build --no-cache --build-arg SPRING_PROFILES_ACTIVE=prod -t my-springboot-app:v1.0.0 .
	  ```
	  
	  构建完成后，可通过 `docker images` 查看已生成的镜像。
- ### 3. 第三步：运行容器（核心参数详解）
  镜像构建完成后，通过`docker run`命令启动容器，核心参数必须掌握，以下是完整的生产级启动命令，附带参数详解：
	- ```bash
	  docker run -d \
	  # 容器名称，全局唯一，用于容器间通信、管理
	  --name springboot-app \
	  # 端口映射：宿主机端口:容器内端口，外部通过宿主机IP:8080访问服务
	  -p 8080:8080 \
	  # 环境变量注入：配置文件、运行参数，无需修改镜像即可切换配置
	  -e SPRING_PROFILES_ACTIVE=prod \
	  -e MYSQL_HOST=mysql-container \
	  -e MYSQL_PORT=3306 \
	  # 数据持久化：宿主机目录/数据卷:容器内目录，容器删除数据不丢失
	  -v ./logs:/app/logs \
	  -v ./config:/app/config \
	  # 资源限制：避免容器占用宿主机全部资源，生产环境必须设置
	  --memory 1g \
	  --cpus 1 \
	  # 网络配置：加入自定义网桥，实现与其他容器通信
	  --network app-network \
	  # 容器重启策略：宿主机重启、容器异常退出时自动重启，生产环境必须设置
	  --restart always \
	  # 镜像名:版本号
	  my-springboot-app:v1.0.0 \
	  # 覆盖CMD默认参数，自定义JVM参数
	  -Xms1g -Xmx2g
	  ```
	- #### 核心管理命令
	  ```bash
	  # 查看运行中的容器
	  docker ps
	  # 查看所有容器（包括停止的）
	  docker ps -a
	  # 查看容器日志（排查问题必备）
	  docker logs -f --tail 100 springboot-app
	  # 进入容器内部
	  docker exec -it springboot-app /bin/sh
	  # 停止/启动/重启容器
	  docker stop springboot-app
	  docker start springboot-app
	  docker restart springboot-app
	  # 删除容器（停止后才能删除，加-f强制删除运行中的容器）
	  docker rm -f springboot-app
	  # 删除镜像
	  docker rmi my-springboot-app:v1.0.0
	  ```
- ### 4. 数据持久化核心说明
  容器的文件系统是临时的，容器删除后，所有写入容器内的数据都会丢失，**必须通过挂载实现数据持久化**，两种挂载方式适用场景如下：
	- **绑定挂载（Bind Mount）**：`-v 宿主机绝对路径:容器内路径`，适合开发环境，挂载本地代码、配置文件，修改本地文件容器内实时生效
	- **Docker 数据卷（Volume）**：`-v 卷名:容器内路径`，Docker官方推荐，适合生产环境，用于持久化MySQL、Redis等中间件数据，由Docker统一管理，性能更高、隔离性更好
	  ```bash
	  # 创建数据卷
	  docker volume create mysql-data
	  # 挂载数据卷到MySQL容器
	  docker run -d -v mysql-data:/var/lib/mysql mysql:8.0
	  ```
	  
	  ---
- ## 四、容器功能拆分的核心决策方法论（一个容器该做什么？）
  新手最容易踩的坑：把所有服务（Nginx+Java+MySQL+Redis）都塞到一个容器里，完全违背Docker的设计理念，以下是标准化的拆分决策体系。
- ### 1. 核心黄金原则：单一职责
  **一个容器只运行一个核心进程，只负责一个独立的业务/技术功能**，进程的生命周期与容器完全绑定，主进程退出，容器就应该停止。
- ### 2. 拆分决策的4个核心维度
  | 拆分维度 | 判断标准 | 拆分示例 |
  |----------|----------|----------|
  | 业务模块边界 | 不同的业务域、独立的迭代节奏，拆分到不同容器 | 订单服务、用户服务、支付服务，各自一个容器 |
  | 技术栈/依赖 | 不同的开发语言、运行环境、依赖版本，必须拆分 | Java后端、Python数据分析、Node.js前端，各自一个容器 |
  | 扩缩容需求 | 流量压力不同、需要独立扩容的模块，必须拆分 | 秒杀服务流量大，单独拆分为容器，可独立扩容，不影响其他服务 |
  | 生命周期/故障隔离 | 不同的启停频率、故障影响范围，必须拆分 | MySQL、Redis等中间件，与业务服务拆分，业务服务重启不影响中间件；Nginx与后端服务拆分，Nginx故障不影响后端业务运行 |
- ### 3. 拆分决策Checklist（5个问题帮你判断）
  只要有一个问题的答案是「是」，就应该拆分为独立容器：
	- 1. 这个功能是否需要和其他功能独立迭代、单独发布？
	- 2. 这个功能是否需要独立扩缩容，和其他模块的流量压力不同？
	- 3. 这个功能故障后，是否需要不影响其他功能的正常运行？
	- 4. 这个功能是否使用了和其他模块不同的技术栈、依赖版本？
	- 5. 这个功能是否是通用组件，会被多个其他模块复用？
- ### 4. 正例&反例
	- ❌ 反例：一个容器内同时运行Nginx、SpringBoot、MySQL、Redis，所有功能耦合
		- 问题：任何一个组件故障，整个容器崩溃；无法单独扩容SpringBoot；更新代码需要重启整个容器，包括MySQL，导致业务中断
	- ✅ 正例：拆分为4个独立容器
	  1. Nginx容器：负责反向代理、负载均衡、静态资源服务
	  2. SpringBoot业务容器：负责核心业务逻辑
	  3. MySQL容器：负责数据存储
	  4. Redis容器：负责缓存、会话存储
		- 优势：独立扩缩容、故障隔离、独立迭代、运维便捷
- ### 5. 特殊情况说明
  仅当两个进程**强绑定、同生共死、无独立扩缩容需求**时，可在一个容器内运行多个进程（需用supervisor等进程管理工具），比如：
	- 业务主进程 + 日志采集边车（与主进程强绑定，不可独立部署）
	- 代理进程 + 业务进程，二者必须共存才能提供服务
	  除此之外，生产环境严格遵循「一容器一进程」原则。
- ---
- ## 五、容器间通信全方案（单机+集群）
  容器间通信的核心是**Docker网络**，Docker提供了多种网络模式，适配不同的开发、生产场景，以下按优先级排序讲解。
- ### 前置说明：避坑提醒
  1. 不推荐使用默认的`bridge`网桥：默认网桥无DNS解析能力，容器间只能通过IP互相访问，IP重启后会变化，维护成本极高
  2. 不推荐使用`--link`指令：已被官方废弃，功能有限，无法动态更新
  3. 90%的开发&单机部署场景，首选**自定义桥接网络**；多机集群场景，首选**Overlay网络/CNI网络插件**
- ### 1. 单机容器通信3种主流方案（开发/单机部署首选）
	- #### 方案1：自定义桥接网络（90%场景首选，官方推荐）
	  collapsed:: true
	  **核心优势**：自带DNS解析，同一网络内的容器可直接通过「容器名/服务名」互相访问，无需固定IP，隔离性好，配置简单。
		- ##### 完整操作步骤
		  ```bash
		  # 1. 创建自定义桥接网络（只需创建一次）
		  docker network create app-network
		  
		  # 2. 启动容器时，加入该网络（--network 指定网络名）
		  # 启动MySQL容器，加入app-network，容器名为mysql-container
		  docker run -d \
		  --name mysql-container \
		  --network app-network \
		  -e MYSQL_ROOT_PASSWORD=123456 \
		  -v mysql-data:/var/lib/mysql \
		  mysql:8.0
		  
		  # 启动Redis容器，加入app-network，容器名为redis-container
		  docker run -d \
		  --name redis-container \
		  --network app-network \
		  redis:7.2-alpine
		  
		  # 启动SpringBoot容器，加入同一个app-network
		  docker run -d \
		  --name springboot-app \
		  --network app-network \
		  -p 8080:8080 \
		  -e MYSQL_HOST=mysql-container \
		  -e MYSQL_PORT=3306 \
		  -e REDIS_HOST=redis-container \
		  -e REDIS_PORT=6379 \
		  my-springboot-app:v1.0.0
		  ```
		- ##### 核心说明
			- 同一自定义网络内的容器，所有端口互相开放，无需做端口映射；只有需要对外提供服务的容器，才需要用`-p`做端口映射
			- 容器内访问其他容器，直接用「容器名:容器内端口」即可，比如SpringBoot配置文件中，数据库地址写`mysql-container:3306`，无需写IP
			- 不同自定义网络之间的容器默认隔离，无法互相访问，如需互通，可通过`docker network connect`将容器加入多个网络
	- #### 方案2：Host网络模式（极致性能场景）
	  collapsed:: true
	  **核心原理**：容器直接共享宿主机的网络栈，容器内的端口直接绑定宿主机的端口，无需端口映射，网络性能无损耗。
	  ```bash
	  # 启动容器时指定 --network host
	  docker run -d \
	  --name springboot-app \
	  --network host \
	  --restart always \
	  my-springboot-app:v1.0.0
	  ```
	  **适用场景**：对网络性能要求极高的场景，如网关、压测工具、高并发代理服务
	  **注意事项**：
		- 端口冲突：容器内的端口直接占用宿主机端口，无法重复绑定
		- 隔离性差：打破了容器的网络隔离，生产环境谨慎使用
		- 仅支持Linux系统，Windows/Mac的Docker Desktop不支持原生host网络
	- #### 方案3：Container 网络模式（共享网络栈）
	  **核心原理**：多个容器共享同一个容器的网络栈，共用同一个IP和端口，互相之间通过`127.0.0.1`访问。
	  ```bash
	  # 先启动一个主容器
	  docker run -d --name main-container nginx:alpine
	  # 启动其他容器，共享主容器的网络栈
	  docker run -d --name sidecar-container --network container:main-container curlimages/curl
	  ```
	  **适用场景**：边车模式（Sidecar），比如日志采集、链路追踪、代理转发，需要和主容器共享网络的场景
	  **注意事项**：主容器停止，所有共享网络的容器都会失去网络连接
- ### 2. 多机集群容器通信方案（生产集群部署）
  当容器分布在多台服务器上时，单机网络无法满足需求，需使用跨主机网络方案：
  1. **Docker Swarm 原生 Overlay 网络**：Docker官方自带的集群编排方案，无需额外安装插件，一键创建跨主机Overlay网络，自带DNS解析、负载均衡，适合中小规模集群，学习成本低
  2. **Kubernetes CNI 网络插件**：大规模生产集群首选，常用插件有Calico、Flannel、Cilium，提供跨节点的容器网络互通、网络策略、安全隔离，适配大规模微服务集群
- ### 3. 实战示例：Docker Compose 一键实现多容器通信
  开发环境中，Docker Compose是多容器管理的首选工具，自动创建自定义网络，无需手动执行网络命令，一键启动所有关联容器。
	- 在项目根目录创建`docker-compose.yml`文件，内容如下：
	  collapsed:: true
		- ```yaml
		  # Compose文件版本，适配Docker 20+版本
		  version: '3.8'
		  
		  # 服务定义：每个服务对应一个容器
		  services:
		  # MySQL服务
		  mysql:
		    image: mysql:8.0
		    container_name: mysql-container
		    # 环境变量
		    environment:
		      MYSQL_ROOT_PASSWORD: 123456
		      MYSQL_DATABASE: app_db
		    # 数据持久化
		    volumes:
		      - mysql-data:/var/lib/mysql
		      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
		    # 重启策略
		    restart: always
		    # 健康检查，确保MySQL启动完成后，再启动依赖它的服务
		    healthcheck:
		      test: ["CMD", "mysqladmin", "ping", "-uroot", "-p123456"]
		      interval: 5s
		      timeout: 10s
		      retries: 5
		  
		  # Redis服务
		  redis:
		    image: redis:7.2-alpine
		    container_name: redis-container
		    restart: always
		    volumes:
		      - redis-data:/data
		  
		  # SpringBoot业务服务
		  app:
		    # 基于当前目录的Dockerfile构建镜像
		    build: .
		    image: my-springboot-app:v1.0.0
		    container_name: springboot-app
		    # 端口映射，仅对外暴露的服务需要
		    ports:
		      - "8080:8080"
		    # 环境变量，直接用服务名(mysql/redis)作为主机地址访问
		    environment:
		      SPRING_PROFILES_ACTIVE: prod
		      MYSQL_HOST: mysql
		      MYSQL_PORT: 3306
		      MYSQL_USER: root
		      MYSQL_PASSWORD: 123456
		      REDIS_HOST: redis
		      REDIS_PORT: 6379
		    # 依赖关系：先启动mysql、redis，再启动app
		    depends_on:
		      mysql:
		        condition: service_healthy
		      redis:
		        condition: service_started
		    restart: always
		    # 资源限制
		    deploy:
		      resources:
		        limits:
		          cpus: '1'
		          memory: 1G
		  
		  # 数据卷定义，统一管理持久化数据
		  volumes:
		  mysql-data:
		  redis-data:
		  
		  # 网络定义：Compose会自动创建一个自定义网桥，所有服务自动加入该网络
		  # 无需手动创建，默认自带DNS解析，服务之间可通过服务名互相访问
		  networks:
		  default:
		    name: app-network
		  ```
	- #### Compose核心操作命令
	  collapsed:: true
		- ```bash
		  # 一键构建镜像+启动所有容器（后台运行）
		  docker compose up -d --build
		  
		  # 查看所有服务运行状态
		  docker compose ps
		  
		  # 查看服务日志
		  docker compose logs -f --tail 100 app
		  
		  # 停止所有服务
		  docker compose stop
		  
		  # 停止并删除所有容器、网络（保留数据卷）
		  docker compose down
		  
		  # 停止并删除所有容器、网络、数据卷（谨慎使用，会清空所有持久化数据）
		  docker compose down -v
		  ```
		  
		  ---
- ## 六、镜像迁移与跨环境部署全流程
  Docker的核心优势就是跨环境一致性，镜像迁移主要分为**离线迁移**和**在线迁移**两种方案，适配不同的网络环境。
- ### 1. 方案1：离线迁移（导出/导入镜像包，无网络环境适用）
  collapsed:: true
  适用于内网隔离、无公网的服务器环境，通过镜像压缩包实现跨机器迁移。
	- ```bash
	  # 1. 开发机/构建机：将镜像导出为tar压缩包
	  # 可同时导出多个镜像到一个压缩包
	  docker save -o my-app-images.tar my-springboot-app:v1.0.0 mysql:8.0 redis:7.2-alpine
	  
	  # 2. 将tar包拷贝到目标服务器（U盘、scp、内网传输等）
	  
	  # 3. 目标服务器：导入镜像包
	  docker load -i my-app-images.tar
	  
	  # 4. 验证导入成功，即可正常启动容器
	  docker images
	  ```
- ### 2. 方案2：在线迁移（私有镜像仓库，正规项目首选）
  适用于有网络的环境，是企业级项目的标准方案，通过镜像仓库实现镜像的存储、分发、版本管理，类似Git代码仓库。
	- #### 常用镜像仓库
	  collapsed:: true
		- 公共仓库：Docker Hub、阿里云ACR、腾讯云TCR、华为云SWR
		- 私有自建：Harbor（企业级首选，支持权限管理、镜像扫描、同步）
	- #### 完整操作步骤
	  collapsed:: true
		- ```bash
		  # 1. 给镜像打标签，格式为：仓库地址/命名空间/镜像名:版本号
		  # 示例：阿里云ACR仓库
		  docker tag my-springboot-app:v1.0.0 registry.cn-beijing.aliyuncs.com/my-project/my-springboot-app:v1.0.0
		  
		  # 2. 登录镜像仓库
		  docker login registry.cn-beijing.aliyuncs.com -u 用户名 -p 密码
		  
		  # 3. 推送镜像到仓库
		  docker push registry.cn-beijing.aliyuncs.com/my-project/my-springboot-app:v1.0.0
		  
		  # 4. 目标服务器登录仓库，拉取镜像
		  docker login registry.cn-beijing.aliyuncs.com -u 只读用户名 -p 密码
		  docker pull registry.cn-beijing.aliyuncs.com/my-project/my-springboot-app:v1.0.0
		  
		  # 5. 目标服务器直接用拉取的镜像启动容器即可
		  ```
	- #### 镜像版本管理规范
	  collapsed:: true
		- 禁止使用`latest`标签：latest是浮动标签，每次推送都会覆盖，会导致环境不一致，无法回滚
		- 推荐版本号格式：`v主版本.次版本.修订号`（如v1.0.0、v1.0.1），或`git commit哈希值`，可追溯、可回滚
		- 环境区分：可通过标签后缀区分环境，如v1.0.0-prod、v1.0.0-test
- ### 3. 生产环境部署方案选型
  | 部署方案 | 适用场景 | 优势 | 劣势 |
  |----------|----------|------|------|
  | Docker Compose 单机部署 | 中小项目、单服务器、测试环境 | 简单易用、零额外依赖、一键启停、维护成本低 | 不支持多机集群、自动扩缩容能力弱 |
  | Docker Swarm 集群部署 | 中小规模集群、多节点、轻量微服务 | Docker原生支持、无需额外组件、学习成本低、兼容Compose配置 | 大规模集群能力弱于K8s、生态不够丰富 |
  | Kubernetes（K8s）集群部署 | 大规模微服务、企业级生产环境、高可用要求 | 生态完善、强大的扩缩容、自愈、滚动更新能力、适配复杂业务 | 学习成本高、运维复杂、需要专业的团队维护 |
- ---
- ## 七、开发&生产环境最佳实践&避坑指南
- ### 1. 开发环境最佳实践
	- 1. **代码热更新**：通过绑定挂载将本地代码挂载到容器内，修改代码无需重新构建镜像，比如前端项目挂载src目录，Java项目挂载配置文件
	- 2. **统一开发环境**：团队所有成员使用同一套Docker Compose配置，无需在本地安装MySQL、Redis等中间件，一键启动开发环境
	- 3. **构建缓存优化**：Dockerfile中优先复制依赖配置文件（pom.xml、package.json），再复制源代码，充分利用构建缓存，提升构建速度
	- 4. **多架构兼容**：开发用Mac M系列（ARM架构），生产用Linux x86架构，使用`docker buildx`构建多架构镜像，避免架构不兼容问题
- ### 2. 生产环境核心最佳实践
	- 1. **安全加固**
		- 禁止使用root用户运行容器，所有容器必须使用非root用户
		- 优先使用精简基础镜像（alpine、distroless），减少镜像体积和漏洞面
		- 镜像构建完成后进行漏洞扫描（Trivy、Clair），禁止存在高危漏洞的镜像上线
		- 严格控制容器权限，禁止使用`--privileged`特权模式
	- 2. **稳定性保障**
		- 所有容器必须设置CPU、内存资源限制，避免单个容器占用宿主机全部资源
		- 必须设置重启策略`--restart always`，保障异常退出时自动恢复
		- 必须实现数据持久化，禁止将业务数据、日志写入容器可写层
		- 配置健康检查，Docker自动检测容器状态，异常时自动重启
	- 3. **可观测性**
		- 容器日志统一输出到stdout/stderr，禁止写入容器内的日志文件，通过ELK、Loki等工具统一采集存储
		- 容器内暴露 metrics 指标，通过Prometheus采集监控，保障可观测性
	- 4. **不可变基础设施**
		- 镜像一旦构建完成，不可修改，配置通过环境变量、配置挂载注入，禁止进入容器修改配置、代码
		- 版本更新通过构建新镜像、替换容器实现，而非修改现有容器
- ### 3. 高频踩坑避坑指南
	- 1. **容器启动后秒退**：核心原因是容器内没有前台运行的主进程，CMD/ENTRYPOINT必须以前台方式运行，不能后台启动
	- 2. **环境不一致**：使用latest标签导致镜像版本混乱，必须固定镜像版本号
	- 3. **数据丢失**：未做数据持久化，容器删除后数据丢失，所有需要持久化的数据必须挂载到宿主机
	- 4. **端口冲突**：宿主机端口被占用，启动容器时提示端口绑定失败，通过`netstat -tulpn`查看占用端口的进程，更换映射端口
	- 5. **容器间无法通信**：未加入同一个自定义网络，或使用了容器IP而非容器名访问，IP重启后会变化，必须使用容器名/服务名访问
	- 6. **镜像体积过大**：未使用多阶段构建，把构建环境、源码、缓存都打包到了最终镜像中，必须使用多阶段构建，仅保留运行所需的文件
	- 7. **时区不一致**：容器内时区为UTC，与宿主机相差8小时，启动容器时通过`-v /etc/localtime:/etc/localtime:ro`挂载宿主机时区，或在Dockerfile中设置时区