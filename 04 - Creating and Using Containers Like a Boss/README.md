1. docker version
作用：检查 Docker 客户端 (CLI) 和 Docker 引擎 (daemon) 之间能否正常通信
输出 Client：本地 docker 命令行工具版本信息
输出 Server：后台 Docker 引擎服务版本信息
✅出现 Server = CLI 成功连通引擎；没有 Server 输出 = 引擎未启动 / 连接失败
2. docker info
作用：查看 Docker 引擎完整运行配置信息
获取内容：容器数量、镜像数量、存储驱动、网络配置、系统资源、镜像仓库等绝大多数引擎参数。
3. docker 命令语法结构
旧格式（仍然兼容可用）
docker <command> (options)
例：docker run
新版分层格式（主流推荐）
docker <command> <sub‑command> (options)
例：docker container run，主命令 + 子命令，结构更清晰


Docker：主命令 + 子命令（命令分层）
Docker 在 1.13 版本之后做了命令重构，把命令做分组：
docker 【主命令】 【子命令】 (--选项/参数)
旧语法：直接 docker <命令>，属于扁平式，现在依然兼容，但不推荐写新代码。
1. 什么是主命令（分组）
主命令 = 资源大类，代表你要操作哪一类 Docker 对象：
container 容器
image 镜像
network 网络
volume 数据卷
system 系统 / 整体
buildx 构建
2. 什么是子命令
子命令 = 要对该资源执行的动作：run、ls、rm、inspect、stop、start……




Docker 容器常用指令笔记
镜像：模板；容器：镜像运行出来的实例
重点：run= 新建 + 启动；start= 启动已存在旧容器，不会新建
1️⃣ 创建并启动容器前台运行（占用终端，日志直接输出屏幕，Ctrl+C 会停止容器）bashdocker container run --publish 80:80 nginx
后台运行（-d /--detach，不霸占终端，推荐）bash# --publish 端口映射，--name 给容器自定义名字
docker container run --publish 80:80 --detach --name webhost nginx
2️⃣ 查看容器bash# 只看【正在运行Up】的容器
docker container ls

# -a 查看全部容器：运行中 + 已经停止Exited的容器
docker container ls -a
3️⃣ 停止容器（优雅关闭，容器保留，不删除）支持写容器 ID 前缀 或者 容器名字bash# 使用ID前缀
docker container stop 4c8

# 使用容器名字
docker container stop webhost

前台 run 模式，终端按 Ctrl + C，等价于 stop，容器变为 Exited，不会删除。
4️⃣ 重新启动已经停止的旧容器（不要用 run！run 会新建）bashdocker container start 4c8
docker container start webhost
5️⃣ 删除容器（容器必须先停止；彻底清除容器数据）bash# 删除单个容器，ID/名字
docker container rm 4c8

# 一次性删除多个停止的容器，空格隔开ID前缀
docker container rm 4c8 eae

容器还在运行，直接 rm 会报错；强制删除运行中的容器：docker container rm -f webhost
6️⃣ 排查查看命令（辅助）bash# 查看容器日志
docker container logs webhost

# 查看容器内部进程（不能裸敲，必须传ID/名字）
docker container top webhost

# 实时查看容器CPU、内存占用
docker stats
❗高频踩坑要点
反复执行docker container run → 每一次都会创建全新容器实例，产生大量 Exited 废弃容器。
stop ≠ rm：stop 只是关闭进程，容器实体还在磁盘；rm 才是真正删掉。
ls默认过滤掉停止容器，想看全部必须加 -a。
ID 前缀：只要本机可以唯一识别，写前 3‑4 位即可。


