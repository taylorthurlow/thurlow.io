---
layout: post
title: Frictionless deployment secrets using environment variables
categories: ruby
date: 2019-11-12 22:15:00 -0800
---
**Note (Jun 13, 2023):** After a number of years playing with different configuration management methodologies, I've largely settled on the idea that YAML kinda sucks in a lot of ways, and this solution does nothing to help with that. Take the contents of this article as you will, as everything regarding environment variables really still rings true, but I would focus primarily on my final suggestion to check out [`dotenv`](https://github.com/bkeepers/dotenv). There are libraries for the vast majority of languages and platforms out there, usually with the same name, that achieve largely the same result. How you ingest these environment variables into your application, though, I will leave as an exercise to the reader.

At work I find myself in a position which I'm sure many people find themselves in - as a developer, I'd rather configure my Ruby and Rails applications using the typical YAML file. Here's a database configuration, for example:

```yaml
# config/database.yml

default: &default
  adapter: sqlserver
  pool: 5
  timeout: 5000
  host: localhost
  port: 4501
  username: username
  password: password

development:
  <<: *default
  database: my_app_development

test:
  <<: *default
  database: my_app_test
```

From a dev-ops perspective, this is less than ideal: your build pipeline has to manage its own set of configuration files, you need to know where they're supposed to go, and you have to manage them in a way that doesn't reveal sensitive information to people (and software!) that it doesn't need to be shown to.

The lovely people doing the hard work to deploy your code would probably prefer storing that kind of information in an environment variable. They're far more convenient for a few reasons:

* Easily set in a build pipeline
* Available for use outside of the context of your application
* No need to commit a file to a secure repository or configuration file management system

And if we're feeling selfish, there's a cool benenfit to developers; being able to temporarily change a configuration variable when running your application, without having to edit a file. Let's say I want to run a Rails server like usual, but pointed at a different database host. Instead of modifying `database.yml` and replacing the value, I can just run the server this way:

```bash
RAILS_DB_HOST=stage-db.example.com bin/rails server
```

And now we have the same Rails server but with a customized database hostname.

## But how do you do it?

Setting this up is really simple - we take advantage of two things:

* [`ENV.fetch`](https://ruby-doc.org/core-2.6.5/ENV.html#method-c-fetch) - We can fetch the value of an environment variable this way, and crucially, provide a default value if the variable is not set. The first argument to the method call is the name of the environment variable, and the second is the default value. You can also provide a block argument in place of the default value, and the return value of the block argument will be used as the default value.
* [Embedded Ruby](https://ruby-doc.org/stdlib-2.6.5/libdoc/erb/rdoc/ERB.html) - You know it as ERB. We can use ERB parsing to place our configuration values in our configuration files automatically. Rails will help us out with this - but we can always write a tiny bit of code and do it ourselves.

### Writing a configuration file

Here's the same configuration file from before, but rewritten to use ENV variable fetching and default values.

```yaml
# config/database.yml

default: &default
  adapter: sqlserver
  pool: <%= ENV.fetch("RAILS_DB_POOL", 5) %>
  timeout: <%= ENV.fetch("RAILS_DB_TIMEOUT", 5000) %>
  host: <%= ENV.fetch("RAILS_DB_HOST", "localhost") %>
  port: <%= ENV.fetch("RAILS_DB_PORT", 4501) %>
  username: <%= ENV.fetch("RAILS_DB_USERNAME", "username") %>
  password: <%= ENV.fetch("RAILS_DB_PASSWORD", "password") %>
  database: <%= ENV.fetch("RAILS_DB_NAME", "") %>

development:
  <<: *default
  database: <%= ENV.fetch("RAILS_DB_NAME", "my_app_development") %>

test:
  <<: *default
  database: <%= ENV.fetch("RAILS_DB_NAME", "my_app_test") %>
  
production:
  <<: *default
  database: <%= ENV.fetch("RAILS_DB_NAME", "my_app_production") %>
```

In Rails, this just works! Rails' default YAML configuration files are all run through the ERB templating engine before they are read by the application, so no extra effort is required to actually interpret the ERB tags we've included. A few things to note here:

* I've deliberately hard-coded the value of `adapter` because there's no sense in being able to change that, and the default value of `database` is an empty string. I've done this because Ruby will raise an exception if `RAILS_DB_NAME` isn't set when calling `ENV.fetch`. This is okay though, because the environment-specific configuration option will override the default one anyways.
* We need to add the `production` section here, because now the deploy is using this exact file, instead of one written separately to be run in production. Normally we could get away with not defining production settings in our local configuration files, but now because they're shared, we need to include it here. This is a good thing! Now we don't need to look in two different places for the same information.

### Evaluating ERB tags in non-Rails configuration files

As I mentioned, default Rails configuration files will automatically be run through the ERB templating engine - no extra work needed on the part of the developer. However, if you have any other non-default configuration files in your application, say a file for storing the URL to an API your app is using, Rails won't take care of that for you.

No problem, though, we can do it ourselves! In a Rails application, add something like the following to a new initializer file (`config/initializers`) - but you'll need to change the code whereever the configuration file is read and loaded:

```ruby
# config/initializers/some_api.rb

config_path = Rails.root.join("config", "some_api_config.yml")
file_contents = File.read(config_path)
erb_document = ERB.new(file_contents)
config = erb_document.result
SOME_API_CONFIG = YAML.load(config)[Rails.env]
```

This code will load, parse and evaluate the contents of `config/some_api_config.yml`, including the ERB tags inside, and then store the environment-specific configuration in `SOME_API_CONFIG`. If you're not using Rails, just modify the existing YAML loading code to use ERB like I have above.

You're all done!

## Some extra crap, and extracurriculars

### ENV.fetch returns a string

`ENV.fetch` returns a string value. Let me explain the potential issue with an example:

```ruby
YAML.load("thing: 123")
# => {"thing"=>123}

YAML.load("thing: '123'")
# => {"thing"=>"123"}

YAML.load("thing: 123xyz")
# => {"thing"=>"123xyz"}
```

Loading a YAML file in the traditional way will intelligently cast your variables to the appropriate types. Quoted strings always become strings, but unquoted strings which are actually numbers will be converted properly. Unquoted strings with non-numeric characters in them will become strings.

Using `ENV.fetch` will always return a string - so be sure to convert the resulting value to the expected datatype within the ERB tags in the configuration file. Most of the time this should just involve an extra `#to_i` or `#to_f`.

### dotenv is cool

[`dotenv`](https://github.com/bkeepers/dotenv) can come in handy when in a development environment. Essentially, it loads environment variables from a `.env` file when starting a Rails server (or really any other application or application server, there's bound to be one for your language or application stack of choice). It's worth giving the Github page a read!

<br>

Happy hacking!