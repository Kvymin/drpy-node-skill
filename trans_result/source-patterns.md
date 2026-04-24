# drpy 源文件编写模式分析

## 一、源文件结构

所有源文件遵循统一结构：

```js
/*
@header({
  searchable: 1,
  filterable: 1,
  quickSearch: 0,
  title: '源名称[标签]',
  '类型': '影视',  // 影视|漫画|小说|听书
  lang: 'ds'
})
*/

var rule = {
    // ... 规则定义
};
```

`@header` 声明源元数据（标签、类型、搜索能力），框架解析用于前端展示。

## 二、五种编写模式

### 模式 1: 纯模板继承（最精简）

**代表**：`樱花动漫[优].js` — 仅7行自定义字段

```js
var rule = {
    title: '樱花动漫',
    模板: 'mxpro',
    host: 'http://www.yinghuadm.cn',
    url: '/show_fyclass--------fypage---.html',
    searchUrl: '/search_**----------fypage---.html',
    class_parse: '.navbar-items li:gt(1):lt(6);a&&Text;a&&href;_(.*?)\.html',
    tab_exclude: '排序',
    searchable: 2, quickSearch: 0, filterable: 0,
}
```

**优点**：代码量极少，利用模板完整能力  
**适用**：标准CMS网站，模板可直接匹配  
**注意**：必须验证class_parse、url、searchUrl与真实站点结构匹配

### 模式 2: 纯字符串规则

**代表**：`DJ音乐[听].js`、`360影视[官].js`（部分字段）

```js
// DJ音乐 - 混合字符串+函数
var rule = {
    推荐: '*',                                    // 继承一级
    一级: '.list_musiclist tr:gt(0);a&&title;img&&src;.cor999:eq(1)&&Text;a&&href',
    二级: '*',                                    // 跳过二级（直链嗅探）
    搜索: '*;*;*;.sc_1&&Text;*',                 // 部分继承一级
    lazy: async function () { /* 解析音乐链接 */ },
}

// 360影视 - 字符串+async混合
var rule = {
    推荐: 'json:data;title;cover;comment;cat+ent_id;description',  // JSON模式
    一级: 'json:data.movies;title;cover;pubdate;id;description',
    搜索: 'json:data.longData.rows;titleTxt||titlealias;cover;cat_name;cat_id+en_id;description',
    二级: async function () { /* 复杂的签名API逻辑 */ },
}
```

**优点**：简单直观，无需写函数  
**适用**：页面结构稳定的DOM解析  
**注意**：CSS选择器需精确；JSON模式用 `json:` 前缀

### 模式 3: async 函数驱动

**代表**：`独播库[优].js`、`番茄小说[书].js`

```js
// 独播库 - 全async函数
var rule = {
    host: 'https://api.dbokutv.com',
    // 自定义工具方法
    get: async function(path) {
        let url = this.getSignedUrl(path);
        let resp = await _fetch(url, { headers: this.headers });
        return JSON.parse(await resp.text());
    },
    class_parse: async function() { /* 返回静态分类 */ },
    推荐: async function() { let json = await this.get('/home'); ... },
    一级: async function(tid, pg) { ... },
    二级: async function() { ... },
    lazy: async function() { ... },
    // 加解密方法
    getSignedUrl: function(path) { ... },
    decodeData: function(data) { ... },
};
```

**核心规则**：
1. `this.input` 是 URL 不是响应，必须 `await request(this.input)`
2. `this.host` = rule.host
3. 纯数字 vod_id 必须设 `detailUrl`
4. POST 用 `body` 参数
5. `searchUrl` 必须带 `**`，否则 `this.KEY` 为空

**优点**：灵活处理复杂逻辑（签名、加密、API调用）  
**适用**：非标CMS、API驱动站、有反爬措施的网站  

### 模式 4: js: 内联代码

**代表**：`短视2` 模板中的一级规则

