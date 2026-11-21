# Hermes 升级记录：v0.21.0 → v0.23.3

升级时间：2026-11-21 18:00~18:47 CST
执行主机：k6（`wlq-NucBox-K6`），双实例部署（default + chaitin-qa）

## 版本对照

| 项 | 升级前 | 升级后 |
|---|---|---|
| Hermes | v0.21.0 (2026.8.31) | `vgit.caafe53` · upstream `caafe538`（= 官方 v0.23.3，2026-11-20）|
| Python | 3.13.13 | 3.14.9（pm 托管运行时）|
| OpenAI SDK | 2.24.0 | 2.54.0 |
| 依赖数 | 34 | 45 |
| 平台路径 | `gateway/platforms/` | `plugins/platforms/` |
| venv 机制 | venv313 单层 | pm 包管理层（隔离运行时）|

## 升级方式：本地探针浅克隆 + 局域网 fetch

github.com 的 git 通道在代理下只有 **4 MB/分**（68 KB/s），全量 fetch 3.4 万提交需要 4~8 小时，不可行。

改用：

```bash
# 1) 浅克隆官方 tag 到探针目录（430M，几分钟）
git clone --depth 1 --branch v0.23.3 https://github.com/NousResearch/hermes-agent /root/hermes-v0233-probe

# 2) 从本地探针 fetch 进主仓库（局域网，2 秒）
cd /root/.hermes/hermes-agent
git fetch --no-tags --depth 1 /root/hermes-v0233-probe caafe5389ae585ef77387a962aee05b1bcf17029
git checkout -b upgrade-v0.23.3 caafe5389ae
```

**副作用**：主仓库变成浅克隆（3 个 shallow 起点），完整历史只保留在 `upgrade-v0.21.0` 分支上。因此 v0.23.3 分支**无法推送到远端**（远端解包失败），本分支即为替代备份通道。

## 依赖同步

`sync()` 第一步 `pm.venv_is_current()` 要连 GitHub，直连卡死在 SYN-SENT。必须同时给：

```bash
HTTPS_PROXY=http://127.0.0.1:7897 HTTP_PROXY=http://127.0.0.1:7897 \
UV_DEFAULT_INDEX=https://pypi.tuna.tsinghua.edu.cn/simple \
  ./venv313/bin/python -c "from hermes_cli.venv_sync import sync; print(sync())"
# → {'state': 'synced', 'ok': True}
```

PyPI 直连在国内是死路（清华 TUNA 0.96s / 官方源 26s+）。

## 本地补丁移植结论

**5 个本地补丁里只有 ② 需要移植，其余上游都已用别的机制覆盖。**

| # | 补丁 | v0.23.3 实测 | 结论 |
|---|---|---|---|
| ① | qqbot 私聊审批 `chat_type in {"c2c","dm"}` | 上游已含同样判断与注释 | 不用移植 |
| ② | qqbot 无限重连 `MAX_RECONNECT_ATTEMPTS` | 仍 `= 100`；守卫点 3 → **4 处** | **已移植**（本分支 `0001-*.patch`）|
| ③ | `status.py` profile 凭据隔离 | `_render_platforms` 整体重写，改读 `load_gateway_config()`，不再用 `os.getenv` | 等效已修 |
| ④ | `external_autonomous_write` 白名单 | 配置键 0 处引用，但**内容写入已不再按所有权设门槛**（上游注释：「Content writes are not gated by ownership」）| 需求已被覆盖 |
| ⑤ | 浏览器栈统一 AgentBridge | 内建 `browser_*` 工具已被上游删除，`tools/browser_tool.py` 只剩兼容桩；旧工具引用数全 0 | 等效已完成 |

### 判定铁律

**grep 关键词没命中 ≠ 补丁未移植。** 升级前用 `gh api` 搜关键词得到「3 个未移植」全部判错，因为上游换了实现方式：

- ③ 换了整个数据源（env → config），关键词自然搜不到
- ④ 直接取消了那道门槛，补丁要解决的问题消失了
- ⑤ 删掉了旧工具，补丁要删的东西已经不存在

