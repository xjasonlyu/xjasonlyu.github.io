---
title: "How to Enable Giscus Comments System in Hugo"
date: 2023-04-18T18:11:48+08:00
Description: ""
Tags: [hugo, theme, github, comments, giscus]
Categories: [Hugo, Blog, GitHub]
DisableComments: false
toc: true
---

## Giscus介绍

[Giscus](https://giscus.app/)是一个基于GitHub Discussions的开源评论系统，非常好用！

- [Open source](https://github.com/giscus/giscus). 🌏
- No tracking, no ads, always free. 📡 🚫
- No database needed. All data is stored in GitHub Discussions. :octocat:
- Supports [custom themes](https://github.com/giscus/giscus/blob/main/ADVANCED-USAGE.md#data-theme)! 🌗
- Supports [multiple languages](https://github.com/giscus/giscus/blob/main/CONTRIBUTING.md#adding-localizations). 🌐
- [Extensively configurable](https://github.com/giscus/giscus/blob/main/ADVANCED-USAGE.md). 🔧
- Automatically fetches new comments and edits from GitHub. 🔃
- [Can be self-hosted](https://github.com/giscus/giscus/blob/main/SELF-HOSTING.md)! 🤳

然而，不是所有的Hugo主题都支持这个评论系统，或者说大部分都不支持。🫠

所以，本篇使用了一种比较hack的方式（即不修改主题源代码），为原先不支持giscus的主题开启giscus评论系统的支持。

> 注意，本教程以目前所使用的[Anatole](https://github.com/lxndrblz/anatole)主题为例，其他主题会略有不同，但原理是相似的。

## Hack原理解释

首先，大部分主题应该是有`Disqus`或者其他的评论支持的，然后它们判断是否开启评论区一般是这么实现的：

```html
{{- if .Site.DisqusShortname -}}
<div id="comment">
    <h2>{{ i18n "comments" }}</h2>
    {{ template "_internal/disqus.html" . }}
</div>
{{- end -}}
{{- if .Site.Params.utterances.repo -}}
<div id="comment">
    <h2>{{ i18n "comments" }}</h2>
    {{ partial "comments/utterances.html" . }}
</div>
{{- end -}}
```

可以看到，他们通过判断config文件中是否存在相关评论系统的配置来开启对应的Comments System。

所以这就比较好办了，因为我们可以通过[Hugo Template](https://gohugo.io/templates/base/)的`layouts/partials`这类目录，覆写掉原先的comments代码，而不用去修改主题源代码。

## 添加giscus的HTML模版

这里以修改`utterances`为`giscus`评论系统为例，只需要在本地`layouts/partials/comments`目录下创建`utterances.html`文件，即可替换掉相关源码内容。

文件里面可以添加如下代码：

```html
<script 
    src="https://giscus.app/client.js"
    data-repo="[ENTER REPO HERE]"
    data-repo-id="[ENTER REPO ID HERE]"
    data-category="[ENTER CATEGORY NAME HERE]"
    data-category-id="[ENTER CATEGORY ID HERE]"
    data-mapping="pathname"
    data-strict="0"
    data-reactions-enabled="1"
    data-emit-metadata="0"
    data-input-position="bottom"
    data-theme="preferred_color_scheme"
    data-lang="en"
    crossorigin="anonymous"
    async
></script>
```

## 修改配置文件

为了开启Giscus支持，还是需要在配置文件中配置Hack的评论系统的，相当于一个激活开关：

```yaml
# Hack to use giscus
utterances:
    repo: anything here
```

然后不出意外的话，就OK了！

## 自动更换主题

现在giscus是配置好了，但是如果主题原先是支持`Light/Dark`这种模式的话，会发现giscus的主题不会随着网站主题的颜色变化而变化，~~这是不能忍的~~。

但是好在我们可以添加JS脚本来解决这个问题，例如将原有的代码更换为以下代码时，可以在访问页面时自动适配当前主题。

```html
<script>
    let giscusTheme = localStorage.getItem("theme");
    let giscusAttributes = {
        "src": "https://giscus.app/client.js",
        "data-repo": "[ENTER REPO HERE]",
        "data-repo-id": "[ENTER REPO ID HERE]",
        "data-category": "[ENTER CATEGORY NAME HERE]",
        "data-category-id": "[ENTER CATEGORY ID HERE]",
        "data-mapping": "pathname",
        "data-reactions-enabled": "1",
        "data-emit-metadata": "0",
        "data-theme": giscusTheme,
        "data-lang": "en",
        "crossorigin": "anonymous",
        "async": "",
    };

    let giscusScript = document.createElement("script");
    Object.entries(giscusAttributes).forEach(([key, value]) => giscusScript.setAttribute(key, value));
    document.getElementById("comment").appendChild(giscusScript);
</script>
```

但是又可以发现，当我们手动切换主题时，giscus的主题是不会跟着更新的。

> TBC

Ref: [https://github.com/giscus/giscus/issues/336](https://github.com/giscus/giscus/issues/336)
