---
title: "Collection of Hugo Shortcodes"
date: 2023-04-17T14:24:47+08:00
Tags: [hugo, theme, github, gist, shortcodes]
Categories: [Hugo, Blog, GitHub]
DisableComments: false
toc: true
---

Here are a few handy tricks for Hugo, along with some practical improvements.

## Gist

Hugo can embed Gist code blocks directly using its built-in `shortcodes`.

### Display All Gist Files

```text
{{</* gist jmooring 50a7482715eac222e230d1e64dd9a89b */>}}
```

{{< gist jmooring 50a7482715eac222e230d1e64dd9a89b >}}

### Display a Specific File

```text
{{</* gist jmooring 50a7482715eac222e230d1e64dd9a89b 1-template.html */>}}
```

{{< gist jmooring 50a7482715eac222e230d1e64dd9a89b 1-template.html >}}

Ref: [Hugo Shortcodes](https://gohugo.io/content-management/shortcodes/)

## GitHub

Hugo itself doesn't support displaying code files from GitHub directly, so we'll need to add a custom shortcode.

### Add the Shortcode

Create a `github.html` file under `layouts/shortcodes`, then paste in the following code:

```text
{{ $dataJ := getJSON "https://api.github.com/repos/"  (.Get "repo")  "/contents/"  (.Get "file")  }}
{{ $con := base64Decode $dataJ.content }}

{{ highlight $con (.Get "lang") (.Get "options") }}
```

### Display GitHub Code

Simply call the shortcode like this:

```text
{{</* github repo="username/repo-name" file="/path/to/file" lang="language" options="highlight-options" */>}}
```

Ref: [https://github.com/haideralipunjabi/hugo-shortcodes/tree/master/github](https://github.com/haideralipunjabi/hugo-shortcodes/tree/master/github)
