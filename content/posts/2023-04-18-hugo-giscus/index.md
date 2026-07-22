---
title: "How to Enable Giscus Comments System in Hugo"
date: 2023-04-18T18:11:48+08:00
Description: ""
Tags: [hugo, theme, github, comments, giscus]
Categories: [Hugo, Blog, GitHub]
DisableComments: false
toc: true
---

## Introduction to Giscus

[Giscus](https://giscus.app/) is an open-source comment system powered by GitHub Discussions, and it works really well!

- [Open source](https://github.com/giscus/giscus).
- No tracking, no ads, always free.
- No database needed. All data is stored in GitHub Discussions.
- Supports [custom themes](https://github.com/giscus/giscus/blob/main/ADVANCED-USAGE.md#data-theme)!
- Supports [multiple languages](https://github.com/giscus/giscus/blob/main/CONTRIBUTING.md#adding-localizations).
- [Extensively configurable](https://github.com/giscus/giscus/blob/main/ADVANCED-USAGE.md).
- Automatically fetches new comments and edits from GitHub.
- [Can be self-hosted](https://github.com/giscus/giscus/blob/main/SELF-HOSTING.md)!

However, not every Hugo theme supports this comment system. In fact, most of them don't. 🫠

So, in this post, I'll use a somewhat hacky approach (without modifying the theme's source code) to add Giscus support to a theme that doesn't support it out of the box.

> Note: this tutorial uses the [Anatole](https://github.com/lxndrblz/anatole) theme I'm currently using as an example. Other themes will be slightly different, but the basic idea is the same.

## How the Hack Works

First, most themes already support `Disqus` or another comment system. They usually decide whether to enable comments with logic like this:

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

As we can see, the theme enables the corresponding comment system by checking whether its configuration exists in the config file.

That makes things much easier: we can use directories such as `layouts/partials` from [Hugo Templates](https://gohugo.io/templates/base/) to override the original comments code without modifying the theme's source code.

## Add the Giscus HTML Template

Here, we'll replace the `utterances` comment system with `giscus` as an example. Simply create an `utterances.html` file under the local `layouts/partials/comments` directory, and it will override the corresponding source file.

Add the following code to the file:

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

## Update the Config File

To enable Giscus support, we still need to configure our hacked comment system in the config file. Think of it as an activation switch:

```yaml
utterances:
    repo: anything here
```

And if nothing unexpected happens, that's it!

## Switch Themes Automatically

Giscus is now configured, but if the theme already supports `Light/Dark` modes, you'll notice that the Giscus theme doesn't change along with the site's color scheme. ~~This is unacceptable.~~

Luckily, we can fix this with some JavaScript. For example, replacing the original code with the following makes Giscus adapt to the current theme when the page loads.

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

But then there's another problem: when we switch the site theme manually, the Giscus theme still doesn't update with it.

So, we need to listen for clicks on the theme switcher and update the Giscus theme accordingly. Here's the final code:

```html
<script>
    const getStoredTheme = () => localStorage.getItem("theme") === "dark" ? "dark" : "light";

    const setGiscusTheme = () => {
        const sendMessage = (message) => {
            const iframe = document.querySelector('iframe.giscus-frame');
            if (iframe) {
                iframe.contentWindow.postMessage({ giscus: message }, 'https://giscus.app');
            }
        }
        sendMessage({ setConfig: { theme: getStoredTheme() } })
    }

    document.addEventListener("DOMContentLoaded", () => {
        const giscusAttributes = {
            "src": "https://giscus.app/client.js",
            "data-repo": "[ENTER REPO HERE]",
            "data-repo-id": "[ENTER REPO ID HERE]",
            "data-category": "[ENTER CATEGORY NAME HERE]",
            "data-category-id": "[ENTER CATEGORY ID HERE]",
            "data-mapping": "pathname",
            "data-strict": "0",
            "data-reactions-enabled": "1",
            "data-emit-metadata": "0",
            "data-input-position": "bottom",
            "data-theme": getStoredTheme(),
            "data-lang": "en",
            "data-loading": "lazy",
            "crossorigin": "anonymous",
            "async": "",
        };

        // Dynamically create script tag.
        const giscusScript = document.createElement("script");
        Object.entries(giscusAttributes).forEach(
            ([key, value]) => giscusScript.setAttribute(key, value));
        document.getElementById("comment").appendChild(giscusScript);

        // Update giscus theme when the theme switcher is clicked.
        const themeSwitcher = document.querySelector(".themeswitch");
        if (themeSwitcher) {
            themeSwitcher.addEventListener("click", setGiscusTheme);
        }
    });
</script>
```

Done and dusted! 🎉

Ref: [https://github.com/giscus/giscus/issues/336](https://github.com/giscus/giscus/issues/336)
