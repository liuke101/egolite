# ego-browser：WSL 里驱动 Windows 上的 ego lite

让跑在 Linux/WSL 里的 AI agent 直接操作 Windows 上安装的 **ego lite** 浏览器。
Claude Code、Codex、Cursor、OpenCode、DeepSeek Harness，或任何能执行 shell 命令
的自定义 agent 都能用。

目录名 `egolite`；`SKILL.md` 不绑定任何 harness，只依赖 shell、`/mnt/c` 和
WSL interop。

## 原理

```
agent (Linux/WSL)
   └─ scripts/ego-browser  ── WSL interop ──▶  ego-browser.exe (Windows 进程)
                                                   └─ 正在运行的 ego lite
                                                      （用户真实 profile / 登录态 / Space）
```

- 不需要插件、端口、扩展或 CDP：脚本整个进程在 Windows 侧执行，由它自己连正在运行的
  ego lite。WSL 的网络模式（NAT / Mirrored）都不影响。
- 不复制一份浏览器 API 文档：skill 第一步执行 `ego-browser skill`，读的是随已安装版本
  发布、版本匹配的用法指南；ego lite 升级后指南自动更新。
- `scripts/winpath` 把 Linux 路径转成 Windows 可用的
  `//wsl.localhost/<distro>/...` 或 `C:/...`，用于截图、上传、下载往返工作区。

## 目录结构

```
egolite/
└── ego-browser/                # skill bundle（整个目录就是安装单位）
    ├── SKILL.md                # 触发描述 + 使用规则（agent 读）
    └── scripts/
        ├── ego-browser         # WSL → Windows 桥：参数/stdin/退出码原样透传
        ├── winpath             # Linux 路径 → Windows 路径
        └── ego-skill-dir       # 定位 ego lite 官方技能 bundle（读 references/）
```

## 前置条件

1. Windows 上已安装 ego lite，并完成首次 onboarding（Windows 里 `ego-browser`
   命令可用）。
2. agent 的 shell 在 WSL/Linux：`/mnt/c` 可访问，interop 开启
   （`cat /proc/sys/fs/binfmt_misc/WSLInterop` 包含 `enabled`）。
3. agent 在 Windows 或 macOS 原生运行时不适用——直接用 ego lite 自带的 skill，
   见 `SKILL.md` 的 **8. Other hosts**。

## 与官方 ego-browser 技能的关系

本技能不是官方技能的改写，也不是替代品，而是**WSL 桥接层 + 官方技能的入口**：

| 内容 | 来源 |
|---|---|
| 先 `ego-browser skill` 读版本匹配指南、结尾标记校验、截断处理 | 沿用官方 SKILL.md 的 bootstrap 约定 |
| 浏览器 API、Space 生命周期、`handOff()`、`claim/takeOver` 限制、profile 选择 | 官方运行指南（`ego-browser skill` 输出，每次实时读取） |
| 清理 cookie/缓存/storage 的破坏性说明 | 官方 `references/clearing-state.md`：技能内写了精简规则，并指向原文 |
| WSL 桥接、`winpath`、跨 harness 安装、其他宿主、故障排查 | 本技能新增 |
| 官方 `references/`、`learnings/`、`agents/openai.yaml`、macOS 安装脚本 | **不复制**，用 `scripts/ego-skill-dir` 定位后直接读原文件 |

设计原因：API 随 ego lite 版本变化，复制一份必然漂移；官方技能随安装包发布，而
`active_version_dir` 是升级后不变的稳定路径，所以让 agent 读原文，本技能只补它没覆盖
的 WSL 侧。没有逐字复制官方文案（只引用了 `[ego-browser:skill] end of guide`
这个协议标记），避免许可和版本同步问题。

已知缺口：官方 bundle 里给 Codex 用的 `agents/openai.yaml`（显示名、图标、
default_prompt）没有照搬；本技能的 SKILL.md 面向通用 `SKILL.md` 协议。

## 安装

技能的本质是「一个目录里有 `SKILL.md`」。把 `ego-browser/` 放进 harness 会扫描的技能
根即可：

| harness | 建议位置 |
|---|---|
| DeepSeek Harness | `~/.dsh/skills/` 或 `<项目>/.dsh/skills/` |
| Claude Code | `~/.claude/skills/` 或 `<项目>/.claude/skills/` |
| Codex / Cursor / OpenCode / 其他 | 查其文档中的 skills 目录；同样要求 `<name>/SKILL.md` 结构 |
| 不扫描技能目录的 harness | 直接把 `SKILL.md` 的绝对路径交给 agent，让它读取并执行 |

```bash
# 以 DeepSeek Harness 为例
mkdir -p ~/.dsh/skills
cp -r egolite/ego-browser ~/.dsh/skills/
chmod +x ~/.dsh/skills/ego-browser/scripts/*
```

技能里所有相对路径都相对 `SKILL.md` 所在目录，所以复制到任何位置都不需要改内容。
不要和 ego lite 官方自带的 `ego-skills/ego-browser` 同时装进同一个技能根：两者同名，
本包是它的 WSL 桥接版。

## 验证

```bash
EB=egolite/ego-browser/scripts/ego-browser
"$EB" --version              # ego-browser 0.6.3.17 / chromium / node
"$EB" skill | tail -1        # [ego-browser:skill] end of guide v2.2.1
egolite/ego-browser/scripts/winpath "$PWD/out.png"
# -> //wsl.localhost/<distro>/<cwd>/out.png
egolite/ego-browser/scripts/ego-skill-dir
# -> .../active_version_dir/ego-skills/ego-browser  （官方技能原文目录）
```

在任意支持 skill 的 agent 里说「用浏览器打开 example.com」，agent 应加载技能，并在
ego lite 的 Space 中执行。

## 限制

- **脚本跑在 Windows 侧**：`process.platform` 是 `win32`，cwd 是 ego lite 安装目录，
  脚本里只能用 Windows 路径。
- **只支持 WSL/Linux 桥接**：agent 在无 interop 的容器、远程机器里时无法触达用户浏览器；
  skill 要求明确报告，而不是换成 curl 或 Playwright 假装完成。
- **会动真浏览器**：真实 profile 与会话，发帖、支付、删除、改设置必须在对话中明确确认；
  登录/验证码用 `handOff()` 交还给用户。
- **ego lite 升级**：桥优先用
  `C:\Users\<用户>\.local\share\ego\active_version_dir\ego-browser.exe`（升级后路径不变），
  找不到时回退到 `Program Files` 下最新版本目录；可用 `EGOLITE_BROWSER_EXE` 显式指定。
- **多实例**：桥会把 `--ego-server-name` 等参数原样透传，用于选择命名实例。

## 卸载

删除技能根下的 `ego-browser/` 目录即可。

## 本机验证记录（2026-11-18）

- 环境：WSL2 + `networkingMode=Mirrored`，interop 开启；ego lite `0.6.3.17`
  （Chromium 156 / Node v24.22.1），装在 `C:\Program Files\Citro Labs\ego lite`。
- 桥接：`--version`、`skill`（v2.2.1 指南，结尾标记完整）。
- 端到端：全新会话给中性任务「打开 example.com 读标题和首段」，agent 自动选择本技能，
  在 Space 5 完成并 `finish()` 关闭，未使用 curl 或其他浏览器工具。
- 文件往返：`winpath` 输出的 `//wsl.localhost/...` 可直接用于
  `page.screenshot({ path })`，截图成功落回 Linux 工作区。
- 已安装（示例）：`/home/lk/Test/.dsh/skills/ego-browser`。
