# xhs-content-pool

小红书 AI 爆款内容池。Mac mini 每天早上 05:00 用 `xiaohongshu2.0-skill` 从关键词搜索和指定 AI 博主主页抓取候选笔记，并发布为公开完整数据表。关键词源使用最近 168 小时窗口；关注博主源接收过去 48 小时发布且点赞数不低于 100 的笔记。

同时会把当天新入库且已补全详情的合格笔记写入 Mac mini 的 Obsidian vault：

```text
Terry-Knowledge/小红书信源/YYYY/MM/YYYY-MM-DD/*.md
```

当天新增的视频笔记还会生成一份轻量的封面标题素材：

```text
Terry-Knowledge/自媒体100天/AI自媒体号/自媒体沉淀/Bases素材/YYYY/MM/YYYY-MM-DD/*.md
```

根目录的 `小红书视频爆款素材.base` 自动汇总所有日期。每份素材包含封面、标题、封面爆款原因、标题爆款原因、封面标题配合方式、原因标签和互动数据，并关联回完整小红书笔记。分析复用每日 MiniMax 摘要调用，不额外增加一次模型请求。可用 `XHS_BASES_MATERIAL_DIR` 覆盖素材目录。

公开池地址：

```text
https://raw.githubusercontent.com/zhuangjunhong21-blip/xhs-content-pool/main/pool-notes.json
https://raw.githubusercontent.com/zhuangjunhong21-blip/xhs-content-pool/main/pool-notes-full.json
https://raw.githubusercontent.com/zhuangjunhong21-blip/xhs-content-pool/main/pool-notes-full.jsonl
```

## 入池规则

- 关注博主：在 `config/followed-authors.json` 中配置主页链接；每天串行访问，近 48 小时且点赞 `>=100` 的笔记进入独立详情队列
- 关键词：`AI工具`、`codex`、`Claude Code`、`vibe coding`、`skill`、`AI工作流`、`obsidian`、`提示词`、`workbuddy`
- 搜索/入库时间窗口：严格最近 168 小时发布
- 搜索前置过滤：小红书 Provider 在截取每次搜索返回数量前，先按 `note_id` 去重，再过滤超过 168 小时和低于 100 赞的候选
- 详情队列二次校验：只给当天新入库候选抓详情；明确超过 168 小时的候选不占详情抓取名额；无法解析的候选仍保留
- 抓取 Provider：`xiaohongshu2.0-patchright`；`fetch` run state 记录 Provider 与本地健康状态，未来可替换读取层而不改变筛选、SQLite 或分发逻辑
- 热度阈值：点赞数 `>= 100`
- 去重键：小红书 `note_id`
- 来源字段：`sourceChannels` 区分 `keyword_search` / `followed_author`；同一笔记可同时保留两个来源，但只抓一次详情
- 关注博主：默认通过签名 Web API 读取作品，需在独立环境安装 `xiaohongshu-cli==0.6.4`；API 单账号失败时才有限回退到浏览器主页
- 内容范围：标题、正文、封面、原始链接、作者名、发布时间、互动数、标签、命中关键词、选题评分、视觉摘要、视频口播转写、视频画面理解、内容摘要、高赞原因
- 分发规则：GitHub 是完整远端数据表；Obsidian 每日文件夹只写当天首次满足规则的内容，以 `firstQualifiedAt` 为准，不再写近 3 天滚动快照
- 媒体理解：通过 OpenClaw `minimax-direct/MiniMax-M3` 对当天新增笔记封面做视觉摘要；视频先批量判断口播是否能提供 Terry 可直接应用的额外价值，通过后才用 DashScope `paraformer-v2` 文件识别。X 与小红书共用 `~/.terry-automation/asr-budget/` 的每日 3 小时预算；纯视觉演示、广告、Vlog、泛观点、深度工程内容或正文已足够完整的视频不进入付费 ASR；英文转写会翻译成中文，再由 `minimax-direct/MiniMax-N3` 校对 AI 工具术语
- `enrich_media.py` 在 DashScope 调用边界会强制检查当天预算预约，直接调用也无法绕过；仅排障时可显式设置 `XHS_ASR_GUARD_OVERRIDE=1`
- 内容理解：媒体增强后调用 OpenClaw 文本模型生成 `contentSummary`、`highLikeReason`、`contentTopicTags`；视频额外生成 `coverHookReason`、`titleHookReason`、`titleCoverSynergy`、`hookTags`，用于 GitHub 数据表、Obsidian 每日新增阅读、Bases 封面标题素材库和后续 agent 选题复用

## 数据边界

这个公开池不包含小红书登录态、cookie、`xsec_token`、评论、评论者信息、本机路径、原始视频/抽帧文件或任何凭据。
