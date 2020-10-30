---
layout: post
title:  "Ruby on Rails: Scopes"
date:   2017-06-18 12:00:00 -0800
categories: ruby
---
Scopes are super useful - and easy to write. Rails 4 and newer make it really easy. Rails 3 and older do have scopes, but I won't cover them here.  Let's say I have a `timelog` model which, among other things, has a `start_time` and `end_time`. If I wanted to scope the hard way, using the usual syntax, it would look something like:

~~~ruby
# app/models/timelog.rb

scope = Timelog.all
scope = scope.where('start_time > ?', start_time) unless start_time.blank?
scope = scope.where('end_time < ?', end_time) unless end_time.blank?
~~~

It would be pretty lame to have to do that every time you're looking for `timelogs` in a particular date range. Scopes makes this super easy. Let's tweak the previous code block a bit. In our model file, `timelog.rb`, we can place this new block of code somewhere near the top. I like to do it below my validations and callbacks, but it's up to you.

~~~ruby
# app/models/timelog.rb

scope :between, -> (start_time, end_time) {
  scope = all
  scope = scope.where('start_time > ? ', start_time) unless start_time.blank?
  scope = scope.where('end_time < ?', end_time) unless end_time.blank?
  return scope
}
~~~

Pretty similar save a few things here and there - but now look how easy it is to call this scope using a rails console:

~~~plain
2.3.1 :001 > Timelog.between(2.months.ago, Time.now)
  Timelog Load (10.9ms)  SELECT "timelogs".* FROM "timelogs" WHERE (start_time > '2017-04-18 06:47:25.635676' ) AND (end_time < '2017-06-18 06:47:25.635821')  ORDER BY "timelogs"."start_time" DESC
 => #<ActiveRecord::Relation [...]>
2.3.1 :002 > Timelog.between(2.months.ago, Time.now).count
   (0.8ms)  SELECT COUNT(*) FROM "timelogs" WHERE (start_time > '2017-04-18 06:48:30.847610' ) AND (end_time < '2017-06-18 06:48:30.847920')
 => 579
~~~

What's even better, is that the scope will accept `nil` for either of the bounds. So, if you wanted to get all the timelogs older than 2 weeks, it's as simple as:

~~~plain
2.3.1 :001 > Timelog.between(nil, 2.weeks.ago)
~~~

It's worth using scopes, clearly, but also avoid overusing them. Happy hacking!
