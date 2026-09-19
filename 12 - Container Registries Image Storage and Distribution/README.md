# Docker Mastery 第12章 Container Registries 镜像仓库 课程笔记
> 本章是**高优先级重点章节**，前面咱们聊过，属于面试高频、工程必备。
> 主题：镜像仓库（Registry）原理、Docker Hub、自建私有仓库、垃圾回收 + 和Swarm配合使用。

## 视频分段拆解
### 001 Docker Hub Digging Deeper（Docker Hub深入）
1. Docker Hub 是Docker官方公共镜像仓库，存放公共基础镜像（ubuntu、postgres、nginx等）
2. 镜像命名规则：`[仓库地址]/[命名空间]/镜像名:标签tag`
   - 简写：`nginx:1.24` 等价 `docker.io/library/nginx:1.24`（默认docker.io就是Docker Hub）
3. 两种镜像：官方镜像（library/xxx）、个人/组织镜像
4. 镜像标签Tag：
   - `latest`只是默认标签，**不代表最新稳定版**，生产不要依赖latest
   - 一个镜像可以打多个tag，多个tag可以指向同一个镜像哈希（digest）
5. 镜像摘要 digest：`sha256:xxx`，是镜像唯一指纹，比tag更可靠，可防止镜像被篡改

### 002 Understanding Docker Registry（理解Registry底层）
1. Docker Registry 是**开源镜像仓库服务**（Docker Hub底层就是基于它）
   - Registry = 存储镜像层文件 + 提供API，供docker pull / push
   - 注意区分：**Registry（仓库服务程序） vs Repository（某个镜像仓库，如myapp） vs Image（镜像）**
2. 分层存储：镜像由多层组成，仓库单独存储每一层；多个镜像共享相同层，节省空间
3. 垃圾回收 GC（Garbage Collection）
   - 作用：删除没有被任何tag引用的旧镜像层，释放磁盘空间
   - ⚠️ 重要：GC执行前**必须先停止registry服务**，不然会损坏镜像数据
4. 镜像仓库镜像可做**镜像镜像站（Mirror）**，加速国内拉取（Docker Hub加速器）

### 003 Run a Private Docker Registry（搭建私有Registry）
实操：用容器直接启动官方`registry:2`镜像，自建私有仓库
```bash
# 最简启动私有仓库命令
docker run -d -p 5000:5000 --name registry registry:2
```
- 本地推送示例：
```bash
docker tag myimage:1.0 localhost:5000/myimage:1.0
docker push localhost:5000/myimage:1.0
```
- 默认http不安全，生产要配置TLS证书；如果内网http，需要在docker守护进程配置`insecure-registries`放行
- 持久化：挂载volume，否则容器删除镜像全部丢失

### 004 Assignment: Secure Docker Registry 作业：安全私有仓库
作业目标：给私有Registry增加安全能力
1. TLS HTTPS加密（https访问）
2. 账号密码认证（基础认证）
> 考点：裸http的私有仓库只能内网测试，生产必须HTTPS+身份认证

### 005 Using Docker Registry With Swarm（Swarm + 私有仓库联合使用）
1. Swarm集群节点拉取镜像逻辑：**所有集群节点都需要能访问镜像仓库**
2. 集群节点拉取私有仓库镜像两种方式：
   - `docker login` 在每个节点登录仓库
   - 在`docker service create`时使用 `--with-registry-auth`，把登录凭证分发到集群节点
3. 完整流程：本地构建镜像 → tag打私有仓库地址 → push到私有registry → swarm创建service，集群节点自动拉取镜像部署

### 006 Third Party Image Registries 第三方镜像仓库
列举市面上托管镜像仓库：
- 阿里云ACR、腾讯TCR、Harbor（企业级私有仓库，比原生registry功能更强：漏洞扫描、项目权限、镜像签名）
> 原生docker registry功能简单；企业生产一般用Harbor

# ✅ 本章核心知识点汇总
1. **概念区分**
    - Registry：镜像仓库服务程序
    - Repository：镜像仓库内某一组镜像（如`localhost:5000/myapp`）
    - Tag：镜像版本标签；Digest(sha256)：镜像唯一不可篡改标识
2. Docker Hub：公共仓库，`latest`标签有坑，生产优先用固定版本号或者digest
3. 自建私有仓库：`registry:2`容器一键启动，默认http，生产要HTTPS+认证+持久化存储
4. 垃圾回收GC：清理无用镜像层，**停机执行**
5. Swarm使用私有仓库关键点：`--with-registry-auth`，集群节点分发登录凭证
6. Harbor是企业级增强版私有仓库，原生registry只适合简单实验

## 📝 核心命令速记
```powershell
# 打标签
docker tag myimg:v1 localhost:5000/myimg:v1
# 推送镜像
docker push localhost:5000/myimg:v1
# 拉取
docker pull localhost:5000/myimg:v1
# swarm创建服务带上仓库认证
docker service create --with-registry-auth ... localhost:5000/myimg:v1
```

# Run a Private Docker Registry Recap 课程总结页笔记
> 这是本节全部实验的完整复盘清单，就是这节课要掌握的全部操作流程

## 1. 启动 registry 容器（最简版本，无数据持久化）
```bash
docker container run -d -p 5000:5000 --name registry registry
```
- `-d`后台运行，端口映射5000，容器名叫registry
- ⚠️ 缺点：容器删除，仓库内镜像全部丢失

## 2. 给已有镜像打标签，推送到私有仓库
```bash
# 打标签，格式必须带上仓库地址:端口
docker tag hello-world 127.0.0.1:5000/hello-world

# 上传镜像到私有仓库
docker push 127.0.0.1:5000/hello-world
```
> 核心：`docker tag`不是复制文件，只是给镜像新增一个别名，push才会上传副本到仓库。

## 3. 删除本地缓存镜像，再从私有仓库拉回来（验证仓库生效）
```bash
# 删除本地hello-world原始标签
docker image remove hello-world

# 删除本地私有仓库版本镜像
docker image remove 127.0.0.1:5000/hello-world

# 从私有仓库重新拉取
docker pull 127.0.0.1:5000/hello-world
```
实验目的：**证明镜像已经保存在Registry仓库，本地删掉镜像，依然可以取回**。

## 4. 使用bind mount绑定挂载重建registry（持久化数据，重点！）
```bash
docker container run -d -p 5000:5000 --name registry -v $(pwd)/registry-data:/var/lib/registry registry
```
- `-v $(pwd)/registry-data:/var/lib/registry`
- 把仓库镜像数据挂载到宿主机当前目录的`registry-data`文件夹
- 就算删除registry容器，宿主机文件夹里镜像数据还在，重建仓库可以复用

# 📖 本节核心考点总结
1. 私有仓库本质：一个Docker容器，用来存放镜像
2. 推镜像前置条件：镜像tag必须带上`仓库IP:端口`
3. `localhost/127.0.0.1`访问http仓库不需要改docker安全配置；跨机器IP访问必须配置`insecure-registries`
4. 两种存储：
   - 不带 `-v`：数据存在容器内部，容器删数据丢失
   - `-v`绑定挂载：数据存在宿主机目录，实现持久化
5. 实验验证思路：push → 删除本地镜像 → pull取回，确认仓库正常工作

