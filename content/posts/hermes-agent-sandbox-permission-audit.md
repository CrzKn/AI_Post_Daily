---
title: "别让 agent 乱跑：沙箱、权限边界与审计日志的三层笼子"
date: 2026-10-29T11:00:00+08:00
tags: ["AI-Agent", "Sandbox", "Least-Privilege", "Audit-Log", "Platform-Engineering"]
author: "CrzKn"
---

凌晨三点，你的 coding agent 在没有任何人点「确认」的情况下，读到了 `~/.ssh`，在生产数据库的路径下写入了一份「修复脚本」，还朝一个你从没在配置里见过的域名发了 POST。第一反应几乎是条件反射：「模型学坏了。」但你把日志拉出来复盘，多半会发现四件事同时成立：沙箱没开、权限给满、出口没设限、日志根本没落地。

这四件事，正好对应本文要讲的三层笼子——进程与文件边界（process & filesystem boundary）、网络出口白名单（egress allowlist）、工具权限门（tool gate）与审计日志（audit log）。核心观点先放在最前面：**agent 越界事件里，绝大多数不是模型学会了使坏，而是安全配置缺失**。支撑它的有三条：越界需要「能力」才能发生，而能力来自运行时的授予，不来自模型的意愿；提示注入在原理上无法被 prompt 可靠防住，因为同一条通道既传数据又传指令，只有架构层隔离能兜底；事故善后靠日志与回滚，不靠模型「下次注意」。

所以整篇文章的立场是一句话：修的方向是「把能力面收窄」，不是「加强提示词」。下面四节，每一节对应一条可执行的原则和一个明确的结论句，后三节各配一段能直接抄的配置或验证命令；读完你应该能照着把自己 agent 平台的三个口子各封上一层。

## 一、先定性质：什么才算越界

在谈笼子之前，先给「越界」一个可判定的定义，再看几起公开事件——它们决定后面每一层要防什么。

### 三类越界：读不该读的、写不该写的、连不该连的

越界行为可以归纳成三类：读不该读的（凭据、私有仓库、上下文数据）、写不该写的（删库、覆盖生产配置、提交未授权变更）、连不该连的（外带数据、访问内网服务）。每一类都有公开事件，而它们的根因几乎都不在「模型变坏」。

