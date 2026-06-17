# xhs-content-pool

小红书 AI 爆款内容池。Mac mini 每天早上 05:00 用 `xiaohongshu2.0-skill` 抓取候选笔记，严格筛选最近 72 小时发布且点赞数不低于 100 的 AI 工具 / Codex / Claude Code / Agent / 工作流相关内容，并发布为公开完整数据表。

同时会把当天新入库且已补全详情的合格笔记写入 Mac mini 的 Obsidian vault：

```text
Terry-Knowledge/小红书信源/YYYY/MM/YYYY-MM-DD/*.md
```

公开池地址：

```text
https://raw.githubusercontent.com/zhuangjunhong21-blip/xhs-content-pool/main/pool-notes.json
https://raw.githubusercontent.com/zhuangjunhong21-blip/xhs-content-pool/main/pool-notes-full.json
https://raw.githubusercontent.com/zhuangjunhong21-blip/xhs-content-pool/main/pool-notes-full.jsonl
```

## 入池规则

- 关键词：`AI工具`、`codex`、`Claude Code`、`vibe coding`、`skill`、`agent`、`模型`、`AI工作流`
- 搜索/入库时间窗口：严格最近 72 小时发布
- 搜索前置过滤：小红书 2.0 在截取每次搜索返回数量前，先按 `note_id` 去重，再过滤超过 72 小时和低于 100 赞的候选
- 详情队列二次校验：只给当天新入库候选抓详情；明确超过 72 小时的候选不占详情抓取名额；无法解析的候选仍保留
- 热度阈值：点赞数 `>= 100`
- 去重键：小红书 `note_id`
- 内容范围：标题、正文、封面、原始链接、作者名、发布时间、互动数、标签、命中关键词、选题评分、视觉摘要、视频口播转写、字幕 OCR、视频画面理解
- 分发规则：GitHub 是完整远端数据表；Obsidian 每日文件夹只写当天新增内容，不再写近 3 天滚动快照
- 视觉理解：通过 OpenClaw `minimax-direct/MiniMax-M3` 对当天新增笔记封面做视觉摘要；视频内容优先使用口播转写和字幕 OCR，抽帧只分析画面风格和封面元素；当前只按 `note_id` 与分析状态避免重复处理，不做二级媒体指纹去重

## 数据边界

这个公开池不包含小红书登录态、cookie、`xsec_token`、评论、评论者信息、本机路径、原始视频/抽帧文件或任何凭据。
