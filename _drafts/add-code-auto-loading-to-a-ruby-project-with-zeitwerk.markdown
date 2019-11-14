---
layout: post
title:  "Add code auto-loading to a Ruby project with Zeitwerk"
date:   2019-11-13 21:15:00 -0800
categories: ruby
---
[zeitwerk](https://github.com/fxn/zeitwerk) is a Ruby code auto-loader written to replace Rails' code auto-loading method with the release of Rails 6. While it's a relatively new tool, and most of the discussion about it is within the context of Rails, it's really easy to use it in your own non-Rails Ruby projects.

## Why use Zeitwerk?

TODO

## Basics

### Project structure

A good Ruby project has a good project directory and file structure. The most generally-accepted structure is what the Zeitwerk documentation calls a "conventional file structure", and it's the structure that'll need to be followed (generally) when using Zeitwerk to load code automatically. Here's the example given by the Zeitwerk documentation:

```plain
lib/my_gem.rb         -> MyGem
lib/my_gem/foo.rb     -> MyGem::Foo
lib/my_gem/bar_baz.rb -> MyGem::BarBaz
lib/my_gem/woo/zoo.rb -> MyGem::Woo::Zoo
```

In this structure, the directories and files are named based on the modules or classes that they define. Your project doesn't necessarily have to be a gem.

Zeitwerk also allows adding multiple directories to the root namespace. Rails is a great example of this, that you're probably familiar with:

```plain
app/models/user.rb                        -> User
app/controllers/admin/users_controller.rb -> Admin::UsersController
```

In this example, both `app/models` and `app/controllers` act as directories belonging to the root namespace. This allows you to structure your project files in a more abstract way, while still maintaining code auto-loading functionality.

### Setting up 

To add auto-loading to your existing code, you'll need to add some code before the definition of the main module of your gem or application. In a Rack application, this would be either `config.ru`, or the main Ruby file which is included within `config.ru` to start the application. In a gem, this would be whichever file first defines the outermost module - the one named the same as the gem.

```ruby
require "zeitwerk"

loader = Zeitwerk::Loader.for_gem
loader.setup 

module CoolGem
  # your application code
end
```

That's it! Now Zeitwerk is doing all the hard work. Any constants that are referenced within your application will now be loaded on-demand, rather than through `require` statements.

### Remove unnecessary require statements

Now that we don't need to use `require` statements to require code within our own project, we can now remove them. Remember, this does not include require statements for:

* Other third-party gems your application depends on
* Ruby core and standard libraries (think `yaml`, `json`, or `fileutils`)

Those require statements aren't the ones that are a pain, though!

## Eager loading code

By default, Zeitwerk will load definitions for constants used in your application *on-demand*. This is fine, and maybe the desired effect, especially if you have a lot of code to load, and startup-time performance is a concern. If it's not, or you have a reasonably-sized project, I'd suggest enabling eager loading.

Eager loading will evaluate the entire source tree and load all code automatically, rather than doing it on-demand. This also comes with the added benefit of knowing whether or not your application is structured properly, and all code can be loaded, even if it isn't necessarily reached during runtime - all at startup.

### Enabling eager loading

To enable eager loading, call `#eager_load` on the Zeitwerk loader after the end of the main module definition. 

```ruby
require "zeitwerk"

loader = Zeitwerk::Loader.for_gem
loader.setup 

module CoolGem
  # your application code
end

loader.eager_load
```

## Auto reloading code

TODO
