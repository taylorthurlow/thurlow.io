---
layout: post
title:  "Using factory_bot without Ruby on Rails"
date:   2018-10-31 12:00:00 -0800
categories: rails ruby factory_bot
---
# Preface about factory\_bot
In my work on Ruby on Rails projects, I'm a fan of using Thoughtbot's [factory\_bot](https://github.com/thoughtbot/factory_bot) as a replacement for the built-in concept of fixtures. If you're unfamiliar, the premise is pretty simple. Rails' fixtures are nice, but are really limited, and tend to hamstring developers into writing unmaintainable tests. A fixture looks something like this:

~~~~ruby
class Song < ApplicationRecord
  # has ActiveRecord attributes title, artist, year
  validates :title, :artist, :year, presence: true
end
~~~~

~~~~yaml
a_cool_song:
  title: All My Friends
  artist: LCD Soundsystem
  year: 2007
~~~~

It's easy to see that in a complex system, it would be necessary to either create a bunch of different fixtures or modify them in each test scenario. Doable for smaller projects, but it quickly becomes unweildy. Factories from factory\_bot are like supercharged fixtures. Instead of defining a bunch of fixtures, we define a single factory for our `Song` model:

~~~~ruby
FactoryBot.define do
  factory :song do
    title { 'All My Friends' }
    artist { 'LCD Soundsystem' }
    year { 2007 }
  end
end
~~~~

In our test scenarios, creating an instance of `Song`, even with potentially different attributes, is super easy:

~~~~ruby
pry(main)> FactoryBot.create(:song)
=> #<Song:0x00007ff42b84b008 @artist="LCD Soundsystem", @title="All My Friends", @year=2007>

pry(main)> FactoryBot.create(:song, title: 'call the police', year: 2017)
=> #<Song:0x00007ff42a321510 @artist="LCD Soundsystem", @title="call the police", @year=2017>
pry(main)>
~~~~

This is only scratching the surface of factory\_bot's capabilities. For more, check out its [getting started document](https://www.rubydoc.info/gems/factory_bot/file/GETTING_STARTED.md).

# Adapting factory\_bot to work with plain Ruby
factory\_bot was written and continues to be maintained with a huge focus on supporting Ruby on Rails - which is fine. It works wonderfully there. With just a couple small tweaks we can make it work nicely with POROs (Plain Old Ruby Objects). In some sense, it already does work, but there are a couple things to do to make it easier to work with.

## Fixing a broken call to ActiveRecord
Trying to use a factory out of the box might look like this:

~~~~ruby
class Song
  attr_accessor :title, :artist, :year
end

pry(main)> FactoryBot.create(:song)
NoMethodError: undefined method `save!' for #<Song:0x00007f915fae6aa8>
~~~~

factory\_bot is trying to call `ActiveRecord::Persistence#save!`. We're not using ActiveRecord, so we need to skip this. To do so, we can add `skip_create` to our factory definition.

~~~~ruby
FactoryBot.define do
  factory :song do
    skip_create

    title { 'All My Friends' }
    artist { 'LCD Soundsystem' }
    year { 2007 }
  end
end

pry(main)> FactoryBot.create(:song)
=> #<Song:0x00007ff42b84b008 @artist="LCD Soundsystem", @title="All My Friends", @year=2007>
~~~~

## Fixing object initialization
factory\_bot's actions are the same as they are in a Rails context, but it's important to understand exactly what it does to create the object. The first thing that factory\_bot does is actually instantiate the object, and run its `initialize` method, if it exists. This is sometimes a problem - factory\_bot won't pass any parameters into the call to `Song.new`, even if it's required. Consider a `Song` class like this:

~~~~ruby
class Song
  attr_accessor :title, :artist, :year

  def initialize(something)
    @title = "A Placeholder Title: #{something}"
  end
end
~~~~

What happens if we use our factory as-is?

~~~~ruby
pry(main)> FactoryBot.create(:song)
ArgumentError: wrong number of arguments (given 0, expected 1)
from test.rb:7:in `initialize'
~~~~

Clearly, factory\_bot is just calling `Song.new` and calling it a day. We can fix this with `initialize_with`, though:

~~~~ruby
FactoryBot.define do
  factory :song do
    skip_create

    title { 'All My Friends' }
    artist { 'LCD Soundsystem' }
    year { 2007 }

    initialize_with { new('foo') }
  end
end

pry(main)> FactoryBot.create(:song)
=> #<Song:0x00007ff42b84b008 @artist="LCD Soundsystem", @title="All My Friends", @year=2007>
~~~~

factory\_bot is now constructing the object in the way that we want, and everything works. You might notice that despite the `initialize` method setting the `@title` instance variable to some value, the value of `'All My Friends'` takes prescendence. This is because the `initialize` method is evaluated first, and then factory\_bot does simple assignment to the instance variables (`@title = 'All My Friends'` and so on...). This is why we need to include `attr_accessor` or `attr_writer` for all of the instance variables we define in our factory.

## Associations with other objects
You can still use factory\_bot's `association` method to call other factories, the same as in Rails. As long as there as an instance variable with the same name, it should work just fine.

~~~~ruby
class Album
  attr_accessor :title, :artist, :year
end

class Song
  attr_accessor :title, :album
end

FactoryBot.define do
  factory :album do
    skip_create

    title { 'Sound of Silver' }
    artist { 'LCD Soundsystem' }
    year { 2007 }
  end

  factory :song do
    skip_create

    title { 'All My Friends' }
    album { association(:album) }
  end
end
~~~~

Create the association automatically because we didn't pass it into the `create` call:

~~~~ruby
pry(main)> s = FactoryBot.create(:song)
=> #<Song:0x00007f9f612eec28
 @album=#<Album:0x00007f9f639f38f0 @artist="LCD Soundsystem", @title="Sound of Silver", @year=2007>,
 @title="All My Friends">
pry(main)> s.album
=> #<Album:0x00007f9f639f38f0 @artist="LCD Soundsystem", @title="Sound of Silver", @year=2007>
~~~~

Set the association to an existing object by passing it into the `create` call as a named parameter:

~~~~ruby
pry(main)> a = FactoryBot.create(:album)
=> #<Album:0x00007f9f639a0df8 @artist="LCD Soundsystem", @title="Sound of Silver", @year=2007>
pry(main)> s = FactoryBot.create(:song, album: a)
=> #<Song:0x00007f9f61366f20
 @album=#<Album:0x00007f9f639a0df8 @artist="LCD Soundsystem", @title="Sound of Silver", @year=2007>,
 @title="All My Friends">
~~~~

Note that the memory address in the `Album` object identifier is `0x00007f9f639a0df8` in both cases. This means we know for a fact that the same instance of `Album` we created first was assigned to the `@album` instance variable on the `Song` instance, as we wanted.

That's all for now, happy hacking!
