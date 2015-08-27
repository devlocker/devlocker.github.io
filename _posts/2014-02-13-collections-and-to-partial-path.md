---
layout: post
title: Collections and to_partial_path
category: code
---

Rendering collections of Active Record models is really quite clean. Calling:

{% highlight ruby %}
<%= render @article.posts %>
{% endhighlight %}

Effectively calls:
{% highlight ruby %}
<%= render partial: "posts/post", locals: { post: post } %>
{% endhighlight %}

Internally, the `render` method calls `to_partial_path` on each item of the
collection to figure out which partial to render (in this case, posts/post).
ActiveRecord models inherit a method which returns a partial name based on the
name of the model. However, it is possible to overwrite said behavior. Plain
Ruby objects are able to implement the same behaviour.

{% highlight ruby %}
class Article < ActiveRecord::Base
  def to_partial_path
    "articles/summary"
  end
end

class Spaceship
  def name
    "Serenity"
  end

  def to_partial_path
    "transports/firefly"
  end
end

class Train
  def to_partial_path
    "transports/train"
  end
end

# In the view
<%= render @transports %>
{% endhighlight %}

Lets use this fact to render out an event feed. Each event has a `type` column
(comment, star, upload, etc.). Each event in the feed is displayed differently,
and has a corresponding partial in the `events` folder. Instead of doing a
switch on type in the view or model, we can take advantage of `to_partial_path`
to dynamically figure out the correct partial to render.

{% highlight ruby %}
class Event < ActiveRecord::Base
  def to_partial_path
    "events/#{type}"
  end
end

....
<%= render @events %>
{% endhighlight %}
