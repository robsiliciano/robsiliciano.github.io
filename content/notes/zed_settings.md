+++
title = "Zed Settings"
date = "2026-01-21"
+++

# Typesetting

I'm a huge fan of Astral's `uv` and `ruff`, so I've been excited for `ty`, their new [type checker and language server](https://docs.astral.sh/ty/). It's pretty easy to [use `ty` as Zed's language server](https://docs.astral.sh/ty/editors/#zed), which comes [ready to go](https://zed.dev/docs/languages/python#configure-python-language-servers-in-zed) in Zed.

```json,name=settings.json
  "languages": {
    "Python": {
      "language_servers": ["ty"],
    },
  },
```

You can check that your language server is running by the language server icon in the lower left. Zed says the icon is a lightning bolt, but it also kind of looks like a sideways paper ticket. 

# Wrap Guides

Zed has a nicely unobtrusive wrap guide that shows a vertical line at specified character counts.

```json,name=settings.json
  "show_wrap_guides": true,
  "wrap_guides": [80],
```

I like to set mine at 80 as a warning sign, then let `ruff` enforce [Black's 88 character](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html#line-length) limit. (Ruff formatting is included with Zed.) If you wanted, you could have lines at both 80 and 88 characters by adding a second wrap guide to the list.
