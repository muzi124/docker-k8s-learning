# 课程：Making It Easier with Docker Compose The Multi‑Container Tool
> 课程出自 Docker Mastery，**主题：Docker Compose 多容器编排工具**

## 📖 课程整体学习目标
解决原生 `docker run` 管理多容器的痛点：多个容器要敲大量命令、网络、端口、依赖、启动顺序很难维护、环境难以复现。
学习使用 **docker‑compose.yml(YAML)**，用一个声明式文件定义整套多容器应用，一条命令完成启动、停止、重建、清理整套服务。

## 📝核心知识点笔记
### 1. Docker Compose 基础概念
1. **什么是 Compose**：本地多容器应用编排工具，只用于开发/测试环境，**不用于生产（生产用Swarm/K8s）**。
2. **核心文件：`docker‑compose.yml`（YAML格式）**
    - YAML语法：缩进敏感，对比JSON；官方YAML快速参考
    - 文件版本演进：Compose V1 vs V2，新旧文件格式差异，现在推荐 Compose Specification（V2新版）
3. Compose自动行为：
    - 自动创建**独立网络**，同一个compose内服务可直接用**服务名做主机名**互相访问，不需要IP
    - 自动管理volumes数据卷、端口映射、环境变量

### 2. compose.yml 文件四大核心字段
- `services`：定义各个容器服务（核心）
  - `image`：直接使用镜像；`build`：本地Dockerfile构建镜像
  - `ports`：端口映射
  - `volumes`：挂载数据卷，持久化数据
  - `environment`：环境变量
  - `depends_on`：定义服务依赖，控制**容器启动顺序**；⚠️只保证容器先启动，**不保证程序就绪（数据库完全初始化）**
- `volumes`：顶层声明命名卷，供services引用
- `networks`：自定义网络配置
- `configs/secrets`：配置与敏感信息

### 3. Compose常用命令
```bash
# 启动整套服务（后台 -d）
docker compose up -d
# 查看运行状态
docker compose ps
# 查看日志
docker compose logs -f
# 停止服务，不删除容器、卷
docker compose stop
# 停止并删除容器、网络（保留数据卷）
docker compose down
# down并删除数据卷
docker compose down -v
# 重新build镜像再启动
docker compose up --build
```
> 注意：V1旧版命令为 `docker‑compose`（横杠），新版V2集成到docker cli，`docker compose`（空格）

### 4. 重点坑点与注意事项
1. `depends_on` **只控制容器启动先后，不会等待应用就绪**（比如mysql容器启动成功 ≠ mysql服务可以接收连接），代码层面要做重试逻辑。
2. `links` 是**废弃旧特性，不要使用**，Compose默认网络已经支持服务名DNS解析。
3. Compose适合开发本地环境；**不直接用于生产部署**，生产使用Docker Swarm或者Kubernetes。
4. YAML格式严格，缩进错误会直接解析失败。



## ✨课程总结
> Docker Compose核心价值：**把一堆docker run命令，收敛到一份yaml配置文件**。
1. 用yaml声明整套应用所有容器、网络、存储；
2. 单条命令管理整套应用生命周期；
3. 本地开发快速复现环境，团队共享配置；
4. 局限：本地开发工具，**不是生产集群方案**；`depends_on`仅控制容器启动顺序，不等同服务就绪。