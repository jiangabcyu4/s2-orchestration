---
name: s2-orchestration
description: 长任务 S2 多 Agent 编排执行。长任务（>5 分钟或需多源调研/多文件/多步骤）时触发，用「可靠任务书 + 结果落盘 + 查文件等待 + 分块读写 + 断点续传」把任务拆成并行子 agent 跑完，避免卡死、丢结果、图片化中断、重复交付。关键词：长任务、多Agent、拆分、调研、演讲稿、报告、可研、方案、编排、S2、落盘、断点续传。
---

# 长任务 S2 多 Agent 编排执行

> 沉淀自多次长任务踩坑复盘（融资演讲稿、可研报告、图片化中断等）。核心目标：让「多源调研 + 文档生成 + 交付」这类长任务**一次跑通、不卡死、不丢结果、不图片化中断、不重复交付**。

## 一、触发条件

满足任一即启用：
- 任务预计 > 5 分钟，或需多源联网调研
- 需生成/重构文件（docx/xlsx/pptx/pdf/md）并交付
- 需拆分多个子任务并行执行

## 二、核心铁律（执行前默念）

1. **先确认范围**：增量 vs 全量、交付格式、基准文件是哪个。
2. **子 agent 结果必须落盘**（`/tmp/xxx.txt`），整合时只 `read` 文件，绝不依赖 completion event 文本。
3. **等待只看"产物文件是否齐全"，不看"完成事件数"**（完成事件只来一次，数错就死等）。
4. **数据不用 Kimi/聚合摘要**，关键数字 `web_fetch` 官方原始出处交叉验证。
5. **文档生成用 `shutil.copy(基准)` + 替换段落文本**，严禁 `Document()` 空建 + `add_paragraph()`。
6. **邮件只在最终完成后发一次**（`send_email_once.py` 去重）。
7. **交付前重新打开验证**（段落数/字符数/格式/zip）。
8. **每步写状态文件**，支持断点续传。
9. **控制单次工具输出**：exec 用 head/tail/grep 分段，read 用 offset/limit 分块，避免长输出触发"图片化"。

## 三、任务书模板（spawn 子 agent 必须含 5 要素）

```
【目标】一句话说清这个子 agent 要产出什么
【数据源】不用 Kimi，用 web_fetch 抓权威源；标注置信度
【落盘】结果必须写入 /tmp/<维度>.txt（强制，不写就视为失败）
【验证标准】结果需包含哪几项关键数据/结论
【完成信号】写文件后返回"已写入 <路径> + 字节数"
```

用 `sessions_spawn(mode="run")` 并行启动，记录每个 childSessionKey。

## 四、可靠等待机制（卡死高发区，最关键）

- ❌ **不要数 completion event**——完成事件只来一次，把已完成的当 pending 再 yield = 死等（曾空转 27 分钟）。
- ✅ **每收到一个完成事件，就 `ls` 重查产物文件清单**：预期 N 个 `/tmp/*.txt`，N 个都在且非空 → 全部完成，立即进整合。
- ✅ 缺哪个才等哪个；`sessions_yield` 用于等待，但**从不因为"觉得还差几个"而无限等**。
- ⏱️ 超时兜底：某子 agent 超过预期时长（如 15 分钟）文件仍未出现 → 主动重查/重试/报告，不干等。
- ⏰ 定时查文件兜底（cron）：派完子任务后，用 cron 每 ~90s（schedule everyMs=90000，sessionTarget=main，payload.kind=systemEvent）注入一次"检查 /tmp 产物文件是否齐全"的事件；文件齐全立即进整合，不再依赖 sessions_yield 完成事件（该事件在本环境经常不唤醒主会话，已多次空转卡住）。全部文件到位、整合完成后删除该 cron。
- 📣 进度通知：距上次报进度 > 8 分钟仍无实质进展 → 主动发一条；阶段完成各报一次。

## 五、完整流程（8 步）

