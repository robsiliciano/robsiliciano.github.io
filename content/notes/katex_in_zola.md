+++
title = "Enabling KaTeX in Zola"
date = 2026-03-03
description = "It's hard to write notes on statistical analysis and data reconfiguration without well-rendered math! Here's how I set up LaTeX web rendering through KaTeX, including copy support and LaTeX environments."
[extra]
latex = true
+++

Hey, check this out:

\\[ \Phi(x) = \frac{1}{\sqrt{2 \pi}} \int_{-\infty}^x e^{-\frac{t^2}{2}} \\]

I just got LaTeX rendering on here! I guess that's a _standard normal_ blog feature?

# Adding KaTeX

I'm using [KaTeX](https://katex.org), a Javascript library for rendering LaTeX math formulas to websites. Specifically, I'm using client-side rendering, where I send the formula `\\(1 + 1\\)` to your browser, and it runs the KaTeX code to create \\(1 + 1\\). You can also use the `$$` syntax, but I'm trying to move to the directional `\[` or `\(` syntax.

Turns out, it's pretty easy to add KaTeX support to a website. You can just add the [Javascript file and CSS](https://katex.org/docs/browser#starter-template) to your HTML, referencing the CDN.

For a Markdown blog, you'd also want [autorender support](https://katex.org/docs/autorender#usage), which lets you write the typical LaTeX syntax in Markdown, i.e., `$`, `$$`, `\(`, and `\[`.

# Being Stingy with the Scripts

If you have a dialup internet connection, my pages might have gotten really hard to load all of a sudden! I've gone from a relatively small static HTML page to including thousands of lines of CSS and Javascript. While necessary for client-side LaTeX rendering, many of my notes don't involve any equations!

Fortunately for all those times you have a poor connection, I can only include KaTeX on the pages that actually need it. Overall, it's just two steps:

1. Wrap the KaTeX tags in a `{% if page.extra.katex %}` – `{% endif %}` block.
2. Add `katex = true` to the `[extra]` block in the page's front matter (the "+++" block).

```md
+++
title = "Your title"

# The rest of your front matter

[extra]
katex = true
+++
```

Now go check the source for this page and any older one. You'll see KaTeX included here, but not used on any of those earlier notes that didn't need it.

# Font Sizes

If you try this yourself, you'll notice that inline formulas, like \\(1 + 1\\), look a bit too big. Turns out, that sizing is intentional, and KaTeX specifically [wants to use a font 21% larger](https://katex.org/docs/font#font-size-and-lengths) than regular text. While the sizing looks a bit silly for something simple like \\(1 + 1\\), it helps readability when there's a lot of tiny superscripts or subscripts: \\( X_i^2 \\).

It's a matter of taste, and I prefer to have the formula sizes match my text sizes. You can change the sizing in your CSS.
```css
.katex { font-size: 1rem; }
```
Just remember to have this setting _after_ you include the KaTeX CSS.

# Copy-Paste Support

Try copying this formula:

\\[ \Phi(x) = \frac{1}{\sqrt{2 \pi}} \int_{-\infty}^x e^{-\frac{t^2}{2}} \\]

You should get back exactly the LaTeX that generates it:
```
$ \Phi(x) = \frac{1}{\sqrt{2 \pi}} \int_{-\infty}^x e^{-\frac{t^2}{2}} $
```

You won't get this on base KaTeX however; copying the formula would give a collection of symbols that aren't really useful:
```
Φ(x)= 
2π
​	
 
1
​	
 ∫ 
−∞
x
​	
 e 
− 
2
t 
2
 
​	
 
 ```

KaTeX provides the [copy-tex](https://github.com/KaTeX/KaTeX/tree/main/contrib/copy-tex) extension, which substitutes in the LaTeX whenever you try to copy KaTeX in the browser. To enable this extension, you just need to include one more script along with the base KaTeX script and CSS.

# Blockier LaTeX

So this is all great for formulas, but what if you need some `\begin{}\end{}` environment, like for an `equation`? You wouldn't use the equation delimiters in normal LaTeX, so the KaTeX we have so far won't help either.

KaTeX supports these blocks through `<script type="math/tex"></script>`. You'll need to add the [`math/tex` extension](https://github.com/KaTeX/KaTeX/tree/main/contrib/mathtex-script-type), just like you added the autorender and copy extensions. This code will search for the `math/tex` scripts and render them.

At this point, we need to use [Zola shortcodes](https://www.getzola.org/documentation/content/shortcodes/), which are essentially custom functions or blocks that expand into HTML.

To create your KaTeX shortcode, create a `katex.html` in the `templates/shortcodes` directory.

```html,name=templates/shortcodes/katex.html
<script type="math/tex mode=display">
    {{ body | safe }}
</script>
```

Since we want "display" (not inline) rendering, we need to have `mode=display` in the type somewhere. The [script](https://github.com/KaTeX/KaTeX/blob/main/contrib/mathtex-script-type/mathtex-script-type.js) is pretty permissive, so you could do all sorts of things like `mode = display, math/tex` and still trigger the autorender.

Anyways, this shortcode has no arguments but uses [body](https://www.getzola.org/documentation/content/shortcodes/#shortcodes-with-body) for the contents between the tags.

You can call it with
```
{%/* katex() */%}
\begin{align}
  y &= (x + 1)^2 \\
    &= x^2 + 2x + 1
\end{align}
{%/* end */%}
```

This block will nicely render the multi-line equation!

{% katex() %}
\begin{align}
  y &= (x + 1)^2 \\
    &= x^2 + 2x + 1
\end{align}
{% end %}


## The Alternative: All Shortcodes

You can also use shortcodes as a replacement for the delimiters. The current KaTeX setup reads through my text, finds the delimiters, and replaces the LaTeX code with a rendered formula. I prefer this way, since I can write nice Markdown for my notes.

The other way to use KaTeX would be to explicitly call it with a shortcode block each time. I find this a lot clunkier than using `\\(1 + 1\\)`, but it's more general and customizable if you need something different. (If you want to go with only shortcodes, you'll want to modify the one above to allow inline rendering too.)

# LaTeX Coverage

One last thing: KaTeX might not support everything you could do with LaTeX. Check out their [supported functions](https://katex.org/docs/supported.html) if you're running into issues.
