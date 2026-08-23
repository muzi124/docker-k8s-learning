# Docker镜像完整学习笔记（整合版）

## 1. docker pull 下载镜像

### 语法

```
docker pull [镜像名]:[tag标签]
```

- 镜像名：仓库名称，如`nginx`、`mysql`
- tag：版本标签，**不写tag默认拉取`latest`最新版，生产环境禁止使用latest，版本会漂移**

### 示例

```
# 拉取最新nginx
docker pull nginx

# 指定版本（推荐写法）
docker pull nginx:1.27

# 拉取私有仓库镜像，需要先登录docker hub
docker login
docker pull 用户名/镜像名:版本
```

## 2. 本地镜像查看

```
# 查看本地全部镜像
docker images
# 等价简写
docker image ls
```

| 字段 | 说明 |
| --- | --- |
| REPOSITORY | 镜像名称 |
| TAG | 版本标签 |
| IMAGE ID | 镜像唯一ID |
| CREATED | 镜像创建时间 |
| SIZE | 镜像大小 |

## 3. 删除镜像

```
docker rmi 镜像ID
# 强制删除镜像（镜像被容器占用时报错，加-f）
docker rmi -f 镜像ID
# 删除多个镜像，空格分隔ID
docker rmi id1 id2
```

## 4. 搜索镜像（命令行查询Docker Hub）

```
docker search nginx
```

> 
> OFFICIAL=true 代表官方镜像，优先选用，安全性更高

## 5. 镜像离线导出&导入

```
# 导出镜像为tar包
docker save nginx:1.27 -o nginx127.tar

# 从tar包导入镜像
docker load -i nginx127.tar
```

## 6. docker history 查看镜像分层历史

```
# 查看镜像构建分层历史
docker history nginx:latest

# --no‑trunc 不截断输出完整命令，必用参数
docker history --no-trunc nginx:latest
```

### 输出字段

| 列 | 含义 |
| --- | --- |
| IMAGE | 镜像层ID，`<missing>`属于正常现象 |
| CREATED | 层创建时间 |
| CREATED BY | 生成该层执行的指令，对应Dockerfile |
| SIZE | 该分层占用磁盘大小 |
| COMMENT | 注释信息 |

### ⭐  说明

> 
> 从Docker Hub pull下载的镜像，中间分层不会保存完整ID，显示`<missing>`，**不是错误，不影响镜像使用**。
> 本地使用`docker build`自己构建的镜像，所有分层都会显示完整ID。

## 7. Docker镜像分层核心原理

> 
> Docker镜像 = **多层只读层堆叠**，history输出第一行是镜像最上层，最后一行是底层基础系统。

### 分层的作用

1. **层复用，节省磁盘**
多个镜像共用基础层，磁盘只保存一份，下载速度更快。
2. **构建缓存，加速build**
Dockerfile构建镜像，指令没有改动直接复用旧分层，不用重复执行。
3. **容器读写分离**
镜像全部是只读层；容器启动，会在镜像最上层新增一层**可读写容器层**。
容器所有修改（写文件、改配置）全部写在这一层；删除容器，读写层销毁，底层镜像不受任何改动。

### 为什么部分分层SIZE=0B，带有`#(nop)`

`nop = no‑operation`，**不写入磁盘任何文件，仅修改镜像元数据（配置信息），所以占用0字节**。

| Dockerfile指令 | 分层大小 | 说明 |
| --- | --- | --- |
| `CMD / EXPOSE / ENV / LABEL / MAINTAINER` | 0B | 元数据操作，只改配置，不写文件 |
| `RUN / COPY / ADD` | 有实际大小 | 新增、修改磁盘文件，产生数据占用存储 |

> 
> 示例解读截图nginx分层：
> 
> 
> - `EXPOSE 80 443`：0B，**仅仅文档声明端口，不会自动开放端口，端口映射需要`docker run -p`**
> - `CMD ["nginx","-g","daemon off;"]`：0B，设置容器默认启动命令
> - `RUN apt-key adv ...`：58.8MB，安装软件，写入大量文件
> - `ln -sf /dev/stdout /var/log/nginx/access.log`：22B，创建日志软链接，让docker logs可以收集nginx日志

