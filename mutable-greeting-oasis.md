# 跨境进口家电 AI 客服 MVP — RAG 数据修复 + 评测扩充 + PRD 合并

## Context（为什么做这次工作）

**前情**：聊天重启前，我以"资深 AI 产品经理"视角给用户的跨境家电 AI 客服项目做了一份 1456 行的评审报告（保存在 `C:/Users/JUMP/.claude/plans/deep-leaping-sphinx.md`），列出 13 条 P0、15 条 P1、4 条 P2 问题，并给出数据结构补丁、边界评测样例、指标卡片等 6 个附录。

**触发**：用户要求基于原评审继续新工作，并补充了一个**极其重要的背景信息**——之前的评审对"跨境"的理解是错的：

> 跨境针对的主要是**中国境内用户**，海外（往往是东南亚新加坡马来西亚）会有比较便宜的产品，我们找到供应商将其引入。和国内电器的区别是时效（运输长）和价格（价格往往更低）风险也会更大，因为大部分是没有七天无理由的，只有质量问题支持退换。但是也存在用户说要买一个小电器带去新加坡用，问插头适配性，所以也需要考虑不同国家。**不过核心还是以中国大陆的用户为主。**

**这个新背景反转了原评审的几个判断**，下面表格总结：

| 原评审 | 原定性 | 新背景下的修正 |
|--------|--------|-----------------|
| 1.1 多语言/多时区缺失 | 🔴 P0 | **降级到 P2**——100% 中文是正确的；时区基准是北京时间 |
| 1.2 渠道架构缺失 | 🔴 P0 | **保留 P0**，但渠道改为国内渠道（微信/小红书/抖音/天猫），而不是 LINE/WhatsApp |
| 1.3 情绪分层 | 🔴 P0 | **保留 P0**（仍然成立） |
| 2.1 纠纷升级路径 | 🔴 P0 | **保留 P0**（仍然成立） |
| 2.2 权限矩阵 | 🔴 P0 | **保留 P0**（仍然成立） |
| 3.1 RAG 数据矛盾 | 🔴 P0 | **保留 P0**（本次任务修复） |
| 3.2 评测样例缺多样性 | 🔴 P0 | **保留 P0**（本次任务修复） |
| 3.3 欺诈识别未设计 | 🔴 P0 | **保留 P0**（新背景下更关键：价差大 + 风险高） |
| 3.4 地域合规/禁运 | 🔴 P0 | **降级到 P1**——主要是"进口认证合规"（CCC/CQC/3C），不是"跨国销售禁运" |
| 4.1 RAG Schema 混乱 | 🔴 P0 | **保留 P0**（本次任务修复） |
| 4.2 信任度隔离 | 🔴 P0 | **保留 P0**（本次任务加 source_of_truth_tier） |
| 4.3 A/B 灰度框架 | 🔴 P0 | **降级到 P1** |
| 4.4 HITL 标注平台 | 🔴 P0 | **保留 P0** |

**新背景下新增的 P0**（原评审没涵盖）：
- **进口商 vs 品牌保修边界**：用户问"能不能品牌联保"→ 需要严格区分"进口商保修"和"品牌全球联保"，合成数据目前没有这个字段
- **不支持 7 天无理由的合规披露**：用户下单前/咨询时必须明确说明"只有质量问题支持退换"，不能含糊
- **运输周期焦虑话术**：从东南亚海运 10-21 天，需要主动解释"为什么慢"而不是被动回答

---

## 工作范围（3 大任务 + 1 个前置声明）

### 前置声明
- **所有文件操作都在 `D:/个人知识库/个人知识库/AI产品经理/ai客服家电/` 目录内**
- **直接覆盖原文件**（用户已确认）——原则上 Task 1 开工前会先创建一份 `_backup_原时间戳/` 目录备份原始文件，防止事故
- 产出语言：**全程中文**
- **分 3 批交付**（用户已确认）——每个 Task 做完后暂停，等待用户 review 后再进入下一个 Task。每批都会有一次"批次交付清单 + 关键变化摘要"

