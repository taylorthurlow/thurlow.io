---
layout: post
title: Remote Selenium WebDriver servers with Rails, Capybara, and RSpec
categories: rails,ruby
date: 2020-11-05 16:02:22 -08:00
---
At work, I'm working on getting our Rails projects deployed on new Kubernetes infrastructure, and we're taking the opportunity to fix some problems with our current CI pipeline - the biggest of which is a lack of tests. The first app we're deploying is a greenfield (yet to be used in production) Rails app which uses RSpec and includes Rails 5.1/6-style "system specs" which use Capybara to drive a web browser.

Getting JavaScript-enabled system specs working when you run the tests locally isn't too bad - mostly just install a few gems, configure them per their respective READMEs, and off you go. This was the case with this new Rails app, but seeing as we wanted tests to be fully supported in our new CI pipeline, we needed to figure out how to deal with system specs, and by extension, Selenium.

My goal with this post is to explain how to set up your specs so they run locally when you're developing locally, but on an arbitrary Selenium server when you desire that (like during CI).

# Selenium WebDriver server setup

Because we're using Docker to set up this testing CI environment, we can use [docker-selenium](https://github.com/SeleniumHQ/docker-selenium), the official Selenium Docker image repository. Because my local machine isn't the CI server, I'll need to set up a Selenium server straight from the Docker image:

```bash
$> docker run --name=selenium -p '4444:4444/tcp' -p '5900:5900/tcp' 'selenium/standalone-chrome-debug'
```

This command will download the container and run it with two ports open - `4444`  is the Selenium WebDriver server, and `5900` is a VNC server which you can use to visually inspect the browser if you want. This is why I used the `standalone-chrome-debug` selenium image instead of the `standalone-chrome` image - the non-debug version doesn't come with a VNC viewer. In production, you can use the non-debug version. You can use a VNC client to open `vnc://localhost:5900`, using the password `secret`.

# Configuring Rails

You'll need a few gems, some or all of which you may already have:

* [Capybara](https://github.com/teamcapybara/capybara), a DSL for testing frameworks used to manipulate web drivers like Selenium (`v3.33.0` at the time of writing)
* [selenium-webdriver](https://github.com/SeleniumHQ/selenium/tree/trunk/rb), the Ruby bindings for controlling Selenium WebDriver (`v3.142.7` at the time of writing)
* [rspec-rails](https://github.com/rspec/rspec-rails), my Ruby testing library of choice (`v4.0.1` at the time of writing)

If you use Minitest instead of RSpec, things should be largely the same, but you're on your own as far as getting things working. Differences between this guide and the same process for Minitest should be mostly superficial.

