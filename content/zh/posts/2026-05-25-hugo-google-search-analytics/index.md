---
title: "让 Hugo 博客被 Google 搜到：Search Console、Sitemap 与 GA4 配置记录"
date: 2026-05-25
tags: [Hugo, PaperMod, Google Search Console, Sitemap, Google Analytics, GA4, SEO, GitHub Pages]
categories: [技术实践]
description: "记录一次 Hugo + PaperMod + GitHub Pages 博客接入 Google Search Console、sitemap、站点验证与 Google Analytics 4 的配置过程，并解释这些配置分别解决什么问题。"
---

这篇笔记记录一下我今天给 Hugo 博客补 Google 搜索和访问统计配置的过程。之前我对 Google Search Console、sitemap、Google Analytics 这些概念并不是很熟，配置过程中发现它们看起来都和“搜索”有关，但实际上解决的是完全不同的问题。

简单概括：

- **Google Search Console**：看网站在 Google 搜索里的表现，包括收录、搜索关键词、曝光和点击。
- **sitemap.xml**：给搜索引擎看的站点地图，告诉它这个站有哪些页面。
- **robots.txt**：告诉搜索引擎哪些路径可以抓取，并声明 sitemap 的位置。
- **Google Analytics 4**：统计真实访问行为，不只限于 Google 搜索流量。

下面以 Hugo + PaperMod + GitHub Pages 为例，把这些配置梳理一遍。

## 站点基础配置

首先，`baseURL` 必须写成线上真实地址。我的博客部署在 GitHub Pages 的 `/blog/` 子路径下，所以配置是：

```yaml
baseURL: "https://leemojiang.github.io/blog/"
title: "LEE's Blog"
theme: ["PaperMod"]
languageCode: "en-us"
publishDir: "public"
```

这里最容易出错的是 `baseURL`。如果站点实际部署在：

```txt
https://leemojiang.github.io/blog/
```

那就不能写成：

```yaml
baseURL: "https://leemojiang.github.io/"
```

否则生成出来的 canonical、sitemap、RSS、静态资源路径都有可能不对。

## SEO 基础信息

PaperMod 会读取 `params` 里的站点描述、关键词、作者等信息，用于生成页面的 meta 信息、OpenGraph、Twitter Card 和结构化数据。

```yaml
params:
  env: production
  author: "LEE"
  description: "LEE 的个人博客，记录 LLM、Agent、AI 编程、工程实践、FPV、旅行与个人研究笔记。"
  keywords: [LEE, Blog, LLM, Agent, AI 编程, 工程实践, FPV, Hugo, PaperMod]
  images: []
  schema:
    publisherType: Person
    sameAs:
      - "https://github.com/leemojiang"
```

这里的 `env: production` 很重要。PaperMod 在生产环境下会输出一些和 SEO 相关的内容，例如：

```html
<meta name="robots" content="index, follow">
<link rel="canonical" href="...">
<meta property="og:title" content="...">
<script type="application/ld+json">...</script>
```

其中：

- `robots: index, follow` 表示允许搜索引擎索引页面并跟随页面链接。
- `canonical` 表示当前页面的规范 URL，避免重复路径导致搜索引擎误判。
- `OpenGraph/Twitter Card` 用于社交平台分享预览。
- `schema JSON-LD` 是结构化数据，有助于搜索引擎理解页面类型和作者信息。

## Google Search Console 验证

Search Console 的第一步不是提交页面，而是证明“这个网站是我的”。Google 提供了几种验证方式，我这次用了两种：

1. HTML meta tag
2. HTML file upload

### Meta Tag 验证

Google 会给一段类似这样的代码：

```html
<meta name="google-site-verification" content="1KV4a977NZpLcPW85cn0mitZeYAdiIFP_Pu5OJW152c" />
```

在 Hugo/PaperMod 里，不需要手写整段 HTML。只要把 `content` 里面的值放到配置里：

```yaml
params:
  analytics:
    google:
      SiteVerificationTag: "1KV4a977NZpLcPW85cn0mitZeYAdiIFP_Pu5OJW152c"
    bing:
      SiteVerificationTag: ""
    yandex:
      SiteVerificationTag: ""
```

构建后页面里会自动生成：

```html
<meta name="google-site-verification" content="1KV4a977NZpLcPW85cn0mitZeYAdiIFP_Pu5OJW152c">
```

这个配置只用于 Google 站点所有权验证。Bing、Yandex 也有类似机制，但它们需要各自的 verification tag，不能直接复用 Google 的值。

### HTML 文件验证

Google 还可能让你下载一个文件，例如：

