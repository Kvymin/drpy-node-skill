# drpy-node Skill Pack

本仓库是一组面向 Claude Code 的 drpy-node 专项能力包，用来让 AI 更稳定地完成 DS 源创建、修复、播放调试、全流程验证和仓库发布。

核心目标：

```text
用户意图 → 人格路由 → drpy-node skill 决策 → MCP 工具执行 → L1/L2/L3 证据 → 修复或发布结果
```

这不是单纯的提示词集合。正确使用方式是：人格文件负责判断路线和边界，skill 负责专业流程，`drpy-node-mcp` 负责真实执行与验证。

---

## 目录结构

```text
drpyS-soul.md                         # drpy-node 写源专家人格文件
drpy-node-source-create/              # 新建 DS 源 skill
drpy-node-source-workflow/            # 源修复、诊断、自动全流程总控 skill
drpy-node-play-debug/                 # 播放链路与 lazy 专项 skill
drpy-node-repo-upload/                # 仓库上传、替换、标签与公开状态 skill
```

每个 skill 目录通常包含：

```text
SKILL.md                              # skill 主流程与边界
test-prompts.json                     # Darwin/行为验证用例
references/                           # 可复用判断标准或速查资料
```

---

## 核心组件

### `drpyS-soul.md`

`drpyS-soul.md` 是 drpy-node 写源专家人格文件，负责把用户请求路由到正确的 skill 和 MCP 工具。

它的职责是：

- 判断用户是在新建源、修已有源、调播放，还是上传仓库。
- 识别只读、普通执行、自主全流程等模式。
- 约束 AI 不要只按提示词“模拟完成”，而要调用对应 skill/MCP 获取证据。
- 统一 L1/L2/L3 验证口径、A/B/C 上传建议和 blocker 分类。

它不是执行层；执行仍应通过 drpy-node skills 和 MCP 工具完成。

### `drpy-node-source-create`

用于从 URL 或站点信息新建 DS 源。

适用场景：

- “帮我写个 drpy 源”
- “根据这个站做 DS 源”
- “这个站像 mxpro，帮我新建源”
- “React/Vue 站，从零写源”

重点能力：

- Step 0 站点可用性预检。
- 模板/CMS 判断。
- HTML、签名 API、SPA/API、特殊内容站分流。
- `drpy_write_file` 写源，`drpy_check_syntax` / `validate_spider` / `test_spider_interface` / `evaluate_spider_source` 验证。
- 自主模式下可迭代到 `evaluate_spider_source == 100`，再生成 repo-upload handoff。

### `drpy-node-source-workflow`

用于已有源诊断、修复、评分和总控编排。

适用场景：

- “这个源 evaluate 低分”
- “详情为空、播放不通，先诊断”
- “搜索和播放都有问题”
- “给这个 URL 自动做源、修到 100、上传仓库”

重点能力：

- 作为总控处理：评估 → 定断点 → 分流 → 修复 → 复测 → 上传建议。
- 区分普通接口问题、evaluate 串联问题、播放链问题。
- 播放问题在 detail 稳定后转 `drpy-node-play-debug`。
- 仓库动作转 `drpy-node-repo-upload`，不在 workflow 内直接上传。
- 自主全流程下编排 URL → alive check → create → repair → play-debug → L3=100 → upload。

### `drpy-node-play-debug`

用于播放链路和 lazy 专项排障。

适用场景：

- “play 返回 parse:0 但打不开”
- “play.html 被当直链了”
- “player_data 加密了”
- “浏览器能播，fetch 找不到 m3u8”
- “lazy 不对，帮我修”

重点能力：

- 区分直链、解析、播放页、iframe、player_* 配置、特殊协议。
- 识别假通过：例如 `parse:0` 返回站内 `/play/*.html`。
- 必须先确认 detail 稳定，有真实 `play_url + flag`。
- 用 `test_spider_interface(play)` 和媒体证据复测。
- 自主模式中只做低风险 lazy 最小修复；登录、验证码、DRM、强 headless 检测必须停手。

### `drpy-node-repo-upload`

用于仓库上传、替换、改标签、切公开/私密和上传前校验。

适用场景：

- “把这个源上传仓库”
- “只改仓库标签，不要重新上传”
- “上传前检查能不能传”
- “修到 100 后自动上传，标签 ds，公开”

重点能力：