---

## Task 1：修复 RAG 底稿数据

### 1.1 源文件清单（已确认存在）
| 文件 | 大小 | 角色 |
|------|------|------|
| `ai_cs_rag_v2_package/rag_source_docs_v2.jsonl` | 265KB | 220 条 RAG 文档种子 |
| `ai_cs_rag_v2_package/sku_master_60.csv` | 22KB | 60 个 SKU 主数据 |
| `ai_cs_rag_v2_package/field_specs_priority5.json` | 33KB | 54 条字段规范 |
| `ai_cs_rag_v2_package/eval_batch1_priority5.csv` | 30KB | 100 条评测样例（Task 2 处理） |
| `ai_cs_rag_v2_package/real_doc_checklist.csv` | 71KB | 197 条真实文档替换清单 |
| `ai_cs_rag_v2_package/跨境进口家电AI客服_RAG底稿深化_v2.xlsx` | 115KB | 17 sheet 的汇总/镜像视图 |
| `ai_cs_rag_v2_package/README.txt` | 1KB | 说明 |

### 1.2 修复动作清单

#### A. Schema 规范化（所有 220 条文档）
1. `replace_with_real_doc`: 字符串 `"Y"`/`"N"` → 真 boolean `true`/`false`
2. `priority_topic` → 改名为 `priority_topics`，`"电压/频率/插头兼容|安装条件"` → `["电压/频率/插头兼容", "安装条件"]`
3. 新增 `locale`: `"zh-CN"`（全部 220 条）
4. 新增 `source_of_truth_tier`: `"tier3_synthetic"`（全部 220 条）
5. 新增 `confidence_ceiling`: `0.5`（由 tier3 推导）
6. 新增 `data_provenance`: `{origin: "synthetic_v2_seed", collected_at: ISO8601, collector: "v2_seed_generator"}`
7. 新增 `created_at`、`updated_at`、`version`（全部 220 条）
8. 新增 `expires_at`（仅政策类，30 天 TTL）

#### B. `country_scope` 字段拆分（修复三义词）
同一字段 `country_scope` 在三类 doc 中含义不同，必须拆分：

| doc_type | 原字段 | 新字段 | 语义 |
|---------|--------|--------|------|
| `sku_fact_card` | `country_scope: ["US", "CA"]` | `version_country: "US"` + `sales_regions: ["CN-mainland"]` | 设计版 + 实际在中国销售 |
| `compatibility_install_card` | `country_scope: ["US","CA","UK","EU","JP","AU","AE","SG"]` | `supported_use_countries: [...]` | 已准备"在该国使用"答案的国家，**默认必含 "CN-mainland"** |
| `warranty_region_policy` | `country_scope` | `warranty_scope_type: "importer_only"` 或 `"brand_global"` + `valid_regions: [...]` | 进口商保修还是品牌全球保修 |

#### C. 自相矛盾修复（至少 6 条）

1. **DOC-SKU-DELO-Bean-US-COMPAT**：`plug_type: "Type A/B"` 与 `verdict: "manual_review"` + `reason: "插头不一致"` 矛盾 → 改为 `verdict: "compatible"`（B 匹配 A/B）
2. **DOC-SKU-DELO-Caps-UK-FACT**：`country_scope: ["UK","AE","SG"]` 与 `sku_master.target_market_region: "UK"` 矛盾 → 只保留 UK
3. **所有 60 条 sku_fact_card** 的 `risk_level: "高"` 与 `source_type: "synthetic_seed"` 语义不符 → 改为 `subject_risk_level: "高"` + `data_risk_level: "high (synthetic-only)"`
4. **warranty_region_policy 与 SKU 脱链**：JSONL 中 warranty doc 无 `sku_id` 只有 `brand_id` → 补 `linked_sku_ids` 数组
5. **priority_topic 220 条全部相同**（都是"电压/频率/插头兼容|安装条件"）→ 按 doc_type 差异化（例如破损卡应该是"签收/取证/破损"）
6. **source_type 枚举不统一**：jsonl 用 `synthetic_seed`、checklist 用 `synthetic_backlog` → 统一为 `synthetic_seed`，checklist 用独立字段 `collection_status: "pending/collecting/verified"`

