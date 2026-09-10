1. **前置：Swarm 需求与环境说明**
   - 介绍 Swarm 适用场景、环境要求；提到 Play-with-Docker 平台的 Jan 2025 版本 bug 提示。
2. **基础概念与命令（DM-S08-Commands.txt）**
   - **Node 节点**：分为 Manager 管理节点（Raft 共识，维护集群状态，调度任务）、Worker 工作节点（只跑容器任务）。Manager 多节点可做集群高可用，同一时间只有 1 个 Leader。
   - **Service 服务**：Swarm 的最小部署单元，定义容器镜像、副本数、资源；
   - **Task 任务**：Service 调度后，在节点上实际运行的容器实例。
   - 两种服务模式：
     - Replicated（默认）：指定副本数量，集群自动分散调度；
     - Global：每个节点都运行 1 个实例（日志采集、监控代理）。
3. **03 跨节点扩容：Overlay 覆盖网络**
   - Overlay 网络：Swarm 专属跨主机虚拟网络，实现不同机器上容器互通，支持加密；
   - Swarm 初始化自动创建`ingress` overlay 网络，用于对外暴露服务端口；
   - 容器内部 DNS：使用服务名互相访问，不用硬编码 IP。
4. **04 路由网格 Routing Mesh**
   - Routing Mesh（路由网格）：Swarm 核心负载均衡能力。任意节点 IP + 端口，都可以访问服务，流量自动转发到对应任务容器；
   - 不管副本运行在哪台机器，访问集群任意节点端口都能接入；内置四层负载均衡。
5. **06 作业：创建多服务集群（Assignment）**
   - 实操：用 Swarm 部署多服务应用，模拟前后端 / 数据库组合；
6. **07 作业答案：多服务集群部署演示**
   - 演示部署、查看状态、排错；
7. **08 Swarm Stack 与生产部署**
   - Stack：基于 Compose yaml 一次性批量部署多个 Swarm 服务；`docker stack deploy`；
   - `docker stack` vs `docker-compose`：compose 是单机开发，stack 是集群生产；
   - 生产环境最佳实践，滚动更新、故障自动重建。
8. **09 & 10 Secrets：Swarm 密钥管理**
   - Docker Secrets：安全保存密码、密钥、证书这类敏感数据；
   - 密钥不会写进 yaml、不会存环境变量；只挂载到目标容器内存文件`/run/secrets/`；
   - 密钥只下发到需要该密钥的服务容器，限制泄露风险；
   - 实操：创建 secret，在 stack/compose 里引用 secret。