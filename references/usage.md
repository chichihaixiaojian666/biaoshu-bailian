# 执行细节（操作手册）

本文件是**执行任务时的完整操作指引**——做解读/制作/合规、生成报告、处理边界情形时按需查阅。接口契约与返回字段见 [api.md](api.md)。

> ⚠️ **数据外发与知情同意**：解读/制作/合规都会把用户提供的文件**上传至招采猫云端**（`biaoshu.zhiliaobiaoxun.com`）处理，此类文件常含商业、报价与个人信息；**上传文件与结果会以账户身份留存在招采猫服务器**（结果/成品约 7 天过期，可登录官网查看管理）。**首次上传前必须确认用户知悉并同意**（完整披露见 SKILL.md「⚠️ 权限与数据说明」）。

## ⚠️ 输出约定（必须遵守，除非用户明确说不要）

运行 `zcm.py` 时**老老实实把脚本输出原样给用户看**，不得为了「省事/抽字段」把它藏起来：

1. **实时进度照常显示**：解读/抽包/生成/合规都会把百分比+阶段（如 `[20%] 智能解读中`）打到 **stderr**。**不要 `2>` 重定向、不要吞掉**——用户要看到进度推进。需要后台实时播报时用 `progress-stream` + Monitor，而非把进度倒进文件。
2. **完整打印结果文件的绝对全路径**：每次产出后，必须把以下文件的**绝对路径**明确告知用户：
   - 智能解读结果/报告（`*_智能解读.html` 等）
   - 成品标书 `*_投标文件.docx`
   - 合规审查结果/报告（`*_合规审查.html` 等）

   脚本本身已打印这些路径（`generate`/`result` 成功后打 `已下载成品标书：<全路径>` 到 stderr，并把路径打到 stdout）；**别用 `>`/`2>` 把它们重定向掉**。若用 `--no-wait`，完成后须主动补取结果/出报告并打印全部全路径。

> 反例（禁止）：`python3 scripts/zcm.py generate <pid> > out.json 2> log` —— 这会同时藏掉进度和成品全路径。

3. **凭证/积分类提示不要把命令与 exit 码原样抛给用户**：缺 Key（exit 2）、积分不足（402）这类脚本输出是给你（助手）看的提示。你应当把它翻译成一句「用户下一步该做什么」（去官网拿 Key / 打开绑定或充值链接），保存 Key 的 `login` 命令由你代跑——不要让用户自己敲命令。
4. **本手册里的一切命令永不面向用户**（SKILL.md 第一铁律）：命令只在后台执行；向用户介绍功能或举例时，用 SKILL.md 各功能「使用示例」的场景话术（用户怎么说 → 得到什么），不要把本文件的命令、参数、代码块贴进回复。

## 用法速查（完整流程）

```bash
python3 scripts/zcm.py login --app-key bk_live_xxx   # 1. 配凭证（存 ~/.zcm/config.json；或环境变量）
python3 scripts/zcm.py me                            #    连通 + 余额自检
python3 scripts/zcm.py interpret 招标文件.pdf --report html   # 2. 解读 → project_id（+解读报告）
python3 scripts/zcm.py packages <project_id>         # 3. 抽包（多包才需选包）
python3 scripts/zcm.py generate <project_id>         # 4. 生成成品标书（扣积分）
python3 scripts/zcm.py compliance <project_id> 投标文件.docx --report html --name 招标文件.pdf   # 5. 可选：合规审查
```
招标文件支持 `.pdf/.doc/.docx`；投标文件 `.doc/.docx`。全部自动轮询、实时播报后端进度。各步详解见下。