### Dockerfile写代码避坑（分层优化）

每一条RUN会生成一层镜像，尽量合并RUN，减少镜像层数：

```
# ❌ 不好，产生两层
RUN apt update
RUN apt install nginx

# ✅推荐，合并为一层
RUN apt update && apt install -y nginx
```

## 8. 镜像加速器（国内下载慢必配）

编辑 `/etc/docker/daemon.json`

```
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```

生效命令：

```
systemctl daemon-reload
systemctl restart docker
```

## 9. 镜像启动容器（扩展）

镜像下载完成，通过`run`基于镜像创建运行容器

```
# -d后台运行，-p宿主机端口:容器内部端口
docker run -d -p 8080:80 nginx:1.27
```

## 📝面试速记

1. `docker pull`下载镜像；`docker images`查看本地镜像；`docker rmi`删除镜像
2. `docker history`查看镜像分层；`<missing>`是拉取远程镜像正常现象
3. `#(nop)`元数据指令ENV/EXPOSE/CMD，size=0，不会写磁盘
4. 镜像全部只读；容器新增一层读写层，容器删除不会破坏镜像
5. EXPOSE只是提示，不会自动开放端口；生产环境镜像tag禁止使用latest

# docker image tag + push 解析
## 1、`docker image tag` 命令
```bash
docker image tag nginx bretfisher/nginx
```
帮助文档格式：
```
docker image tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```
- **SOURCE_IMAGE**：源镜像（本地已经存在的镜像）
- **TARGET_IMAGE**：新仓库名/新标签名

这条命令含义：
拿本地 `nginx:latest`，打一个新标签：`bretfisher/nginx:latest`。
> 不会复制镜像实体，**只是新增一个别名，IMAGE‑ID不变，不占用额外磁盘**。

看列表：
```
bretfisher/nginx   latest   db079554b4d2
nginx              latest   db079554b4d2
nginx              mainline db079554b4d2
```
三个记录，**完全同一个IMAGE‑ID，磁盘只存一份镜像**。

> `bretfisher/nginx` 这种格式：`用户名/仓库名`，这是为推送到Docker Hub做准备。
> 推送到hub的镜像，名字格式必须是 `dockerhub用户名/镜像名`。

## 2、`docker image push bretfisher/nginx`
```bash
docker image push bretfisher/nginx
```
作用：把本地打了标签的镜像，上传推送到Docker Hub远程仓库。

报错：`requested access to the resource is denied`
> **访问资源被拒绝**
### 报错原因
1. 本地镜像标签是 `bretfisher/nginx`，`bretfisher` 是别人的DockerHub账号，**你没有权限往别人账号仓库推送镜像**。
2. 想要push成功：
    - ① docker login 登录**你自己的DockerHub账号**
    - ② tag的时候，写成你自己的用户名：`docker image tag nginx 你的hub账号/nginx`
    - ③ 再执行 push

### 完整正确示例
```bash
# 1 登录docker hub，输入账号密码
docker login

# 2 打标签，用户名替换成你自己hub账号
docker image tag nginx myhubname/nginx

# 3 推送到远程仓库
docker image push myhubname/nginx
```

## 3、梳理tag、本地镜像、push的完整流程
1. pull：从远程拉镜像到本地
2. docker image tag：**本地给镜像贴别名（只是指针，不复制数据）**，名字格式必须匹配你的hub账号
3. docker push：把这个标签对应的镜像分层上传到Docker Hub
4. 别人就可以 `docker pull myhubname/nginx` 下载你的镜像

> 关键点：**tag只是改本地的名字；push依靠这个名字知道上传到哪个远程仓库。**

# 语法：docker image tag 源镜像[:tag] 目标镜像[:tag]
docker image tag nginx:latest myname/nginx:v1