docker container run --publish 8080:80 --name webhost -d nginx:1.11 nginx -T
逐段解析
docker container run
创建并启动一个新容器。
--publish 8080:80（简写 -p 8080:80）
标注：change host listening port
端口映射：宿主机端口:容器内部端口
含义：访问本机的 8080 端口，流量转发到容器内部的 80 端口。
修改宿主机端口就改冒号前面数字，修改容器内端口改冒号后面。
--name webhost
给容器起名字叫 webhost，后续操作（stop/rm/logs）可以直接用名字，不用容器 ID。
-d
--detach，后台守护模式运行容器，容器在后台跑，不占用当前终端。
nginx:1.11
标注：change version of image
镜像名：标签 (tag)，指定用 nginx 镜像的 1.11 版本。
修改镜像版本，就改冒号后面的版本号。
nginx -T
标注：change CMD run on start



docker container inspect 容器名/容器ID
# 简写
docker inspect 容器名/容器ID

作用
查看容器完整底层元数据，输出一大段 JSON 格式信息。
docker ps只能看到精简信息；inspect看全部内部配置。

字段	含义
Id	容器完整 ID（长 ID，ps 只显示前 12 位缩写）
Created	容器创建时间
Path	容器入口程序，启动时执行的程序，mysql 这里是docker‑entrypoint.sh脚本
Args	传给入口程序的参数，这里传入mysqld，告诉脚本最终要启动 mysql 服务
State	容器状态块Status:running → 容器正在运行；还能拿到启动时间、退出码、是否重启等

# 实时查看容器资源占用
docker container stats
# 简写
docker stats

# 指定单个/多个容器
docker stats mysql01 nginx


核心作用
实时监控容器CPU、内存、磁盘 IO、网络 IO资源占用，相当于容器版的 top。
ps 看状态；top 看容器内部进程；stats 看资源消耗。



# docker -i -t 参数 & run / exec 笔记
## `-i` （--interactive）
> Keep STDIN open，**保持标准输入打开**
- 作用：让容器接收键盘输入。没有`‑i`，你敲键盘容器收不到。
- 只 `-i`：有输入流，但没有终端，没有命令提示符，体验很差。

## `-t`（--tty，pseudo‑tty 伪终端）
> Allocate a pseudo‑TTY，分配一个虚拟终端，类似ssh登录后的终端界面
- 作用：模拟真实终端，出现 `root@xxx:/#` 这种命令提示符，支持光标、回退键。
- 只 `-t`：有漂亮提示符，但你敲的字符传不进容器。

## `-it` 组合（最常用）
**`‑i + ‑t` 必须配对使用**：既接收键盘输入，又提供终端交互界面。
> 只要需要进入容器敲命令，几乎都写 `-it`。

---

## `docker container run -it` vs `docker container exec -it`（重点笔记）
1. **`docker container run -it 镜像名 命令`**
> 创建**全新容器**，启动容器并且直接进入交互shell
```bash
docker container run -it --name proxy nginx bash
```
- 行为：从镜像新建一个容器，容器启动后直接执行`bash`，进入容器内部。
- ⚠️注意：**最后面的bash会覆盖镜像默认启动程序**。上面这条，nginx服务不会自动启动，只启动bash shell。退出bash，容器直接停止。

2. **`docker container exec -it 容器名 命令`**
> 在**已经存在、正在运行的容器**里面，额外新开一个交互终端，不影响容器原本主进程。
```bash
docker container exec -it proxy bash
```
- 行为：容器已经在跑，只是附加进去一个shell；退出bash，容器继续正常运行。

### 核心区别表格
|命令|场景|容器状态变化|
|---|---|---|
|`run -it`|新建容器，同时进入shell|创建+启动容器；退出shell，容器停止|
|`exec -it`|进入**已经在运行**的旧容器|容器本来就在跑；退出shell，容器继续运行|

> 记忆口诀：
> - run：**新建容器**；exec：**钻进已经跑起来的老容器**

## 坑点笔记（考试/实操高频）
1. `docker run -it nginx bash`，因为覆盖了默认启动命令，**nginx不会启动**，容器里面只有bash。
2. 生产环境启动nginx/mysql一般不加`‑it`，后台守护运行（`‑d`）；排查问题才用`exec ‑it`进去看。
3. `-i`负责收键盘；`‑t`负责出终端提示符；二者一般成对写`‑it`。

