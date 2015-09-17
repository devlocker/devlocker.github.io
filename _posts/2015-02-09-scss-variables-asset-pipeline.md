---
layout: post
title: Persisting SASS Variables in the Asset Pipeline
category: code
author: patrick
---

Here's a technique for setting up your Rails app to share SCSS variables across
all of your `.scss` partials.

The way I setup my SCSS structure for my Rails projects looks something like
this.

{% highlight css %}
app/
  assets/
    stylesheets/
      application.css
      project_name.scss
      base/
        _buttons.scss
        _forms.scss
        _typography.scss
        _variables.scss
      components/
        _dropdown.scss
        _navbar.scss
      mixins/
        _some_mixins.scss

{% endhighlight %}

I also create a file called `project_name.scss` where I import all of my files.

{% highlight css %}
@import "base/variables"
@import "mixins/some_mixins"

//Base
@import "base/buttons"
@import "base/forms"
@import "base/typography"

// Components
@import "base/dropdowns"
...

{% endhighlight %}

And then I just include that file in `application.css`

{% highlight css %}
/*
 *= require "project_name"
 */
{% endhighlight %}

Now I'd love to be able to use variables or mixins declared in
`base/_variables.scss` in all of my other files. But going into each file 
and manually importing seems tedious and repetitive. But if I try to
start my Rails app, I will quickly run into undefined variable name errors.


Luckily, there's an easy fix. Simply rename `application.css` to
`application.css.scss` and everything will work just fine.
