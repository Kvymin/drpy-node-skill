# 播放 lazy 模板默认逻辑摘要

> 来源：drpy-node 引擎模板系统中的内置 lazy 逻辑。
> 用于排查时判断当前源的 lazy 行为是否符合模板默认策略。

---

## 1. common_lazy（通用免嗅探）

模板系统通用免嗅探 lazy 的核心逻辑：

```js
let html = await request(input);
let hconf = html.match(/r player_.*?=(.*?)</)[1];
let json = JSON5.parse(hconf);
let url = json.url;

if (json.encrypt == '1') {
  url = unescape(url);
} else if (json.encrypt == '2') {
  url = unescape(base64Decode(url));
}

if (/\.(m3u8|mp4|m4a|mp3)/.test(url)) {
  result = { parse: 0, jx: 0, url };
} else {
  result = url && url.startsWith('http') && tellIsJx(url)
    ? { parse: 0, jx: 1, url }
    : input;
}
```

### 行为解读
| 条件 | 返回 | 说明 |
|---|---|---|
| 解析到 m3u8/mp4/m4a/mp3 | `{parse:0, jx:0, url}` | 直链，无需解析 |
| 解析到站外解析链接 + tellIsJx | `{parse:0, jx:1, url}` | 站外解析器处理 |
| 都不满足 | `input`（回退原链接） | 交给默认嗅探 |

### 加密处理
| encrypt 值 | 处理方式 |
|---|---|
| `'1'` | `unescape(url)` |
| `'2'` | `base64Decode(url)` → `unescape()` |

---

## 2. def_lazy（默认嗅探）

```js
return { parse: 1, url: input, js: '' }
```

### 行为解读
- 站点播放页本身交给前端/外部解析器处理
- `parse:1 + 网站播放页链接` 不一定是错，可能正是模板默认策略

---

## 3. cj_lazy（采集站）

### 逻辑
- 直链 m3u8/mp4 → `parse:0`
- `parse_url` 以 `json:` 开头 → 先请求 JSON 解析接口再取 `json.url`
- 否则拼接 `rule.parse_url + input`

### 排查要点
- 检查 `rule.parse_url` 是否配置
- 检查是否走 `json:` 解析接口
- 检查返回 JSON 里是否真的有 `url` 字段

---

## 快速判断对照表

| 播放页特征 | 推荐策略 |
|---|---|
| 页面有 `player_*` JSON 配置 | common_lazy：解析 player 配置 |
| 播放页直接交给解析系统 | def_lazy：`parse:1` |
| 采集站/解析接口 | cj_lazy：检查 parse_url |
| 漫画/小说/投屏 | 特殊协议（pics:///novel:///push://）|

---

## 4. playParseAfter 后处理机制

lazy 返回后，框架调用 `playParseAfter()` 做最终判断：

```js
parse = SPECIAL_URL.test(playUrl) || /^(push:)/.test(playUrl) || 
        /\.(m3u8|mp4|m4a|mp3)/.test(playUrl) ? 0 : 1;
jx = tellIsJx(playUrl);
```

**含义**：
- 即使 lazy 返回 `{parse:1, url}`，如果 url 以 m3u8/mp4/m4a/mp3 结尾 → 框架自动覆盖为 parse:0
- 即使 lazy 返回纯字符串 URL（无对象包裹）→ 框架自动处理后返回

---

## 5. 常见加密模式对照

| 加密方式 | 特征 | 处理 |
|---------|------|------|
| `encrypt: '1'` | player_* JSON 中 | `unescape(url)` |
| `encrypt: '2'` | player_* JSON 中 | `base64Decode(url)` → `unescape()` |
| URL 非 http 开头 | 如裸 base64 字符串 | `base64Decode(input)` |
| 携带 sign/timestamp | URL 含 `?sign=...&t=...` | 需逆向签名算法 |
| 动态 wasm 解密 | 页面加载 wasm | 调用 wasm 或预解密映射 |

---

## 6. 特殊协议处理

| 内容类型 | 协议 | lazy 返回格式 |
|---------|------|-------------|
| 漫画 | `pics://` | `{parse:0, url: 'pics://url1&&url2&&url3'}` |
| 小说 | `novel://` | `{parse:0, url: 'novel://' + JSON.stringify({title, content})}` |
| 投屏 | `push://` | `{parse:0, url: 'push://...'}` |

---

## 7. 多线路混合处理

当一个源有多条线路（混合直链 + 解析），不应一刀切：

```js
lazy: async function () {
    let {input} = this;
    if (/\.(m3u8|mp4)/.test(input)) return {parse: 0, url: input};  // 直链线路
    if (/webplay/.test(input)) {
        let html = await request(input);
        let iframe = pdfh(html, 'iframe&&src');
        return {parse: 0, url: iframe};
    }
    return {parse: 1, url: input};  // 需解析线路
}
```
