# xhs-content-pool

小红书 AI 爆款内容池。Mac mini 每天下午用 `xiaohongshu2.0-skill` 抓取候选笔记，严格筛选最近 3 天发布且点赞数不低于 100 的 AI 工具 / Codex / Claude Code / Agent / 工作流相关内容，并发布为公开 JSON。

公开池地址：

```text
https://raw.githubusercontent.com/zhuangjunhong21-blip/xhs-content-pool/main/pool-notes.json
```

## 入池规则

- 关键词：`AI工具`、`codex`、`Claude Code`、`vibe coding`、`skill`、`agent`、`模型`、`AI工作流`
- 时间窗口：严格最近 3 天发布
- 搜索前置过滤：小红书 2.0 在截取每次搜索返回数量前，先按 `note_id` 去重，再过滤超过 72 小时和低于 100 赞的候选
- 详情队列二次校验：明确超过 3 天的候选不占详情抓取名额；无法解析的候选仍保留
- 热度阈值：点赞数 `>= 100`
- 去重键：小红书 `note_id`
- 内容范围：标题、正文、封面、原始链接、作者名、发布时间、互动数、标签、命中关键词

## 数据边界

这个公开池不包含小红书登录态、cookie、`xsec_token`、评论、评论者信息、本机路径或任何凭据。
