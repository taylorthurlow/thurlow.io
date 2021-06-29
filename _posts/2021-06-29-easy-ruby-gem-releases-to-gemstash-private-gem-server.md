---
layout: post
title: Easy Ruby gem releases to Gemstash private gem server
categories: ruby
date: 2021-06-29 12:31:35 -07:00
---
bundle config set --global https://gem.example.com $FETCH_API_KEY
bundle config set --global https://gem.example.com/private $PUSH_API_KEY
bundle config set --global gem.push_key my_push_key