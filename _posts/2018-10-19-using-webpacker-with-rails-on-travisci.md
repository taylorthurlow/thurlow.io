---
layout: post
title: Using Webpacker with Rails on TravisCI
categories: ruby
date: 2018-10-19 12:00:00 -0800
---
> **Note (Jun 13, 2023):** I'm adding this note here to warn you. The writing has been on the wall for at least a year now, but past here be dragons. Webpacker was a valiant effort at Rails-ifying the monster that is webpack, but it ultimately did not get fleshed out in the way that it needed to be, and things were eventually dropped in support of [cssbundling-rails](https://github.com/rails/cssbundling-rails) and [jsbundling-rails](https://github.com/rails/jsbundling-rails).
>
> From a JavaScript perspective, I strongly suggest avoiding webpack unless you know for a fact that you need the extreme level of configurability it offers. Webpacker as a concept should absolutely not be something you spend any time on unless you need to maintain an app that currently relies on it. As someone who generally finds themselves working on primarily server-side-rendered applications with complex but largely superficial JavaScript needs, switching to the aforementioned *bundling-rails gems and `esbuild` has been a fantastic experience.

I recently finished reworking my site's CSS using [Tailwind CSS](https://tailwindcss.com). Everything went smoothly locally - especially cool since I have no prior experience using Webpacker, or even Yarn. Tests were passing with no issues. After pushing my finished changes, TravisCI immediately broke - complaining about something wrong with Webpacker:

~~~~plain
$ bundle exec rake webpacker:compile
...
Compilation failed:
/home/travis/build/taylorthurlow/thurlow.io/vendor/bundle/ruby/2.5.0/gems/webpacker-3.5.5/lib/webpacker/webpack_runner.rb:11:in `exec': No such file or directory - /home/travis/build/taylorthurlow/thurlow.io/node_modules/.bin/webpack (Errno::ENOENT)
  from /home/travis/build/taylorthurlow/thurlow.io/vendor/bundle/ruby/2.5.0/gems/webpacker-3.5.5/lib/webpacker/webpack_runner.rb:11:in `block in run'
  from /home/travis/build/taylorthurlow/thurlow.io/vendor/bundle/ruby/2.5.0/gems/webpacker-3.5.5/lib/webpacker/webpack_runner.rb:10:in `chdir'
  from /home/travis/build/taylorthurlow/thurlow.io/vendor/bundle/ruby/2.5.0/gems/webpacker-3.5.5/lib/webpacker/webpack_runner.rb:10:in `run'
  from /home/travis/build/taylorthurlow/thurlow.io/vendor/bundle/ruby/2.5.0/gems/webpacker-3.5.5/lib/webpacker/runner.rb:6:in `run'
  from ./bin/webpack:15:in `<main>'
~~~~

To me, it wasn't exactly clear what was going on. It looked to me, at first, like the Webpacker gem just wasn't being installed, but the gem was in the gemfile, and after all, this worked locally. Here's the `travis.yml` that eventually solved my problem, with some of my stuff removed for brevity:

~~~~yaml
language: ruby
rvm:
  - 2.5.1

env:
  global:
    - NODE_ENV=test

before_script:
  - bundle exec rake db:schema:load RAILS_ENV=test

cache:
  bundler: true
  directories:
    - node_modules
  yarn: true

install:
  - bundle install
  - nvm install node
  - node -v
  - npm i -g yarn
  - yarn

script:
  - bundle exec rails webpacker:compile
  - bundle exec rake
~~~~

I'm not sure which of these changes fixed my problem, but it's likely the entire `install` section which wasn't there before. I hope this is helpful in getting Webpacker working properly on TravisCI.

Happy hacking!

