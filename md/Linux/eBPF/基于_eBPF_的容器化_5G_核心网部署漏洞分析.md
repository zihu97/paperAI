# 2603.19867v1.pdf
# Vulnerability Analysis of eBPF-enabled Containerized Deployments of 5G Core Networks

## 基于 eBPF 的容器化 5G 核心网部署漏洞分析

### 中文摘要
本文针对采用 eBPF 技术加速数据面处理的容器化 5G 核心网部署进行了深入的漏洞分析。虽然 eBPF 在提升 5G 用户面功能（UPF）
性能和可观测性方面表现出色，但其强大的内核编程能力也带来了显著的安全隐患。本文在 Open5GS 环境下构建了真实的实验平台，
详细探讨并实现了四种攻击场景：进程追踪、拒绝服务（DoS）、敏感信息窃取以及 Bash 命令注入。实验结果证明，攻击者可以
利用 eBPF 的特权指令突破容器隔离，威胁 5G 网络的安全运行。最后，本文提出了相应的防御和缓解策略，为未来 5G/6G 网络的安全
加固提供了参考。

### 各章节主旨内容
1.  **引言 (Introduction)**：介绍 5G 服务化架构（SBA）及容器化趋势，阐述 eBPF 在高性能 5G 网络（如 UPF）中的应用背景
    及其引入的安全风险。
2.  **系统架构 (System Architecture)**：描述 5G 独立组网（SA）架构，详细说明 eUPF（基于 eBPF 加速的 UPF）的实现原理
    及其在 Linux 内核中的工作流程。
3.  **威胁模型与实验设置 (Threat Model and Experimental Setup)**：定义攻击者模型（受损容器），介绍基于 Ubuntu 24.04、
    Linux 6.14 内核及 Open5GS 的测试平台配置。
4.  **漏洞分析 (Vulnerability Analysis)**：核心章节，详细剖析并演示了利用 eBPF helper 函数实现追踪、DoS、信息窃取和
    Bash 注入的具体攻击路径。
5.  **可能的缓解措施 (Possible Mitigations)**：探讨从虚拟化隔离、特权限制到精细化权限模型（LSM）及供应链安全等多种
    防御方案。
6.  **结论 (Conclusion)**：总结研究成果，强调在追求高性能的同时必须兼顾 eBPF 的安全管控，并展望未来的研究方向。

### 中文结论
研究表明，共享内核的容器化 5G 部署天然容易受到来自 eBPF 的跨容器攻击。由于 eUPF 需要 `CAP_SYS_ADMIN` 等高特权，
攻击者一旦控制受损容器，即可利用 eBPF 绕过命名空间隔离，执行从被动嗅探到主动干预（如篡改管理员脚本）的各类攻击。
尽管 eBPF 对 5G 性能至关重要，但当前的“全有或全无”特权模型无法满足安全需求，亟需引入基于 LSM 的精细化访问控制
（如 Hook 限制和 Helper 函数白名单）来保障网络安全。

---

## 章节详解

### 第一章：引言 (Introduction)
**设计思想与痛点分析**：
5G 网络为了支持超可靠低延迟通信（uRLLC）和增强型移动宽带（eMBB），对数据面（UPF）的转发性能要求极高。传统的
容器网络栈在处理高并发小包时存在显著的上下文切换开销。eBPF（特别是 XDP）通过在内核驱动层直接处理报文，
避免了数据包在内核协议栈与用户态之间的多次拷贝，将处理效率提升了 30% 以上。
然而，作者敏锐地发现，业界在疯狂追求性能提升的同时，忽略了 eBPF 的安全侧写。在云原生多租户环境下，多个 5G 
网络功能（NF）共享同一物理主机的内核，eBPF 的内核态运行特性成为了突破容器边界的天然利器。

**关键技术背景**：
- **XDP (eXpress Data Path)**：在网络驱动层提供可编程数据面。
- **5G NF 容器化**：AMF、SMF、UPF 等组件以容器形式部署，通常需要复杂的特权配置。

### 第二章：系统架构 (System Architecture)
**深度拆解**：
本文采用 3GPP 定义的 5G SA 架构，重点在于 **eUPF**（eBPF-based User Plane Function）。
- **数据面流转**：eUPF 通过 XDP 挂载在主机的网络接口上。当 N3 接口收到来自 gNB 的 GTP-U 封装包时，XDP 
  程序直接在内核中进行 GTP 头部解析、TEID 查找、路由转发（N6 接口）或反向封装（N6 到 N3）。
