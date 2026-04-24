# drpy API 与全局函数参考

> 来源：drpy-node-mcp `get_drpy_libs_info` + 源码分析

## 一、核心请求函数

| 函数 | 说明 |
|---|---|
| `request(url, options?)` | 主HTTP请求，返回响应体(string)。options: `{headers, method, data/body, timeout, encoding}` |
| `post(url, options?)` | POST请求快捷方式 |
| `req(url, options?)` | 底层请求封装，返回 `{content, headers, code}` |
| `fetch(url, options?)` | fetch兼容的请求 |
| `reqs(urls, options?)` | 批量请求 |

## 二、解析函数

### HTML/XML 解析

| 函数 | 说明 |
|---|---|
| `pdfh(htmlOrNode, rule)` | 提取节点文本/属性。rule: `a&&Text` / `img&&src` / `.title&&Text` |
| `pd(htmlOrNode, rule, baseUrl?)` | 提取并标准化URL(自动解析相对路径) |
| `pdfa(htmlOrNode, selector)` | 获取节点列表(纯CSS选择器) |
| `pdfl(htmlOrNode, rule)` | 选集列表解析 |
| `jsp(baseUrl?)` | 创建jsoup解析器实例 |
| `jsoup` | jsoup类 |

### JSON 解析

| 函数 | 说明 |
|---|---|
| `pjfh(json, rule)` | JSON模式pdfh |
| `pjfa(json, rule)` | JSON模式pdfa |
| `pj(json, rule)` | JSON模式pd |

## 三、async 函数 this 上下文

当使用 async function 时，引擎通过 Proxy 注入 this：

```js
// 必须从 this 解构
一级: async function () {
    let { input, MY_URL, HOST, MY_CATE, MY_PAGE, MY_FL,
          pdfa, pdfh, pd, pjfa, pjfh, pj, KEY, fetch_params } = this;
    // input = 渲染后的URL
    // MY_CATE = 分类ID
    // MY_PAGE = 当前页码
    // MY_FL = 筛选条件对象
    // KEY = 搜索关键词
}
```

## 四、URL 工具

| 函数 | 说明 |
|---|---|
| `urljoin(base, path)` | 拼接/解析相对URL |
| `buildUrl(url, obj)` | 构建带query的URL |
| `getQuery(url)` | 解析query参数 |
| `parseQueryString(qs)` | query字符串→对象 |
| `buildQueryString(params)` | 对象→query字符串 |
| `urlDeal(url)` | URL处理 |
| `tellIsJx(url)` | 判断是否为解析URL |
| `encodeUrl(str)` | URL编码 |
| `urlencode(str)` | URL编码别名 |

## 五、结果格式化

| 函数 | 说明 |
|---|---|
| `setResult(d)` | 格式化视频列表返回。`d` 是 `{title, url, desc, pic_url, content}[]` |
| `setHomeResult(d)` | 首页结果格式化 |
| `setResult2(d)` | 替代格式化 |

`setResult` 内部将字段映射：
- `title` → `vod_name`
- `url` → `vod_id`
- `desc` → `vod_remarks`
- `pic_url` / `img` → `vod_pic`
- `content` → `vod_content`

## 六、加密与编解码

| 函数 | 说明 |
|---|---|
| `base64Encode(str)` / `base64Decode(str)` | Base64 |
| `md5(str)` / `md5X(str)` | MD5 |
| `aesX(data, key, iv)` / `aes(data, key, iv)` | AES加解密 |
| `desX` / `des` | DES加解密 |
| `rsaX` / `RSA` | RSA加解密 |
| `rc4Encrypt/Decrypt` / `rc4` | RC4 |
| `gzip(str)` / `ungzip(str)` | Gzip压缩 |
| `CryptoJS` | 完整CryptoJS库 |
| `JSEncrypt` / `NODERSA` | RSA库 |
| `forge` | node-forge库 |
| `CryptoJSW` | WASM加速版CryptoJS |

## 七、User-Agent 常量

| 常量 | 值 |
|---|---|
| `MOBILE_UA` | `Mozilla/5.0 (Linux; Android 12; ...)` |
| `PC_UA` | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/...` |
| `UC_UA` | UC浏览器UA |
| `IOS_UA` | iOS Safari UA |
| `UA` | 默认UA |
| `randomUa.generateUa()` | 随机UA生成 |

## 八、其他工具函数

| 函数 | 说明 |
|---|---|
| `log(msg)` / `print(msg)` | 日志输出 |
| `sleep(ms)` | 异步等待 |
| `deepCopy(obj)` | 深拷贝 |
| `computeHash(data)` | 计算哈希 |
| `jsonToCookie(obj)` / `cookieToJson(str)` | Cookie转换 |
| `toBeijingTime(ts?)` | 转北京时间 |
| `fixAdM3u8Ai(url, content, headers?)` | 修复带广告的m3u8 |
| `forceOrder(list)` | 强制排序 |
| `ENV.get(key)` / `ENV.set(key, val)` | 环境变量 |
| `local.get(store, key)` / `local.set(store, key, val)` | 本地持久化存储 |
| `OcrApi(img)` | OCR识别 |
| `simplecc` | 简繁转换 |
| `DataBase` / `database` | SQLite数据库 |
| `pupWebview` | Puppeteer浏览器实例(如可用) |
| `batchFetch(urls)` | 批量并发请求 |
