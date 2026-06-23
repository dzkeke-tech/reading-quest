# CLAUDE.md — 阅读探险队 Reading Quest

> 这是 Claude Code 在本仓库工作时**必须先读**的项目记忆文件。
> 每次开始任务前先通读本文件，结束后如有重要决策/踩坑，**追加写回本文件**（这是项目的长期记忆）。

---

## 一句话简介

「阅读探险队 / Reading Quest」是一个**全家共用的阅读习惯打卡游戏**，单文件 HTML 应用，部署在 GitHub Pages，后端用 Cloudflare Worker + KV 做云同步和 AI 调用。中英文双语界面。

---

## 家庭成员（profiles）

| id | 名字 | 角色 band | 说明 |
|----|------|-----------|------|
| charles | Charles | big（大童） | 儿子，约9岁，MAP Reading RIT 218，Lexile 790–1120L，小学中高年级 |
| kylie | Kylie | young（小童） | 女儿，约6岁，学龄前/低年级 |
| dad | Kevin | adult（大人） | 爸爸（榜样，读成人书） |
| mom | Jenny | adult（大人） | 妈妈（榜样，读成人书） |

**关键**：这是**全家**应用，不是纯儿童应用。大人读的不是童书，问答难度要有成人级；哥哥的难度会随等级升级。

---

## 技术架构

- **前端**：单个 `index.html`（约 2000 行，HTML+CSS+JS 全在一个文件里）。仓库根目录文件名必须是 `index.html`。
- **部署**：GitHub Pages。仓库 `dzkeke-tech/reading-quest`，分支 `main`，根目录。线上地址 `https://dzkeke-tech.github.io/reading-quest/`。推送后 GitHub Actions 自动构建部署（约1分钟）。
- **后端**：Cloudflare Worker `reading-quest-ai`，地址 `https://reading-quest-ai.dzkeke.workers.dev`
  - `POST /ai` → 代理 Gemini（gemini-2.5-flash），用于书名分类、AI 鼓励语、成长点评
  - `GET/PUT /sync` → Cloudflare KV 读写，云同步
  - KV 绑定名 `RQ_DATA`，KV key = `family_data`
  - Worker 源码：`reading-quest-worker.js`（改了要手动在 Cloudflare 控制台 Deploy，GitHub 不会自动部署 Worker）
- **PWA**：`manifest.json` + `icon-180/192/512.png`，可加到主屏幕。

### 环境约束
- 中国大陆网络直连 **访问不了 workers.dev**，iPad 等设备需要代理才能同步；离线时本地可用，联网后同步。
- Claude 的 bash 环境**访问不了 github.io**（会返回 "Host not in allowlist"），所以**不能用 curl 线上文件来判断线上版本**。要确认线上版本，让用户在浏览器控制台输入 `APP_VERSION`。

---

## 数据模型（localStorage + 云端 KV，结构相同）

- 全局状态 `S = load()`，存储 key = `readingquest_v1`
- `S.profiles[id]`：每个成员
  - `points`：**可花余额**（兑换奖励扣这个）
  - `earned`：**累计赚取**（只增不减，**等级/伙伴解锁用这个算**，兑换不影响）⚠️ 见下方"积分与等级"
  - `goals{zh,en}`：每日中/英文目标分钟
  - `lexile{min,max}|null`：英文 Lexile 区间（仅 charles 有）
  - `history[]`：每次打卡记录（含 book/lang/fic/genre/minutes/mode/progress/pages/finished/note/points/answers）
  - `reading[]`：在读的长书（连载未读完的）
  - `badges[]`、`streak`、`lastDate`、`totalCheckins`、`booksDone`
- `S.rewards[]`、`S.redemptions[]`、`S.settings`、`S.order`

### 云同步逻辑（极其重要，别改坏）
- `cloudGet()` 返回：`undefined`=连接失败/未配置；`null`=云端空（首次，应把本地推上去初始化）；`object`=云端有数据（覆盖本地）。
- **`null` 和 `undefined` 语义绝不能混淆**——曾因把 `null` 当失败处理导致一直"无法连接云端"。
- `WORKER_URL` 必须在 script 顶层定义（约第715行），不能放函数里。
- 当前同步是"**云端覆盖本地**"，离线打卡有被覆盖风险（merge 合并模式尚未实现，见 TODO）。

---

## 积分规则（finishCheckin 函数，约第1335行起）

- 完成打卡 base = **+3**
- 时长：达成当日该语言目标后，超出部分每 2 分钟 +1（不封顶；未达标不计时长分）
- 首次达成当日语言目标 = **+5**
- 难度（仅英文且已知 Lexile）：刚刚好/挑战 +8，偏简单 +2；难度未知 +4
- 广度（和上次不同 虚构/非虚构 或体裁）= +5
- 双语（当天中英文都读）= +5
- 每回答一个问题 = **+3**（按实际作答数；纯笔记不给此分）
- 读书笔记写满字数 = **+5**（固定）
- 连击：≥3天 +8，≥7天 +15
- 读完一本书（按**全书总页数**）：≤50页 +10 / 50–100 +25 / 100–200 +60 / 200+ +100
  - 完整模式视为读完；连载模式需勾"读完了"。读完判定时**必须填总页数**。

**改积分规则时两处必须同步改**：① `finishCheckin` 里的计分逻辑；② 家长设置里"📐 积分规则"说明卡（约第1860行）。否则显示和实际对不上。

### 积分与等级（曾踩坑）
- 等级 `levelOf(p)` 用 **`p.earned`**（累计赚取），不是 `p.points`。LV_STEP=60。
- 兑换奖励只扣 `points`，**不动 `earned`**，所以兑换后等级不下降。
- 打卡加分时 `points` 和 `earned` **都要加**。
- 手动改积分时，`earned` 要保证不低于新积分。