1. **接收+范围确认**：增量/全量、格式、基准文件；写 `任务_<名>_state.json`。
2. **拆分**：调研阶段并行 spawn N 个维度 agent；撰写/整合单 agent；交付单 agent。
3. **并行调研**：按第三节任务书 spawn；结果落盘。
4. **可靠等待**：按第四节查文件齐全度。
5. **整合**：单 agent read 所有文件，校验非空/含关键数据，汇总。
6. **生成**：`shutil.copy(基准)` + 替换段落；中文字体设 ascii/hAnsi/eastAsia。
7. **验证**：重新打开，段落数/字符数/格式/zip 完整性。
8. **幂等交付**：云文档（同任务更新同一 doc）+ 文件附件（MD/DOCX 分条发）+ 邮件仅终版一次。

## 六、"图片化"应对（已确认根因）

- **现象**：会话上下文变大后，工具输出被转成图片，上下文显示 `(see attached image)`，纯文本模型读不到。
- **根因**：渲染/回传层**环境级 bug/回归**，**无独立阈值配置**；会话级状态触发（连 7 字节输出都会图片化）。
- **图片是内嵌 base64**（`{type:"image",data:"<base64>"}`），随 transcript JSON 持久化，**不落独立 PNG**。
- **应对**：
  1. 治本：`contextPruning(mode=cache-ttl, ttl=5m)` + `compaction(mode=safeguard, keepRecentTokens=20000)` + 主动 `/compact`。
  2. 治标：控制单次输出（分块读）、大上下文下沉子 agent、图片/OCR 走视觉桥。
  3. 应急：已图片化 → `/new` 开新会话，关键结论先落盘。
  4. 兜底：子 agent 结果写文件 + completion event 文本。

**视觉方案优先级**（图片/OCR/文档解析走视觉桥时按序选）：
- **方案1（优先/默认，token 消耗小）**：百度 OCR（免费零 token，纯文字识别首选）。
- **方案2（可选，token 消耗大）**：deepseek-v4-flash-vision-exp（精度更高、表格/复杂图更强；因 token 消耗大，作为"可选择项"，需要高精度时手动切换）。
- **兜底**：MinerU（PDF/文档解析）+ glm-4v-flash（图像理解兜底）。

## 七、断点续传

`任务_<名>_state.json` 最小字段：
```json
{"phase":"research|integrate|generate|deliver|done",
 "baseFile":"...","format":"docx|md",
 "subtasks":[{"dim":"硫资源","file":"/tmp/research_sulfur.txt","status":"done"}],
 "expectedFiles":["/tmp/research_sulfur.txt"]}
```
主会话重启后第一件事：读状态文件 → 判断 phase → 从断点继续，不重新拆分。

## 八、常见坑表

| # | 坑 | 根因 | 修复 |
|---|---|---|---|
| 1 | 等待卡死 | 数 completion event 死等 | 查产物文件齐全度 |
| 2 | 结果回传丢失 | 依赖 completion event 文本 | 结果强制落盘，整合只 read |
| 3 | 空文档/格式乱 | Document() 空建 + add_paragraph | shutil.copy 基准 + 替换段落 |
| 4 | 多 agent 覆盖文件 | 未锁定基准 | 任务前锁定唯一基准 |
| 5 | 数据虚构 | Kimi 聚合 | web_fetch 权威源交叉验证 |
| 6 | 邮件乱发 | 每版都发 | 只终版发一次 + 去重脚本 |
| 7 | 图片化中断 | 会话级环境 bug | compaction/pruning + 分块读 + 拆子 agent |
| 8 | 断点丢失 | 状态不落地 | 每步写 state.json |
| 9 | 云文档版本堆一堆 | 每迭代 create 新 doc | 同任务更新同一 doc |
| 10 | key 截断 | 复制了掩码显示 | 用完整 key，注意"…"是掩码 |

## 九、成功标准（Done）

- [ ] 状态文件 phase=done
- [ ] 产物文件非空、通过交付前验证
- [ ] 云文档已上传 + 所有权已授用户
- [ ] 邮件已发（仅一次，有去重日志）
- [ ] 复盘已写入 TOOLS.md