### 示例对比
```bash
# 创建新容器，直接进去bash（容器不会跑nginx）
docker run -it --name proxy nginx bash

# 正确：后台启动nginx容器，之后再exec钻进去
docker run -d --name proxy nginx
docker exec -it proxy bash
```

## 补充退出
- 容器shell里面输入 `exit` 退出：
  - run‑it创建的：exit后容器停止
  - exec‑it进入的：exit后容器继续运行



  # docker container start 命令解析
> 命令：`docker container start --help`
> 作用：查看帮助文档，**启动一个或多个已经停止的容器**（只能启动已存在、被stop停止的容器，不能新建容器）

**语法格式**
```bash
docker container start [OPTIONS] CONTAINER [CONTAINER...]
```
- `[OPTIONS]`：可选参数
- `CONTAINER`：容器ID / 容器名字，可以一次性写多个，同时启动多个容器

---
## Options 参数（截图里的选项）
| 参数 | 全称 | 说明 | 常用场景 |
|---|---|---|---|
| `-a` | `--attach` | 挂载容器标准输出/错误输出，把容器日志打印到当前终端，同时转发信号 | 启动后直接看容器控制台输出日志，按`Ctrl+C`会停止容器 |
| | `--checkpoint string` | 从指定检查点恢复容器 | 高级功能，容器快照恢复，几乎日常不用 |
| | `--checkpoint‑dir string` | 自定义检查点快照存储目录 | 高级快照功能，很少用 |
| | `--detach‑keys string` | 自定义脱离容器终端的快捷键 | 默认 `Ctrl+p, Ctrl+q`，修改退出快捷键 |
| `--help` | `--help` | 打印帮助信息 | 就是你现在执行的这条 |
| `-i` | `--interactive` | 开启容器标准输入STDIN，保持输入打开 | 配合 `-a`，启动后可以向容器输入命令交互 |

### 重点区分
> ⚠️ `docker container start` VS `docker run`
1. `docker run`：**新建+启动容器**，基于镜像创建全新容器
2. `docker container start`：**启动已经存在、被停止的旧容器**，复用之前的容器状态

### 常用示例
```bash
# 启动容器（后台静默启动，最常用）
docker container start nginx

# 启动并且把日志输出到当前终端
docker container start -a nginx

# 启动并开启交互输入
docker container start -ai nginx

# 一次性启动多个容器
docker container start nginx mysql-redis
```

### 小提示
- `-ai` 组合：`-a`看输出 + `-i`允许输入，启动后进入容器；
- 如果只想后台跑容器，**不加任何参数**，直接 `docker container start 容器名`，容器后台运行，终端不会卡住。

### 补充对比笔记（方便记）
|命令|作用|
|----|----|
|`docker container stop`|停止正在运行的容器|
|`docker container start`|启动已经停止的旧容器|
|`docker container restart`|重启容器（先stop再start）|

# docker container exec 命令笔记
> 命令：`docker container exec --help`
> 核心作用：**在一个正在运行的容器内部执行一条命令**
> ⚠️ 前提：容器必须处于运行状态，停止的容器不能用 exec

**语法格式**
```bash
docker container exec [OPTIONS] CONTAINER COMMAND [ARG...]
```
- `[OPTIONS]`：可选参数
- `CONTAINER`：容器ID / 容器名称
- `COMMAND`：要在容器里面执行的命令（比如 bash、ls、mysql）
- `[ARG...]`：命令附加参数

