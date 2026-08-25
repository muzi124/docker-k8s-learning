# 07‑Dockerfile ENTRYPOINT 章节学习笔记
> 课程主题：Dockerfile中 `CMD` / `ENTRYPOINT`，**构建时(Buildtime) vs 运行时(Runtime)**，Linux PID1进程、信号SIGTERM/SIGKILL优雅停机问题。

## 一、Buildtime（构建时） vs Runtime（容器运行时）
|指令|执行时机|说明|
|---|---|---|
|`RUN`|**Buildtime（docker build构建镜像阶段）**|构建镜像时执行，结果写入镜像层；安装软件、编译代码。容器启动后**不会再执行**。|
|`CMD` / `ENTRYPOINT`|**Runtime（docker run启动容器阶段）**|构建镜像时不运行，**只有容器启动的时候才执行**，定义容器启动要跑什么程序。|

> 核心误区：很多新手把启动应用写在`RUN`里面，build的时候执行完就结束，容器启动后什么都不跑。

## 二、CMD 和 ENTRYPOINT 两种写法
### 1. Exec格式（数组格式，✅推荐）
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
ENTRYPOINT ["python", "app.py"]
```
- 直接启动程序，**没有shell包装**；应用直接成为容器内PID 1进程。
- 可以接收docker stop发出的SIGTERM信号，实现优雅关闭，生产优先使用。

### 2. Shell格式（不推荐）
```dockerfile
CMD nginx -g daemon off;
ENTRYPOINT python app.py
```
- 实际内部会用 `/bin/sh -c` 包裹运行。
- **PID1是shell，不是你的业务程序**；shell不会转发信号，`docker stop`收不到优雅关闭信号，超时后被暴力SIGKILL杀掉，会出现异常退出码137/143。

## 三、CMD 和 ENTRYPOINT 行为区别
1. **CMD**：容器默认命令。`docker run 镜像 后面跟的参数`会**完全覆盖掉CMD**。
```dockerfile
CMD ["echo","hello"]
```
`docker run myimage echo world` →输出world，CMD被完全替换。

2. **ENTRYPOINT**：容器固定入口。`docker run`后面追加的参数**作为参数传给ENTRYPOINT**，不会覆盖主程序。
```dockerfile
ENTRYPOINT ["ls"]
```
`docker run myimage -l /etc` →实际执行：`ls -l /etc`

3. **CMD + ENTRYPOINT组合用法**
ENTRYPOINT写固定主程序，CMD写**默认参数**。
```dockerfile
ENTRYPOINT ["ls"]
CMD ["-l"]
```
- 默认运行：`ls -l`
- docker run myimage -a →执行`ls -a`，CMD默认参数被覆盖。

> 典型案例：postgres、mysql官方镜像大量使用ENTRYPOINT。

## 四、PID1与Linux信号（课程重点难点）
- `docker stop`：首先发送 `SIGTERM` 给容器PID1进程，给程序时间优雅关闭；等待宽限期（默认10s），还没退出就发送`SIGKILL`强制杀死。
- Shell格式坑：PID1是`/bin/sh`，shell不把SIGTERM转发给子进程业务程序，业务收不到关闭信号，直接等到超时被暴力杀掉。
- ✅解决：使用Exec数组格式 `["程序","参数"]`，业务程序直接成为PID1，接收信号，优雅停机。

> 如果必须shell脚本作为入口，脚本内要用`exec`把业务程序提升为PID1，转发信号。
```bash
# shell脚本内
exec "$@"
```

## 五、docker run如何覆盖ENTRYPOINT
`--entrypoint` 参数，可以覆盖镜像内的ENTRYPOINT。
```bash
# 临时覆盖入口，进入容器调试，不跑原本应用
docker run --entrypoint /bin/bash myimage
```

## 六、笔记总结要点
1. `RUN`构建时执行；`CMD/ENTRYPOINT`容器启动时执行。
2. 优先**Exec数组格式**，避免shell格式，解决信号、优雅停机问题。
3. CMD：会被run后的参数完全覆盖；ENTRYPOINT：run参数追加作为它的参数。
4. 组合模式：ENTRYPOINT固定程序，CMD提供默认参数。
5. PID1进程必须是业务程序，否则`docker stop`无法优雅关闭容器。
6. `--entrypoint`可以在run时重写镜像的入口，用来调试容器。

## 七、常见踩坑清单
1. 在RUN里面写启动命令：构建阶段执行完毕，容器启动不会运行。
2. 使用shell格式CMD/ENTRYPOINT，docker stop无法优雅关闭，直接强杀。
3. 混淆CMD和ENTRYPOINT，搞不清什么时候参数会被覆盖/追加。
4. shell脚本作为入口脚本，忘记`exec`，业务进程收不到信号。

# 踩坑清单 + 对应解法
## 坑1：在 RUN 里面写启动命令：构建阶段执行完毕，容器启动不会运行
### 错误示例
```dockerfile
# ❌错误：RUN只在docker build构建时执行，容器run的时候不会再跑这行
RUN nginx -g "daemon off;"
```
构建镜像的时候nginx启动，构建结束进程就销毁；容器启动没有任何前台程序，直接退出。

### ✅解法
启动应用的命令必须写在 **CMD / ENTRYPOINT**（运行时执行），`RUN`只用来做安装、编译、下载等构建操作。
```dockerfile
RUN apt update && apt install nginx -y   # 构建阶段：安装软件
CMD ["nginx","-g","daemon off;"]        # 运行阶段：容器启动时跑程序
```

---

## 坑2：使用 shell 格式 CMD/ENTRYPOINT，docker stop 无法优雅关闭，直接强杀
### 错误示例（shell格式，不带数组）
```dockerfile
# ❌shell格式，底层是 /bin/sh -c "nginx -g daemon off;"
# PID1是sh，不是nginx，收不到SIGTERM信号
CMD nginx -g daemon off;
```
执行`docker stop`，信号发给shell，shell不转发给nginx，等待10秒后被SIGKILL暴力杀死，日志退出码 `137`。

### ✅解法：使用Exec数组格式（推荐生产）
```dockerfile
# ✅exec数组格式，nginx直接成为PID1，接收SIGTERM优雅停机
CMD ["nginx","-g","daemon off;"]
```

> 注意：数组内部全部是字符串，参数要拆开写，不要把一整行写进第一个数组元素。

---

## 坑3：混淆 CMD 和 ENTRYPOINT，搞不清什么时候参数会被覆盖 / 追加
### 3‑1 CMD：docker run后面传参数，**完全覆盖CMD**
错误理解：以为run后面的参数会追加。
```dockerfile
CMD ["echo","hello"]
```
```bash
# run后面参数直接替换整个CMD，输出world，不是 hello world
docker run myimage echo world
```

### 3‑2 ENTRYPOINT：docker run后面传参数，**追加传给ENTRYPOINT，不会覆盖主程序**
```dockerfile
ENTRYPOINT ["echo"]
```
```bash
# 实际执行 echo world
docker run myimage world
```

### ✅标准组合用法：ENTRYPOINT + CMD（默认参数）
ENTRYPOINT写固定主程序，CMD放默认参数。run传参就覆盖CMD，追加到ENTRYPOINT。
```dockerfile
ENTRYPOINT ["ls"]
CMD ["-l"]
```
- 默认启动：`ls -l`
- `docker run myimage -a` →执行 `ls -a`

> 记忆口诀：
> CMD = 默认命令，会被run参数整个替换；
> ENTRYPOINT = 固定入口，run参数只是追加它的参数。

---

## 坑4：shell脚本作为入口脚本，忘记exec，业务进程收不到信号
### 错误示例 entrypoint.sh
```bash
#!/bin/bash
python app.py   # ❌没有exec；PID1是bash脚本，python是子进程，收不到docker stop信号
```
Dockerfile
```dockerfile
ENTRYPOINT ["./entrypoint.sh"]
```
`docker stop`信号发给bash，python子进程收不到，超时强杀。

### ✅解法1：脚本末尾用 `exec "$@"`，把业务进程提升为PID1
entrypoint.sh
```bash
#!/bin/bash
# 做一些初始化工作
echo "doing init work..."

