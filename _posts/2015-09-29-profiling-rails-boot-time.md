---
layout: post
title: Profiling Rails Boot Time
category: code
author: patrick
---

Is your Rails app starting to feel sluggish when running common start up
commands like `server`, `console` and `rake`?  Let's take a look at debugging
rails applications that take a long time to boot.

Most of the pain is going to come from the Gemfile and anything in
`config/initializers`. Large Gemfiles and long lists of initializers add to the
time it takes for Rails to boot because Rails has to load all of those files and
run any setup code. There are a few options to diagnose which Gems and
initializers are more costly than others.

## Benchmarking Boot Time

Before we start trying to improve the boot time of our application, we need some
baselines to measure any improvements against.  Let's use the `rake environment`
task as the base for our benchmark - this just loads the Rails environment and
then exits. We can time it with:

{% highlight text %}
$ time bundle exec rake environment
bundle exec rake environment  4.07s user 1.32s system 99% cpu 5.429 total
{% endhighlight %}

We use `bundle exec` to make sure any preloaders (spring, zeus) don't get in the
way and use a preloaded environment.

As a sample, I'm using the popular open source project
[Discourse](https://github.com/discourse/discourse). For an app as big as
Discourse, a 4 second boot time is very good.


## Profiling
We'll look into a few ways to profile what's slow and what's fast. Let's have some
fun and monkey patch `require`. Open up `config/boot.rb` and paste in:

{% highlight ruby %}
require "benchmark"

def require(file_name)
  result = nil

  time = Benchmark.realtime do
    result = super
  end

  if time > 0.1
    puts "#{time} #{file_name}"
  end

  result
end
{% endhighlight %}

This wraps the call to `super` - the actual `require` method - in a call to
`Benchmark#realtime` which returns a float of the total time the code
block took to execute. We add a conditional to print the time and file name if
the file takes over 100ms to load. To get the output sorted simply pipe to
`sort`.


{% highlight text %}
$ time bundle exec rake environment | sort

0.10626332799438387 arel
0.10734090598998591 nokogiri
0.10807372402632609 active_support/time
0.1093888989998959  openid/consumer/idres.rb
0.1137797289993614  active_support/core_ext/hash/conversions
0.11505815701093525 mime/types
0.12525871800608002 pry
0.12973016197793186 byebug/commands
0.13043017499148846 pry-rails
0.1312439759785775  loofah
0.13336818301468156 openid/consumer
0.13918157300213352 active_support/core_ext/object/conversions
0.13963568801409565 rails-html-sanitizer
0.14628734899451956 openid
0.14947060000849888 action_view/helpers/form_tag_helper
0.15580355300335214 rack/openid
0.16811588499695063 byebug/core
0.17389453400392085 omniauth/strategies/open_id
0.18345668498659506 active_support/core_ext/object
0.1887360849941615  rails/configuration
0.1958723379939329  rails/railtie
0.2032998200156726  rails/engine
0.21330312499776483 sprockets/rails/helper
0.21499990700976923 active_record
0.27881946400157176 sprockets/railtie
0.29778293799608946 rails/application
0.31094263499835506 active_record/railtie
0.43013601601705886 rails
1.0511020889971405  rails/all
1.5489416829950642 discourse/config/environment.rb
bundle exec rake environment  4.15s user 1.36s system 99% cpu 5.541 total
{% endhighlight %}


Another great option is the [Bumbler Gem](https://github.com/nevir/Bumbler). It
profiles your application and shows you how long each gem or initializer took to
load. Again, this is for the Discourse app.


{% highlight text %}
$ bumbler
Slow requires:
    101.97  omniauth-facebook
    108.51  sidekiq-statistic
    111.31  better_errors
    123.78  image_optim
    125.08  barber
    127.52  onebox
    141.45  rake
    158.47  rspec-given
    160.70  ember-rails
    182.73  nokogiri
    204.75  sass
    285.01  byebug
    286.85  seed-fu
    288.10  omniauth-openid
    317.03  mail
    623.52  rails

$ bumbler --initializers
Slow requires:
    117.19  active_record.initialize_database
    118.97  ./config/initializers/04-message_bus.rb
    156.36  :finisher_hook
    329.09  :set_routes_reloader_hook
    809.04  ./config/initializers/06-ensure_login_hint.rb
   1143.55  :load_config_initializers
{% endhighlight %}

# Remedies
Now that we know what's slow, there are a few options for dealing with slowness.

## Update Gems

The easiest way, believe it or not, is to simply update to the latest version of
a slow loading gem. I have run into several instances where I was running an
older version that had a dependency on an old version of a slow loading gem
(like nokogiri). In the latest stable version that dependency was removed, and
I was able to shave off over a second of boot time in several different
applications by doing a simple `bundle update gem_name`.

## require: false

By default, any Gem listed in the `Gemfile` gets loaded when Rails starts. This
means boot time usually grows with Gemfile size. However, it is rarely the case
that you need a particular gem to be loaded and available everywhere.

If you only need classes / functions from a gem in a handful of files, it is much
more performant to add `require: false` to the Gem declaration. Then, in any
file(s) you need access to that gem, add a `require gem_name` to the top of the
file.

If you look at the [Discourse
Gemfile](https://github.com/discourse/discourse/blob/master/Gemfile) you will
see that many of the Gem declarations tag on a `require: false`. That is
what helps Discource to boot so quickly despite having 93 gem dependencies.

## Rails inheritance in lib/

This is a potential issue I have seen in a few places where a class was
inheriting from a Rails core class such as `ActiveRecord::Base` or
`ApplicationController` and then being referenced in an initializer. It appeared
that it was requiring Rails all over again (around a 500ms hit). Moving it into
a Gem or into the `app` folder resolved that issue.

## Faster routes...
This is one that I unfortunately have no solution for. A large routes file can
have a serious impact on boot performance. In one app I worked on I've seen it
take over 3 seconds to load the routes. In the bumbler output it is the
`:set_routes_reloader_hook` initializer.