```txt
google26308ecd24c83a7e.html
```

文件内容类似：

```txt
google-site-verification: google26308ecd24c83a7e.html
```

Hugo 不会自动发布项目根目录里的普通文件，所以要把它放到 `static/` 目录：

```txt
static/google26308ecd24c83a7e.html
```

构建后它会出现在：

```txt
public/google26308ecd24c83a7e.html
```

线上访问路径就是：

```txt
https://leemojiang.github.io/blog/google26308ecd24c83a7e.html
```

这一步的作用仍然是验证所有权，不是提交页面，也不是统计访问。

## Sitemap 是什么

`sitemap.xml` 可以理解成给搜索引擎看的站点目录。它会列出站点有哪些页面，以及这些页面大概什么时候更新过。

在 Hugo 里可以这样配置：

```yaml
sitemap:
  changeFreq: weekly
  filename: sitemap.xml
  priority: 0.5
```

我的站点是中英文多语言站点，Hugo 会生成一个 sitemap index：

```txt
https://leemojiang.github.io/blog/sitemap.xml
```

这个主 sitemap 里面会再引用两个子 sitemap：

```txt
https://leemojiang.github.io/blog/en/sitemap.xml
https://leemojiang.github.io/blog/zh/sitemap.xml
```

需要注意的是，英文是默认语言，所以英文页面实际 URL 多数不是 `/blog/en/...`，而是：

```txt
https://leemojiang.github.io/blog/
https://leemojiang.github.io/blog/posts/formula/
```

中文页面则是：

```txt
https://leemojiang.github.io/blog/zh/
https://leemojiang.github.io/blog/zh/posts/...
```

sitemap 的作用是帮助 Google 更系统地发现页面。它不保证收录，也不保证排名，但对新站、小站、个人博客尤其有用，因为这类站点通常外链较少，Google 不一定能很快自己发现所有页面。

## robots.txt 的作用

Hugo 开启下面这个选项后，会生成 `robots.txt`：

```yaml
enableRobotsTXT: true
```

PaperMod 的默认模板会生成类似内容：

```txt
User-agent: *
Disallow:
Sitemap: https://leemojiang.github.io/blog/sitemap.xml
```

含义是：

- `User-agent: *`：对所有搜索引擎爬虫生效。
- `Disallow:` 为空：没有禁止抓取的路径。
- `Sitemap:`：告诉搜索引擎 sitemap 的位置。

不过因为 GitHub Pages 项目站点部署在 `/blog/` 子路径下，搜索引擎对 `robots.txt` 的读取逻辑可能更偏向域名根路径。因此最稳妥的方式还是在 Google Search Console 里手动提交 sitemap：

```txt
https://leemojiang.github.io/blog/sitemap.xml
```

## Search Console 里应该提交什么

先说一下 Search Console 到底是做什么的。

Google Search Console 不是访问统计工具，也不是站内搜索工具。它更像是 Google 搜索和你的网站之间的“控制台”。它主要回答这些问题：

- Google 能不能访问我的网站？
- 哪些页面已经被 Google 收录？
- 哪些页面因为 404、重定向、重复内容、`noindex` 等原因没有被收录？
- 用户在 Google 里搜索了哪些关键词时看到了我的页面？
- 我的页面在搜索结果里曝光了多少次、被点击了多少次、平均排名大概是多少？
- sitemap 有没有被 Google 正确读取？

所以 Search Console 关注的是**搜索引擎视角**：Google 如何发现、抓取、理解和展示我的页面。它不负责统计所有访问流量，也不能告诉你用户从 GitHub、微信、直接输入网址等入口访问了多少次。这些属于 GA4 这类 analytics 工具的范围。

Search Console 里有两个常用动作：

1. **提交 sitemap**：告诉 Google 这个站点有哪些页面，让它更系统地发现内容。
2. **URL Inspection / 请求编入索引**：对某一个具体 URL 请求 Google 重新抓取和评估，适合首页、刚发布的新文章、刚修复的页面。

提交 sitemap 不是“把网站加入 Google 搜索”的唯一入口。即使不提交 sitemap，Google 也可能通过外链发现你的页面。但对个人博客这种新站、小站来说，主动提交 sitemap 会更稳，也更容易在 Search Console 里看到抓取和索引状态。

如果 Search Console 添加的资源是：

```txt
https://leemojiang.github.io/blog/
```

那么 Sitemaps 页面里提交：

```txt
sitemap.xml
```

或者直接提交完整地址：

```txt
https://leemojiang.github.io/blog/sitemap.xml
```

不要提交：

```txt
blog/sitemap.xml
```

