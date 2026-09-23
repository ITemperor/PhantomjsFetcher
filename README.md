# PhantomjsFetcher · JS 渲染页面抓取器

> 用 PhantomJS 起一个本地渲染服务，让 Python（Tornado）能拿到**执行完 JS 之后**的 HTML。

**原始实现**：源自 [pyspider](https://github.com/binux/pyspider) 作者 Binux 的方案，本仓库为学习留档的一份拷贝。

**维护状态**：⚠️ **已归档**。PhantomJS 本体在 2018 年停止维护，现代项目请直接看第六节的替代方案。

---

## 一、解决什么问题

普通的 `requests` 只能拿到服务端吐出来的原始 HTML。页面靠 JS 拼数据时（当年的 SPA、
懒加载列表、评论区异步加载），拿到的就是个空壳。

这个项目做的事情很直白：**用真实浏览器内核把页面跑一遍，再把渲染后的 DOM 交给你**。

## 二、架构

```
┌─────────────────┐   POST JSON    ┌──────────────────┐
│ Python 抓取脚本 │ ─────────────► │ phantomjs 服务    │
│  (tornado)      │ ◄───────────── │  (真实 WebKit)    │
└─────────────────┘  渲染后的 HTML  └──────────────────┘
```

| 文件 | 角色 |
|---|---|
| `phantomjs_fetcher.js` | PhantomJS 端 HTTP 服务，收 JSON 请求 → 开页面 → 执行 JS → 回吐 HTML |
| `tornado_fetcher.py` | Python 客户端，`Fetcher` 类，支持连接池、同步/异步、超时与自定义 UA |

## 三、怎么用

**第 1 步：起渲染服务**

```bash
phantomjs phantomjs_fetcher.js 12306
```

> 需要先装 [PhantomJS](http://phantomjs.org/download.html) 并放进 PATH。

**第 2 步：Python 里调用**

```bash
pip install tornado
```

```python
from tornado_fetcher import Fetcher

fetcher = Fetcher(
    user_agent='phantomjs',                    # UA
    phantomjs_proxy='http://localhost:12306',  # 渲染服务地址
    pool_size=10,                              # 并发连接数
    async=False                                # True 则走 Tornado 异步
)

# 拿渲染后的 HTML
fetcher.fetch(url)

# 渲染完再额外执行一段 JS（比如滚动到底触发懒加载）
fetcher.fetch(url, js_script='setTimeout("function(){window.scrollTo(0,100000)}", 1000)')
```

## 四、可调参数

| 参数 | 说明 |
|---|---|
| `user_agent` | 请求 UA |
| `phantomjs_proxy` | 渲染服务监听地址，端口与第 1 步一致 |
| `pool_size` | 并发上限（`CurlAsyncHTTPClient` 连接池） |
| `async` | 是否异步模式，`False` 时同步阻塞返回 |
| 默认请求项 | `timeout=120`、`use_gzip=True`、`allow_redirects=True` |

## 五、与 pyspider 的关系

这套方案最早出现在 **pyspider** 里作为可选抓取组件。本仓库把它单独拎出来，
方便不跑整套框架、只想拿一个 JS 渲染能力的人直接用。

参考：<https://github.com/binux/pyspider>

## 六、现代替代方案（推荐）

PhantomJS 停更后，社区已经换血，新项目不要再用它：

| 场景 | 现在的做法 |
|---|---|
| 要渲染 JS | **Playwright**（Chromium / Firefox / WebKit，微软维护） |
| 只要少量异步数据 | 直接抓接口（`requests` + 浏览器 Network 面板抄 XHR） |
| 想轻量一点 | `requests-html` / `httpx` + `html artillery` 之类 |
| 大规模并发抓取 | Playwright + asyncio 池 / Scrapy + playwright 中间件 |

用 Playwright 重写只需几行，且不用单独维护一个 PhantomJS 进程：

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    page = p.chromium.launch().new_page()
    page.goto(url)
    page.wait_for_timeout(1000)
    html = page.content()
```

## 七、许可

原作者保留权利（文件头注明 Author: Binux），本仓库仅做学习留档与中文注释。

---

## 免费赞助

这套东西是白送的：**不收费、不锁功能、不塞广告**。如果它帮你省了时间、或者多赚了钱，
可以扫码请 Emperor 喝杯茶 —— 完全自愿，不打赏也照样用、照样更新。

<p align="center">
  <img src="assets/sponsor-qr.png" alt="免费赞助 · Emperor、| 说事-不闲聊" width="280">
</p>

<p align="center"><sub>扫码可备注一句你在做什么类目，方便后续针对性更新</sub></p>