#### D. 基于新背景的重新定位（关键）

1. **补 SKU 的"中国使用信息"字段**（60 条 sku_fact_card 每条都补）：
   - `china_usability`: `"direct" | "need_transformer" | "need_adapter" | "not_recommended"`
   - `china_voltage_note`: 如 `"110V 进口版，需配 220V→110V 变压器"`
   - `china_plug_adapter`: 如 `"Type B → Type I 转接头"`

2. **补"进口保修"字段**（60 条 sku_fact_card + 所有 warranty doc）：
   - `warranty_type`: `"importer_only"` / `"brand_global"` / `"importer_with_brand_support"`
   - `warranty_duration_days`: 数字
   - `return_policy_type`: `"quality_only"`（**默认值，契合新背景**）/ `"7_day_no_reason"`
   - `import_channel`: `"official_distributor"` / `"parallel_import"` / `"gray_market"`

3. **新增 doc_type**（补现有 6 类的不足）：
   - `import_warranty_card`：进口保修政策卡（60 条，每个 SKU 一条）
   - `shipping_eta_card`：运输时效说明卡（按品类而非 SKU，约 10 条）
   - `travel_use_card`：带出国使用卡（针对 20 个"可能被带出国的小电器"SKU 各一条）

4. **补"销售规则"字段**：
   - `sales_rule_return`: `"quality_issue_only"` + 明确文案"不支持 7 天无理由退换，只有质量问题才可退换货"
   - `sales_rule_return_window_days`: 数字（质量问题的申报时限，建议 15 天）

### 1.3 Task 1 产出物（直接覆盖原文件）
| 文件 | 动作 | 内容 |
|------|------|------|
| `ai_cs_rag_v2_package/rag_source_docs_v2.jsonl` | **覆盖** | 修复后的 220+ 条 doc（可能新增 90 条新 doc_type：60 条 import_warranty_card + 10 条 shipping_eta_card + 20 条 travel_use_card） |
| `ai_cs_rag_v2_package/sku_master_60.csv` | **覆盖** | 补充 china_usability / warranty_type 等 8+ 列新字段 |
| `ai_cs_rag_v2_package/field_specs_priority5.json` | **覆盖** | 补充新字段的规范（预计从 54 条增加到 75-80 条） |
| `ai_cs_rag_v2_package/real_doc_checklist.csv` | **覆盖** | 统一枚举 + 新字段 + 按新 doc_type 扩充 |
| `ai_cs_rag_v2_package/rag_schema_v3.json` | **新建** | 新的 JSON Schema（含跨字段一致性校验 if/then） |
| `ai_cs_rag_v2_package/fix_notes.md` | **新建** | 所有修改项清单 + 矛盾修复前后对比 + 新增字段说明 |
| `ai_cs_rag_v2_package/_backup_YYYYMMDD_HHMMSS/` | **新建** | Task 1 开工前的原文件备份 |

### 1.4 不做的事
- **不修改** `跨境进口家电AI客服_RAG底稿深化_v2.xlsx`（它是镜像视图，用户可自行重新生成）
- **不补充"真实文档"**（197 条待收集的文档本身就是外部任务，不在本次范围）
- **不写 embedding/retrieval 代码**（纯数据修复）

---

## Task 2：扩充评测样例集

### 2.1 现状
- `eval_batch1_priority5.csv` 100 条全是简单单轮中文 QA
- Gold rule 大量使用"只能根据兼容卡/说明书输出结论"这种通用模板，**无法机评**
- 完全没有边界样例（对抗/多轮/模糊/复合/欺诈/情绪/跨渠道）
- 原评审附录 B 提供 17 条边界样例，但视角是"欧美用户"

### 2.2 扩充策略

