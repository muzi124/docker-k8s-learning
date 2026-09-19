# Docker Security 学习笔记（课程：Docker Mastery - Docker Security Good Defaults and Tools）
> 本章节讲解Docker容器安全基线、核心隔离机制、安全工具、最佳实践，覆盖从内核隔离到镜像、运行时安全全套内容。

## 一、开篇：Docker安全十大要点（Section 001）
容器安全≠虚拟机安全，容器共享宿主机内核，安全边界比VM更弱，需要一套专门的安全控制手段。
1. 最小权限原则是容器安全的核心
2. 安全不是单一开关，是多层防御：内核隔离、镜像、运行时、主机、合规审计

## 二、Docker底层安全基础：Cgroups & Namespaces（002）
这是Docker隔离的两大内核基石：
1. **Namespaces（命名空间）**：隔离资源视图。容器内进程看不到宿主机和其他容器的进程、网络、挂载、用户等。
    - PID、网络、挂载、用户、IPC、主机名命名空间
2. **Cgroups（控制组）**：限制资源用量。限制容器CPU、内存、IO，防止容器耗尽宿主机资源（DoS）
> ⚠️ 注意：Namespace只做**视图隔离**，不是安全沙箱；内核漏洞会突破隔离。Cgroups只做资源限制，不防代码执行。

配套：Docker Engine本身内置的默认安全机制，引擎层面自带部分安全保护，但**默认配置并不等于足够安全**。

## 三、内核安全增强：AppArmor & Seccomp（003）
Linux内核安全模块，对容器进程做限制：
1. **AppArmor**：强制访问控制，配置文件定义进程能读写哪些文件、访问哪些路径。Docker自带默认AppArmor profile。
2. **Seccomp（安全计算模式）**：过滤系统调用。容器进程可以使用的内核syscall白名单，阻止容器调用高危系统调用。
> 很多人会`--security-opt=seccomp=unconfined`关闭它，这会大幅降低容器安全性，生产禁止随意关闭。

## 四、安全基准扫描：CIS Benchmark & Docker Bench（004）
1. **CIS Docker Benchmark**：业界标准Docker安全合规检查清单，给出主机、daemon、容器、镜像的安全基线。
2. **Docker Bench for Security**：官方脚本工具，一键扫描宿主机Docker环境，对照CIS基线输出安全风险报告。
> 用途：上线前自动化巡检，找出daemon配置漏洞、不安全容器启动参数、权限问题。

## 五、镜像安全：Dockerfile 使用 USER 非root用户（005）
**最重要最佳实践之一：容器内不要用root运行应用**
1. 很多基础镜像默认root，如果容器被入侵，攻击者拿到容器root，就有机会逃逸到宿主机。
2. Dockerfile 用 `USER` 指令，提前创建普通用户，应用进程以普通用户身份启动。
3. 注意：要处理文件目录权限，避免业务无读写权限。

## 六、User Namespaces（用户命名空间）（006）
宿主机UID映射：容器内root（uid=0），映射到宿主机一个普通非root uid。
- 开启后：就算容器内拿到root权限，在宿主机上只是普通用户，大幅降低容器逃逸风险。
- 缺点：部分存储、卷挂载场景兼容性问题，需要规划后再启用。

## 七、镜像漏洞扫描（007）
1. **Snyk**：镜像/代码漏洞扫描工具，扫描镜像OS包、应用依赖的CVE漏洞。
2. CVE数据库：漏洞信息来源，用于比对镜像内安装软件包版本。
3. **左移安全 Shift-Left Security**：安全检查放到开发阶段，在CI构建镜像时就扫描漏洞，不要等到上线后才发现。
> 核心思想：问题越早发现，修复成本越低。提交代码、构建镜像阶段自动扫描。

## 八、运行时安全：Content Trust、Sysdig Falco（008）
1. **Docker Content Trust（DCT）**：镜像签名机制。对镜像做数字签名，拉取镜像时校验签名，只允许可信、未篡改的镜像运行，防止镜像被投毒。
2. **Sysdig Falco**：容器运行时行为监控工具。持续审计容器行为，当出现异常行为（比如容器内创建宿主机敏感文件、提权、访问机密）触发告警。
> 属于**运行时检测**：镜像扫描是静态检查；Falco监控容器正在运行时的行为。

## 九、Rootless Docker（无根Docker，009）
Rootless模式：Docker daemon**不需要root权限启动**，由普通用户运行dockerd。
- 传统Docker：daemon以root运行，`docker.sock`权限一旦泄露=宿主机root沦陷。
- Rootless：dockerd进程本身是普通用户，降低daemon被劫持带来的宿主机权限风险。
- 局限：部分网络、存储驱动有兼容性限制。

## 十、安全Top10：容器 vs 虚拟机差异（010）
容器和VM安全模型差异：
- VM：完整独立内核，隔离强，隔离成本高。
- 容器：共享宿主机内核，隔离轻量，攻击面在内核。
> 容器安全重心：内核加固、限制系统调用、最小权限、镜像漏洞扫描、运行时审计。

## 十一、Distroless 极简镜像（011）
Google Distroless镜像：镜像里**只有应用程序和运行依赖，没有shell、包管理器、ls/curl等工具**。
- 优势：
  1. 攻击面极小，就算应用被攻破，容器内没有工具供攻击者进一步探测、提权。
  2. 体积更小，漏洞数量大幅减少。
- 缺点：无法直接`docker exec`进容器调试，调试需要额外方案。

## 十二、集群安全：Swarm & Kubernetes（012）
Docker Swarm和K8s在容器安全上的异同：
- 底层容器安全机制（namespace、seccomp、apparmor、USER）两者都支持。
- K8s扩展更多安全能力：Pod安全策略/PSP、网络策略、RBAC、准入控制。
> 容器底层安全是基础，编排层是额外的安全控制，两者叠加防御。

# ✅ 整体最佳实践总结
1. **权限最小化**：容器内尽量不用root；优先Dockerfile USER；必要时开启User Namespaces；生产优先Rootless Docker。
2. **内核加固**：启用Seccomp和AppArmor，不要随意关闭。
3. **镜像安全**：CI阶段左移扫描漏洞；使用Distroless极简镜像；启用镜像签名Content Trust。
4. **基线检查**：用Docker Bench定期扫描宿主机，遵循CIS基线。
5. **运行时防护**：Falco监控异常行为，实时告警。
6. **分层防御**：内核隔离、镜像构建、镜像分发、运行监控、编排权限，多层防护，不依赖单一安全手段。

