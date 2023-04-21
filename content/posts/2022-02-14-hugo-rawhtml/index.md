---
title: "How to Embed Raw Html Code in Hugo"
date: 2022-02-14T14:15:50+08:00
Description: ""
Tags: [HTML, hugo, rawhtml]
Categories: [Hugo, Blog]
DisableComments: false
aliases: [rawhtml]
---

有时候受限于Markdown的语法或者功能，需要添加一些额外的组件，这时候直接内嵌进去HTML代码即可，如：

```html
<iframe 
    src="https://www.youtube.com/embed/aWzlQ2N6qqg"
    title="YouTube video player"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowfullscreen>
</iframe>
```

测试一下Hugo内嵌HTML代码的效果：

<iframe src="https://www.youtube.com/embed/aWzlQ2N6qqg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Ref: [https://anaulin.org/blog/hugo-raw-html-shortcode/](https://anaulin.org/blog/hugo-raw-html-shortcode/)