---
## Options 参数对照表
| 短参数 | 完整参数 | 释义 | 使用场景 |
|---|---|---|---|
| `-d` | `--detach` | 后台模式，后台执行这条命令，不输出到终端 | 执行不需要看输出的后台任务 |
| | `--detach‑keys string` | 自定义脱离容器终端快捷键 | 极少用，修改退出快捷键 |
| `-e` | `--env list` | 设置环境变量 | 执行命令时临时注入环境变量 |
| `--help` | `--help` | 打印帮助文档 | 就是当前这条查看帮助 |
| `-i` | `--interactive` | 保持标准输入STDIN打开，接收键盘输入 | 交互必备，配合 `-t` 使用 |
| | `--privileged` | 给这条命令最高特权权限 | 容器内要操作内核硬件时使用，谨慎使用 |
| `-t` | `--tty` | 分配一个伪终端 | 模拟终端窗口，显示命令提示符 |
| `-u` | `--user string` | 指定执行命令的用户/UID | 不用root，用普通用户执行命令 |

### 🔥最经典组合：`-it`
- `-i`：保持输入流
- `-t`：分配伪终端
**两者几乎永远成对使用 `-it`**，用来进入容器内部终端。

### 常用示例
```bash
# 进入mysql容器内部，打开bash终端（最常用）
docker container exec -it mysql bash

# 进入容器直接执行一条命令，不进入交互shell
docker container exec mysql ls /

# 后台执行命令，不占用终端
docker container exec -d nginx touch /tmp/test.txt

# 指定普通用户执行命令
docker container exec -u 1000 -it nginx bash

# 临时设置环境变量执行命令
docker container exec -e "NAME=test" -it nginx env
```

---
## 易混淆对比（重点记笔记）
|命令|条件|作用|
|---|---|---|
|`docker container start`|容器已停止|启动已经存在的容器，让容器跑起来|
|`docker container exec`|容器**正在运行**|进到运行中的容器里面执行命令，不会重启容器|
|`docker run`|无容器|基于镜像新建容器并且直接启动|

> 易错坑：
> 1. exec 只能操作**运行中**容器，如果容器stop了，exec直接报错，需要先start启动容器。
> 2. `-it` 缺一不可：只有 `-i` 没有 `-t` 没有命令行提示符；只有 `-t` 没有 `-i` 无法输入文字。

### 补充小知识点
```bash
# 如果容器里面没有bash（比如alpine轻量镜像），换成sh
docker exec -it nginx sh
```

docker container inspect --format "{{ .NetworkSettings.IPAddress }}" webhost

docker container inspect：获取容器完整元数据（默认输出大段 JSON）
--format（简写 -f）：模板过滤，只提取指定字段，避免刷屏大 JSON
"{{ .NetworkSettings.IPAddress }}"：Go 模板语法
.NetworkSettings 网络配置节点
.IPAddress 获取容器内部 IP 地址
webhost：容器名字
作用：直接打印出容器 webhost 的内网 IP，不会输出一大堆 JSON

# 获取容器IP
docker inspect -f "{{ .NetworkSettings.IPAddress }}" webhost

# 获取容器网关
docker inspect -f "{{ .NetworkSettings.Gateway }}" webhost

# 获取容器端口映射
docker inspect -f "{{ .NetworkSettings.Ports }}" webhost

# 获取容器状态
docker inspect -f "{{ .State.Status }}" webhost


作用	命令	说明
查看所有网络	docker network ls	列出 docker 全部网络，driver、scope、名称
查看网络详情	docker network inspect <网络名>	看网段、网关、接入了哪些容器
创建自定义网络	docker network create --driver <驱动> <网络名>	默认驱动bridge；最常用：docker network create my‑net
运行中容器接入网络	docker network connect <网络名> <容器名>	热插拔网卡，容器不用重启，给容器多挂一块虚拟网卡，加入另一个网络
运行中容器脱离网络	docker network disconnect <网络名> <容器名>	热拔网卡，把容器从某个网络剥离，无需重启容器


# 删除未被使用的网络
docker network prune

# 删除指定网络（网络里不能有正在运行的容器）
docker network rm my‑app‑net


docker container run -d --name my_nginx --network my_app_net nginx


