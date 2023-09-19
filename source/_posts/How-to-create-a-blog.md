---
title: How to create a blog
date: 2023-09-19 16:02:57
tags:
---



# 整体结构

（参考链接：https://blog.cuijiacai.com/blog-building/）

托管仓库：github + git

博客框架：hexo + nodejs

建站：Netlify

Netlify是一个国外的免费的提供静态网站部署服务的平台，能够将托管 GitHub，GitLab 等上的静态网站部署上线。至于我们为什么不使用`github`自带的`gitpage`，原因很简单，访问速度慢。此外，Netlify还有很多别的功能支持，这里不作剧透，可以自行探索。

PS：为了后续`netlify`建站方便，在`package.json`里面添加一个命令：

```json
{
    // ......
    "scripts": {
        "build": "hexo generate",
        "clean": "hexo clean",
        "deploy": "hexo deploy",
        "server": "hexo server",
        "netlify": "npm run clean && npm run build" // 这一行为新加
    },
    // ......
}
```