- 上传前给出 A/B/C 档和 L1/L2/L3 依据。
- `house_verify` 检查仓库连接。
- `house_file(upload/replace/update_tags/toggle_visibility/info)` 执行并核验。
- 自主最终版上传必须满足：L3=100、A 档、已预授权、路径明确、tags 明确、is_public 明确、目标对象明确。

---

## Skill 与 MCP 的关系

一句话：

```text
Skill 决策，MCP 执行。
```

skill 不替代 MCP。skill 提供流程、分流、边界和判断标准；MCP 工具负责真实读写、抓取、验证和上传。

常用 MCP 工具分层：

- 源文件管理：`list_sources`、`drpy_read_file`、`drpy_write_file`、`drpy_edit_file`
- 语法结构：`drpy_check_syntax`、`validate_spider`、`get_resolved_rule`
- 站点分析：`fetch_spider_url`、`guess_spider_template`、`analyze_website_structure`、`debug_spider_rule`
- 播放调试：`extract_iframe_src`、`test_spider_interface(play)`、Playwright network
- 接口验证：`test_spider_interface(home/category/detail/search/play)`、`evaluate_spider_source`
- 仓库发布：`house_verify`、`house_file(list/info/upload/replace/update_tags/toggle_visibility)`

人格文件和 skill 都要求：凡是涉及可用性、修复完成、播放真实可播、是否上传，都必须能落到 MCP 证据上，不能只按提示词口头判断。

---

## 验证等级

所有结论都应标注证据等级。

- L1：`drpy_check_syntax` + `validate_spider`
  - 能支持：语法/结构可进入下一步。
  - 不能支持：源可用、已修好、建议最终上传。
- L2：`test_spider_interface` 单接口
  - 能支持：某个接口真实通/断，定位断点。
  - 不能支持：全链路稳定、最终版可发布。
- L3：`evaluate_spider_source`
  - 能支持：首页→一级→二级→播放→搜索串联评分。
  - 不能支持：站点长期稳定。

`evaluate_spider_source` 分数模型：

- home：20
- category：20
- detail：25
- play：25
- search：10
- 总分：100

自主最终版和自动上传场景必须以 L3 为准，且目标是 `evaluate_spider_source == 100`。

---

## 自主全流程模式

当用户明确表达以下意图时，进入自主全流程模式：

- “自动完成”
- “修到 100”
- “满分后上传”
- “不要中途问我”
- “给网址做源并上传”

目标链路：

```text
URL
 ↓
alive check
 ↓
站型判断
 ↓
source-create 建源
 ↓
L1/L2/L3 验证
 ↓
按丢分接口最小修复
 ↓
必要时 play-debug 播放专项
 ↓
L3=100
 ↓
repo-upload 上传
 ↓
house_file(info) 核验
 ↓
最终回报
```

自主模式下可以连续执行低风险动作：站点预检、建源、改源、验证、播放专项、L3=100 后上传。

但遇到以下 blocker 必须停手：

- `broken_site`
  - 典型表现：DNS/连接失败、超时、长期 5xx、错误页、空壳且无可复现 API。
  - 动作：不建源、不上传。
- `hard_anti_bot`
  - 典型表现：验证码、强 headless 检测、DRM/WASM 等无法低风险复现。
  - 动作：不绕过，等用户确认。
- `missing_credentials`
  - 典型表现：需要登录、Cookie、Authorization、Token。
  - 动作：等用户提供凭据。
- `high_risk_change`
  - 典型表现：大段重写、多线路全局 lazy 改造、复杂签名逆向。
  - 动作：输出方案，等确认。
- `ambiguous_upload`
  - 典型表现：tags、is_public、目标对象、file_id/cid 冲突或多候选。
  - 动作：暂停确认目标。
- `score_below_target`
  - 典型表现：用户要求最终版/100 分，但 L3 未达标且无法继续低风险修。
  - 动作：报告断点和下一步。

---

## 典型任务流转

### 1. 给 URL 新建源，不上传

```text
用户给 URL
 ↓
drpy-node-source-create
 ↓
fetch_spider_url / guess_spider_template / analyze_website_structure
 ↓
选择路线 A/B/C/D
 ↓
drpy_write_file
 ↓
L1 + L2 + 必要时 L3
 ↓
返回源文件、验证结果和后续建议
```

### 2. 给 URL 自动做源、修到 100、上传

