---
name: drpy-node-source-create
description: 适用于 drpy-node 新建 DS 源。用户提到“新建源”“写个 drpy 源”“分析这个站做 DS 源”“创建新规则”“从零开始做源”时使用。专注于站点分析、模板判断、规则生成与初步验证，优先走模板继承与最小覆盖路线，避免一开始就把修复、仓库上传等杂事混在一起。
---

# drpy-node Source Create

## 新增重点：模板继承核查清单（强制执行）

当站点明显命中模板时，不要一上来就手写 `一级/搜索/推荐/二级`。必须先完成这份核查清单。

### 核查顺序
1. `class_parse` 是否残留并覆盖 `class_name/class_url`
2. `double` 是否导致首页推荐为空
3. `url` 是否是真实分类模板，而不是首页导航表层链接
4. `searchUrl` 是否是真实搜索模板
5. 首页推荐节点是否真实存在
6. 分类页第 1 / 2 页翻页结构是否真实匹配 `fyclass/fypage`
7. 在必要时先删除手写 `一级/搜索`，优先验证模板内置规则

---

## 一条强规则：模板内置优先

### 禁止事项
当模板站未完成模板继承核查前，**禁止**一上来就大面积手写：
- `推荐`
- `一级`
- `搜索`
- `二级`

### 正确顺序
优先顺序应为：
1. 先修模板继承残留项
2. 先确认真实 `url/searchUrl`
3. 先验证模板内置链路
4. 只有模板内置规则确认不通，才允许最小覆盖

---

## 模板继承的四个高频坑

### 1. 模板残留 `class_parse` 会压住 `class_name/class_url`
如果继承模板后准备使用：
```js
class_name: '电影&电视剧',
class_url: '1&2'
```
则必须检查模板是否自带 `class_parse`。

若模板残留的 `class_parse` 仍在生效，可能导致首页 `class` 为空，自动评估拿不到分类 ID。

#### 处理原则
优先显式补：
```js
class_parse: ''
```
让 `class_name/class_url` 真正接管。

---

### 2. 首页推荐为空时，要检查 `double`
如果首页真实存在推荐卡片节点，但 `home.list` 仍为空，要考虑模板默认推荐可能按双层定位处理。

对于实际是单层推荐结构的站点，应优先尝试：
```js
double: false
```

#### 处理原则
- 真实推荐是单层 → 优先 `double:false`
- 真实推荐是分组+分组内卡片 → 再考虑双层

---

### 3. 分类 `url` 不能只看首页导航表面链接
不要看到首页导航是 `/list/1.html`，就直接推断 `url` 模板是 `/list/fyclass_fypage.html`。

必须验证：
- 分类页真实地址
- 翻页第 2 页地址
- `fyclass` 与 `fypage` 的真实落位

#### 已验证经验
有些站首页导航是 `/list/1.html`，但真正可用的分类模板可能是：
```js
/top/fyclass--------fypage---.html
```
另一些站则可能是：
```js
/index.php/vod/type/id/fyclass/page/fypage.html
```

所以：
**分类 `url` 必须通过真实网页和翻页结构验证，不可只靠表层导航推断。**

---

### 4. 要区分“规则不通”与“自动评估串联没接上”
如果 `evaluate_spider_source` 结果很差，不要立刻断言整份源不可用。

应优先用：
- `test_spider_interface(category)`
- `test_spider_interface(detail)`
- `test_spider_interface(play)`

分别验证单接口是否真实可用。

#### 处理原则
- 如果单接口通，但自动评估不通 → 优先怀疑自动串联条件（如首页 class、模板残留覆盖、`double`、错误分类 URL）
- 不要把“评估器没串起来”误判成“规则完全不通”