---

## 题库（QUESTIONS，约第358行起）

按 **虚构/非虚构 × 完整/连载** 四象限选题库：`open` / `serial` / `nonfic` / `nonfic_serial`。
- **非虚构问主旨/论点/知识/事实vs观点/批判思考，绝不问"主角/情节/故事发展"**（曾踩坑：《人的境况》被问"主人公怎么了"）。
- 难度分 4 档（tier）：young 起步 tier1，big 起步 tier2，adult 起步 tier3；随等级每升3级 +1 档，最高 tier4。
- tier4 是成人/高阶题（文学分析、论证重构、辨析前提假设）。
- `MIN_CHARS`（开放题字数下限）：zh `{1:30,2:50,3:80,4:120}`，en（词数）`{1:20,2:35,3:50,4:80}`。
- `NOTE_MIN`（读书笔记字数下限）：`{1:60,2:100,3:150,4:200}`。
- serial/nonfic_serial 只有 tier1–3，tier4 时 buildQuestions 会**回退**到最高可用 tier（别删这个回退逻辑）。

### 打卡校验规则
- **答题 或 读书笔记，至少完成一样**（也可都做）。纯笔记可不答题、不选选择题。
- 校验失败用持久橙色提示条 `#finishWarn`（`warnFinish()`），不只用一闪而过的 toast。
- `finishCheckin` 开头会**直接从 DOM 读 textarea/输入框的值回填**（避免键盘听写/输入法没触发 oninput 导致答案漏更新）。别删这个回填逻辑。

---

## 版本标记机制

- 顶部 `const APP_VERSION = "2026-06-18-vXX"`，同时 `window.APP_VERSION`，家庭页底部也显示。
- **每次改动必须升版本号**（如 v13d → v13e 或 v14）。
- 用户验证线上是否更新：浏览器控制台输入 `APP_VERSION`，或看页面底部版本号。
- 当前版本：**2026-06-18-v13d**

---

## 🚨 测试规范（铁律，违反过多次，务必遵守）

**`node --check`（语法检查）只能抓语法错误，抓不到 `xxx is not defined` 这类运行时错误。** 历史上多次因为只做语法检查就交付，导致"点了没反应""提交卡死"。

**每次改完代码，交付前必须跑完整执行测试**：

1. 提取 `<script>` 内容，搭一个 DOM-stub 测试夹具（stub 出 document/localStorage/fetch 等）。
2. **真实执行**关键路径，不只是语法检查：
   - `finishCheckin` 全路径（虚构/非虚构 × 完整/连载 × 大人/小孩），验证积分增加、写入 history。
   - 所有 `renderX` 页面渲染（family/dash/rewards/shelf/history/parent/report）不报错。
   - **扫描所有 `onclick="fnName("` 里调用的函数，逐个确认已定义**（专门防"点了没反应"）。
3. **测试夹具每个用例之间要清空 DOM stub**（`document._e={}`），否则用例间状态串味会产生假阳性/假阴性。
4. 改了积分/题库/校验逻辑，要专门加针对性测试用例。

只有完整执行测试全绿，才能交付。

---

## 交付与部署流程

1. 改 `index.html`（文件名就是 index.html，别叫别的）。
2. 升 `APP_VERSION`。
3. 跑完整执行测试（见上），全绿。
4. 改了 Worker 才需动 `reading-quest-worker.js`，且要提醒用户去 Cloudflare 控制台手动 Deploy。
5. 提交 + 推送到 `main`。GitHub Actions 自动部署 Pages。
6. 告诉用户：上传/部署完，浏览器控制台查 `APP_VERSION` 或看页面底部版本号确认。

---

## 已知 TODO / 路线图

- [ ] **云同步改 merge 合并模式**（当前是云端覆盖本地，离线打卡有数据丢失风险）。这是最重要的待办。
- [ ] **导出分享卡片**：报告页"🖼️ 导出分享卡片"按钮目前是占位（提示"即将上线"）。要做成设计感强、可截图发朋友圈/小红书的精美卡片。可用 canvas 或 html-to-image 方案。
- [ ] 题库持续扩充、按使用反馈打磨。

---

## 历史踩坑清单（别重蹈覆辙）

1. **只做语法检查就交付** → 多次交付了带 `ReferenceError` 的坏代码（删了 `openParent`、删了 `const t=todayStr()`）。必须完整执行测试。
2. **闭包函数 `toString()` 内联进 onclick** → `doIt is not defined`，兑换奖励点了没反应。用全局回调暂存替代（见 `parentGate`/`submitGate`）。
3. **`null` vs `undefined` 在 sync 里混淆** → 一直"无法连接云端"。
4. **大段 str_replace 时误删相邻函数**（openParent 被报告代码替换吃掉）。改完要扫一遍关键函数还在不在。
5. **非虚构用了故事题库** → 问哲学书"主角怎么了"。按四象限选题库。
6. **等级用 points 算** → 兑换后等级下降。改用 earned。
7. **新功能渲染漏了**（读书笔记存了但记录页没渲染）。加字段后要确认所有展示处都渲染。
8. **用 curl 判断线上版本** → bash 环境访问不了 github.io，得到错误结论。让用户在浏览器查 APP_VERSION。

---

## 用户偏好

- 用户 Yi，技术背景（GitHub `dzkeke-tech`），在上海/香港时区，Mac + 代理（Clash）。
- 主要用中文沟通。
- **强烈反感来回反复改、浪费时间和 token**。要求交付前充分验证。
- 重大改动**落地前先过一眼**（先给改进建议/方案，确认后再实现），不要无人审查直接改生产。
