# session-extractor

按需把 **agent 会话记录**转换成结构化 Markdown 到 `.session-extractor/` 目录的 **skill**。
是 [`claude-session-log`](https://github.com/AndyKong2020/awesome-claude-infra/tree/main/plugins/claude-session-log) /
[`opencode-session-log`](https://github.com/AndyKong2020/opencode-session-log) 的**简化、非实时、skill 化**版本：

- **不用 hook、不实时**：由用户/agent 主动触发，一次性扫描当前会话并重建 Markdown。
- **多次触发增量幂等**：同一 session 始终写到同一目录，整体覆盖刷新，不漂移、不重复。
- **两种形态**：`structured`（默认，meta/summary 双树）与 `flat`（每 agent 一个自包含大 md）。
- **多 agent / subagent / 多 session**。
- **跨平台**：统一中间表示（IR）+ 适配器，已支持 **Claude Code / Codex / Opencode** 三平台（均在本机真实数据上端到端验证，含 subagent / 增量）。

## 安装

作为 Claude Code 插件（marketplace 外链）：

```bash
/plugin marketplace add AndyKong2020/<marketplace>
/plugin install session-extractor@<marketplace>
```

或直接克隆本仓库使用其中的脚本（零运行时依赖，仅 Python3 标准库）。

## 使用

由 skill 触发（描述匹配「导出/归档/快照当前会话为 markdown」），或手动运行：

```bash
python3 skills/session-extractor/scripts/extract.py [--platform claude|codex|opencode] [--mode structured|flat|both]
                                                    [--session ID | --all] [--out DIR]
                                                    [--cwd DIR] [--config-dir DIR] [--quiet]
```

平台由 agent **自行判断**并经 `--platform` 显式传入（skill 场景，最可靠、无嵌套歧义）；省略 `--platform` 时回退按环境变量自动探测（standalone 便利）。默认导出当前会话、`structured`、到 `<cwd>/.session-extractor`。详见
[`skills/session-extractor/SKILL.md`](skills/session-extractor/SKILL.md)。

## 输出结构

结构对齐 `.agents-log`（meta 双树 + 全局 index/state/locks 均在 `meta/` 下）：
```
.session-extractor/
├─ summary/<platform>/<时间戳>/                # structured：人读总览（单本地时间戳文件夹）
│  ├─ summary.md  usage.json
│  └─ agents/<key>/{summary.md,usage.json}
├─ meta/                                       # structured：详细+取证+索引+状态
│  ├─ index.md                                # 全局索引（所有会话）
│  ├─ state/<platform>__<sid>.json            # 幂等状态（产物相对路径）
│  ├─ locks/                                  # 会话级文件锁
│  └─ sessions/<platform>/YYYY/MM/<sid>/
│     ├─ index.md  merged/session.md
│     ├─ agents/<key>/session.md
│     └─ artifacts/{shared,<key>}/rendered/
└─ <platform>/<时间戳>/                         # flat：每 agent 一个大 md（单本地时间戳文件夹）
   ├─ index.md  main.md  <subagent>.md  usage.json
```

## 架构

```
skills/session-extractor/scripts/
├─ extract.py                 # CLI 入口与编排
└─ sessionlog/
   ├─ ir.py                   # 归一化中间表示 Session/Agent/Event/Block/Usage
   ├─ util.py                 # 纯函数工具
   ├─ discover.py             # 平台探测 + 公共助手
   ├─ render.py               # IR -> Markdown（平台无关）+ 产物外溢/去重
   ├─ layout.py               # structured/flat 目录布局 + 幂等编排 + 全局索引
   ├─ state.py                # per-session 状态 + 文件锁
   └─ adapters/
      ├─ claude.py            # Claude Code transcript+telemetry -> IR
      ├─ codex.py             # Codex rollout(+subagent 文件) -> IR
      └─ opencode.py          # Opencode 只读 SQLite(session/message/part) -> IR
```

**渲染器只认 IR**，三平台差异全部收敛在各自适配器里——这是「一套渲染服务三平台」的关键。

## 设计原则

- **不写死本地环境**：路径走 `$CLAUDE_CONFIG_DIR` / `$CODEX_HOME` / `$XDG_DATA_HOME` 等 env + 约定回退。
- **缺权威信号即 fail loud**：不堆冗余兜底（避免代码全靠兜底跑起来掩盖问题）；对*可选*输入才用合法默认。
- **只读**原始会话数据。

## 测试

```bash
python3 -m unittest discover -s skills/session-extractor/scripts/tests
```

## 平台支持

| 平台 | 状态 | 存储 | env 回退探测信号（省略 --platform 时）|
|---|---|---|---|
| Claude Code | ✅ 已支持 | `${CLAUDE_CONFIG_DIR:-~/.claude}/projects/<enc>/<sid>.jsonl` | `CLAUDE_CODE_SESSION_ID` |
| Codex | ✅ 已支持 | `${CODEX_HOME:-~/.codex}/sessions/YYYY/MM/DD/rollout-*.jsonl` | `CODEX_SANDBOX` / `CODEX_THREAD_ID` |
| Opencode | ✅ 已支持 | `${XDG_DATA_HOME:-~/.local/share}/opencode/opencode.db`（只读 SQLite） | `OPENCODE=1`（带 PID 存活性闸门）|

三平台均已在本机真实数据上完整验证（含 subagent、增量触发、客户端路由）。
