---
layout: post
title: Using Sidekiq for sending emails asynchronously
---

[Sidekiq](https://github.com/mperham/sidekiq) is an awesome gem written by [Mike Perham](http://www.mikeperham.com/) used for processing background jobs in Ruby. Sidekiq uses threads to process multiple jobs within the same process at the same time. It is highly compatible with [Resque](https://github.com/resque/resque) as it shares the exact same message format as Resque.

I have been playing around with Sidekiq for a while and I almost always use the it for sending emails from my application asynchronously. Most of the emails sent from an application are notifications about a certain events and they should always be sent asynchronously. Sending emails asynchronously will reduce your application response time as the main application thread is not waiting on sending the email.

# Sidekiq Dependencies

Sidekiq uses [Redis](http://redis.io/) for data storage and holds all the job data. You can install redis using [Homebrew](http://brew.sh/) on a Mac.

{% highlight bash %}
$ brew install redis
 # To have launchd start redis at login:
$ ln -sfv /usr/local/opt/redis/*.plist ~/Library/LaunchAgents
 # Then to load redis now:
$ launchctl load ~/Library/LaunchAgents/homebrew.mxcl.redis.plist
 # Or, if you don't want/need launchctl, you can just run:
$ redis-server /usr/local/etc/redis.conf
{% endhighlight %}

To install redis on an ubuntu machine follow the instructions [here](http://redis.io/download) or [here](https://www.digitalocean.com/community/articles/how-to-install-and-use-redis).

# Sidekiq Installation

Just add the gem to your Gemfile. The sinatra gem is required to access the web console for monitoring the state of the installation.

{% highlight ruby %}
gem 'sinatra', '>= 1.3.0', :require => nil
gem 'sidekiq'
{% endhighlight %}

Start sidekiq from the root directory of your Rails app.

{% highlight bash %}
$ bundle exec sidekiq
{% endhighlight %}

Make sure you're redis server is up and running as well.

# Asynchronus emails using sidekiq

The easiest way to send the email asynchronously is by using the [Delayed Extension](https://github.com/mperham/sidekiq/wiki/Delayed-Extensions). You can specify a delay interval as well. The email will be sent using a separate thread without blocking the main execution thread. 

{% highlight ruby %}
def invite_person
	UserMailer.delay.invite_person(@user.id)
 	# UserMailer.delay_for(5.minutes).invite_person(@user.id)
end
{% endhighlight %}

Another way is to create your own worker process. Add a <code>app/workers</code> directory within your Rails application and create a new worker there.

{% highlight ruby %}
class EmailSender
  include Sidekiq::Worker
  def perform(id)
	UserMailer.invite_person(id).deliver
  end
end
{% endhighlight %}

Finally replace your call to the <code>ActionMailer</code> with the newly created <code>EmailSender</code>.

{% highlight ruby %}
def invite_person
	EmailSender.perform_async(@user.id)
end
{% endhighlight %}

Its not a good idea to save state in Sidekiq. We should only pass simple identifiers to it and load the actual objects at the time of processing. This would prevent us having stale objects. 

#Deployment

The deployment setup for also pretty straight forward. Just add the following to you Capistrano <code>deploy.rb</code> file.

{% highlight ruby %}
require "sidekiq/capistrano"
{% endhighlight %}

This should be sufficient for setting up and starting Sidekiq on the each application server. You should also read the [rake file](https://github.com/mperham/sidekiq/blob/master/lib/sidekiq/tasks/sidekiq.rake) which makes this magic happen.

For more advanced settings, please refer to the document [here](https://github.com/mperham/sidekiq/wiki/Advanced-Options)

Personally, I am super happy with the performance and feature set of Sidekiq. It has everything I need to process background jobs right now. There are some other great libraries like [Resque](https://github.com/resque/resque), [Stalker](https://github.com/han/stalker), [Delayed Job](https://github.com/collectiveidea/delayed_job) for processing background jobs. Check them out as well and pick the one that works for you.