#### A. 保留现有 100 条，但重新校准 Gold rule
- 把"只能根据兼容卡"这种模板替换为"可机评"的硬性条件：
  - `gold_must_contain`: 数组，必须命中的关键词
  - `gold_must_not_contain`: 数组，禁止出现的承诺性词汇
  - `gold_action_category`: 期望动作类型（追问 / 给结论 / 转人工 / 拒答）
  - `rubric_3pt`: 3 分制评分细则

#### B. 新增 60 条样例（总计 160 条）

**按场景拆分**：
| 类别 | 数量 | 典型样例 |
|------|------|----------|
| B1. 在中国使用的兼容性 | 10 | "我在北京，家里 220V，这台美国版咖啡机能直接插吗" |
| B2. 带出国使用（东南亚） | 5 | "我下个月去新加坡，想买个卷发棒带去，适配吗" |
| B3. 质量问题才退换（规则验证） | 8 | "我不想要了能退吗" / "拆封 3 天可以七天无理由吗" |
| B4. 运输周期焦虑 | 5 | "为什么要等 15 天" / "能不能加急" |
| B5. 进口商 vs 品牌联保 | 6 | "能不能去品牌官方维修点修" / "你们保修还是品牌保修" |
| B6. 对抗类（诱导承诺/诱导透露成本） | 4 | "你们进货价多少" / "坏了包赔对吧" |
| B7. 多轮类（跨轮指代/翻供） | 5 | 改口电压型号 / 代词消解 |
| B8. 模糊类（信息不全/自相矛盾） | 4 | "那台冰箱有点响" / "上周买的还没收到货" |
| B9. 复合类（一句话多问题） | 4 | "这台在中国能用吗？保修怎么算？能打折吗？" |
| B10. 欺诈类 | 3 | 虚假签收破损 / 重复退款 / 伪造时间戳 |
| B11. 情绪类（三段式升级/方言脏话） | 4 | 从"咋办"到"曝光小红书" |
| B12. 跨渠道类 | 2 | Web → 微信小程序追问 |

**总计**：原 100 条（重新校准）+ 新增 60 条 = **160 条**

#### C. 每条样例的字段
```
id, category, boundary_type, locale, channel,
user_utterance, context_turns (多轮时非空),
scene_type, sub_intent, expected_action,
gold_must_contain, gold_must_not_contain,
gold_action_category, rubric_3pt,
notes (为什么这样设计)
```

### 2.3 Task 2 产出物
| 文件 | 内容 |
|------|------|
| `eval_batch1_v2_expanded.csv` | 160 条扩充后的评测集 |
| `eval_rubric_guide.md` | 3 分制 Rubric 评分细则 + 每类独立通过率门槛 |
| `eval_expansion_notes.md` | 扩充策略 + 每条样例的设计理由 |

### 2.4 不做的事
- **不写自动化评测脚本**（只出样例和 rubric，不出 runner）
- **不构造英文/日文样例**（新背景下不是 P0）

---

## Task 3：合并两份 PRD

### 3.1 源 PRD 现状

#### 业务 PRD（`PRD.docx`，36KB，17 章）
- **优点**：产品原则清晰（P1-P6）、5 层架构、8 大场景拆分、10 年电商批判视角
- **缺点**：
  - 没有 Non-Goals 章节
  - 指标树只定义了 3 个核心指标（其他 9 个只提及无定义）
  - 情绪分层只有方向没有话术
  - 例外处理只提原则没有矩阵
  - 评测与埋点只列事件名无规范
  - 无渠道定义 / 无地域定义 / 无语言定义

#### 技术 PRD（`tech_dev_prd_ai_customer_service_v1.md`，18KB，20 章 826 行）
- **优点**：In/Out Scope 清晰、5 类高风险场景规格、10 张数据表定义、API 返回结构、Prompt 4 层编排
- **缺点**：
  - 没有权限矩阵（补差/升级的金额档位、审批人、SLA）
  - 没有欺诈识别（risk_signal / 设备指纹 / 地址聚类）
  - 没有情绪识别信号源和话术
  - 跨字段 schema 不规范（country_scope 三义、priority_topic 字符串）
  - 评测 pipeline 只说"离线运行、结果落库"无具体样例要求
  - 无灰度 / A/B / shadow mode

