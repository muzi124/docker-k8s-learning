# Docker Swarm 章节笔记（09‑Swarm Intro and Creating a 3‑Node Swarm Cluster）

> 
> 课程：Docker Mastery，主题：**Docker Swarm 集群编排，搭建3节点Swarm集群**
> Compose 是单机多容器；**Swarm 是多机器集群编排**，Docker官方原生集群工具。

## 一、Swarm 是什么

1. Docker Swarm = Docker内置的集群模式，不需要额外安装软件。
2. 把**多台Docker主机（物理机/虚拟机/云服务器）组成一个集群**，统一管理。
3. 概念区分
   - **Manager节点（管理节点）**：集群控制中心，调度任务、保存集群状态，使用Raft一致性协议做多节点数据同步。可以多个Manager实现高可用。
   - **Worker节点（工作节点）**：只负责跑容器业务，接收Manager分配过来的任务。
4. 服务（service）：Swarm的最小部署单元，定义容器镜像、副本数量、网络、卷；Swarm自动把副本分发到集群不同机器。
5. Raft共识算法：Manager多节点之间保证集群数据一致，选主，故障自动切换。

> 
> Compose：单机器，本地开发测试；Swarm：多机器集群，生产部署。

## 二、核心命令

```
# 初始化swarm，把本机变成第一个manager节点
docker swarm init

# 获取worker节点加入集群的token
docker swarm join-token worker

# 获取manager节点加入集群的token
docker swarm join-token manager

# 其他机器执行，加入集群
docker swarm join --token <token> 管理节点IP:2377

# 查看集群节点列表
docker node ls

# 创建服务（类似compose里service，但跑在多机器集群）
docker service create

# 列出集群服务
docker service ls

# 扩容副本数量
docker service scale

# 部署compose文件到swarm集群：docker stack deploy
docker stack deploy -c docker-compose.yml myapp
```

### 重要端口（课程文档 005 Docker‑Swarm‑Firewall‑Ports）

Swarm集群需要防火墙开放端口：

- `2377/tcp`：集群管理通信
- `7946 tcp/udp`：节点发现
- `4789/udp`：overlay覆盖网络（集群跨机器容器网络）

> 
> 云服务器（DigitalOcean等）必须放行这些端口，否则节点无法通信。

## 三、本章节实验目标：搭建3节点集群

3台机器：1个Manager管理节点 + 2个Worker工作节点，组成3‑Node Swarm Cluster。
实验步骤流程：

1. 准备3台虚拟机/云主机，每台安装Docker。
2. 在第一台机器执行 `docker swarm init`，初始化集群，生成join token。
3. 另外两台机器使用 `docker swarm join`，拿着token加入集群。
4. `docker node ls` 查看3个节点状态。
5. 创建service，测试副本调度，观察容器被自动分配到不同机器。
6. 演示stack部署：把compose文件部署到swarm集群。

> 
> 课程里面演示DigitalOcean云平台，使用SSH密钥登录云主机。

## 四、Overlay网络（集群网络）

- 单机用bridge网桥；**跨机器集群使用 overlay网络**。
- 不同机器上的容器，可以直接通过服务名字互相访问，和compose服务名DNS类似，但跨主机。

## 五、Compose vs Swarm Stack

1. `docker compose up`：单机运行，不支持多机器。
2. `docker stack deploy -c compose.yml`：**把同一个yaml部署到Swarm多机器集群**。> 
> 注意：compose里面部分字段在stack部署会被忽略（比如 `build` 构建镜像，swarm只能使用已经存在的镜像）。

## 六、Raft协议关键点（幻灯片重点）

- Manager节点内部使用Raft保证集群元数据一致。
- 高可用Manager建议奇数个（1、3、5），方便投票选主。
- 多数Manager存活，集群才可用。

## 七、本章节学习重点总结

1. Swarm 是Docker原生集群，用于多服务器统一调度容器，Compose用于单机。
2. 节点角色：Manager（管理调度）、Worker（执行业务容器）。
3. 搭建集群流程：`swarm init` → 获取token → 其他节点 `swarm join`。
4. 防火墙必须开放Swarm专用端口，否则集群节点通信失败。
5. Service是Swarm部署单元，可以做副本扩容；`stack deploy`复用compose yaml部署集群。
6. Overlay网络实现跨主机容器互通；Manager使用Raft共识保证集群高可用。

## 八、现实现状补充

> 
> 现实工作中，现在企业大多使用K8s，Docker Swarm用的少；但Udemy课程学习目的是理解集群编排思想：**多机器、副本调度、服务、集群网络、高可用**，这些概念可以迁移到K8s。

### 易踩坑点

1. 云服务器安全组不放行swarm端口，节点加入集群超时。
2. stack deploy不能使用build，镜像必须提前构建推送。
3. Manager节点建议奇数台，Raft投票机制。
4. swarm init输出的token要保存，节点加入集群必须使用。

