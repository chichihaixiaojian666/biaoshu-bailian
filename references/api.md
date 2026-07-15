# 招采猫开放 API 契约参考

> **契约兼容标注（skill biaoshu-bailian 2.0.7）**
> - 适配后端 API：`/api/open/v1`
> - 契约核对日期：2026-07-06（后端字段/枚举变化时更新此处并 bump 版本）
> - 关键枚举快照：`risk_level ∈ {high, review, tip}` · `result_type ∈ {suspected, detected}` · `priority ∈ {high, medium, low}`
> - 渲染兼容策略：`report.py` 同时兼容文档值（高/中/低）与实测值、证据多形态、缺字段不崩——契约小幅漂移只需 PATCH，不触发 MAJOR。

> ⚠️ **数据外发与知情同意**：本文档所有接口均为招采猫云端服务——上传的招标/投标文件**常含商业、报价与个人信息**，将发送至 `biaoshu.zhiliaobiaoxun.com` 处理并消耗账户积分；**上传的文件与产出的结果会留存在招采猫服务器**（任务结果与成品 .docx 约 7 天后过期，数据以 App Key 所属账户身份存于平台、可登录官网查看管理）。**首次上传前必须确认用户知悉并同意**；完整披露见 SKILL.md「⚠️ 权限与数据说明」。

`scripts/zcm.py` 已封装下列全部端点；本文档供需要直接发请求、排查错误或理解返回结构时查阅。
所有契约均经后端源码 + 本地实跑核实。

