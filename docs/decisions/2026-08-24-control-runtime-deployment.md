# Control、Server 与远程 Runtime 的部署所有权

- 状态：Accepted
- 日期：2026-08-24
- 最后核验：2026-08-28，当前 control、deploy-host、Runtime 与 Codex Desktop Bridge 实现
- 适用边界：原生 systemd、Docker deploy-host、macOS Runtime 和 Desktop App 连接所有权
- 失效条件：部署所有者、制品拓扑或 Desktop Bridge 被新的 Accepted ADR 明确取代

本文描述当前所有权边界。版本、协议和持久化判断同时遵循[版本、兼容与迁移所有权](./2026-08-27-versioning-and-compatibility.md)与[Codex Desktop App 任务所有权](./2026-08-23-unified-workspace-chat.md)。

## 问题

Linux 公网 Web 需要统一认证、持久化和部署入口，但 Codex App 项目、任务与 writer 只存在于用户 Mac。Linux 复制 App 状态或自启 Codex App Server 会形成第二权威。容器与 systemd 若同时拥有同一服务、数据库或 Runtime 激活，也会破坏更新和回滚的唯一生命周期所有者。

## 决定

交付 `cc-connect-control`、`cc-connect-server`、Linux `cc-connect-deploy-host` 和 macOS `cc-connect-runtime` 四类制品。原生安装由 systemd 管理 control；容器安装由 systemd 只管理 deploy-host，deploy-host 管理 control 容器，control 通过私有 Unix Socket 监管 server。两条 Linux 通道不得共用持久目录。

control 是认证、设备、server、Release、执行槽、日志和跨重启 activation 的唯一生命周期所有者。Codex Desktop App 独占项目、task、Turn、历史和 writer；Runtime 只代理经过审核的 App tools 语义能力。SessionManager 只保存平台用户到 App task ID 的选择关系，服务器不复制对话正文。

Runtime 使用出站 TLS/WebSocket 和 Ed25519 challenge-response 连接 control。由于当前 App 私有 Socket 的执行上下文限制，Runtime 不由 launchd 后台启动；安装器负责验签、安装和配对，然后必须在当前 Codex App 交互终端中启动 launcher。launcher re-exec 为 App 内置 Node supervisor；supervisor 是持续生命周期所有者，不随启动终端挂断或单个 worker 退出而结束。worker 仅使用继承的双向 FD，不启动 App Server。

systemd 模式下 control 拥有签名 Release 的在线更新、回滚和跨重启 activation。Docker 模式下 control 拥有部署业务事务，deploy-host 独占 control 容器切换和 watchdog 回滚。control 不挂载 Docker Socket，只能通过 peer credential 校验的 Unix Socket 请求固定 Release/tag 操作。

control 与 deploy-host 使用唯一容器宿主协议指纹；不匹配只返回 `update_required`。Runtime protocol 只承载项目、任务、快照、等待、发送、创建和元数据操作。候选确认同时校验正在运行的 control 编译版本、激活目标和宿主持久状态，旧镜像不能确认新目标。

Runtime 连接代际以 `control.db` 的最后 checkpoint 为下界。每次连接拥有独立 context；断线会取消并等待该代 RPC 和 task 观察，旧响应不得写入新连接。设备撤销会同步关闭活动连接。设备连接事件写入 `control.db.audit_events`，作为 Web 查询和实时日志的持久来源。

Release manifest 固定仓库、workflow、tag、commit、八个目标制品及协议/数据库元数据。control、deploy-host 和 Runtime 均验证 GitHub OIDC/Sigstore identity、manifest 和 SHA-256；deploy-host 还验证固定 GHCR 镜像签名，不提供未签名 fallback。

## 被考虑的替代方案

- Linux 或 macOS Runtime 自启 Codex App Server：无法复用当前 App writer，会形成第二任务权威。
- launchd 后台直连 App tools Socket：当前 App 会拒绝该执行上下文，因此安装器移除旧 `dev.cc-connect.runtime` LaunchAgent。
- VPN 或反向穿透到 macOS：出站 TLS 长连接已覆盖需求，入站网络只会扩大设备攻击面。
- systemd 分别管理 control 和 server：部署事务无法原子协调业务活动、日志、候选健康和数据库恢复。
- 把 Docker Socket 交给 control：扩大容器权限并产生第二个宿主生命周期所有者。

## 兼容与迁移

旧 management token、业务进程公开 TCP、`cc-connect web`、CLI update、`cc-connect daemon`、服务器 `template_project`、本地 App Server 路径、WorkspaceChat 独立协议和 Runtime LaunchAgent 均删除，不保留双协议或兼容解析。

`control.db` 使用显式事务迁移保存永久控制状态。`workspace_chat.db` 已从当前架构删除；旧部署必须停服后精确删除数据库、sidecar 和已核验附件目录，不做迁移或重建。

## 事务与生命周期

systemd 模式下 DeploymentManager 锁定 Release 和机器级执行槽，再验签、检查活动操作、检查磁盘、暂存在线 Runtime、备份 `control.db` 并写 activation record。进入 server stop 后运行不可取消；随后切换 `current` 并激活 Runtime，交给 systemd 启动候选 control。

候选 control 恢复 Runtime 连接并确认激活；Runtime 在确认前保留本地 watchdog。候选健康后提交 run 并删除 activation；候选失败时稳定 helper 恢复旧槽、数据库和 Runtime。Docker 通道使用独立 `container-activation.json`，由 deploy-host 切换已验证 digest 并在超时后恢复 previous 状态。

Runtime 关闭只结束代理连接和 task 观察，不关闭 Codex App 或任务。App Socket 断开、Runtime 更新和 worker 退出都由 Node supervisor 统一清理旧代，再从 `current` Release 建立新代际；终端 `SIGHUP` 不结束 supervisor。Runtime 激活按目标 tag 幂等：重复请求或目标已是 `current` 时仍建立或复用 pending activation，并通过同一 confirm/rollback 生命周期收口。control 只把设备标为离线，不能启动替代 writer。

## 架构风险目录

主要风险包括多 Mac 隔离、App 终端生命周期、Socket/schema 变化、设备离线、活动操作阻断、签名摘要、数据库备份、两阶段恢复、宿主 peer credential、固定命令边界、watchdog、执行槽、非 root 容器、loopback 暴露和持久目录契约。验证按实际影响选择，但不得用历史成功代替当前 Release、服务和 App 连接状态。