正确顺序：**先读目标函数在新版里的实现 → 再决定要不要移植**。

## 坑与修复

### 1. npm 装错目录导致向量记忆不可用

pm 有两层 venv，长得像但完全不同：

```
installs/<h>/pm-runtime/generations/<id>/    ← 看着像应用 venv，装这里无效
installs/<h>/environments/<h2>/venv/         ← 进程真正 map 的
```

装错**不报错**，只是功能静默不可用。判定只认运行中进程：

```bash
PID=$(systemctl show hermes-gateway -p MainPID --value)
tr '\0' '\n' < /proc/$PID/maps | grep -oE '/root/.hermes/[^ ]*site-packages' | sort -u
```

`vector_memory` 插件需要 `numpy`，缺了会让 `is_available()` 裸 `except` 返回 False，表现为
`Memory provider 'vector_memory' is selected but reports unavailable`，日志里无 traceback。

修法：

```bash
uv pip install --python <environments venv>/bin/python numpy
```

### 2. `is_available()` 吞异常，别信警告文本

该方法实现为 `try: return health()['healthy'] except: return False`。看警告文本会往「凭据/环境变量」方向排查，实际是依赖缺失。直接调 `_get_vm().health()` 拿真实异常。

### 3. `_warn_once` 每进程只警告一次

重启后日志里没有告警 ≠ 已修好，要用进程启动时间界定窗口，并按 PID 归属：

```bash
S=$(systemctl show hermes-gateway -p ExecMainStartTimestamp --value)
journalctl -u hermes-gateway --since "$S" --no-pager | grep -c 'unavailable'   # 期望 0
journalctl ... | grep -i 'unavailable' | grep -oE 'python\[[0-9]+\]' | sort | uniq -c
```

### 4. 不要替用户跑 `hermes gateway install --force`

在 profile 的 `HERMES_HOME` 下执行会装成 **user service** 并立即 start，与已有 system service
抢同一 bot token → flap loop。收拾走纯文件 `mv`（会话内 `systemctl --user` 常连不上 dbus）。

### 5. 重启只能用户在网关外做

`setsid nohup` 无效（命令进程是 gateway 子进程，SIGTERM 连坐）；`systemd-run` 也会被拦
（守卫会读被引用脚本的内容再判断）。只能交用户在网关外执行或 `/restart`。

## 升级后验收结果

```
hermes-gateway              active  PID 2857743  NRestarts=0
hermes-gateway-chaitin-qa   active  PID 2856523  NRestarts=0

default  ✓ qqbot connected  ✓ dingtalk connected  ✓ weixin connected
QA       ✓ qqbot connected  ✓ dingtalk connected

向量记忆  实测召回成功（hybrid，score 0.98+）
记忆库    default 553 条 / QA 66 条（与升级前一致）
ERROR     两实例均 0（排除已知无害噪音）
```

## 已知非阻塞项

| 项 | 说明 |
|---|---|
| `install out of sync (ffmpeg/node/npm/pnpm/ripgrep)` | v0.23.3 新增可选工具，无核心功能依赖 |
| `session_reset.mode: both` 失效 | v0.23 改为只认 `/new` 与 `/reset`；要恢复闲置重置需 `hermes plugins install hermes-session-reset-policy` |
| systemd unit `TimeoutStopSec=90s` < `drain_timeout 210s` | 关机时可能被 SIGKILL；用 drop-in 覆盖而非 `gateway install --force` |
| `skills.external_autonomous_write` 配置键惰性 | v0.23.3 不读它，留着无害（记录当初意图） |
| STANDALONE gateway 提示 | 双实例部署是刻意设计，忽略 `gateway migrate --multiplex` 建议 |

## 回滚

```bash
cd /root/.hermes/hermes-agent && git checkout upgrade-v0.21.0
systemctl restart hermes-gateway-chaitin-qa hermes-gateway
```

完整备份：`/root/backups/hermes-upgrade-20261121_174635`（git bundle 748M + state.db 1.4G + 两记忆库）
补丁文件：`/root/backups/hermes-patch-v0233/0001-k6-patch-qqbot-v0.23.3.patch`（已验证可反向还原）
