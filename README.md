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

根目录的 `小红书视频爆款素材.base` 继续作为底层归档结构。日常浏览入口是主力机的“爆款灵感看板”，不再要求用户打开旧 Bases 视图。每份素材包含封面、标题、封面爆款原因、标题爆款原因、封面标题配合方式、原因标签、互动数据，以及 MiniMax 输出的结构化封面版式分类。

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
- 媒体理解：通过 OpenClaw `minimax-direct/MiniMax-M3` 对当天新增笔记封面做视觉摘要；当天新入库视频全部使用 DashScope `paraformer-v2` 文件识别，英文转写翻译成中文，再由 `minimax-direct/MiniMax-N3` 校对 AI 工具术语。共享 ASR 账本继续记录预计时长，但默认只用于统计，不截断小红书转写
- 当天新入库视频默认全部进入 ASR；`select_video_asr.py` 记录预计时长并创建调用预约，但不再做价值跳过或预算截断。只有显式设置 `XHS_VIDEO_ASR_SELECTION_MODE=value` 或 `XHS_VIDEO_ASR_ENFORCE_DAILY_BUDGET=1` 才恢复对应限制
- `enrich_media.py` 在 DashScope 调用边界会强制检查当天预约，直接调用也无法绕过；仅排障时可显式设置 `XHS_ASR_GUARD_OVERRIDE=1`
- 内容理解：媒体增强后调用 OpenClaw 文本模型生成 `contentSummary`、`highLikeReason`、`contentTopicTags`；视频额外生成 `coverHookReason`、`titleHookReason`、`titleCoverSynergy`、`hookTags`，用于 GitHub 数据表、Obsidian 每日新增阅读、Bases 封面标题素材库和后续 agent 选题复用
- 封面分类：`enrich_cover_layouts.py` 独立调用 `minimax-direct/MiniMax-M3`，只写 `coverLayout`、`coverTags`、识别依据、置信度、模型和版本字段。它在每日脚本中是非阻塞步骤，失败不会改变抓取、构建或同步的最终状态

## 封面分类验证与回填

默认每日只处理最近 7 天仍待识别或封面已变化的素材，单次最多 40 张；图片哈希与分类版本一致时直接命中缓存。

```bash
# 只处理 32 张人工标注样本
python3 scripts/enrich_cover_layouts.py \
  --gold tests/fixtures/cover-layout-gold.json

# 验证主分类一致率与“左右分区”准确率
python3 scripts/enrich_cover_layouts.py \
  --evaluate-gold tests/fixtures/cover-layout-gold.json

# 精准回填现有 Bases 素材，不属于每日任务
python3 scripts/enrich_cover_layouts.py \
  --bases-root "$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/Terry-Knowledge/自媒体100天/AI自媒体号/自媒体沉淀/Bases素材" \
  --limit 0

# 预览并把数据库分类原子写回 Bases；正式写入会先备份原 Markdown
python3 scripts/backfill_bases_cover_layouts.py --dry-run
python3 scripts/backfill_bases_cover_layouts.py
```

MiniMax 超时或服务不可用时不写分类字段，素材在看板显示为“待识别”，后续每日任务会继续补齐。连续两次返回非法结构化结果时写为“待复核”，方便人工检查。

## 数据边界

这个公开池不包含小红书登录态、cookie、`xsec_token`、评论、评论者信息、本机路径、原始视频/抽帧文件或任何凭据。
