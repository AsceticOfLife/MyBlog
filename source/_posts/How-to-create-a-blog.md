---
title: 如何创建自己的博客
date: 2023-09-19 16:02:57
tags:
- 搭建博客
- hexo主题
- 引用资源
---



（参考链接：https://blog.cuijiacai.com/blog-building/）

**托管仓库：**github + git  （博客存放地点）

**博客框架：**hexo + nodejs （编写博客以及生成静态网站）

**博客建站：**Netlify + 配置域名 （建立静态网页以及通过域名访问）

<!--more-->

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

**更换主题：**为了方便将主题添加到github上，所以将主题作为submodule
1.在themes文件夹下添加主题：(以yilia-plus为例)

```git
cd themes
git submodule add https://github.com/JoeyBling/hexo-theme-yilia-plus
git commit -m "add submodule: yilia-plus"
```

2.修改_config.yml文件：

```
theme: hexo-themes-yilia-plus
```

3.保存到github仓库：

```
git push
```

**引用资源：**
1.打开Hexo提供的资源文件夹功能，方便使用相对路径引用资源：
修改_config.yml文件，修改如下：

```
post_asset_folder: true
```

之后Hexo 将会在我们每一次通过 `hexo new <title>` 命令创建新文章时自动创建一个同名文件夹，于是我们便可以将文章所引用的相关资源放到这个同名文件夹下，然后通过相对路径引用。

2.相对路径引用方法：
通过常规的 markdown 语法和相对路径来引用图片和其它资源可能会导致它们在存档页或者主页上显示不正确。我们可以通过使用 Hexo 提供的标签插件来解决这个问题：

```
{% asset_path slug %}
{% asset_img slug [title] %}
{% asset_link slug [title] %}
```

比如说：当你打开文章资源文件夹功能后，你把一个 `example.jpg` 图片放在了你的资源文件夹中，如果通过使用相对路径的常规 markdown 语法 `![](/example.jpg)` ，它将 *不会* 出现在首页上。（但是它会在文章中按你期待的方式工作）

**！！！注意：** 如果已经开启了文章的资源文件夹功能，当使用 MarkDown 语法引用相对路径下的资源时，只需 `./资源名称`，不用在引用路径中添加同名文件夹目录层级。

正确的引用图片方式是使用下列的标签插件而不是 markdown ：

```
{% asset_img example.jpg This is an example image %}
```

通过这种方式，图片将会同时出现在文章和主页以及归档页中。





