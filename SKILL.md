---
name: bcc-cli
description: 用包拆拆命令行工具 bcc 把视频转成剧本(docx/md/html)、分镜表(xlsx/html)或风格指南(md)。当用户提供视频文件并要求「拆剧本/生成分镜/分镜表/storyboard/风格研究/风格指南」，或提到 bcc / 包拆拆 / @aixlb/bcc CLI 时使用。这是给其他大模型/Agent 对接 bcc 的集成说明：覆盖安装、配置、四条命令、stdout/stderr 机器契约与退出码。
---

# bcc — 包拆拆命令行 (给 Agent 的对接指南)

`bcc`（npm 包 `@aixlb/bcc`）是「包拆拆 / XLB Storyboard」的纯 Node 命令行，把视频喂给多模态大模型，产出三类成品：

| 命令 | 作用 | 默认产物 | 可选格式 |
|------|------|----------|----------|
| `bcc script <视频>` | 拆**剧本** | `.docx` | docx, md, txt, html |
| `bcc storyboard <视频>` | 拆**分镜表** | `.xlsx` | xlsx, html |
| `bcc research <视频...>` | 多视频合成**风格指南** | `.md` | md, txt |
| `bcc config <子命令>` | API Key / 模型 / 代理设置 | — | — |

## 前置：安装

```bash
npm i -g @aixlb/bcc      # 需 Node 20+；安装后命令为 bcc
# 若 npm 用国内镜像、提示找不到包，直连官方源：
npm i -g @aixlb/bcc --registry https://registry.npmjs.org/
```

视频探测/截帧依赖随包的 ffmpeg-static / ffprobe-static，**无需系统装 ffmpeg**。验证：`bcc --version`。

## 机器契约（对接最关键的部分）

把 bcc 当工具调用时，按这套约定解析，绝不会出错：

- **stdout 只输出产物的绝对路径**，每行一个文件。多格式导出就是多行。用最后的非空行（或全部行）作为结果文件。
- **stderr 是进度和人类提示**（百分比、"已写入 ..."、诊断）。解析结果时忽略它，或加 `--quiet` 让它闭嘴。
- **退出码**：`0` 成功；`1` 失败（用法错误 / 连接失败 / 文件不存在等）。`config test` 有任一模型 FAIL 也返回 `1`。
- 失败信息写到 stderr，形如 `错误: ...`（用法类）或 `失败: ...`（运行类）。

Agent 调用范式（务必带 `--quiet`，并捕获 stdout）：

```bash
out=$(bcc script ./a.mp4 --out ./剧本.docx --quiet) || { echo "bcc 失败"; exit 1; }
# $out 即生成的 .docx 绝对路径
```

## 前置：配置 API Key（首次必做）

`script` / `storyboard` / `research` 都要联网调模型，先配好至少一个 provider 的 Key：

```bash
bcc config set gemini.api_key sk-xxxx     # 申请: https://aistudio.google.com/apikey
bcc config test                           # 测试所有已配 Key 的模型连通性
bcc config list                           # 查看全部配置（密钥自动打码）
```

默认 provider 是 gemini。Key 写进 `config.yaml`（与桌面端 GUI 共用同一份配置与项目库）。常用键见 [reference.md](reference.md)。

## 数据根 / 配置文件位置

`config.yaml` 与项目库的根目录按以下优先级解析（前者优先）：

1. `--data-dir <应用目录>` 参数
2. `BCC_DATA_DIR` 环境变量
3. 当前工作目录（cwd 下存在 `config.yaml` 或 `data/` 时）
4. 平台 userData（与安装版 GUI 共用，需 productName = `XLB Storyboard`）

给定的目录是「应用目录」，内含 `config.yaml` 与 `data/`。无状态/容器化调用时建议显式 `--data-dir ./work` 或 `BCC_DATA_DIR=./work` 固定下来。

## 常用调用

```bash
# 拆剧本：一次出 docx + md + html 三种
bcc script ./a.mp4 --format docx,md,html --quiet

# 拆分镜：出 xlsx 与可视化 html
bcc storyboard ./a.mp4 --out ./分镜.xlsx --format xlsx,html --quiet

# 风格研究：多条视频合成一份风格指南
bcc research a.mp4 b.mp4 c.mp4 --out 风格指南.md --quiet
bcc research --input-dir ./clips --out 风格指南.md --quiet   # 扫目录里所有视频

# 复用已存在的项目（用 projectId 替代视频路径）
bcc script <projectId> --out 剧本.docx
bcc research --project <researchId> --out 风格指南.md
```

`--out` 的语义：给文件名（带扩展名）就按扩展名定格式；给目录就用默认文件名落在该目录；配合 `--format a,b` 批量导出多份。完整参数表、配置键清单、各命令细节见 **[reference.md](reference.md)**。

## 决策提示（给 Agent）

- 用户给**单个视频**要「剧本/台词/对白」→ `bcc script`。
- 要「分镜/镜头表/storyboard/逐镜头」→ `bcc storyboard`。
- 给**多个视频**要「风格/调性/统一指南/参考」→ `bcc research`。
- 报 `fetch failed` / 连接超时 → 网络到 Google 不通，提示用户配 `gemini.proxy` 或设 `HTTPS_PROXY`，再 `bcc config test` 复测。
- 报「找不到视频文件或项目」→ 确认传的是存在的视频路径或正确的 projectId。
- 始终对调用结果用退出码判断成败，用 stdout 取产物路径，不要解析 stderr 文案。
