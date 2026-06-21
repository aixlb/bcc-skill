# bcc-skill

给大模型 / Agent 用的 **bcc** 对接 skill —— 用[包拆拆](https://www.npmjs.com/package/@aixlb/bcc)命令行把视频转成**剧本** / **分镜表** / **风格指南**。

## 安装这个 skill

```bash
npx skills add aixlb/bcc-skill
```

## 它依赖的 CLI

skill 教 Agent 调用的是 `bcc` 命令（npm 包 `@aixlb/bcc`）：

```bash
npm i -g @aixlb/bcc      # 需 Node 20+
bcc --version
```

## 内容

- [SKILL.md](SKILL.md) —— 触发描述 + 对接核心（安装、配置、机器契约、决策提示）
- [reference.md](reference.md) —— 完整命令 / 配置键 / 退出码参考

skill 的核心约定：**stdout 只输出产物绝对路径、stderr 走进度、退出码 0/1**，便于脚本与 Agent 捕获。

---

Created by **@玩AI的小笼包**
