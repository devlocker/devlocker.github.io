---
layout: post
title: Tailing Logs in Capistrano
category: operations
---

Quite often you might find yourself wanting to tail the logs for your remote
servers. SSHing into the remote machine and manually tailing the logs is a bit
cumbersome, so I'm going to show you how to do it with Capistrano (v3).

It's actually quite simple. Create a file called `lib/capistrano/tasks/tail.cap`
and add the following code.

{% highlight ruby %}
namespace :tail do
  desc "tail rails logs" 
  task :rails do
    on roles(:app) do
      execute "tail -f #{fetch(:shared_path)}/log/#{fetch(:rails_env)}.log"
    end
  end
end
{% endhighlight %}

The task can be invoked by running:

{% highlight ruby %}
cap production tail:rails
{% endhighlight %}
