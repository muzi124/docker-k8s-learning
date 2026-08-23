# Docker 持久化存储学习笔记
章节：Persistent Data Volumes, Volumes, Volumes
> 核心问题：**容器是临时的，容器删除，容器内部数据默认全部丢失。持久化就是把数据脱离容器生命周期保存下来**

## 一、3种持久化存储方式
### 1. Volume（Docker管理卷，生产首选）
- 存储位置：Docker管理宿主机文件系统（Linux：`/var/lib/docker/volumes/`），用户不建议直接手动修改该目录文件
- 特点
  - 完全由docker管理，跨平台兼容（Windows/macOS/Linux）
  - 容器删除，**卷数据不会自动删除**
  - 适合：数据库、业务数据
- 分类
  1. 匿名卷：不指定名字，`-v /container/path`，docker自动生成随机id名称，容器删除后需要手动清理
  2. 命名卷：`-v myvol:/container/path`，自定义卷名，推荐生产使用
- 常用命令
```bash
docker volume ls                 # 列出所有卷
docker volume create myvol       # 创建命名卷
docker volume inspect myvol      # 查看卷详情
docker volume rm myvol           # 删除指定卷
docker volume prune              # 清理所有未被容器使用的匿名卷
```
- Dockerfile `VOLUME`指令
```dockerfile
VOLUME ["/var/lib/postgresql/data"]
```
> 注意：Dockerfile VOLUME只能声明容器内路径，**不能指定宿主机路径**；宿主机路径只能在`docker run`时指定。

### 2. Bind Mount 绑定挂载（开发环境首选）
- 原理：直接把**宿主机指定文件夹/文件**映射到容器内部路径
- 语法：`-v /宿主机绝对路径:/容器内路径`
- 特点
  - 宿主机文件修改，容器内立刻生效；容器内修改也会直接改动宿主机文件
  - 路径必须写绝对路径，Windows/macOS要注意路径格式
  - 适合：本地开发，代码热更新（如Jekyll静态网站案例）
- 缺点：强依赖宿主机目录，可移植性差，不适合生产环境。

### 3. tmpfs 内存挂载（临时存储）
- 数据存宿主机内存，不会写入磁盘，容器销毁数据直接消失
- 适合：敏感临时数据，不需要持久化
```bash
docker run --tmpfs /container/path
```

## 二、-v 的两种含义（极易混淆）
1. `-v 卷名:容器路径` → Volume（docker托管卷）
2. `-v 宿主机绝对路径:容器路径` → Bind Mount绑定挂载
> 判断规则：冒号左边是名字=volume；左边是路径(包含`/`)=bind mount

## 三、实战案例（课程里面的例子）
### 案例1：PostgreSQL数据库持久化
- 数据库容器重建、升级镜像，只要挂载同一个命名卷，数据不会丢失
- 作业考点：升级postgres镜像版本，数据库数据保留
> 坑：不要把数据库数据做bind mount，不同操作系统文件权限容易出问题，优先用Volume。

### 案例2：Jekyll静态网站（Bind Mount开发案例）
- 将宿主机项目代码目录bind mount挂载到容器
- 在宿主机修改代码，容器内部服务立刻感知变化，不用重新build镜像，开发效率高。

### 案例3：多容器共享数据
- 多个容器挂载同一个Volume，实现容器之间数据共享。
- 坑点：**文件权限问题**，不同容器内部用户id不一致，会出现读写拒绝。

## 四、常见踩点总结
1. 删除容器不会自动删除Volume，需要手动`docker volume rm`或`prune`清理，长期不清理会占用磁盘。
2. Dockerfile写`VOLUME`，后续镜像继承该指令，无法在子Dockerfile取消该挂载。
3. Bind Mount宿主机不存在目录：docker会自动创建这个文件夹。
4. 权限问题：bind mount在Windows、Linux混合环境经常遇到权限报错，数据库尽量不用bind mount。
5. 不要手动修改`/var/lib/docker/volumes`下的原始文件，容易损坏数据。

## 五、选型速查表
|存储类型|使用场景|环境|
|---|---|---|
|Volume|数据库、业务持久数据|生产环境✅|
|Bind Mount|本地代码开发调试|开发环境✅，生产❌|
|tmpfs|临时敏感数据，不需要落盘|临时场景✅|

## 六、补充
- `--mount`参数：另一种挂载写法，语义更清晰，兼容docker swarm，可替代`-v`。
```bash
# volume示例
docker run --mount type=volume,source=myvol,target=/data
# bind mount示例
docker run --mount type=bind,source=/host/path,target=/container/path
```

