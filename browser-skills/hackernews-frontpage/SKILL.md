---
name: hackernews-frontpage
description: Scrape the Hacker News front page (titles, points, comment counts).
host: news.ycombinator.com
trusted: true
source: human
version: 1.0.0
args: []
triggers:
  - scrape hacker news frontpage
  - scrape hn frontpage
  - get hn top stories
  - latest hacker news stories
---

# Hacker News 首页抓取器

抓取 Hacker News (`news.ycombinator.com`) 首页，并以 JSON 格式返回
热门 30 条故事。每条故事包含排名、标题、链接 URL、分数
和评论数。

## 使用方法

```
$ $B skill run hackernews-frontpage
{
  "stories": [
    { "rank": 1, "title": "...", "url": "...", "points": 412, "comments": 87 },
    ...
  ],
  "count": 30
}
```

## 工作原理

1. 通过守护进程导航到 `https://news.ycombinator.com`。
2. 读取页面 HTML。
3. 将每条故事行（HN 稳定的 `tr.athing` 结构）解析为类型化的 `Story` 记录。
4. 在 stdout 上输出单个 JSON 文档。

## 为什么这是参考技能

`hackernews-frontpage` 是最小的有意义的浏览器技能：无需认证、
HTML 稳定、输出确定、便于文件测试。每个 Phase 1 组件（SDK、作用域令牌、三层查找、spawn 生命周期）都通过 `$B skill run hackernews-frontpage` 和捆绑的 `script.test.ts` 进行验证。

当 HN 的 HTML 结构变更导致我们的选择器失效时，测试会在用户察觉之前对捕获的 fixture 失败。这就是其意义所在。