- **特权要求**：eUPF 容器必须具备 `CAP_SYS_ADMIN` 和 `CAP_NET_ADMIN` 权限，以便加载和附加 eBPF/XDP 程序。
  这一设计虽然解决了 5G 转发性能的痛点，但正如作者指出的，这赋予了容器修改内核行为的能力，构成了安全隐患的基础。

### 第三章：威胁模型与实验设置 (Threat Model and Experimental Setup)
**实现细节**：
- **攻击入口**：假设一个 eUPF 容器已被攻破（通过供应链攻击或漏洞利用）。攻击者以此容器为跳板。
- **技术栈**：Ubuntu 24.04 (LTS), Kernel 6.14（前瞻性研究）, Open5GS（5G 核心网实现）, UERANSIM（UE/gNB 模拟器）。
- **权限配置**：容器挂载了 `/sys/fs/bpf`，并拥有完整的 `CAP_SYS_ADMIN` 权限。

### 第四章：漏洞分析 (Vulnerability Analysis)
本章是论文的核心，作者展示了 eBPF 如何被“武器化”。

#### A. 追踪攻击 (Tracing Attack)
**技术原理**：
攻击者在内核 `raw_tracepoint/sys_enter` 处挂载 eBPF 程序。利用 `bpf_get_current_comm()` 获取进程名，
`bpf_get_current_pid_tgid()` 获取 PID。
**影响**：
这打破了容器的进程隔离。攻击者可以实时枚举主机上运行的所有 5G 服务进程（如 `open5gs-amfd`），
为后续精准攻击进行侦察。

#### B. 拒绝服务攻击 (DoS)
**设计逻辑**：
利用 `bpf_send_signal(9)` helper 函数。攻击者编写 eBPF 程序，当检测到目标 5G 进程（如 SMF）执行 
`sys_read` 等系统调用时，直接发送 `SIGKILL` 信号。
**后果**：
这种攻击在内核态执行，完全绕过了用户态的防御。即使 Kubernetes 尝试重启 Pod，eBPF 程序仍会持续杀掉
新进程，导致 `CrashLoopBackOff`，使 5G 核心网彻底瘫痪。

#### C. 信息窃取 (Information Theft)
**攻击流程**：
1. **Open 探测**：在 `openat` 系统调用入口，利用 `bpf_probe_read_user_str()` 检查文件名。如果匹配 
   `id_rsa` 或 5G 加密密钥路径，记录其 PID。
2. **Read 拦截**：在 `read` 系统调用退出（exit）时，利用 `bpf_probe_read_user()` 从用户空间缓冲区
   直接读取文件内容并将其泄露。
**对比与权衡**：
这种方式不需要修改目标容器的文件系统，纯粹从内存层面窃取数据，隐蔽性极高。

#### D. Bash 注入 (Bash Injection)
**核心机制**：
这是最危险的主动干预攻击。攻击者利用 `bpf_probe_write_user()` 直接覆写目标进程的内存缓冲区。
当管理员或自动化脚本执行 `bash script.sh` 时，eBPF 程序拦截 `read` 调用，将原本的脚本内容替换为
恶意命令（如创建反弹 shell），并配合 `bpf_override_return()` 修正返回值，确保 Bash 解释器不会因
长度不匹配而崩溃。

### 第五章：可能的缓解措施 (Possible Mitigations)
**权衡分析 (Trade-offs)**：
1. **虚拟化 (VM-based Isolation)**：将每个 NF 放入独立的虚拟机。优点是内核级隔离，缺点是性能开销大，
   不符合云原生轻量化趋势。
2. **权限限制**：移除 `CAP_SYS_ADMIN`。虽然安全，但会导致 eUPF 无法工作，性能倒退。
3. **LSM 精细化控制 (推荐)**：这是作者推崇的方案。利用 SELinux 或 AppArmor，对 eBPF 的行为进行
   精细约束：
   - **Hook 限制**：限制特定容器只能挂载到特定的 XDP Hook，禁止挂载到追踪点。
   - **Helper 函数白名单**：禁止生产环境容器使用 `bpf_probe_write_user` 或 `bpf_send_signal`。

### 第六章：结论与展望 (Conclusion)
作者强调 eBPF 是 5G 性能的功臣，但也可能是安全的掘墓人。未来的 5G 部署必须建立一套完整的安全评估框架，
将内核态的可编程安全性纳入 NFV（网络功能虚拟化）的标准考量中。