因为它可能会被拼成：

```txt
https://leemojiang.github.io/blog/blog/sitemap.xml
```

这个地址当然是 404。

如果 Search Console 添加的是根域名资源：

```txt
https://leemojiang.github.io/
```

那才应该提交：

```txt
blog/sitemap.xml
```

所以 sitemap 提交失败时，首先要检查两个东西：

1. Search Console 当前选中的 property 是哪个 URL 前缀。
2. 提交的 sitemap 路径是否和这个前缀重复或缺失。

## 站内搜索配置

PaperMod 的站内搜索不是 Google 搜索，而是本地浏览器里的 Fuse.js 搜索。它依赖 Hugo 生成 `index.json`。

首页输出要包含 JSON：

```yaml
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

然后给每种语言配置搜索参数：

```yaml
params:
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    threshold: 0.4
    distance: 1000
    minMatchCharLength: 0
    limit: 10
    keys: ["title", "permalink", "summary", "content"]
```

这个搜索只发生在博客内部，不会影响 Google 收录。它解决的是“用户进入我的博客后，如何搜索站内文章”的问题。

## Google Analytics 4 是什么

Search Console 和 GA4 容易混淆，但它们不是一回事。

Search Console 只看 Google 搜索相关的数据：

- 用户搜索了什么关键词。
- 我的页面在 Google 结果里出现了多少次。
- 有多少人从 Google 搜索结果点进来。
- 哪些页面被收录，哪些页面有索引问题。

GA4 看的是网站真实访问行为：

- 访问量。
- 访问页面。
- 访问来源。
- 国家/地区。
- 设备和浏览器。
- 实时访问。
- 页面滚动、出站点击等事件。

也就是说，GA4 不只看 Google 搜索流量。用户从 GitHub README、社交平台、直接输入网址、其他网站链接点进来，只要页面加载了 GA4 脚本，都可能被统计到。

## GA4 配置

Google Analytics 创建 Web data stream 后，会给一段代码：

```html
<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-R22BREPQN8"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-R22BREPQN8');
</script>
```

在 Hugo 里不需要手动把这段脚本粘进模板。Hugo 支持 `services.googleAnalytics.ID`：

```yaml
services:
  googleAnalytics:
    ID: "G-R22BREPQN8"
```

PaperMod 在生产环境下会调用 Hugo 的 Google Analytics 模板，构建后 HTML 里会出现：

```html
<script async src="https://www.googletagmanager.com/gtag/js?id=G-R22BREPQN8"></script>
```

以及：

```html
gtag("config","G-R22BREPQN8")
```

这说明 GA4 配置已经生效。部署到 GitHub Pages 后，访问线上页面，就可以在 GA4 的实时报告里看到访问数据。

## GitHub Pages 自己能不能看访问量

GitHub 仓库有 `Insights -> Traffic`，但它主要是仓库访问和 clone 数据，并不是完整的 GitHub Pages 博客访问统计。

如果想看博客页面的真实访问量，还是需要接入额外统计工具，例如：

- Google Analytics 4
- Cloudflare Web Analytics
- Plausible
- Umami
- GoatCounter

我目前的选择是先只用：

- Google Search Console
- Google Analytics 4

前者看搜索表现，后者看访问行为。个人博客早期没有必要一次性接入太多统计脚本，否则页面会多加载第三方 JS，统计口径也容易混乱。

## 最后检查

每次改完配置，可以先本地构建：

```powershell
hugo --minify
```

然后检查生成文件：

```powershell
rg -n "google-site-verification|G-R22BREPQN8|googletagmanager" public/index.html public/zh/index.html
```

检查 sitemap：

```powershell
Get-Content -Raw public\sitemap.xml
Get-Content -Raw public\zh\sitemap.xml
Get-Content -Raw public\en\sitemap.xml
```

部署后再检查线上地址：

```txt
https://leemojiang.github.io/blog/sitemap.xml
https://leemojiang.github.io/blog/robots.txt
https://leemojiang.github.io/blog/google26308ecd24c83a7e.html
```

如果这些地址都能访问，再回到 Google Search Console 里提交 sitemap：

```txt
https://leemojiang.github.io/blog/sitemap.xml
```

这套配置完成后，整个链路大概就是：

```txt
sitemap / robots.txt -> 帮搜索引擎发现页面
Search Console -> 看 Google 搜索表现和收录状态
GA4 -> 看真实访问行为
PaperMod 站内搜索 -> 用户在博客内部搜索文章
```

它们名字相似，但职责完全不同。理解这一点之后，Hugo 博客的搜索和统计配置就清晰很多了。