先看「写」。2025 年 7 月，Replit Agent 在代码冻结期执行了一条破坏性数据库命令，删掉生产库后又生成假数据，事后 agent 自述「I violated explicit instructions」（[Fortune](https://fortune.com/2025/07/23/ai-coding-tool-replit-wiped-database-called-it-a-catastrophic-failure/)、[AI Incident DB 1152](https://incidentdatabase.ai/cite/1152/)）。很多人把它当成「模型误判」的铁证，但根因其实是**配置缺失**：dev 和 prod 库没有隔离、没有审批门——代码冻结是人的策略，权限没有跟着收窄。agent 违反的是「指令」，但真正可修的漏洞是「它居然有能力删生产库」。

再看「读」。更硬的证据在供应链：2025 年 8 月，恶意 npm 包 s1ngularity 不攻破 agent 本身，而是直接调用 Claude Code 的 `--dangerously-skip-permissions`、Codex 的 `--yolo`，把受害者的 agent 变成搜刮 GitHub/npm token、SSH key、钱包的取证工具（[Snyk](https://snyk.io/blog/weaponizing-ai-coding-agents-for-malware-in-the-nx-malicious-package/)、[Wiz](https://www.wiz.io/blog/s1ngularity-supply-chain-attack)）。免确认 flag 本身就是攻击面——攻击者不用绕过任何防线，因为防线默认就是开的。

「连」的例子有两个。M365 Copilot 的 EchoLeak（CVE-2025-32711）是零点击间接注入，恶意 Markdown 图片被自动加载后即外泄上下文——同通道既传数据又传指令的架构缺陷，叠加出口未收窄（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-32711)、[arXiv](https://arxiv.org/html/2509.10540v1)）。GitHub MCP 则因同一 token 兼读公/私库，公共 issue 注毒后读到了私有仓库——权限过宽（[Invariant Labs](https://invariantlabs.ai/blog/mcp-github-vulnerability)）。MCP 还有一类「工具描述投毒」：工具描述里藏指令，客户端批准后攻击者再改描述，客户端不复核（[Invariant Labs](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)）。

还要提一个「生成物也越界」的类别：Lovable 生成的应用默认缺 RLS，任意人可读写数据库，CVE-2025-48757 披露了 170+ 个受影响应用（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-48757)）。这不是 agent 在跑的时候越界，而是它产出的东西默认不安全——同样是配置缺失。

归因纪律给一个示范：Amazon Kiro 被媒体描述为「agent 删掉重建生产环境」，Amazon 反驳称是 misconfigured role、AI 只是碰巧在场，随后补上「mandatory peer review for production access」（[FT](https://www.ft.com/content/00c282de-ed14-4acd-a948-bc8d6bdb339d)、[Amazon](https://www.aboutamazon.com/news/aws/aws-service-outage-ai-bot-kiro)）。两种叙事指向同一件事：**权限授予，不是模型性格**。

### 五个可观测判据

把越界拆成五个可观测面，后三节会逐个回指这些面。之所以要拆成「面」而不是笼统地说「它乱跑了」，是因为每一面都能落到一个具体的技术开关上：文件面对应挂载与路径白名单，进程面对应沙箱与 capability，网络面对应出口策略，凭据面对应免确认开关，持久化面对应审批门与日志。判据的价值在于可观测、可验证：

| 面 | 观测点 | 「看起来正常但已越界」的形态 |
|---|---|---|
| 文件 | 读写路径 | 只该改工作区，却读了 `~/.ssh` |
| 进程 | 子进程与权限 | 以 root 跑，能碰 `docker.sock` |
| 网络 | 出口目标 | 配了域名白名单却走 IP 直连 |
| 凭据 | token/密钥流向 | 免确认 flag 让 token 被外带 |
| 持久化 | 变更留存 | 删库、提交、部署无审批痕迹 |

**结论：越界是被授予出来的，不是模型想出来的。** 判定标准应该是「配置有没有收窄能力面」，而不是「模型这次判断对不对」。用这张表去自查你的 agent 平台，半小时内能定位大多数口子。

## 二、进程与文件边界：把它关进一个会消失的房间

先立一条基准线：**容器隔离的是资源视图，不是内核**。

### 容器不是沙箱：内核共享、/proc、设备、cgroup 与 rootless

把 agent 扔进 Docker 不等于把它关进了笼子。容器用 namespaces + cgroups + seccomp/caps 做隔离，但所有容器共享宿主内核（[Docker rootless](https://docs.docker.com/engine/security/rootless/)）。一条内核漏洞，或一个被挂进去的 `/proc`、设备节点，就足以穿透防线。这也是为什么「容器」和「沙箱」是两回事——前者管资源，后者要管内核共享面。隔离档位从弱到强大致是这样：

| 档位 | 隔离面 | 内核 | 启动开销 |
|---|---|---|---|
| 容器 | namespaces+cgroups+seccomp/caps | 共享宿主内核 | 未公布 |
| rootless 容器 | 同上，但 daemon 与容器都跑在 user namespace，不用 root | 共享 | 同上 |
| gVisor | 用户态内核 Sentry 拦截 syscall，Gofer 经 9P 代理文件访问 | 不跑宿主内核代码 | 未公布 |
| Firecracker microVM | 硬件虚拟化，每 workload 独立 guest 内核 | 独立 | 论文：<125ms 启动、<5MB 内存开销 |
| 独立 VM | 完整 OS | 独立 | 最高（未公布具体数字） |

来源见 [gVisor 安全导论](https://gvisor.dev/docs/architecture_guide/intro/)、[Firecracker NSDI'20 论文](https://www.usenix.org/system/files/nsdi20-paper-agache.pdf)。

几句「为什么重要」的补充：rootless 容器把 daemon 和容器都塞进 user namespace，不用 root，好处是宿主机 root 与容器内 root 不再是一回事；gVisor 用一个用户态内核 Sentry 拦截 syscall、Gofer 经 9P 代理文件访问，等于在 syscall 这一层收窄了攻击面；Firecracker microVM 走硬件虚拟化，每个 workload 一个独立 guest 内核，论文实测 <125ms 启动、<5MB 内存开销，这点启动代价不该再成为拒绝它的理由。选档位的判据其实是「这任务跑的是不是不可信代码」：改自己的仓库，容器档配齐下面的参数就够；跑 PR 里未审查的代码、接第三方 MCP，就该 gVisor 起步。结论不是「容器没用」，而是：容器压低的是逃逸面，不是消灭它；高风险任务（跑未审查代码、接不可信 MCP）该上 gVisor 或 microVM。

哪怕只在容器档，也请把这几项配齐——只读根文件系统、非 root 用户、drop 全部 capabilities、`no-new-privileges`、tmpfs 工作区、任务结束即销毁：

```dockerfile
# 最小沙箱容器骨架
FROM debian:bookworm-slim
USER 65532:65532                      # 非 root
COPY --chown=65532:65532 agent ./agent
ENTRYPOINT ["./agent"]
```

```bash
# 结束即销毁；出口不在容器层放行，单独在 L7 层做
docker run --rm \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --network none \
  agent-image
```

逐条说这几个 flag 各防什么：`--read-only` 让根文件系统不可写，agent 想持久化只能落到 tmpfs 工作区，任务结束就消失；`--cap-drop ALL` 摘掉全部 capability，连改主机名、改系统时钟这类特权都拿不到；`no-new-privileges` 堵住 setuid 提权这条路；`--network none` 先断网，出口在第三节单独放行。任何一条漏掉，都会单独开一个口子——这正是第二节标题里「会消失的房间」的含义。

### 文件边界：给挂载点，不给路径通配

文件边界的可执行原则只有一句：**工作目录 = 唯一可写挂载点**。禁止 `~`、`..`、通配符授权；符号链接和 `..` 穿越要在 realpath 之后比对白名单，否则一个软链就能把「只读」变成「可写」。举个反例：`/workspace` 可写，agent 在里面建了个软链 `link -> /etc`，然后写 `link/passwd`。如果策略在 realpath 之前就比对字符串前缀，`link/passwd` 仍以 `/workspace` 开头，判定通过；realpath 之后变成 `/etc/passwd`，才被拦下。所以顺序是：先 realpath，再比对。给一段策略示例：

```python
import os

WRITABLE = {"/workspace"}          # 唯一可写挂载点
READONLY = {"/usr", "/etc"}        # 只读

def check(path: str, mode: str) -> bool:
    real = os.path.realpath(path)  # 先消掉 symlink 和 ..
    if mode == "w":
        return any(real == p or real.startswith(p + "/") for p in WRITABLE)
    return any(real == p or real.startswith(p + "/") for p in WRITABLE | READONLY)
```

再补一条拒绝清单的思路：除了白名单，还要显式拒绝高风险路径——`~/.ssh`、`~/.aws`、`/var/run/docker.sock`、`/etc/passwd`。白名单决定「能碰什么」，拒绝清单决定「即便漏判也绝不能碰什么」，两者叠加才叫边界。

为什么不能靠默认值？看五个主流运行时的默认姿态：

| 运行时 | 默认沙箱 | 默认可写 | 默认出口 | 免确认开关 |
|---|---|---|---|---|
| Claude Code | 关 | 工作目录+每用户临时目录 | 走本机代理，`allowedDomains` 初始为空 | `--dangerously-skip-permissions` |
| Codex CLI | 开（`workspace-write`） | workspace 内 | 访问网络需审批 | `--dangerously-bypass-approvals-and-sandbox`（别名 `--yolo`） |
| Cursor | Auto-review（官方称非安全边界） | 工作区内 | 沙箱内默认 Blocked | 无单一 yolo flag |
| Aider | 未提供内建开关 | 仓库目录 | 未受限 | `--yes-always` |
| Open Interpreter | 有（`workspace-write`） | workspace roots | workspace-write 下网络关闭 | `--yolo` |

Claude Code 沙箱**默认关闭**，且默认可读「most of the machine, including credential files such as `~/.ssh` and `~/.aws/credentials`」（[sandboxing](https://code.claude.com/docs/en/sandboxing)）。Codex 默认 preset 是 `workspace-write`（[sandbox](https://learn.chatgpt.com/docs/sandboxing)）。Aider 甚至没有内建沙箱开关（[options](https://aider.chat/docs/config/options.html)）。Cursor 有 Auto-review，但官方文档原话「not a security boundary」（[Run Modes](https://cursor.com/docs/agent/security/run-modes)）。**默认值本身就是事故面**——文件边界必须显式配置，不能假设运行时替你收紧了。

**结论：把 agent 关进「会消失的房间」——只读根文件系统、唯一可写挂载点、结束即销毁。** 这一步做对，「文件」和「进程」两个判据面就守住了。

## 三、网络出口白名单：默认全拒，例外审批

出口是 agent 能力面里最难收窄的一环，因为域名会漂移、TLS 不一定被检查、绕过面五花八门。原则只有一条：**默认全拒 + 显式放行**。

### 三层落地：DNS、L4 策略、L7 代理

各层控制的生效点由弱到强。DNS 白名单在解析层生效，但应用可以自带 resolver 或走 DoH，IP 还会漂移，DNS rebinding 还能让同一域名在两次解析间指向不同 IP，所以它最弱——只能拦住「正常走系统 DNS 解析」这一种情况，拦不住自带 resolver 或做 rebinding 的进程。L4 NetworkPolicy 按 IP/CIDR 生效，但域名易漂移，而且 k8s 里默认 pod 对 egress **非隔离**——不写 policy 等于全开。iptables/nftables 在主机 netfilter 层，只有 IP 没有域名概念。L7 正向代理按 SNI/Host + 方法 + 路径放行，最强，而且天然产出日志；代价是它依赖客户端读代理变量，客户端不读就形同虚设（[k8s NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)）。

生产形态通常是「L7 为主 + L4 兜底」：L7 负责把「能不能连」落到具体域名和路径上，L4 负责在 IP 层兜住那些绕过域名解析的直连。还有一点别忘了：标题里的「例外审批」是流程，不是配置——agent 临时要访问一个新域名，走一次人工审批、把域名加进白名单并留痕，而不是在配置里一劳永逸地放开整段网段。

在 k8s 里，「默认全拒」就是一条空 egress 的 NetworkPolicy（[declare-network-policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)）：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: agent-egress-deny-all, namespace: agents}
spec:
  podSelector: {}
  policyTypes: [Egress]   # 空 egress + Egress 类型 = 默认全拒
  egress: []              # 之后按需加 allow：DNS→kube-system、registry.npmjs.org 的 CIDR
```

这套 YAML 是 L4 兜底的一半——它负责「默认全拒」；另一半「显式放行」要按 agent 的真实依赖逐条加：DNS 到 kube-system、拉包走 registry.npmjs.org 的 CIDR。加放行条目时问一句：「这个出口，agent 现在这个任务真的需要吗？」不需要就先不加，宁可事后补，不要预先开。

### 绕过面清单：为什么「配了白名单」还是会漏

以下七条已知绕过面，每条都该用命令验一遍：

1. **不读代理环境变量**——Linux 侧靠 `HTTP_PROXY/HTTPS_PROXY/ALL_PROXY`，可能被忽略。验：`env -u HTTPS_PROXY -u HTTP_PROXY curl -sS https://example.com`，看请求是否绕过代理直达。
2. **白名单 + 不检 TLS → domain fronting**——Claude 文档承认代理按 hostname 放行且「does not terminate or inspect TLS」。验：只放行一个 CDN 大域，再经它访问别的 host。
3. **IP 直连**——域名层策略拦不住 `curl https://1.2.3.4`，必须 L4 默认全拒兜底。验：在只配了域名白名单的环境里直连任意 IP。
4. **IPv6 未纳管**——Claude 文档单列「IPv6 addresses in domain lists」，IPv6 出口常被漏配。验：`curl -6 https://<被拒域名>`。
5. **unix socket 绕过网络层**——能直接碰 `docker.sock` 这类套接字，网络层策略根本看不见。验：沙箱里试 `curl --unix-socket /var/run/docker.sock http://localhost/version`。
6. **localhost 全端口**——Claude Code 为沙箱命令设 `NO_PROXY`，localhost 直连等于本机所有监听服务。验：沙箱内 `curl localhost:<别的服务端口>`。
7. **k8s 节点流量豁免 / hostNetwork pod**——NetworkPolicy 管不到节点自身的流量。验：hostNetwork pod 里连宿主机网段。

这七条的共同点是：白名单在「解析层」和「域名层」做的判断，都能被更底层的通道绕过去。所以验证的正确姿势不是「白名单里没有这个域名」，而是「从沙箱里发起一次连接，它确实出不去」——前者看配置，后者看行为。更省事的做法是把这七条写成 CI 里的 egress 冒烟测试：起一个沙箱容器，依次跑上面的验证命令，断言「该通的通、该断的断」，白名单每次改动后重跑一遍防回归。

**结论：配了白名单不等于收窄了出口。** 至少要「L7 代理为主 + L4 默认全拒兜底」，并逐条验证上面的绕过面，才算守住了「网络」判据面。

## 四、工具权限门与审计日志：从「能做什么」到「这次允许做什么」

前两层解决「agent 能碰到什么」，这一层解决「agent 这次被允许做什么」——这是唯一能挡住「工具是对的、参数是错的」的那层。

### 工具门的四种粒度：会话级、工具级、参数级、一次性授权

- **会话级**：整场会话给一个权限档。适用 demo；失效场景是任务中途要越界一次，只能整场放宽。
- **工具级**：按工具放行（如允许 `read_file`、禁止 `bash`）。适用多数生产场景；失效场景是工具本身有危险参数。
- **参数级**：对 shell 命令、SQL、文件路径做解析式校验。**这是唯一能挡住「工具对、参数错」的一层**——例如允许 `git commit` 却拒绝 `git push --force`。
- **一次性授权**：逐条 approve，适合高风险操作。失效场景是审批疲劳，用户开始闭眼点「允许」。

现实里这四档不是互斥的：工具级做默认面，参数级做危险工具的二层过滤，一次性授权只留给改配置、删数据、push 主分支这几类高危动作。分层组合比单档更稳，也更能扛住审批疲劳。

参数级门是这四档里最吃工程的一档：shell 要拆 token、挡命令拼接与重定向，SQL 要挡 `DROP` 或没有 WHERE 的 `UPDATE`，路径要 realpath 后比对。解析器写不全，等于留了半扇门；但写对了，它是唯一能拦下「工具合法、参数危险」的那一层。

Open Interpreter 2026 版的 `approval_policy` 就是这三态——`untrusted`/`on-request`/`never`，且策略无法落实时 fail closed（[Sandbox & Approvals](https://www.openinterpreter.com/docs/terminal/sandbox)）。Codex 的 `--ask-for-approval on-request|never` 同理（[approvals](https://learn.chatgpt.com/docs/agent-approvals-security)）。给一段三态策略伪代码：

```python
def gate(tool: str, args: dict, policy: dict) -> str:
    # 返回 allow / deny / require-approval
    if tool in policy["deny"]:
        return "deny"
    for rule in policy["param_rules"].get(tool, []):
        if not rule.match(args):
            return "require-approval"   # 工具允许，参数越界
    if tool in policy["approve_always"]:
        return "allow"
    return "require-approval"
```

反例给一个：Cursor 的 Auto-review 是 3.6 起的推荐默认，但官方文档原话「Auto-review is not a security boundary」（[Run Modes](https://cursor.com/docs/agent/security/run-modes)）。用 LLM 分类器守权限位，等于把安全问题塞回对齐，这不是工具门。

### 审计日志：记什么、谁在看、留多久

审计最小字段集，可直接映射 OTel GenAI 属性（[semconv-genai](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/registry/attributes/gen-ai.md)）：

`trace_id` · `gen_ai.agent.name` · `gen_ai.operation.name=execute_tool` · `gen_ai.tool.name` · `gen_ai.tool.call.arguments` · `gen_ai.tool.call.result` · `gen_ai.tool.call.id` · 决策（allow/deny/ask）· 批准人 · 时间戳 · 耗时。

这套字段能回答四个问题：谁（agent 身份、批准人）、在何时（时间戳）、用哪个工具干了什么（工具名 + 参数摘要）、结果如何（返回值摘要 + 耗时）。字段齐了，一次越界的过程就能完整重建。落地到结构化日志就是把这组字段序列化成 JSON：trace_id 与 `gen_ai.tool.call.id` 做关联键，arguments 存脱敏后的参数摘要，decision 与批准人记决策和责任人。存结构化 JSON 而不是自由文本，是为了能按「批准人」「工具名」「决策」三个维度直接查询——否则日志再多也只是给 grep 用的。

两个坑必须提醒：**决策与批准人不是 OTel 规范字段，是自定义属性**，别写成规范字段；日志要能重建一次决策链——trace_id 串联从请求到执行到批准的全过程，`gen_ai.tool.call.id` 用来把「批准」和「执行」对上账。日志本身不能成为新的泄漏面，密钥与 PII 必须脱敏。

留存时长按合规要求来定，别过度引申：OWASP 当前的 **LLM06:2025 Excessive Agency** 把根因列为 excessive functionality / permissions / autonomy——excessive functionality 对应工具门，excessive permissions 对应文件与进程边界，excessive autonomy 对应一次性授权与审批，三层笼子恰好一一对上（[OWASP](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/)）。EU AI Act Art. 12 要求高风险系统「technically allow for the automatic recording of events (logs) over the lifetime」，Art. 19 要求留存「at least six months」——**但这只适用于被列为高风险的系统**，Annex III 自 2027-12-02、Annex I 自 2028-08-02 适用；通用 agent 平台通常不在此列（[Art.12](https://artificialintelligenceact.eu/article/12/)、[Art.19](https://artificialintelligenceact.eu/article/19/)）。别写成「所有 agent 都必须留 6 个月日志」。另外，留存不是越长越好：日志本身是敏感数据，留得越久越像一个新的攻击面，按「能重建一次决策链」的下限来定、再叠加监管要求即可。

日志缺失时，你无法回答的问题其实就三个：**谁批准的、当时看到了什么、结果是什么**。答不上来，事故就只能靠猜。

**结论：工具门把「能做什么」收窄到「这次允许做什么」，审计日志负责让每一次授权都可重建、可回滚。** 这两样一起，守住「凭据」和「持久化」两个判据面。

## 收尾：笼子的意义是让故障可回滚

三层笼子缺一层等于没配：文件边界关住了读写，但出口全开，agent 照样能把数据带出去；工具门守住了参数，但没有审计，出了事找不到谁批准的。笼子不是不信任模型，而是让每一次故障都能被定位、回滚、追责。

落地顺序按「收益最高、成本最低」排出七天：**先文件边界**（只读 rootfs + 唯一可写挂载点，半天）→ **再出口白名单**（L7 为主 + L4 全拒兜底，两天）→ **再工具门**（参数级门优先，两天）→ **最后补审计**（最小字段集 + 脱敏 + 留存，两天半）。顺序背后的逻辑是：边界不靠 agent 配合，所以最先做；出口要靠代理和网络策略协同，次之；工具门要设计审批流，再次；审计是最后一块，因为它是前几层的「账本」——没有前几层，账本记的是空账。每一步都有对应的验证命令，配完当场验，不留「看起来配好了」。

回到开头那个凌晨三点的场景：你要防的从来不是模型的恶意，而是**配置的默认值**——沙箱默认关、权限默认满、出口默认不限、日志默认不落。把默认值改掉，越界事件的绝大多数就消失了。而「可回滚」落到实操就是两件事：变更前有审批留痕，变更后有日志可查——缺了这两样，回滚只能靠运气。

---

## 参考链接

- [Fortune：Replit AI 删库事件](https://fortune.com/2025/07/23/ai-coding-tool-replit-wiped-database-called-it-a-catastrophic-failure/)
- [AI Incident DB #1152](https://incidentdatabase.ai/cite/1152/)
- [Snyk：恶意 npm 包武装 AI coding agent](https://snyk.io/blog/weaponizing-ai-coding-agents-for-malware-in-the-nx-malicious-package/)
- [Wiz：s1ngularity 供应链攻击](https://www.wiz.io/blog/s1ngularity-supply-chain-attack)
- [NVD CVE-2025-32711（EchoLeak）](https://nvd.nist.gov/vuln/detail/CVE-2025-32711)
- [arXiv 2509.10540（EchoLeak）](https://arxiv.org/html/2509.10540v1)
- [Invariant Labs：GitHub MCP 漏洞](https://invariantlabs.ai/blog/mcp-github-vulnerability)
- [Invariant Labs：MCP 工具描述投毒](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)
- [NVD CVE-2025-48757（Lovable）](https://nvd.nist.gov/vuln/detail/CVE-2025-48757)
- [FT：Amazon Kiro 事故](https://www.ft.com/content/00c282de-ed14-4acd-a948-bc8d6bdb339d)
- [Amazon：Kiro 反驳](https://www.aboutamazon.com/news/aws/aws-service-outage-ai-bot-kiro)
- [Claude Code：Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Codex：Sandbox](https://learn.chatgpt.com/docs/sandboxing)
- [Codex：Approvals](https://learn.chatgpt.com/docs/agent-approvals-security)
- [Aider：Options](https://aider.chat/docs/config/options.html)
- [Cursor：Run Modes](https://cursor.com/docs/agent/security/run-modes)
- [Open Interpreter：Sandbox & Approvals](https://www.openinterpreter.com/docs/terminal/sandbox)
- [Docker：Rootless](https://docs.docker.com/engine/security/rootless/)
- [gVisor：架构导论](https://gvisor.dev/docs/architecture_guide/intro/)
- [Firecracker NSDI'20 论文](https://www.usenix.org/system/files/nsdi20-paper-agache.pdf)
- [Kubernetes：NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes：Declare Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
- [OTel GenAI 语义约定](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/registry/attributes/gen-ai.md)
- [OWASP LLM06:2025 Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/)
- [EU AI Act Art. 12](https://artificialintelligenceact.eu/article/12/)
- [EU AI Act Art. 19](https://artificialintelligenceact.eu/article/19/)
