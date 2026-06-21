# bcc 命令参考

`bcc` v0.2.0+（npm 包 `@aixlb/bcc`）。`npm i -g @aixlb/bcc` 安装，命令为 `bcc`。

## 全局

```
bcc <命令> [参数]
bcc --help | -h            # 帮助
bcc --version | -v         # 版本
```

公共参数（联网命令通用）：

| 参数 | 简写 | 说明 |
|------|------|------|
| `--out <file>` | `-o` | 输出路径；给扩展名就按扩展名推断格式，给目录就用默认文件名 |
| `--format a,b` | `-f` | 批量导出多种格式（逗号分隔） |
| `--data-dir <dir>` | | 指定应用目录（内含 config.yaml 与 data/） |
| `--quiet` | `-q` | 静默进度（stdout 只剩产物路径） |
| `--verbose` | | 打印数据根等诊断到 stderr |

**输出契约**：stdout 每行一个产物绝对路径；stderr 是进度/提示；退出码 0 成功、1 失败。

## bcc config

```
bcc config list                 # 打印全部配置（JSON，密钥打码）
bcc config                      # 同 list
bcc config get <key>            # 读取单项（点号路径），密钥打码
bcc config set <key> <value>    # 写入单项，按既有值类型强转（数字/布尔/字符串）
bcc config path                 # 打印 config.yaml 路径
bcc config test [provider]      # 测连接：无参=测所有已配 Key 的模型；带参只测该 provider
```

- `set` 的类型强转：原值是数字则要求数字；是布尔则 `true`/`1` 为真；原值为 null/空则 `null` 或空串写回 null。
- `set` 只能改**已存在**的键，未知键报错。
- `config test` 内置三家（gemini / kimi / minimax）并发实测；自定义供应商标 SKIP；任一 FAIL → 退出码 1。
- `--json` 可加在 list/get 上（输出仍是 JSON）。

### 常用配置键

| 键 | 含义 | 示例值 |
|----|------|--------|
| `gemini.api_key` | Gemini Key（默认 provider） | `sk-...` |
| `gemini.model` | Gemini 模型名 | `gemini-3.1-pro-preview` |
| `gemini.video_resolution` | 视频精度 | `low` / `high` |
| `gemini.proxy` | 代理（留空=跟随系统） | `http://127.0.0.1:7890` |
| `kimi.api_key` / `kimi.model` | Moonshot Kimi | `kimi-k2.6` |
| `minimax.api_key` / `minimax.model` | MiniMax | — |
| `script_provider` | 剧本用哪家 | `gemini` / `kimi` |
| `storyboard_provider` | 分镜用哪家 | `gemini` / `kimi` / `openai` |
| `storyboard.scene_threshold` | 切镜阈值 | `0.3` |
| `storyboard.max_shots` | 最大镜头数 | `200` |
| `storyboard.concurrency` | 单视频内分析并发 | `3` |
| `storyboard.prompt` | 自定义分镜 prompt | （留空用默认） |
| `research.video_concurrency` | 研究时并行视频数 | `2`（1..4） |
| `research.synthesis_provider` | 合成 provider | `null`=继承 storyboard_provider |
| `research.mosaic_frames_per_video` | 每视频拼帧数 | `6` |

完整默认值见仓库 `config.example.yaml`。

## bcc script

```
bcc script <video|projectId> [--out file] [--format docx,md,txt,html]
                             [--save-project] [--data-dir d] [--quiet] [--verbose]
```

- 输入是视频文件路径，或已建项目的 `projectId`。
- 默认格式 `docx`；可选 `docx,md,txt,html`（md/txt 内容相同，是带时间戳的剧本纯文本）。
- `--save-project`：把裸视频也建成项目入库（生成 projectId，剧本写进项目 `script.md`）。
- 默认文件名取视频基名。
- stderr 收尾打印「剧本生成完成（N 场，M 字）」。

## bcc storyboard

```
bcc storyboard <video|projectId> [--out file] [--format xlsx,html]
                                 [--save-project] [--data-dir d] [--quiet] [--verbose]
```

- 流程：检测镜头 → 截帧 → 并发分析 → 导出。
- 默认格式 `xlsx`；可选 `xlsx,html`（html 是带关键帧缩略图的可视化分镜表）。
- 给 `projectId` 时帧落进项目帧目录（持久化）；裸视频用临时帧目录，导出后清理。
- 切镜/并发等行为受 `config storyboard.*` 控制。
- stderr 收尾打印「分镜完成（N 镜）」。

## bcc research

```
bcc research <video...> [--input-dir dir] [--project id] [--name 名称]
                        [--out file] [--format md,txt] [--data-dir d] [--quiet] [--verbose]
```

- 对多条视频各自拆分镜，再合成一份《风格指南》。
- 输入三选一/可叠加：直接列多个视频路径；`--input-dir` 扫目录（支持 mp4/mkv/mov/webm/avi/flv/wmv/m4v，去重排序）；`--project <id>` 复用已建研究项目。
- 默认格式 `md`；可选 `md,txt`。
- `--name` 自定义研究项目名（不给则用 `风格研究_<首个视频名>等N条`）。
- 新建研究时 stderr 打印 `研究项目已建: <id>（N 条视频）`，可记下该 id 后续 `--project` 复用。

## 退出码与错误

| 情况 | 退出码 | 输出位置 |
|------|--------|----------|
| 成功 | 0 | 产物路径 → stdout |
| 用法错误（缺参/未知命令/不支持格式） | 1 | `错误: ...` → stderr |
| 运行失败（连接/文件/模型） | 1 | `失败: ...` → stderr |
| `config test` 有模型 FAIL | 1 | 每模型一行 → stdout |

## 集成片段（捕获产物路径）

```bash
# 单格式：直接拿路径
docx=$(bcc script ./a.mp4 --out ./out/剧本.docx --quiet) && echo "生成: $docx"

# 多格式：逐行就是各产物
bcc storyboard ./a.mp4 --format xlsx,html --quiet | while read -r f; do
  echo "产物: $f"
done

# 固定数据根，避免容器内落到平台 userData
BCC_DATA_DIR=./work bcc config test
```
