---
layout: post
title: Remote Selenium WebDriver servers with Rails, Capybara, and RSpec
categories: rails,ruby
date: 2020-11-05 16:02:22 -08:00
---
At work, I'm working on getting our Rails projects deployed on new Kubernetes infrastructure, and we're taking the opportunity to fix some problems with our current CI pipeline - the biggest of which is a lack of tests. The first app we're deploying is a greenfield (yet to be used in production) Rails app which uses RSpec and includes Rails 5.1/6-style "system specs" which use Capybara to drive a web browser.

Getting javascript-enabled system specs working when you run the tests locally isn't too bad - mostly just install a few gems, configure them per their respective READMEs, and off you go. This was the case with this new Rails app, but seeing as we wanted tests to be fully supported in our new CI pipeline, we needed to figure out how to deal with system specs, and by extension, Selenium.

# Selenium WebDriver server setup

Because we're using Docker to set up this testing CI environment, we can use [docker-selenium](https://github.com/SeleniumHQ/docker-selenium), the official Selenium Docker image repository. Because my local machine isn't the CI server, I'll need to set up a Selenium server straight from the Docker image:

```bash
$> docker run --name=selenium -p '4444:4444/tcp' -p '5900:5900/tcp' 'selenium/standalone-chrome-debug'
```