## 目录
- [鉴权与环境](#鉴权与环境)
- [核心模型与约定](#核心模型与约定)
- [8 个端点详情](#8-个端点详情)
- [错误码速查](#错误码速查)
- [注意事项](#注意事项)

---

## 鉴权与环境

- **Base URL（生产）**：`https://biaoshu.zhiliaobiaoxun.com/api/open/v1`
- 每个请求都带鉴权头：

| Header | 值 | 说明 |
|---|---|---|
| `X-App-Key` | App Key | 必填，形如 `bk_live_xxxxx` |
| `Idempotency-Key` | UUID（可选） | 相同 key 24h 内返回同一 `job_id`，不重复扣费 |

- 服务开关：开放 API 受超级管理员『系统设置』总开关控制，**关闭时整层返回 404**。
- 凭证获取：官网 <https://biaoshu.zhiliaobiaoxun.com/> 注册 →『账户 → 开放 API』生成 Key。
  App Key 可随时在『账户 → 开放 API』面板查看；重置后旧 Key 立即失效。

## 核心模型与约定

- **project_id**：统一句柄，由「智能解读」产出，是**唯一的招标文件上传入口**。后续抽包 / 生成 / 合规复用同一 project，不重复解读、不重复计费。
- **job_id**：每个异步任务的对外句柄。提交类接口立即返回 `{ "job_id": "..." }`。
- **任务状态**：`queued` → `running` → `succeeded` / `failed` / `canceled`。
- **上传方式**：本 skill 一律 `multipart/form-data` 直传本地文件（后端另有 `file_url` 入参，**本 skill 不使用**，也不做任何远程抓取）。
- **限流**：每 App Key 默认 60 req/min、同时进行任务 ≤ 3；超限 429。
- **统一错误体**：`{ "error": { "code": "...", "message": "..." } }`
- **计费**：仅在 ③生成（正文逐条 + 导出）发生一次；①解读、②抽包不扣费，仅受限流约束。
- **结果时效**：任务结果与 .docx 默认保留约 7 天，过期取结果返回 404 `result_expired`。⚠️ 这意味着**结果在此期间留存于招采猫服务器**（第三方存储）；上传文件与历史数据以账户身份存于平台，用户可登录官网查看管理——向用户交代结果时请一并说明。

## 8 个端点详情

### `GET /me` — 连通性与余额
```json
{"wallet_balance":1397084,
 "limits":{"rate_per_min":60,"max_concurrent_jobs":3,"running_jobs":0}}
```

### `POST /interpretations` — 智能解读（唯一上传入口）
- 入参：multipart 字段 `file`（.pdf/.doc/.docx）。
- 返回：`{"job_id":"..."}`。
- 结果（`/jobs/{id}/result`）：`{"job_id","service":"interpretation","result":{...}}`。
  `result` 含句柄 `project_id`/`result_id`/`status` + **8 个内容维度 + 控标洞察**，
  完整字段见 [附录 A](#附录-a智能解读结果字段)。**记下 `result.project_id`**。

### `POST /bid-documents/{project_id}/packages` — 抽取分包
- 无 body。返回 `{"job_id":"..."}`。
- 结果：
```json
{"service":"bid_document",
 "result":{"packages":[...],"is_multi_package":true,"package_count":2,
           "suggested_pages":50,"max_total_pages":300}}
```
- 把 `packages` 给用户挑选，收集选中的 `package_ids`。
- `is_multi_package=false` 时可跳过选包，generate 不带 `package_ids`。

### `POST /bid-documents/{project_id}/generate` — 生成成品标书
- 入参 JSON：`{"package_ids":[11,12],"total_pages":80}`（非多包可省略 body 或传 `{}`）。
- 返回 `{"job_id":"..."}`。内部串行「选包 → 抽需求 → 大纲 → 逐条正文 → 导出」，耗时长。
- 进度阶段加权：`select / requirements / outline / content / export`。
- **结果是流式 .docx 二进制**（非 JSON），响应头 `Content-Disposition: attachment; filename="bid_<job_id>.docx"`。

### `POST /projects/{project_id}/compliance-reviews` — 合规审查
- 入参：multipart `bid_files`（一或多份 .doc/.docx）+ 表单字段 `is_blind_bid` / `is_electronic_bid`。
- project 必须已完成解读，否则 409。返回 `{"job_id":"..."}`。
- 结果（`/jobs/{id}/result`）：`result.compliance` 含 `summary`/`issues`/`similarity_issues`/`manual_items` 等，
  完整字段见 [附录 B](#附录-b合规审查结果字段)。

### `GET /jobs/{job_id}` — 查任务状态（轮询用）
```json
{"job_id":"...","service":"interpretation|bid_document|compliance",
 "phase":null,"status":"running",
 "progress":{"percent":20,"stage":"interpreting","stage_label":"智能解读中","updated_at":"..."},
 "error":null,"created_at":"...","updated_at":"..."}
```

### `GET /jobs/{job_id}/result` — 取结果
- 解读/合规返回 JSON；标书制作返回 .docx 二进制流。

### `POST /jobs/{job_id}/cancel` — 取消
- 尽力而为；已过的扣费点不退款。

### 402 insufficient_balance 错误体新增字段

`phone_bound`（bool）；另有 `bind_url` / `recharge_url`（**均携带明文 `bind_key=<app_key>`**）。
🔒 **本 skill 不使用也不转发这些带 Key 的链接**（防凭证经会话记录/截图/链接预览泄露）——积分不足一律引导用户自行登录官网充值（不含参数的普通链接）。

### 积分前置闸门（提交时 402）

积分余额 < 1 时，`POST /interpretations`、`POST /bid-documents/{pid}/generate`、
`POST /projects/{pid}/compliance-reviews` 三个计费入口在**提交时**直接返回 402
`insufficient_balance`（错误体含上述引导字段），充值或绑定手机号领积分后方可操作；
抽包（packages）与查询类接口不受限。skill 侧提交前也会先调 `GET /me` 预检余额。

## 错误码速查

| HTTP | code | 含义与处理 |
|---|---|---|
| 401 | `missing_credentials` / `invalid_credentials` | 缺 `X-App-Key` Header / App Key 不对 → 检查凭证或重置 Key |
| 403 | `account_disabled` | 凭证或用户被停用 |
| 402 | `insufficient_points` | 余额不足，不扣费不产出 → 充值 |
| 404 | `not_found` | 多为开放 API 总开关未开（整层 404）→ 联系管理员开启 |
| 404 | `job_not_found` / `project_not_found` / `result_expired` | 句柄不存在/非本人/结果过期（7 天 TTL） |
| 409 | `invalid_job_state` | 任务未成功就取结果 / 未解读就生成 / 未抽包就 generate |
| 422 | `validation_error` | 文件缺失/类型不支持 / 缺 package_ids |
| 429 | `rate_limited` / `too_many_concurrent_jobs` | 触发限流 → 退避重试（看 `Retry-After`）或减并发 |
| 500 | `internal_error` | 服务端异常 → 重试或反馈 |

任务级失败时 `GET /jobs/{id}` 的 `error.code`：`interpretation_failed` / `generation_failed` / `compliance_failed` / `insufficient_points` / `canceled` / `worker_lost`（服务重启导致，需重新提交）。

## 注意事项

- **唯一上传入口**：招标文件只能经 `/interpretations` 上传并产出 `project_id`；制作与合规都复用它，**不要重复上传同一招标文件**。
- **幂等**：网络重试带相同 `Idempotency-Key`（UUID），避免重复建任务/重复扣费。
- **计费**：扣 App Key 所属用户积分，与网页同价；生成前用 `GET /me` 看 `wallet_balance` 预判。
- **内容质量依赖知识库**：正文质量取决于 owner 租户的公司资料库；资料缺失会致内容退化（不硬失败）。
- **来源标记**：经开放 API 产生的数据标记为 **skill** 来源（网页端为「平台」），便于在网页历史/消费流水里区分。

> 字段口径与根目录《招采猫Skill服务.md》附录 A/B 一致；`scripts/report.py` 据此渲染报告。

---

## 附录 A：智能解读结果字段

`GET /jobs/{id}/result` 的 `result`（`service=interpretation`）：

```json
{
  "project_id": "123", "result_id": 7, "status": "completed",
  "project_info": [...], "compliance": [...], "disqualification": [...],
  "evaluation": [...], "key_requirements": [...], "business_terms": [...],
  "pricing": [...], "procurement_analysis": {...}, "decision_analysis": {...}
}
```

- **project_info[]** 项目基本信息：`field_name` / `field_value` / `source_page` / `source_text`。
- **compliance[]** 合标项（参与资格）：`category` / `requirement_text` / `source_page` / `source_text` / `is_structured`。
- **disqualification[]** 废标项（红线）：在 compliance 字段基础上多 `type`（资格废标/响应性废标/合规废标）。
- **evaluation[]** 评审项：`component` / `item` / `factor` / `score`(满分) / `weight` / `source_page` / `source_text` / `is_structured`。
- **key_requirements[]** 关键要求：`category` / `requirement_text` / `source_page` / `source_text`。
- **business_terms[]** 商务条款：`term_type` / `term_content` / `source_page` / `source_text`。
- **pricing[]** 报价要求：`component` / `requirement_text` / `source_page` / `source_text`。
- **procurement_analysis{}** 采购背景：`analysis_summary` / `procurement_background` / `procurement_objectives` / `procurement_scope_items[]` / `key_constraints[]` / `key_success_metrics[]`(每条 `{name,detail}`，关键成功指标)（缺失字段可为 null/空）。
- **decision_analysis{}** 控标洞察：
  - 顶层：`participation_recommendation`（建议/谨慎/不建议参与）、`control_risk_level`（高/中/低）、`confidence_level`、`summary[]`、`signals[]`、`evidence_items[]`、`actions[]`、`advantaged_supplier_profile[]`、`our_gap_assessment[]`。
  - `signals[]`：`id` / `dimension`（qualification_barrier/technical_targeting/business_barrier/scoring_bias/acceptance_and_performance_risk/pricing_competitiveness_constraint）/ `title` / `risk_level` / `description` / `reasoning` / `evidence_item_ids[]` / `our_stance`（advantage/risk/neutral/unknown）/ `our_stance_reason`。
  - `evidence_items[]`：`id` / `source_category` / `source_page` / `source_text_excerpt` / `why_it_matters`。
  - `actions[]`：`priority`（high/medium/low）/ `action_type` / `recommendation` / `related_signal_ids[]`。

---

## 附录 B：合规审查结果字段

`GET /jobs/{id}/result` 的 `result.compliance`（`service=compliance`）：

```json
{
  "run_id": 42, "status": "completed", "mode": "standalone",
  "document_id": 123, "interpretation_result_id": 7,
  "summary": {...}, "partial_summary": {...}, "bid_files": [...],
  "issues": [...], "similarity_issues": [...], "manual_items": [...],
  "scope_summary_lines": [...], "error_message": null
}
```

- **summary{}** 汇总：`high_count` / `review_count` / `tip_count` / `similarity_count` / `manual_unchecked_count` / `conclusion`(一句话结论) / `conclusion_phase`(full/rules_only/semantic_partial) / `overview_ready` / `semantic_review.state` / `semantic_review.message_zh`。
- **bid_files[]** 被查文件：`id` / `filename` / `content_hash` / `metadata` / `created_at`。
- **issues[]** 合规问题（核心）：`id` / `bid_file_id` / `bid_filename` / `issue_type`(如 `hard_field_presence` 等) / `risk_level` / `result_type` / `title` / `description` / `tender_evidence` / `bid_evidence` / `suggestion` / `confidence`(0-1) / `status` / `user_note`。
  > ⚠️ **实测枚举值**：`risk_level` = **`high`/`review`/`tip`**；`result_type` = **`suspected`/`detected`**。`summary.high_count/review_count/tip_count` 按 `risk_level` 计数。
  > 证据多形态（因引擎而异）：语义类 `tender_evidence/bid_evidence` 主键 **`excerpt`**（另含 `chunk_id`/`section_path`/`section_title`）；硬字段类 `{field,expected_text}`；规则未命中 `{source}`。`report.py` 的 `_ev` 已按 `excerpt > text > field/expected_text > source` 兼容。
- **similarity_issues[]** 多文件雷同（仅多份投标文件时有）：`file_a_id`/`file_b_id` / `file_a_name`/`file_b_name` / `similarity_type`(text_overlap/structure_overlap) / `risk_level` / `title` / `evidence_a{text,page}`/`evidence_b{...}` / `similarity_score`(0-1) / `suggestion` / `status`。
- **manual_items[]** 人工核查清单：`category` / `title`(简短标题) / `description` / `source` / `is_checked` / `note`(备注) / `checked_by` / `checked_at`。
- **scope_summary_lines[]** 检查范围摘要（适合报告开头展示）。

报告推荐布局：总览 → 风险摘要(summary) → 高风险问题(issues 高) → 待人工复核(result_type=semantic) → 格式提示(低) → 多文件相似度 → 人工核查清单。`scripts/report.py` 已实现此布局。
