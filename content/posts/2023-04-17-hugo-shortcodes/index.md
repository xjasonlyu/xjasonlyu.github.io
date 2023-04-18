---
title: "Collection of Hugo Shortcodes"
date: 2023-04-17T14:24:47+08:00
Tags: [hugo, theme, github, gist, shortcodes]
Categories: [Hugo, Blog, GitHub]
DisableComments: false
toc: true
---

整理了一些在 Hugo 下使用比较方便的技巧，以及一些可行的优化。

## Gist

Hugo 可以直接使用其内置的`shortcodes`来嵌入Gist代码块。

### 显示所有的gist文件

```text
{{</* gist jmooring 50a7482715eac222e230d1e64dd9a89b */>}}
```

{{< gist jmooring 50a7482715eac222e230d1e64dd9a89b >}}

### 显示指定的文件

```text
{{</* gist jmooring 50a7482715eac222e230d1e64dd9a89b 1-template.html */>}}
```

{{< gist jmooring 50a7482715eac222e230d1e64dd9a89b 1-template.html >}}

Ref: [Hugo Shortcodes](https://gohugo.io/content-management/shortcodes/)

## GitHub

Hugo本身并不支持直接显示GitHub上的代码文件，所以想要这么做的话，需要添加一个自定义的shortcode。

### 添加Shortcode代码

在`layouts/shortcodes`下创建`github.html`文件，并粘贴如下代码：

```text
{{ $dataJ := getJSON "https://api.github.com/repos/"  (.Get "repo")  "/contents/"  (.Get "file")  }}
{{ $con := base64Decode $dataJ.content }}

{{ highlight $con (.Get "lang") (.Get "options") }}
```

### 显示GitHub代码

直接调用如下shortcode即可：

```text
{{</* github repo="username/repo-name" file="/path/to/file" lang="language" options="highlight-options" */>}}
```

Ref: [https://github.com/haideralipunjabi/hugo-shortcodes/tree/master/github](https://github.com/haideralipunjabi/hugo-shortcodes/tree/master/github)