参数	作用
docker container run	创建并启动容器，等价简写 docker run
-d	后台守护模式运行，不占用终端
--name my_nginx	设置容器名称，后续可直接用名字操作容器，不用长 ID
--network my_app_net	将容器加入自定义网络my_app_net
nginx	使用 nginx 镜像启动容器


# Docker --net‑alias 网络别名 学习笔记
## 完整命令序列
```bash
#1. 创建自定义网络 dude
docker network create dude

#2. 启动第一个es容器，指定网络别名 search
docker container run -d --net dude --net-alias search elasticsearch:2

#3. 启动第二个es容器，同样设置网络别名 search
docker container run -d --net dude --net-alias search elasticsearch:2

#4. alpine容器测试DNS解析别名search
docker container run --rm --net dude alpine nslookup search

#5. centos容器curl访问别名search:9200
docker container run --rm --net dude centos curl -s search:9200
```

## 参数解释
|参数|说明|
|---|---|
|`docker network create dude`|创建自定义网络，网络名叫 `dude`|
|`--net dude`|等价 `--network dude`，把容器加入dude自定义网络|
|`--net‑alias search`|**网络别名**。同一个网络内，多个容器可以共用同一个别名`search`。<br>DNS对该别名返回**全部绑定该别名的容器IP**，实现简单DNS轮询|
|`-d`|后台运行容器|
|`elasticsearch:2`|使用elasticsearch2.x镜像|
|`--rm`|容器执行完命令就**自动删除容器**，用完即扔，不残留垃圾|
|`alpine nslookup search`|alpine小镜像，执行nslookup做域名解析测试|
|`centos curl -s search:9200`|centos镜像执行curl，`‑s`静默模式，不输出进度，访问search别名的9200端口（es默认端口）|

## 输出解读
1. `docker container ls`：可以看到两个elasticsearch容器，**容器名字是随机生成**，没有手动`--name`，但是二者共享同一个网络别名`search`。
2. `nslookup search`输出：
```
Address 1: 172.18.0.2 search.dude
Address 2: 172.18.0.3 search.dude
```
> DNS域名`search`解析出来两个IP，对应两台es容器。Docker内置DNS做轮询。
3. `curl -s search:9200`返回elasticsearch的JSON信息，代表**通过网络别名成功访问服务**，多次执行会轮流访问不同容器，实现简单负载均衡。

## 核心知识点 ✨`--net‑alias`重点
1. `--name`是**容器唯一名字**，一个名字只能属于1个容器；
2. `--net‑alias`是**网络别名，允许多个容器共用同一个别名**；
3. 别名只在**所属自定义网络内部生效**，外部网络不能解析；
4. Docker内置DNS，访问别名时返回全部绑定该别名的容器IP，实现DNS轮询（简易负载均衡）；
5. 默认`bridge`网络**不支持net‑alias，必须自定义网络**。

## 对比总结（笔记重点）
|选项|`--name`|`--net‑alias`|
|---|---|---|
|作用|容器唯一名称|网络DNS别名|
|重复|不能重名|多个容器可以相同别名|
|生效范围|全局容器管理|仅当前自定义网络内部DNS解析|
|DNS解析|可以解析|可以解析，支持多IP轮询|

## 易错点
1. `--net‑alias`只在自定义网络生效，默认bridge网络无效；
2. 别名不能跨网络访问；
3. `--rm`：测试用，执行完命令容器自动销毁，适合临时调试，不要用于长期服务。

## 使用场景
同一组多实例服务（ES集群、后端服务副本），给全部实例设置同一个`net‑alias`，其他服务直接访问别名，docker DNS自动轮询分发请求。

> 注意：只是DNS轮询，**不是真正负载均衡**，没有健康检查，故障容器IP依旧会被DNS返回。

## 配套记忆命令
```bash
docker network inspect dude   #查看网络，可看到net‑alias信息
docker ps                     #查看运行容器
```

