---
layout: post
title: "Broken application logging with a forking application server"
date: 2020-07-12 13:00:00 -0700
categories: ruby
---
At work I recently wrote a small [Sinatra](http://sinatrarb.com/) application, which runs on [Passenger](https://www.phusionpassenger.com/) in the staging environment I was deploying to. Everything went smoothly with the first deploy, but I was having an issue where my log files were being created, but no logs were being written to them. After some investigation I figured out the source of my problem - Passenger's forking processes.

## What is Passenger?

Passenger is a web application server. In a new Rails 5 application, the default application server is [Puma](https://github.com/puma/puma). You might also be familiar with [Unicorn](https://yhbt.net/unicorn/). There are pros and cons to all of these options - [this Scout APM](https://scoutapm.com/blog/which-ruby-app-server-is-right-for-you) article gives a great run-down on the differences between these three options.

Passenger (and web application servers) in general are pieces of software which can allow a traditional webserver (like Apache or Nginx) to serve content from an application, like one running on Rails.

## The forking process "problem"

Some web application servers like Passenger and Puma both perform what is called "process forking". This is a technique that you can read more about [in this help article by Passenger](https://www.phusionpassenger.com/library/indepth/ruby/spawn_methods/). The gist of it is that a web application server with a moderate amount of traffic will inevitably find itself needing to handle multiple HTTP requests, concurrently. One way to do this is to spawn multiple "worker" processes than can each handle their own HTTP request.

Passenger uses what they call "smart" spawning for Ruby applications - this decreases memory overhead, and increases performance. You can read a lot more about this in the link above. There are some caveats to this, however. Part of the act of `fork`ing a system process and creating a new child process means that threads disappear after a `fork` call. So, if your application, or any of the gems it uses, relies on threads to perform work, those threads will no longer be running after the process is forked.

In my case, this is the source of the problem with logging. We use [semantic_logger](https://github.com/rocketjob/semantic_logger), and by extension, [rails_semantic_logger](https://github.com/rocketjob/rails_semantic_logger) in Rails applications. SemanticLogger runs their logger logic in "appenders" which each run in their own threads, created when an application is initialized. Because this thread is dropped when the main process forks off a new child worker, the log appender thread is lost, preventing logs from being written.

## The solution

This problem is sort of already solved, but not in a way that was immediately obvious. Because this is an extremely common problem with Rails apps using rails_semantic_logger (and Puma or Passenger), rails_semantic_logger will actually automatically re-initialize logging threads after a fork occurs. It supports this for Passenger, Resque, and Spring - all software which is affected by this problem.

The catch in my case is that I wasn't using rails_semantic_logger, because this small application was using Sinatra. The solution then is pretty simple - somewhere in the app initialization code, I added this code:

```ruby
if defined?(PhusionPassenger)
  PhusionPassenger.on_event(:starting_worker_process) do |forked|
    SemanticLogger.reopen if forked
  end
end
```

This code checks if we have Passenger avaiable (we might not because we don't run Passenger in development), and if so, calls `SemanticLogger.reopen` to reopen the logger threads whenever Passenger starts a new forked process. There we go - no more issues writing to log files.

Happy hacking!