```js
一级: 'js:let body=input.split("#")[1];let t=Math.round(new Date/1e3).toString();let key=md5("DS"+t+"DCC147D11943AF75");let url=input.split("#")[0];body=body+"&time="+t+"&key="+key;print(body);fetch_params.body=body;let html=post(url,fetch_params);let data=JSON.parse(html);VODS=data.list.map(function(it){it.vod_pic=urljoin2(input.split("/i")[0],it.vod_pic);return it});',
```

**优点**：可在字符串规则中嵌入JS逻辑  
**适用**：需要少量计算但不想写完整async函数的场景  
**注意**：代码在沙箱中执行，通过 `executeJsCodeInSandbox()` 运行

### 模式 5: 网盘资源型

**代表**：`网盘[模板].js`

```js
var rule = {
    host: '',                    // 由hostJs动态获取
    url: '',
    searchUrl: '*',              // 仅搜索
    line_order: ['百度', '夸克', '优汐', '天翼', '123', '移动', '阿里'],
    search_match: true,
    quark_transfer: false,
    
    hostJs: async function () { /* 动态获取有效host */ },
    class_parse: async function () { /* 动态解析分类 */ },
    推荐: async function () { ... },
    一级: async function () { ... },
    二级: async function (ids) { /* 解析网盘分享链接 */ },
    搜索: async function (wd, quick, pg) { ... },
    lazy: async function (flag, id, flags) {
        // 按flag区分不同网盘解析逻辑
        if (flag.startsWith('夸克')) { ... }
        else if (flag.startsWith('百度')) { ... }
        // ...
    },
};
```

**特点**：多网盘资源聚合，lazy根据flag分派不同解析逻辑  
**适用**：百度/夸克/阿里/天翼等网盘资源搜索

## 三、特殊内容类型

### 3.1 漫画类型

```js
// lazy 返回 pics:// 协议
lazy: async function () {
    let html = await request(input);
    let arr = pdfa(html, '.single-content&&img');
    let urls = arr.map(it => pdfh(it, 'img&&data-src'));
    return {
        parse: 0,
        url: 'pics://' + urls.join('&&'),
        js: '',
    };
}
```

### 3.2 小说类型

```js
// lazy 返回 novel:// 协议
lazy: async function () {
    let html = await request(content_url);
    let json = JSON.parse(html);
    let ret = JSON.stringify({ title, content: json.data.content });
    return { parse: 0, url: 'novel://' + ret, js: '' };
}
```

### 3.3 音频/音乐类型

```js
// lazy 返回直链 m4a/mp3
lazy: async function () {
    let html = await request(input);
    let music = html.match(/var\s+music\s*=\s*(\{[\s\S]*?\})/)[1];
    music = JSON5.parse(music);
    input = urljoin(input, "//mp4.djuu.com/" + music.file + ".m4a");
    return input;  // 框架自动判断为 parse:0
}
```

## 四、模板库文件使用

### 4.1 引用方式

```js
// spider/js/_lib.request.js
$.exports = { requestHtml };

// spider/js/番茄小说[书].js
const {requestHtml} = $.require('./_lib.request.js');
```

所有 `_` 前缀文件是库文件，通过 `$.require()` 引用。

### 4.2 核心库文件一览

| 文件 | 用途 |
|---|---|
| `_base.js` | 最简模板（空壳，所有方法返回空） |
| `_base1[画].js` | 漫画基础模板 |
| `_base2[画].js` | 漫画API基础模板 |
| `_360.js` | 360影视签名算法 |
| `_fq.js` | 番茄小说免流代理 |
| `_fq2.js` | 番茄小说免流代理v2 |
| `_fq3.js` | 番茄小说免流代理v3 |
| `_qq.js` | QQ相关接口 |
| `_lib.action.js` | 动作交互库 |
| `_lib.cntv*.js/cjs` | CCTV央视算法（urlparse/wasm/live） |
| `_lib.douyin_*.cjs` | 抖音弹幕/签名 |
| `_lib.douyu.cjs` | 斗鱼弹幕 |
| `_lib.random.js` | 随机工具 |
| `_lib.request.js/cjs` | 增强请求库 |
| `_lib.scan.js` | 扫码库 |
| `_lib.tingyou.js` | 听书工具 |
| `_lib.waf.js` | 长亭WAF绕过（WebAssembly） |

