---
title: 如何创建自己的博客
date: 2023-09-19 16:02:57
tags:
---



（参考链接：https://blog.cuijiacai.com/blog-building/）

**托管仓库：**github + git  （博客存放地点）

**博客框架：**hexo + nodejs （编写博客以及生成静态网站）

**博客建站：**Netlify + 配置域名 （建立静态网页以及通过域名访问）

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



**CDN加速：**ClouldFlare加速

**配置https：**netlify配置https访问