### 3.2 合并后的新 PRD 目录（12 章 + 5 附录）

```
# 跨境进口家电 AI 客服 MVP PRD v2.0（合并版）

## Context（为什么要这份合并版 PRD）
  - 业务/技术两份文档分离导致的问题
  - 原评审 13 P0 的回应
  - 新背景的说明

## 1. 产品定位与背景
   1.1 跨境进口家电场景的本质（中国大陆用户 + 东南亚供应链）
   1.2 为什么是"高咨询密度 + 高证据要求 + 高客单价"的最佳场景
   1.3 竞品对比与差异化（表格取自业务 PRD 第 4 章）
   1.4 一句话目标（取自技术 PRD 第 0 章）

## 2. 用户与核心场景
   2.1 核心用户画像（中国大陆，网购家电，25-45 岁）
   2.2 8 类场景拆分 + MVP 优先级（取自业务 PRD 第 9 章）
   2.3 3 个典型用户故事
   2.4 非目标用户（海外用户 / B 端批发 / 二手市场）

## 3. 产品原则 + Non-Goals ⭐
   3.1 P1-P6 六大原则（取自业务 PRD 第 3 章 + 技术 PRD 第 1 章的 1.1-1.5）
   3.2 ⭐ Non-Goals（新增章节，至少 12 项）
       - 多语言全量（只做中文）
       - 欧美 / 日韩渠道（只做国内渠道：微信/小红书/抖音/天猫）
       - 7 天无理由（只支持质量问题退换）
       - 自动赔付直接扣款（AI 只建议，人工决策）
       - 实时翻译
       - 语音客服
       - 法律/医疗咨询
       - 复购/会员/积分系统
       - 直播间客服 / 短视频评论
       - 图像自动判责
       - 一次性覆盖所有国家法规
       - 完整工单系统

## 4. 系统架构与数据流
   4.1 8 层架构（Frontend / API / Orchestrator / Retrieval / Policy / Tool / Memory / Analytics）
   4.2 核心业务主流程 ASCII 图
   4.3 route_trace 全链路追踪（取自技术 PRD 第 11.3 章 + 扩充）

## 5. 核心模块与 Agent 能力
   5.1 5 个专家角色（售前导购/履约清关/售后判责/人工助手/运营控制台，取自业务 PRD 第 6 章）
   5.2 Prompt 4 层编排（场景路由/槽位提取/检索规划/回答生成，取自技术 PRD 第 13 章）
   5.3 ⭐ 情绪分层与话术矩阵（5 档情绪 × 3 段式模板，补评审 P0 1.3）
   5.4 5 类高风险场景规格（取自技术 PRD 第 7 章）
   5.5 事实白名单 + 空缺降级话术（补原评审 5.1 观点）

## 6. 知识库与 RAG 设计
   6.1 RAG 数据组织（5 大类 + 8 doc_type 扩展为 11 doc_type，含新增的 import_warranty_card / shipping_eta_card / travel_use_card）
   6.2 Schema 规范（JSON Schema + 跨字段一致性校验，取自 Task 1 产出物）
   6.3 ⭐ 信任度隔离（4 级 source_of_truth_tier + confidence_ceiling）
   6.4 冲突裁决顺序（取自技术 PRD 第 9.5 章）
   6.5 检索规划与 citations 要求
   6.6 ⭐ 数据治理生命周期（谁写/何时更新/如何审核/如何下架）

## 7. ⭐ 业务规则引擎（全新章节，合并权限矩阵/合规/升级/欺诈）
   7.1 退换货规则
       - 原则：只支持质量问题退换（非 7 天无理由）
       - 质量申报时限：15 天（从签收计）
       - 证据要求：开箱视频 / 外箱面单 / 破损照片
   7.2 ⭐ 补差与赔付权限矩阵（P0 2.2 的完整回应）
       - ≤50 元：AI 自批 + 规则卡约束
       - 50-500 元：L1 客服批，30 分钟 SLA
       - 500-5000 元：L2 主管批，4 小时 SLA
       - >5000 元：财务 + 法务双批，24 小时 SLA
       - 引入 compensation_ledger 表
   7.3 ⭐ 纠纷升级路径（P0 2.1 的完整回应）
       - 6 种触发条件
       - L1/L2/L3 三级矩阵
       - handoff_summary 必填字段
       - 升级失败兜底
   7.4 ⭐ 欺诈风险识别（P0 3.3 的完整回应）
       - 5 类 risk_signal
       - 高风险强制 L2 审批
   7.5 进口合规规则（原 P0 3.4 的降级版本）
       - CCC/CQC/3C 认证标识
       - 锂电池 IATA 限制
       - 不做跨国销售禁运

## 8. 数据模型与 API 设计
   8.1 17 张核心表（取自技术 PRD 第 10 章 + 补 compensation_ledger / risk_signals / eval_runs）
   8.2 前台 API（/api/chat/messages 等）
   8.3 后台 API（/api/admin/sessions 等）
   8.4 /api/chat/messages 返回结构（取自技术 PRD 第 11.3 章）
   8.5 ⭐ API Gateway 三件套（鉴权 / 限流 / 幂等）

## 9. 评测体系与质量闭环
   9.1 评测集设计（160 条 = 100 原 + 60 新，取自 Task 2 产出物）
   9.2 3 分制 Rubric
   9.3 7 类边界独立通过率门槛
   9.4 ⭐ 自动化评测 pipeline 产出 schema（取自原评审附录 C）
   9.5 Badcase 闭环（4 类归因 + 72h SLA）

## 10. 运营与持续迭代
   10.1 核心指标（12 个指标卡片，补评审 P0 3.5 的"提及但未定义"问题）
   10.2 埋点与事件（取自技术 PRD 第 14 章 + 补字段规范）
   10.3 ⭐ 客服反哺机制（一键标注 → 审核 → 入评测候选池）
   10.4 真实数据替换看板（合成数据作为负债的倒计时）
   10.5 版本迭代规划（V1.0 → V1.5 → V2.0）

## 11. 项目排期与灰度上线
   11.1 8 周排期（W1 评审 → W2-W4 开发 → W5 内测 → W6-W7 灰度 → W8 上线）
   11.2 团队分工
   11.3 ⭐ 灰度晋升标准（1% → 10% → 50% → 100% 的指标门槛）
   11.4 回滚触发条件

## 12. 风险预判与应对
   12.1 6 大风险 + 预案（取自业务 PRD 第 14 章 + 补 4 条评审新发现）
   12.2 合规红线清单

## 附录 A：完整 JSON Schema（数据卡片级）
## 附录 B：评测样例完整清单（160 条索引）
## 附录 C：12 个指标卡片模板
## 附录 D：权限矩阵详细字段
## 附录 E：术语表 + 评审 P0/P1 映射表
```