## 目录
- [第 1 步：凭证](#第-1-步凭证)
- [第 2 步：智能解读](#第-2-步智能解读)
- [第 3 步：抽取分包](#第-3-步抽取分包)
- [第 4 步：生成成品标书](#第-4-步生成成品标书)
- [第 5 步：合规审查](#第-5-步合规审查)
- [报告生成与命名](#报告生成与命名)
- [关键约定](#关键约定)

---

## 第 1 步：凭证

凭证默认存在 **`~/.zcm/config.json`**（skill 目录之外，权限 600，不随 skill 分发）。读取优先级：环境变量 > 凭证文件。`ZCM_CONFIG` 可改凭证文件路径。

**只需 App Key 一项**，由用户**自行到官网获取**（本 skill 不代注册、不收集手机号/验证码）。获取全路径（转述时逐步骤完整给出，链接原样显示完整 URL）：
打开官网 https://biaoshu.zhiliaobiaoxun.com/ → 手机号 + 短信验证码注册并登录（新用户赠积分）→ 点**左侧菜单『开放 API』**，在弹出面板中生成/查看 App Key（形如 `bk_live_xxxxx`，重置后旧 Key 立即失效）。

**配置方式（Key 全程不经对话；不得索要或引导用户在对话中粘贴 Key）**：
1. **凭证文件（唯一引导方式）**：用户自行创建 **`~/.zcm/config.json`**（完整全路径，`~` 为用户主目录），内容模板如下，保存后建议 `chmod 600`：
   ```json
   {"app_key": "bk_live_xxxxx"}
   ```
   （可选字段：`base` 自定义 API 地址、`output_dir` 成品存放目录。）向用户介绍时把**文件全路径和模板**原样给出。
2. **被动兜底**：用户未按指引、**自行**把 Key 粘进对话时——不要复述 Key，由你代跑 `login --app-key <Key>` 保存，然后提醒：「Key 已保存；对话中粘贴的 Key 会留在会话记录里，介意可到官网『开放 API』重置后改用凭证文件方式」。

- 缺凭证时脚本会打印官网获取指引并退出（码 2），把指引转述给用户即可。
- 先 `python3 scripts/zcm.py me` 确认连通与积分余额（生成会扣分）。

### 积分不足（402，给用户自助链接，skill 不代办）

积分不足时脚本会打印引导，照原样转达给用户，由用户**自行登录官网充值**后回到对话继续，App Key 全程不变：充值入口 `https://biaoshu.zhiliaobiaoxun.com/recharge`（用注册手机号登录后操作）。

- 🔒 **凭证保护（强制）**：平台 402 错误体里的 `recharge_url` / `bind_url` **携带明文 `bind_key`（即用户的 App Key）**——**一律不得把这类带 Key 的链接转发进对话**（会话记录、截图、链接预览都可能泄露 Key，他人拿到即可操作该账户）。只给上面这条不含任何参数的普通充值链接。
- ⛔ **禁止**：已有 Key 的用户，别引导他去官网「另注册新账号 / 另生成新 Key 再切换」——积分会留在孤立新账号上、还得换 Key。

## 第 2 步：智能解读

唯一招标文件入口；只在这步传一次，后续全程复用 `project_id`。

```bash
python3 scripts/zcm.py interpret /path/招标文件.pdf      # 仅本地路径
```
- 支持 `.pdf/.doc/.docx`，**≤ 50 MB**（超限脚本提前报错）。自动轮询，结束打印 `project_id`（**记下它**）+ **完整解读结果**。
- **不支持云端链接**：传入 http(s) 链接会被脚本直接拒绝（本 skill 不做任何远程抓取）。用户给的是链接时，请他先自行下载到本地，再提供本地路径。
- **直接把解读结果展示给用户**——含 8 维度 + 控标洞察：项目基本信息 / 合标项 / 废标项 / 评审项 / 关键要求 / 商务条款 / 报价要求 / 采购背景分析 / 控标洞察（`decision_analysis`）。挑重点讲（控标建议、废标红线、评分结构），别只丢 `project_id`。字段口径见 [api.md 附录 A](api.md)。
- 展示后**主动问是否生成解读报告**（见[报告生成与命名](#报告生成与命名)）。

## 第 3 步：抽取分包

```bash
python3 scripts/zcm.py packages <project_id>
```
- 把返回的 `packages` 呈现给用户挑选，收集选中的 `package_ids`。
- `is_multi_package=false` → 跳过选包，第 4 步不带 `--package-ids`。

## 第 4 步：生成成品标书

**唯一扣积分的步骤**，耗时较长。生成前**先问用户存哪**：
- 给了路径 → `-o <路径>`；想长期固定 → `login --output-dir <目录>`。
- 不指定 → 默认 skill 包同级 `biaoshu-bailian-files/`，文件名 `招标文件名_投标文件.docx`（招标文件名从本地缓存取，取不到退化 `bid_<job_id>.docx`）。

```bash
python3 scripts/zcm.py generate <project_id> --package-ids 11,12 --total-pages 80 -o 投标文件.docx
# 非多包：python3 scripts/zcm.py generate <project_id>
```
- 存放目录优先级：`-o` > `ZCM_OUTPUT_DIR` > `login` 存的 `output_dir` > 默认 `biaoshu-bailian-files/`。
- 自动轮询（默认超时 3600s，`--timeout` 可调）。完成后打印**成品完整路径**+所在目录，**两项都告诉用户**。
- ⏱ **生成可能耗时 >10 分钟**（实测 30 页约 15 分钟）。脚本本身轮询不会超时，但**前端/工具调用常有 ~10 分钟上限**会把命令杀掉——**注意：后端任务不受影响、仍在跑，切勿重新提交（会重复扣费）**。长任务推荐：`generate <pid> --no-wait` 拿 `job_id`，再用 `progress-stream <job_id>`（配合 Monitor 后台实时播报）续查到终态，最后 `result <job_id> -o <路径>` 下载并打印全路径。万一命令被杀，用同一 `job_id` 续查即可，不要重发 generate。

## 第 5 步：合规审查

要**两样输入，都要让用户提供**：
1. **招标文件**（`.pdf/.doc/.docx`）→ 经第 2 步解读产出 `project_id`；已解读则复用，不重传。
2. **投标文件**：**一份或多份** `.doc/.docx`，被审查对象（仅本地路径），**每份 ≤ 1024 MB**。

```bash
python3 scripts/zcm.py compliance <project_id> /path/投标A.docx /path/投标B.docx
# 暗标/电子标：加 --blind / --electronic
```
- **不支持云端链接**：传链接会被脚本拒绝，请用户先自行下载到本地。
- **直接把合规结果展示给用户**——含 `summary`（风险计数 + 一句话结论）、`issues[]`（风险等级/招标依据/投标证据/修改建议）、`similarity_issues[]`（多文件雷同）、`manual_items[]`（人工核查清单）。优先讲高风险与结论。字段见 [api.md 附录 B](api.md)。
- `risk_level` 实测为 `high`/`review`/`tip`，脚本输出与报告**已自动转中文**（高风险/待复核/提示），直接用中文呈现。
- 未解读就调 → 409；投标文件缺失/类型不对 → 422（两份输入缺一不可）。
- 展示后**主动问是否生成合规报告**（见下）。

## 报告生成与命名

解读/合规结果可渲染成报告（HTML / Word），零依赖：

```bash
# 随命令一步出（默认 html；要 Word：--report both）
python3 scripts/zcm.py interpret 招标文件.pdf --report html
python3 scripts/zcm.py compliance <pid> 投标.docx --report html --name 招标文件.doc
# 按 job_id 补出
python3 scripts/zcm.py report --job <JOB_ID> --name 招标文件.pdf            # html
python3 scripts/zcm.py report --job <JOB_ID> --name 招标文件.pdf --format both  # +Word
```
- **默认只出 HTML**；用户明确要 Word 才 `docx`/`both`。
- 命名：`招标文件名_智能解读` / `招标文件名_合规审查`。取名优先级：`--name` > 结果自动识别（`original_filename` / `project_info.项目名称` / 本地缓存）> `标签_时间戳`。
  - `interpret` 自动用上传文件名；`generate` 自动用缓存名；**`compliance`/`report --job` 拿不到招标文件名时务必带 `--name`**，否则退化时间戳。
- 报告内容依赖后端按 [api.md 附录 A/B](api.md) 返回完整结果；`/result` 只回句柄或字段空时，报告注明「无明细」而不报错。

## 关键约定

- **必须输出完整路径**：解读报告 / 成品标书 / 合规报告生成后，把**每个文件的完整绝对路径**逐行告诉用户（脚本已用「已生成…/已下载…」打印绝对路径，照搬即可）——**不要只说落在某目录**。
- **进度播报（两阶段，必须这样做才能实时）**：Bash 工具不流式传输 stderr，`--no-wait` + `progress-stream` + Monitor 是唯一能让用户看到实时进度的方式。长任务（interpret / generate / compliance）统一走以下三步：
  1. **提交**（同步，快）：加 `--no-wait`，Bash 运行后立即拿到 `job_id`。
  2. **实时监听**：`Bash(run_in_background=True)` 运行 `python3 scripts/zcm.py progress-stream <job_id>`，再用 Monitor 订阅该进程 stdout——每行状态变更即时通知 Claude，Claude 实时转达给用户（如「5% 准备文档」→「20% 解读中」→「完成」）。Monitor 的 description 用正常任务名，**不带「重试」等临时标签**——即使是 worker_lost 后重新提交的 job，新 job 已正常运行，描述应反映当前状态而非历史原因。
  3. **取结果 + 生成报告 + 输出路径**：Monitor 收到 `[完成]` 后必须主动补齐后处理，三类任务各有对应步骤：
     - `interpret`：`result <job_id>`（提取 project_id）→ `report --job <job_id> --format html`（生成解读报告）→ 输出报告全路径
     - `generate`：`result <job_id> -o <路径>.docx`（下载标书）→ 输出 docx 全路径
     - `compliance`：`result <job_id>`（打合规摘要）→ `report --job <job_id> --format html`（生成合规报告）→ 输出报告全路径
     
     > `--no-wait` 跳过了同步模式的后处理，**AI 必须手动补**，否则报告文件不会生成，用户看不到路径。
  > 仅在用户不需要看进度或调试时才用单命令前台运行（无 `--no-wait`）。`packages` / `me` 等快速命令无需两阶段。
- **断点续查**：`job <job_id>` 查状态、`result <job_id> [-o file]` 取结果、`cancel <job_id>` 取消。
- **幂等**：网络重试给提交命令加 `--idempotency-key <UUID>`，避免重复建任务/重复扣费。
- **续接已有 project**：用户解读后直接说「帮我生成」，沿用 `project_id` 从第 3 步继续，不重传。
- **错误处理**：脚本已把 401/402/404/422/429 转中文。常见——402 余额不足让用户充值；整层 404 多为开放 API 总开关未开，让管理员开启；429 退避重试。完整对照见 [api.md](api.md)。
- **积分不足（402）**：脚本只打印**不含凭证参数的官网充值链接**，照原样转达即可；错误体里带 `bind_key` 的 `recharge_url`/`bind_url` 一律不转发（见第 1 步「凭证保护」）。
