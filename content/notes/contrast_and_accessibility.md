+++
title = "Blog Contrast and Accessibility"
date = 2026-03-11
description = "A quick note on visual accessibility for a blog and how to add support for higher contrast."
+++

You might notice that this blog _isn't_ black text on a white background! It's gray on grey. Or grey on gray, I'm not sure.

I find the dark gray on light gray more comfortable and less harsh than the default black text on a white background. However, it's important to consider the accessibility of the site for other people.

# Contrast Ratios

The [Web Content Accessibility Guidelines](https://www.w3.org/WAI/WCAG22/quickref/?versions=2.0&showtechniques=143#qr-visual-audio-contrast-contrast) (WCAG) give minimal **contrast ratios** for text on a background. Contrast ratios measure the difference between the brighter and darker color. 1:1 is the same and 21:1 is the most different.

You can check the contrast ratio of a foreground and background color using the [WebAIM contrast checker](https://webaim.org/resources/contrastchecker/). My contrast is 8.6:1, which is pretty good and meets the WCAG standards.

# A Bit More Contrast

Fortunately, people can tell their browser that they prefer contrast, and you can add [specific high-contrast](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-contrast) CSS. If the user configured it so they want _more_ contrast, their browser will look for the `@media (prefers-contrast: more)` block.

My CSS looks something like:

```css
html {
  color: #333;
  background-color: #fbfbfb;
}
@media (prefers-contrast: more) {
    html {
        color: #000;
        background-color: #ffffff;
    }
}
```