## 五、hostJs 动态域名

```js
var rule = {
    hostJs: async function () {
        // 从配置接口动态获取有效域名
        let html = await request('https://config.example.com/domains');
        let json = JSON.parse(html);
        let domains = json[paramKey];
        // 测试各域名可用性
        for (let url of domains) {
            let res = await request(url, {method: 'HEAD', timeout: 3000});
            if (res) return url;
        }
        return domains[0];
    },
};
```

用于解决经常更换域名的网站，host在运行时动态获取。

## 六、二级访问前预处理

```js
var rule = {
    二级访问前: async function () {
        // 在执行二级解析前执行此函数
        // 可以返回新URL来改变请求目标
        let url = this.MY_URL + '?full=1';
        return url;  // 返回新URL，框架自动重新请求
    },
};
```

## 七、proxy_rule 代理规则

```js
var rule = {
    proxy_rule: async function (params) {
        // params.url 是请求的URL
        // 返回 [status, contentType, body]
        return [200, 'text/plain', 'proxy content'];
    },
};
```

用于自定义代理请求处理，可以实现图片处理、数据转换等功能。

## 八、完整 rule 字段速查

| 字段 | 类型 | 说明 | 必填 |
|---|---|---|---|
| `title` | string | 源显示名称 | 是 |
| `host` | string | 网站域名 | 是 |
| `类型` | string | 影视/漫画/小说/听书 | 是 |
| `模板` | string | 继承的模板名 | 否 |
| `模板修改` | function | 模板继承前修改 | 否 |
| `url` | string | 分类列表URL模式 | 是 |
| `homeUrl` | string | 首页URL（推荐/分类来源） | 否 |
| `detailUrl` | string | 详情页URL模式 | 数字vid时必填 |
| `searchUrl` | string | 搜索URL模式 | 搜索时必填 |
| `class_name` | string | 静态分类名(&分隔) | 无class_parse时必填 |
| `class_url` | string | 静态分类ID(&分隔) | 配合class_name |
| `class_parse` | string/func | 动态分类解析 | 替代class_name/url |
| `cate_exclude` | string | 分类排除正则 | 否 |
| `headers` | object | 请求头 | 否 |
| `timeout` | number | 超时(ms) | 否(默认5000) |
| `searchable` | 0/1/2 | 搜索能力 | 否 |
| `filterable` | 0/1 | 筛选支持 | 否 |
| `quickSearch` | 0/1 | 快速搜索 | 否 |
| `play_parse` | bool | 启用免嗅探 | 否(default:true) |
| `lazy` | string/func | 播放解析 | 否 |
| `double` | bool | 推荐双层定位 | 否 |
| `limit` | number | 每页条数 | 否 |
| `multi` | number | 多页聚合 | 否 |
| `filter` | string/obj | 筛选配置(gzip压缩) | 否 |
| `filter_url` | string | 筛选URL模板 | 否 |
| `filter_def` | object | 默认筛选值 | 否 |
| `encoding` | string | 页面编码 | 否 |
| `hostJs` | function | 动态获取host | 否 |
| `预处理` | function | 预处理函数 | 否 |
| `二级访问前` | function | 二级前置处理 | 否 |
| `line_order` | array | 线路排序 | 否 |
| `tab_exclude` | string | 线路排除正则 | 否 |
| `tab_rename` | object | 线路重命名 | 否 |
| `search_match` | bool | 搜索结果严格匹配 | 否 |
| `proxy_rule` | string/func | 代理规则 | 否 |
| `play_json` | array/bool | 播放JSON控制 | 否 |
| `sniffer` | bool | 辅助嗅探 | 否 |
| `isVideo` | string | 嗅探正则 | 否 |
