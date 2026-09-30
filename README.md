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

根目录的 `小红书视频爆款素材.base` 继续作为底层归档结构。日常浏览入口是主力机的“爆款灵感看板”，不再要求用户打开旧 Bases 视图。每份素材包含封面、标题、封面爆款原因、标题爆款原因、封面标题配合方式、原因标签和互动数据。

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
- 评论：详情抓取后通过同一登录态的签名 Web API 分页读取一级评论，默认最多 20 页，并保留接口直接附带的回复；评论失败不阻塞入库。匿名化评论原文只写入本地 Obsidian/工作台，不进入公开池
- 高评论表现：每日按此前 30 天、视频与图文各自的评论数前 20% 计算门槛；达标笔记由 GPT-6 Luna（`max`）分析评论量原因、互动机制、讨论质量与可复用互动设计。资料领取型互动会与真实讨论明确区分
- 内容范围：标题、正文、封面、原始链接、作者名、发布时间、互动数、标签、命中关键词、选题评分、视频口播转写、视频结构拆解、内容摘要、高赞原因
- 分发规则：GitHub 是完整远端数据表；Obsidian 每日文件夹只写当天首次满足规则的内容，以 `firstQualifiedAt` 为准，不再写近 3 天滚动快照
- 媒体理解：当天新入库视频全部使用 DashScope `paraformer-v2` 文件识别；英文转写按需翻译成中文，不再做 ASR 语义校对；随后 GPT-6 Luna（`max`）基于转写拆解开头、中间、结尾、爆款机制和可借鉴金句。通用封面视觉摘要和视频抽帧理解已停止；Bases 素材卡片保留专用封面与标题分析
- 当天新入库视频默认全部进入 ASR；`select_video_asr.py` 记录预计时长并创建调用预约，但不再做价值跳过或预算截断。只有显式设置 `XHS_VIDEO_ASR_SELECTION_MODE=value` 或 `XHS_VIDEO_ASR_ENFORCE_DAILY_BUDGET=1` 才恢复对应限制
- `enrich_media.py` 在 DashScope 调用边界会强制检查当天预约，直接调用也无法绕过；仅排障时可显式设置 `XHS_ASR_GUARD_OVERRIDE=1`
- 内容理解：媒体增强后调用 GPT-6 Luna（`max`）生成 `contentSummary`、`highLikeReason`、`contentTopicTags`；视频额外生成标题与封面协同字段，并把视频结构拆解用于 GitHub 数据表、Obsidian 每日新增阅读和后续 agent 选题复用
- 封面分类：每日抓取不再调用模型自动分类。爆款封面看板从 Obsidian `Bases素材/_系统/cover-categories.json` 读取用户维护的分类；历史 `coverLayout` 仅用于首次迁移保留原有归类，新封面默认进入“待整理”，由看板拖拽或批量归类。
- 历史图片维护：签名 CDN 图片地址会过期。主流程完成后以低频方式最多修复 3 条缺少本地附件的旧笔记：重新取详情、下载当前图片并更新对应 Obsidian 笔记；失败不会阻断当天抓取与发布。

## 安全冷却后的详情恢复

如果当天搜索已经完成、详情阶段因访问频繁中断，只恢复数据库中当天已发现但缺少详情的候选，不重新搜索：

```bash
$HOME/.venv/openclaw/bin/python3 ~/xhs-content-pool/scripts/resume_details.py --date YYYY-MM-DD --limit 40 --min-delay 30 --max-delay 60
```

脚本会先检查 `state/risk-control.json`；冷却未结束时直接记录 `skipped_risk_cooldown`，不会打开小红书。运行中再次遇到安全限制会立即停止并保留已成功写入的详情。

## 封面分类管理

封面分类不再执行模型回填或自动补齐。通过工作台“爆款封面分析 → 封面看板 → 管理分类”管理分类；已删除分类的封面会保留素材并回到“待整理”。

## 数据边界

这个公开池不包含小红书登录态、cookie、`xsec_token`、评论原文、评论者信息、本机路径、原始视频/抽帧文件或任何凭据。