```text
用户给 URL + 自动完成/修到100/上传授权
 ↓
drpy-node-source-workflow 总控
 ↓
alive check
 ↓
drpy-node-source-create 建源
 ↓
evaluate_spider_source
 ↓
按丢分接口循环修复
 ↓
play 问题转 drpy-node-play-debug
 ↓
L3=100 后转 drpy-node-repo-upload
 ↓
house_verify + upload + info 核验
 ↓
返回 file_id / cid / tags / is_public
```

### 3. 已有源低分或接口异常

```text
用户给源名/问题
 ↓
drpy-node-source-workflow
 ↓
get_resolved_rule + L1/L2/L3
 ↓
判断断点：home/category/detail/search/play/evaluate 串联
 ↓
最小修复或转 play-debug
 ↓
复测并给上传建议
```

### 4. 播放不通或 lazy 异常

```text
先确认 detail 稳定
 ↓
拿真实 ids / play_url / flag
 ↓
drpy-node-play-debug
 ↓
判断直链/解析/播放页/iframe/player_data/假通过
 ↓
最小修 lazy
 ↓
同一 play_url + flag 复测
 ↓
返回 autonomous_next 或人工下一步
```

### 5. 上传仓库或改标签

```text
用户明确仓库动作
 ↓
drpy-node-repo-upload
 ↓
house_verify
 ↓
L1/L2/L3 + A/B/C 判断
 ↓
确认 path / tags / is_public / auto_replace / 目标对象
 ↓
house_file mutation
 ↓
house_file(info) 核验
```

---

## 上传发布守门

上传前必须区分“技术上能上传”和“现在建议上传”。

- A 档：建议上传
  - 典型依据：L3=100 或目标所需接口完整通过，且无红线。
  - 动作：普通模式确认后上传；自主预授权时可上传。
- B 档：技术可传但不建议
  - 典型依据：语法结构通过，但搜索 fallback、边界问题或整体链路未站稳。
  - 动作：说明风险，用户坚持才传。
- C 档：暂不应上传
  - 典型依据：语法失败、结构无效、detail 空、play 假通过。
  - 动作：不上传，先修复。

自主最终版上传要求更严格：

- 必须有 L3 证据。
- 必须 `evaluate_spider_source == 100`。
- 必须是 A 档。
- 必须明确本地路径、源名、tags、is_public、auto_replace 和目标对象。
- 上传后必须 `house_file(info)` 核验 `file_id/cid/tags/is_public`。

---

## dry-run 与只读模式

如果用户说：

- “只诊断”
- “只给方案”
- “dry-run”
- “不要改文件”
- “不要上传”
- “不要改标签”

则必须进入只读模式。

只读模式允许：

- 读取源文件。
- 分析站点结构。
- 跑非 mutation 验证。
- 给修复计划和拟调用参数。

只读模式禁止：

- `drpy_write_file`
- `drpy_edit_file`
- `house_file(upload/replace/update_tags/toggle_visibility)`

---

## 安全与边界

这套 skill 面向授权的 drpy-node 源开发、调试和仓库维护。

必须停手的情况：

- 需要登录但用户未提供凭据。
- 出现验证码、强风控、Cloudflare/headless 检测、DRM/WASM 等强反爬。
- 需要写死 Cookie、Authorization、Token。
- 需要绕过验证码、登录墙或检测机制。
- 上传目标、标签、公开状态、file_id/cid 有歧义。
- 用户要求 dry-run，却需要 mutation 才能继续。

---

## 行为验证

每个 skill 目录中的 `test-prompts.json` 用于验证行为是否符合预期，特别覆盖：

- 新建源 happy path。
- dry-run 不写文件。
- 坏站早停。
- 强反爬/缺凭据早停。
- 播放假通过识别。
- 自主流程 L3=100 后上传。
- L3<100 时拒绝最终版上传。
- tags/is_public/目标对象不明确时暂停确认。

常用本地校验：

```bash
python -m json.tool drpy-node-source-create/test-prompts.json >/dev/null
python -m json.tool drpy-node-source-workflow/test-prompts.json >/dev/null
python -m json.tool drpy-node-play-debug/test-prompts.json >/dev/null
python -m json.tool drpy-node-repo-upload/test-prompts.json >/dev/null

git diff --check -- drpyS-soul.md drpy-node-source-create drpy-node-source-workflow drpy-node-play-debug drpy-node-repo-upload
```

---

## 一句话总结

```text
drpyS-soul.md 负责路由和边界；四个 drpy-node skills 负责专业流程；drpy-node-mcp 负责真实执行；所有完成结论都必须有 L1/L2/L3 或仓库 info 证据。
```
