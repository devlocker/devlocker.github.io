---
layout: post
title: Useful Vim Settings to Edit Markdown
category: tooling
---

Here's some tips for improving your Vim setup to edit Markdown files.

## Set width to 80 characters

By setting the `textwidth` option, Vim will automatically wrap lines to that
width. 

{% highlight vim %}
:set textwidth=80
{% endhighlight %}

You can also configure to automatically this setting to specific types of files

{% highlight vim %}
au BufRead,BufNewFile *.md setlocal textwidth=80
{% endhighlight %}

To reformat existing lines, select them in visual mode and press `gq`.

## Enable Spellcheck
Avoid spelling mistakes in your markdown files and git commit messages with
this handy option.

{% highlight vim %}
:setlocal spell
{% endhighlight %}

Have Vim always apply this setting:

{% highlight vim %}
autocmd BufRead,BufNewFile *.md setlocal spell
{% endhighlight %}
