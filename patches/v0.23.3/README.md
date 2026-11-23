# Hermes Agent v0.23.3 本地补丁存档

本目录是本机（k6 / 办公室实例）在 **Hermes Agent v0.23.3** 上的全部源码改动，
以补丁形式存档，避免依赖本地浅克隆的祖先链。

## 基线

| 项 | 值 |
|---|---|
| 上游版本 | v0.23.3（2026-11-20）|
| 基线提交 | `caafe5389ae585ef77387a962aee05b1bcf17029` |
| 上游仓库 | https://github.com/NousResearch/hermes-agent |

> 基线提交是上游公开 tag，可从上游仓库完整取得；本目录补丁叠加其上即为本机完整源码。

## 补丁清单（新 → 旧）

| 文件 | 内容 |
|---|---|
| `0001-k6-patch-qqbot-v0.23.3.patch` | qqbot 无限重连移植到 v0.23.3（`MAX_RECONNECT_ATTEMPTS = None` + 3 处守卫）|
| `0002-k6-patch-systemd-TimeoutStopSec-unit.patch` | 修 `shutdown_forensics.py` 误报：unit 不存在时读到了 systemd 内置默认值（90s），应校验 `LoadState`/`FragmentPath` |
| `0003-chore-venv.patch` | 忽略本地 `venv` 兼容软链（`/venv -> venv313`）|
| `0004-fix-dingtalk-validate-business-ack-and-isolate-react.patch` | 钉钉：校验 business ack、隔离 reactions（含 133 行测试）|
| `0005-fix-gateway-quiet-auth-fallback-graceful-dingtalk-shutdown.patch` | 网关：auth 回退静默化（不再刷错误日志）、钉钉优雅关停（auth 存储读失败与队列表未就绪时不再抛栈）|

合计 11 个文件、+239 / -39 行。

## 应用方法

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
git checkout caafe5389ae585ef77387a962aee05b1bcf17029
git am /path/to/patches/v0.23.3/*.patch
```

## 为什么用补丁而非分支

本机仓库是浅克隆，其祖先链与远端 fork 都缺对应对象。
GitHub 不支持从浅克隆推送（`remote: fatal: did not receive expected object`）。

2026-11-23 复核缺口的精确位置：

| 提交 | 角色 | 本地 | 远端 fork |
|---|---|---|---|
| `caafe5389a` | 基线（本机分支根）| ✅ | ❌ |
| `49b1b513` | `caafe5389a` 的真实父 | ✅ | ❌ |
| `535049c2` | `49b1b513` 的真实父 | ❌ | ❌ |

远端 fork 的 `main` 停在 `0a8092ac`（**2026-08-16**），
而基线是 **2026-11-19**，中间约 3 个月的上游提交两边都没有；
补齐需拉取该区间全部历史，代理链路实测约 33KB/s 且此前两次中途断流，不可行。

补丁方式：**45KB、可离线、可逐条 review**，且基线是公开 tag，随时可完整重建。

## 已验证

- `git format-patch` 从真实提交导出，非手工编写
- 主工作区未受任何影响（操作在独立 worktree 中完成）
- 0001–0005 依次 `git am` 到基线后，文件树与本机 `upgrade-v0.23.3` 分支 tip（`ed08327df4a`）**逐字节一致**（`git diff` 为空）
- 应用后测试通过：`run_tests.sh` 三文件 43 项全绿