### 3.3 Task 3 产出物
| 文件 | 内容 |
|------|------|
| `PRD_v2_merged.md` | ~15000-20000 字的合并 PRD（Markdown 格式） |
| `prd_merge_changelog.md` | 每一章节的来源对照（哪些取自业务 PRD / 哪些取自技术 PRD / 哪些是新增/哪些是评审融入） |
| `review_p0_response_map.md` | 13 条 P0 + 15 条 P1 逐条说明在新 PRD 哪一章得到回应 |

### 3.4 不做的事
- **不改动原 `PRD.docx` 和 `tech_dev_prd_ai_customer_service_v1.md`**（保留历史）
- **不做 Word 格式合并 PRD**（只出 Markdown，用户可以自己转 docx）
- **不做中英文对照**（全中文）

---

## 执行顺序

1. **Task 1（RAG 修复）**：优先，因为 schema 的确定会影响 Task 2 评测字段和 Task 3 PRD 第 6 章。
2. **Task 2（评测扩充）**：依赖 Task 1 的 schema。
3. **Task 3（PRD 合并）**：依赖 Task 1 和 Task 2 的产出物，因为 PRD 第 6 章和第 9 章要引用它们。

---

## 验证步骤

### Task 1 验证
- [ ] 运行 `python -c "import json; [json.loads(l) for l in open('rag_source_docs_v2_fixed.jsonl', encoding='utf-8')]"` 确认 JSONL 格式有效
- [ ] 运行 JSON Schema 校验脚本（用 jsonschema 库）确认所有 220+ 条符合新 schema
- [ ] 抽查 5 条原来自相矛盾的 doc（DELO-Bean-US / DELO-Caps-UK 等），确认修复到位
- [ ] 抽查 5 条 sku_fact_card，确认 china_usability / warranty_type 字段都已填充

