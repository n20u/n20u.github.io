---
layout: post
title: 使用 Jekyll 创建 Github Pages
description: 创建 Github Pages 需要掌握的相关知识和技巧方法
categories: 指南
---
以前一直想制作个自己的博客网站，了解到 Github 的静态站点托管服务 Github Pages 后，就开始阅读相关文档，摸索着使用相关工具，利用网上的资源，终于做好了[我的 Github Pages](https://{{ site.username }}.github.io/)。

本文主要总结使用 Jekyll 创建 Github Pages 需要掌握的相关知识，这些知识对于定制化自己的 Github Pages 很有帮助。还有在创建过程中使用的一些技巧方法，这些方法能使 Github Pages 更加完善。

# 知识准备

按重要程度从高到低排列
1. HTML、CSS、JavaScript。这三者构成了网页的前端，HTML 决定网页的结构和内容，CSS 设定网页的表现形式，JavaScript 控制网页的行为。因为 Github Pages 是 Github 的静态站点托管服务，所以没有后端，也就对后端知识没有要求。

2. Markdown。Markdown 是一种轻量级标记语言，它允许人们使用易读易写的纯文本格式编写文档。Github Pages 能渲染展示 Markdown 文档，用其编写博客比较方便。

3. [Github Pages 官方文档](https://docs.github.com/cn/github/working-with-github-pages)，这个文档介绍了创建 Github Pages 的步骤，包括使用 Jekyll 创建 Github Pages 的方法，但内容并不详细，相关知识还需要阅读专门的文档。
![使用 Github Pages]({{ assets_url }}/assets/img/posts/{{page.slug}}/使用 Github Pages.png)
如果想仅做一个页面作为 Github Pages 的首页，则只需要看这篇简单的 [Github Pages 文档](https://pages.github.com/)就够了。

4. [Jekyll 文档](http://jekyllcn.com/docs/home/)，这是由[开源爱好者们](https://github.com/xcatliu/jekyllcn/graphs/contributors)共同翻译的 Jekyll 文档，包括一些内容如：搭建和运行站点、创建以及管理内容、定制站点的展现和外观的一些建议。
![使用 Jekyll 文档]({{ assets_url }}/assets/img/posts/{{page.slug}}/Jekyll 文档.png)

5. [Liquid 文档](https://liquid.bootcss.com/)。Liquid 是一门安全、面向客户的模板语言，用于构建灵活的 web 应用。Jekyll 将 Liquid 作为自身的模版语言，并且添加了许多对象（object）、标记（tag）和过滤器（filter）。
![Liquid 文档]({{ assets_url }}/assets/img/posts/{{page.slug}}/Liquid 文档.png)

6. [YAML 入门教程](https://www.runoob.com/w3cnote/yaml-intro.html)。YAML 可以简单表达清单、散列表，标量等数据形态，特别适合用来表达或编辑数据结构、各种配置文件、倾印调试内容、文件大纲（例如：许多电子邮件标题格式和YAML非常接近）。YAML 的配置文件后缀为 .yml。Jekyll 在创建网站时，会从网站根目录下的 _config.yml 文件加载配置信息。
![YAML 入门教程]({{ assets_url }}/assets/img/posts/{{page.slug}}/YAML 入门教程.png)

7. [SCSS 文档](https://www.sass.hk/docs/)。Sass 是一款强化 CSS 的辅助工具，它在 CSS 语法的基础上增加了变量 (variables)、嵌套 (nested rules)、混合 (mixins)、导入 (inline imports) 等高级功能，这些拓展令 CSS 更加强大与优雅。SCSS 是 Sass 的语法格式之一，这种格式仅在 CSS3 语法的基础上进行拓展，所有 CSS3 语法在 SCSS 中都是通用的，同时加入 Sass 的特色功能。Jekyll 提供了对 Sass 的内建支持。
![SCSS 文档]({{ assets_url }}/assets/img/posts/{{page.slug}}/SCSS 文档.png)

# 技巧方法

1. 使用 jsDelivr 加速访问 Github Pages 的静态资源。jsDelivr 是一个免费的开源 CDN，其[官方文档](https://www.jsdelivr.com/features#gh)介绍了对 Github 的支持。简单介绍如下：<br>
以我的 Github Pages 仓库为例，地址为`https://github.com/{{ site.username }}/{{ site.username }}.github.io`，里面的资源可以直接从`https://cdn.jsdelivr.net/gh/{{ site.username }}/{{ site.username }}.github.io/ + 仓库里的文件路径`来访问。比如仓库里有个 js 文件`assets/js/main.js`，访问它的 CDN 链接为：`https://cdn.jsdelivr.net/gh/{{ site.username }}/{{ site.username }}.github.io/`。

2. 在 Jekyll 中使用 SCSS。首先你需要将所有需要导入的 SCSS 文件放在`sass_dir`下，该路径默认是`<source>/_sass`，然后在主 SCSS 文件中使用`@import`语句导入 SCSS 文件，并将其放在你希望输出文件所在的目录下，如`<source>/assets/css`。Jekyll 将这些文件的输出存放在同一目录下，例如网站源目录下的`/assets/css/styles.scss`，Jekyll 会处理生成网站目标目录下的`/assets/css/styles.css`。最后在 HTML 文件中使用`<link rel="stylesheet" href="/assets/css/styles.css">`引入 CSS 文件。<br>
需要注意的是，因为最终在 HTML 文件中引入的 CSS 文件是 Jekyll 在创建网站时生成的，该文件并不在网站源目录中，所以不能使用 jsDelivr 加速访问。

3. 使用 Gitalk 制作 Github Pages 的评论功能。Gitalk 是一个基于 GitHub Issue 和 Preact 开发的评论插件，其[官方文档](https://github.com/gitalk/gitalk/blob/master/readme-cn.md)介绍了使用方法。<br>
需要注意的是，其官方文档中说用下面的 JavaScript 代码来生成 gitalk 插件：

``` javascript
var gitalk = new Gitalk({
    clientID: 'GitHub Application Client ID',
    clientSecret: 'GitHub Application Client Secret',
    repo: 'GitHub repo',
    owner: 'GitHub repo owner',
    admin: ['GitHub repo owner and collaborators, only these guys can initialize github issues'],
    id: location.pathname,      // Ensure uniqueness and length less than 50
    distractionFreeMode: false  // Facebook-like distraction free mode
})

gitalk.render('gitalk-container')
```

但不能在这段代码外面加个`<script>`标签就直接写入 HTML 文件中，否则浏览器会报`Uncaught SyntaxError: Unexpected identifier`错误，需要在每条语句后面加个分号。

``` html
<script>
    var gitalk = new Gitalk({
        clientID: 'GitHub Application Client ID',
        clientSecret: 'GitHub Application Client Secret',
        repo: 'GitHub repo',
        owner: 'GitHub repo owner',
        admin: ['GitHub repo owner and collaborators, only these guys can initialize github issues'],
        id: location.pathname,      // Ensure uniqueness and length less than 50
        distractionFreeMode: false  // Facebook-like distraction free mode
    });

    gitalk.render('gitalk-container');
</script>
```