# exec：把当前shell进程替换成业务程序，业务程序成为PID1，接收信号
exec python app.py
```

> 如果脚本需要接收外部传入参数：
```bash
exec "$@"
```

### ✅解法2：Dockerfile里直接exec数组，尽量避免shell包装脚本。

---

# 速记小抄，可以贴笔记
1. RUN只管构建安装；启动程序交给 CMD / ENTRYPOINT。
2. 优先用 `["程序","参数"]` exec数组格式，拒绝shell简写格式。
3. CMD：会被run参数完整覆盖；ENTRYPOINT：run参数作为追加参数。
4. shell入口脚本，业务程序前面一定要加 `exec`，让业务进程成为PID1，才能正常响应`docker stop`优雅关闭。
5. 调试容器：`docker run --entrypoint /bin/bash 镜像名`，绕过原有入口。

> 补充小知识点：
> docker stop发送SIGTERM；超时没退出发送SIGKILL；程序被强杀退出码一般是 **137**。出现137优先排查是不是PID1不是业务程序。

# Dockerfile指令分类图完整解读
> 核心思想：区分**构建时(Buildtime：docker build)**、**运行时(Runtime：docker run)**；再区分指令是**追加Additive**还是**覆盖Overwrite**。
> 一句话：
> - Buildtime：`docker build`执行镜像构建，真正改变镜像文件层。
> - Runtime：只是写入镜像元数据，**build阶段不执行**，只有容器启动`docker run`才生效。
> - Both：构建和运行都生效。

## 第一张大图：按执行时机划分
### 🟨 Buildtime（仅构建时 docker build）
只在`docker build`镜像构建阶段执行，**真正修改镜像文件系统**。
|指令|作用|
|---|---|
|`FROM`|基础镜像，镜像起点|
|`ADD / COPY`|拷贝文件到镜像内，写入镜像层|
|`RUN`|构建镜像时执行命令，安装软件，生成镜像层|
|`ARG`|构建时参数，仅build阶段可用，容器运行时不可见|
|`ONBUILD`|触发器，别人基于此镜像再build才触发执行|

> 重点：`RUN`在这里！构建阶段执行，容器run不会再跑。

### 🟩 Both（Build & Run，构建、运行两个阶段都生效）
构建时写入镜像元数据，容器运行时继承生效，**重复写会覆盖旧值**。
`LABEL`、`ENV`、`USER`、`SHELL`、`WORKDIR`
- `ENV`：构建时可以用，容器启动后环境变量也存在。
- `WORKDIR`：build时cd到此目录；容器启动默认工作目录。
- `USER`：build阶段切换用户；容器启动默认以此用户运行。

### 🟦 Runtime（仅运行时 docker run，镜像元数据）
> ⚠️**build构建镜像的时候不会执行！只是存元数据，只有容器启动才生效**。
`EXPOSE`、`VOLUME`、`STOPSIGNAL`、`CMD`、`ENTRYPOINT`、`HEALTHCHECK`

- `EXPOSE`：只是文档声明端口，**不会自动打开端口**，仅元数据。
- `VOLUME`：镜像声明卷，容器启动自动创建卷；build阶段不会生成卷。
- `CMD / ENTRYPOINT`：容器启动命令，build不执行，run才执行。
- `HEALTHCHECK`：容器健康检查，运行时才执行。

---

## 第二张大图：追加Additive vs 覆盖Overwrite
### 1. Additive（追加模式，不能覆盖，多条就叠加）
`FROM`、`ADD`、`COPY`、`RUN`、`ONBUILD`
> 每写一条，就新增一层镜像，**不会覆盖上一条**。
- 多个RUN，会一层层叠加镜像层；
- COPY多次，文件不断往镜像叠加，不会清空前面拷贝的。
> FROM只能写一次有效（多阶段除外）。

### 2. Overwrite（覆盖模式，同指令写多次，后面直接替换前面）
`LABEL`、`ENV`、`SHELL`、`USER`、`WORKDIR`
> 同一个key重复写，**后面直接覆盖前面的值**。
```dockerfile
ENV VERSION=1
ENV VERSION=2
# 最终 VERSION=2，后面覆盖前面
```
```dockerfile
WORKDIR /app
WORKDIR /data
# 最终工作目录是/data，后者覆盖前者
```

### 3. ARG：构建时参数，build阶段生效，容器运行环境看不见。
```dockerfile
ARG VERSION=1.0
RUN echo $VERSION
# 容器启动后，$VERSION变量不存在，只有build时能用
```

### 4. Runtime组全部是覆盖逻辑
`EXPOSE` / `VOLUME` / `STOPSIGNAL` / `CMD` / `ENTRYPOINT` / `HEALTHCHECK`
Dockerfile多次写`CMD`，只有最后一条生效，前面全部被覆盖。
```dockerfile
CMD ["echo","a"]
CMD ["echo","b"]
# 最终只会执行 echo b
```

## 高频易错总结（结合之前踩坑）
1. ❌误区：以为`CMD/ENTRYPOINT`在build的时候执行。
✅真相：属于Runtime，**build只是存元数据，docker run容器启动才执行**。

2. ❌误区：RUN写启动命令。
✅真相：RUN属于Buildtime，构建阶段跑完就结束；容器启动不会再次运行。启动程序写CMD/ENTRYPOINT。

3. ENV、WORKDIR、USER多次书写，**后面覆盖前面**，不是叠加。

4. EXPOSE仅仅文档声明，不会自动发布端口；VOLUME镜像声明，build不会创建卷，容器run才生成。

5. ARG仅仅构建时可用；ENV构建+容器运行都可用。

## 记忆小抄
|类别|指令|特性|
|---|---|---|
|仅Buildtime|FROM,COPY,ADD,RUN,ARG,ONBUILD|改变镜像文件层；ARG容器运行不可见|
|Build+Run|ENV,LABEL,USER,WORKDIR,SHELL|写多次后面覆盖前面|
|仅Runtime|EXPOSE,VOLUME,CMD,ENTRYPOINT,HEALTHCHECK|build不执行，run才生效，多次写取最后一条|
|Additive|RUN/COPY/ADD|每一条新增镜像层，叠加不覆盖|
|Overwrite|ENV/WORKDIR/USER/CMD|重复写，后者覆盖前者|

### 延伸面试考点
> Q：Dockerfile写两次CMD会发生什么？
A：只有最后一条CMD生效，前面全部被覆盖，CMD属于Runtime+Overwrite。

> Q：ARG和ENV区别？
A：ARG仅docker build构建阶段可见；ENV构建和容器运行环境都存在。

> Q：VOLUME写在Dockerfile里面，build的时候会生成卷吗？
A：不会，镜像元数据，**docker run启动容器那一刻才会创建volume**。




## 主题：Dockerfile 的 Shell Form（shell形式） vs Exec Form（exec数组形式）
### 1.两个写法区别
1. **Shell Form（shell形式）**
```dockerfile
CMD python app.py
```
内部会包装成 `/bin/sh -c python app.py`，真正PID1是shell，你的程序是shell的子进程。
> 字幕原话：*But you should know that the shell binary used in the Shell Form can be changed.*
> 翻译：**Shell Form所使用的shell二进制程序是可以修改的**（Dockerfile `SHELL` 指令切换默认shell，比如换成bash、powershell）。

2. **Exec Form（exec数组形式，JSON数组）**
```dockerfile
CMD ["python","app.py"]
```
直接启动程序，**没有shell中间层**，你的程序直接是容器PID1，能够收到停止信号SIGTERM。

### 2.核心知识点对比
| 项目 | Shell Form | Exec Form |
|---|---|---|
| 写法 | 普通字符串 | JSON数组 `[]` |
| 是否经过shell | ✅ 会调用sh -c，可以写`$变量`、管道`|`、`&&` | ❌ 不经过shell，变量、管道不生效 |
| PID1进程 | shell（sh），业务程序是子进程，收不到停止信号 | 业务程序直接PID1，可以接收SIGTERM优雅关闭 |
| 修改shell | 可以用`SHELL [...]`指令更换默认shell | 和SHELL指令无关，直接调用二进制 |