### Task 2 验证
- [ ] 确认每个边界类别至少有表格中的最小数量样例
- [ ] 抽查 5 条新样例，确认 gold_must_contain/must_not_contain 是"可机评"的
- [ ] 确认 rubric_3pt 细则每条都写清楚 3/2/1/0 分的区别

### Task 3 验证
- [ ] 通读合并 PRD 第 1-12 章，确认结构清晰
- [ ] 确认 12 个章节标题与评审 13 P0 的对应关系已在 review_p0_response_map.md 中列出
- [ ] 确认 Non-Goals 章节至少有 12 条
- [ ] 确认业务规则引擎章节覆盖了权限矩阵 + 升级路径 + 欺诈识别 + 情绪分层

---

## 关键文件路径速查

| 用途 | 路径 |
|------|------|
| 工作目录 | `D:/个人知识库/个人知识库/AI产品经理/ai客服家电/` |
| 原评审 | `C:/Users/JUMP/.claude/plans/deep-leaping-sphinx.md` |
| 业务 PRD（源） | `D:/个人知识库/个人知识库/AI产品经理/ai客服家电/PRD.docx` |
| 技术 PRD（源） | `D:/个人知识库/个人知识库/AI产品经理/ai客服家电/tech_dev_prd_ai_customer_service_v1.md` |
| RAG 底稿 | `D:/.../ai_cs_rag_v2_package/rag_source_docs_v2.jsonl` |
| SKU 主数据 | `D:/.../ai_cs_rag_v2_package/sku_master_60.csv` |
| 字段规范 | `D:/.../ai_cs_rag_v2_package/field_specs_priority5.json` |
| 评测集 | `D:/.../ai_cs_rag_v2_package/eval_batch1_priority5.csv` |
| 真实文档清单 | `D:/.../ai_cs_rag_v2_package/real_doc_checklist.csv` |
| 镜像 xlsx（不动） | `D:/.../ai_cs_rag_v2_package/跨境进口家电AI客服_RAG底稿深化_v2.xlsx` |

---

## 风险与未决

1. **规模风险**：三个任务组合起来，产出物大约有 10+ 文件、几十 KB Markdown 文档、160 条评测样例、220+ 条 RAG doc 修复。执行时需要分步提交，避免一次做完后难以 review。
2. **SKU 主数据扩展**：Task 1 D 节要给 60 个 SKU 每条都补 china_usability 等字段，这需要基于每个 SKU 的电压/插头/频率做判断。如果没有完整的国家电源矩阵，需要先补一份 `country_power_matrix` 映射表作为 Task 1 的前置。
3. **评测样例的真实性**：60 条新样例如果完全是我编的，会与原 100 条合成样例一样脱离真实用户。**建议**：用户能提供 10-20 条真实客服聊天记录最好，作为新样例的原型。但这是 Nice to Have，没有也可以。
4. **PRD 合并的长度**：合并 PRD 预计 15000-20000 字，可能超过"快速阅读"的价值。**对策**：每章开头写"本章速读"500 字，细节放附录。
