---
layout: post
title:  "Add code auto-loading to a Ruby project with Zeitwerk"
date:   2019-11-17 12:39:00 -0800
categories: ruby
---
[Zeitwerk](https://github.com/fxn/zeitwerk) is a Ruby code auto-loader written to replace Rails' code auto-loading method with the release of Rails 6. While it's a relatively new tool, and most of the discussion about it is within the context of Rails, it's really easy to use it in your own non-Rails Ruby projects.

## Why use Zeitwerk?

Loading code is a "solved problem", and there's no reason to use a code loader like Zeitwerk other than pure conveience. But, convenient it is! Never worry about code loading order - just set up Zeitwerk and use your application constants without fear of them not being defined properly. Zeitwerk is battle-tested and is the default code loader in Rails 6.

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

If you are developing an application which would benefit from code auto-reloading, like a web application, Zeitwerk supports this out of the box. Before calling `loader.setup`, call `enable_reloading`:

```ruby
# ...
loader = Zeitwerk::Loader.for_gem
loader.enable_reloading 
loader.setup

# ...
```

At this point, any time you call `loader.reload`, Zeitwerk will reload the application's code. In order to trigger this automatically, you'll need a gem which can monitor your files for changes. [`listen`](https://github.com/guard/listen) is perfect for this. 

After installing `listen`, set it up in the same file you do the Zeitwerk setup in, after you've called `loader.setup`:

```ruby
require "zeitwerk"

loader = Zeitwerk::Loader.for_gem
loader.enable_reloading
loader.setup 

Listen.to("lib", "config") { loader.reload }.start

module CoolGem
  # your application code
end

loader.eager_load
```

The listener runs in a separate thread and is non-blocking. Viola, you have auto-reloading code!

Happy hacking!
