---
title: "How to Embed Raw Html Code in Hugo"
date: 2022-02-14T14:15:50+08:00
Description: ""
Tags: [HTML, hugo, rawhtml]
Categories: [Hugo, Blog]
DisableComments: false
aliases: [rawhtml]
---

Sometimes Markdown's syntax and features are a bit limiting, and we need to add some extra components. In that case, we can simply embed the HTML directly, like this:

```html
<iframe 
    src="https://www.youtube.com/embed/aWzlQ2N6qqg"
    title="YouTube video player"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowfullscreen>
</iframe>
```

Let's test how embedded HTML works in Hugo:

<iframe src="https://www.youtube.com/embed/aWzlQ2N6qqg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Ref: [https://anaulin.org/blog/hugo-raw-html-shortcode/](https://anaulin.org/blog/hugo-raw-html-shortcode/)
