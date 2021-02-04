---
layout: post
title: Remote Selenium WebDriver servers with Rails, Capybara, RSpec, and Chrome
categories: ruby
date: 2020-11-05 16:02:22 -08:00
---
At work, I've been spending some time getting our Rails projects deployed on new Kubernetes infrastructure, and we're taking the opportunity to fix some problems with our current CI pipeline - the biggest of which is a lack of tests. The first app we're deploying is a greenfield (yet to be used in production) Rails app which uses RSpec and includes Rails 5.1/6-style "system specs" which use Capybara to drive a web browser.

Getting JavaScript-enabled system specs working when you run the tests locally isn't too bad - mostly just install a few gems, configure them per their respective READMEs, and off you go. This was the case with this new Rails app, but seeing as we wanted tests to be fully supported in our new CI pipeline, we needed to figure out how to deal with system specs, and by extension, Selenium.

My goal with this post is to explain how to set up your specs so they run locally when you're developing locally, but also on an arbitrary Selenium server when you desire that (like during CI). I'll be using Chrome as the driver, but configuration should be minimally different for other browsers.

## Selenium WebDriver server setup

Because we're using Docker to set up this testing CI environment, we can use [docker-selenium](https://github.com/SeleniumHQ/docker-selenium), the official Selenium Docker image repository. Because my local machine isn't the CI server, I'll need to set up a Selenium server straight from the Docker image:

```bash
$> docker run --name=selenium -p '4444:4444/tcp' -p '5900:5900/tcp' 'selenium/standalone-chrome-debug'
```

This command will download the container and run it with two ports open - `4444`  is the Selenium WebDriver server, and `5900` is a VNC server which you can use to visually inspect the browser if you want. This is why I used the `standalone-chrome-debug` selenium image instead of the `standalone-chrome` image - the non-debug version doesn't come with a VNC server. In production, you can use the non-debug version. You can use a VNC client to open `vnc://localhost:5900`, using the password `secret`. If you're on macOS, you can open Finder, press `CMD+k`, and paste in the above URL directly - macOS has a built-in VNC client.

## Configuring Rails

You'll need a few gems, some or all of which you may already have:

* [Capybara](https://github.com/teamcapybara/capybara), a DSL for testing frameworks used to manipulate web drivers like Selenium (`v3.33.0` at the time of writing)
* [selenium-webdriver](https://github.com/SeleniumHQ/selenium/tree/trunk/rb), the Ruby bindings for controlling Selenium WebDriver (`v3.142.7` at the time of writing)
* [webdrivers](https://github.com/titusfortner/webdrivers), an easy way to automatically install, use, and update web drivers for popular browsers (`v4.4.1` at the time of writing)
* [rspec-rails](https://github.com/rspec/rspec-rails), my Ruby testing library of choice (`v4.0.1` at the time of writing)

If you use Minitest instead of RSpec, things should be largely the same, but you're on your own as far as getting things working. Differences between this guide and the same process for Minitest should be mostly superficial.

### Configuring the rails_helper

Following the `require "rspec/rails"` statement in `rails_helper.rb`, add the following requires:

```ruby
# spec/rails_helper.rb

# ...

require "rspec/rails"

require "capybara/rails"
require "capybara/rspec"

Dir[Rails.root.join("spec", "support", "**", "*.rb")].sort.each { |f| require f }

# ...
```

You can omit the complicated `spec/support` directory require logic, but I find it generally useful. We'll be creating `spec/support/capybara.rb` shortly, so if you don't want to do it the way I did, then make sure you require that file.

### Configuring Capybara

Create `spec/support/capybara.rb`. You can treat this as a sort of an RSpec initializer for Capybara. What follows is the entire contents of the file that you should paste in. I will explain each part afterwards.

```ruby
Capybara.register_driver :remote_selenium do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--window-size=1400,1400")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-dev-shm-usage")

  Capybara::Selenium::Driver.new(
    app,
    browser: :chrome,
    url: "http://#{ENV["SELENIUM_HOST"]}:4444/wd/hub",
    options: options,
  )
end

Capybara.register_driver :remote_selenium_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless")
  options.add_argument("--window-size=1400,1400")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-dev-shm-usage")

  Capybara::Selenium::Driver.new(
    app,
    browser: :chrome,
    url: "http://#{ENV["SELENIUM_HOST"]}:4444/wd/hub",
    options: options,
  )
end

Capybara.register_driver :local_selenium do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--window-size=1400,1400")

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

Capybara.register_driver :local_selenium_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless")
  options.add_argument("--window-size=1400,1400")

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

selenium_app_host = ENV.fetch("SELENIUM_APP_HOST") do
  Socket.ip_address_list
        .find(&:ipv4_private?)
        .ip_address
end

Capybara.configure do |config|
  config.server = :puma, { Silent: true }
  config.server_host = selenium_app_host
  config.server_port = 4000
end

RSpec.configure do |config|
  config.before(:each, type: :system) do |example|
    # `Capybara.app_host` is reset in the RSpec before_setup callback defined
    # in `ActionDispatch::SystemTesting::TestHelpers::SetupAndTeardown`, which
    # is annoying as hell, but not easy to "fix". Just set it manually every
    # test run.
    Capybara.app_host = "http://#{Capybara.server_host}:#{Capybara.server_port}"

    # Allow Capybara and WebDrivers to access network if necessary
    driver = if example.metadata[:js]
        locality = ENV["SELENIUM_HOST"].present? ? :remote : :local
        headless = "_headless" if ENV["DISABLE_HEADLESS"].blank?

        "#{locality}_selenium#{headless}".to_sym
      else
        :rack_test
      end

    driven_by driver
  end
end
```

The first part of this file involves creating four different custom Capybara drivers to run our specs. We have four because we need to have a separately configured driver for local runs and remote runs, as well as a separately configured driver for "headless" runs, where the browser runs invisibly, instead of actually opening a desktop window and stealing focus. It's usually annoying, but sometimes it's valuable to be able to see Selenium manipulate the browser yourself.

So we've created four drivers: `remote_selenium_headless`, `remote_selenium`, `local_selenium_headless`, and `local_selenium`. Each needs to be configured slightly differently. The `Selenium::WebDriver::Chrome::Options` object represents command-line flags to the chromedriver:

* [`--window-size=1400,1400`](https://peter.sh/experiments/chromium-command-line-switches/#window-size) - Set the window size to 1400x1400 pixels. This is a reasonable size without being too large, but you can set it to whatever you like. This mostly impacts the size of debugging screenshots, but some tests may fail if you ask Capybara to click on an element which is not currently visible on the page.
* [`--no-sandbox`](https://peter.sh/experiments/chromium-command-line-switches/#no-sandbox) - Disables Chrome's [sandbox](https://chromium.googlesource.com/chromium/src/+/master/docs/design/sandbox.md) functionality, because it has an issue with Docker version `1.10.0` and later. You may see a warning in the Docker container logs about "security and stability" suffering, which doesn't really matter in this case.
* [`--disable-dev-shm-usage`](https://peter.sh/experiments/chromium-command-line-switches/#disable-dev-shm-usage) - The `/dev/shm` shared memory partition is too small on many VM environments, which will cause Chrome to [fail or crash](http://crbug.com/715363). You might be able to get away with omitting this option, or mapping the `/dev/shm` path to the Docker host, but I haven't bothered with it. If your Selenium performance is suffering, it may be worth investigating this further.
* [`--headless`](https://peter.sh/experiments/chromium-command-line-switches/#headless) - Enable Chrome's headless mode which will run Chrome without a UI or display server dependencies.

*Note:* Some guides may suggest using the `--disable-gpu` flag, but [this is no longer necessary on any operating system](https://bugs.chromium.org/p/chromium/issues/detail?id=737678).

You'll also notice that the `:url` key is provided when constructing the remote drivers, but not for the local ones.

```ruby
selenium_app_host = ENV.fetch("SELENIUM_APP_HOST") do
  Socket.ip_address_list
        .find(&:ipv4_private?)
        .ip_address
end

Capybara.configure do |config|
  config.server = :puma, { Silent: true }
  config.server_host = selenium_app_host
  config.server_port = 4000
end
```

The next thing we do is determine the `selenium_app_host` by checking the value of the `SELENIUM_APP_HOST` environment variable, falling back to a little Ruby which obtains the IP address of the machine currently running the code. This is important because our Selenium server needs to know where the Rails app itself is being run - the same server that is running this Ruby code. Typically you won't need to set `SELENIUM_APP_HOST`, but it's there in case the automatic-IP-determining code doesn't work properly. Now that we have the IP/host of the app server, we configure Capybara to use that IP and port 4000 for the puma web server. This is where Capybara will run our Rails application server so the Selenium driver and browser have something to make a request to.

Now that we've created the drivers, we need to configure RSpec to use the correct driver in the correct scenario. Using the `before(:each)` RSpec hook, we can run some code before each spec example of type `:system`. Specs' types are determined either by the subdirectory they are contained within under the `spec` directory (only when RSpec's `infer_spec_type_from_file_location` option is enabled), or directly in the spec file itself:

```ruby
# spec/system/some_example_spec.rb

RSpec.describe "Sign Up", type: :system do
  it "signs up a new user", js: true do
    # ...
  end
end
```

System specs fall into two categories: specs that require JavaScript, and those that don't. System specs that require JavaScript are denoted with the `js: true` tag. If your system spec can avoid using JavaScript, then you should do so. System specs that don't require JavaScript are run using the `:rack_test` driver instead of Selenium. The advantage there is that they run *much* faster.

```ruby
config.before(:each, type: :system) do |example|
  # `Capybara.app_host` is reset in the RSpec before_setup callback defined
  # in `ActionDispatch::SystemTesting::TestHelpers::SetupAndTeardown`, which
  # is annoying as hell, but not easy to "fix". Just set it manually every
  # test run.
  Capybara.app_host = "http://#{Capybara.server_host}:#{Capybara.server_port}"
 
  driver = if example.metadata[:js]
      locality = ENV["SELENIUM_HOST"].present? ? :remote : :local
      headless = "_headless" if ENV["DISABLE_HEADLESS"].blank?

      "#{locality}_selenium#{headless}".to_sym
    else
      :rack_test
    end

  driven_by driver
end
```

Our `before(:each)` block begins with setting the Capybara `app_host` configuration value. This happens here due to the fact that Rails is going to reset it to `127.0.0.1` before every spec runs, whether we like that or not. We use the configuration values we set before to construct the app host for every Capyara system spec.

Next comes a check if our current `example` (the current spec being run) has this `js` flag set to `true`. If it does, this is our signal to use one of our four Selenium drivers. If not, then we can fall back to `:rack_test`.

The conditional assignment to the `driver` variable constructs a symbol name which corresponds to the driver we need to use. We check for the presence of a value in the `SELENIUM_HOST` environment variable, and if there is one, we assume we want to connect to a remote Selenium server, so we pick `remote` as the prefix. We can then tack on a `_headless` suffix by default, skipping it if the `DISABLE_HEADLESS` environment variable has a value (not true/false, mind you).

Now that we've generated the name of the driver we want to use, we can pass that to `driven_by`. It's worth noting that `driven_by` is not a Capybara method, but rather a method provided by Rails. We can pass a Capybara driver name to it if we want, though.

### If you use WebMock

If you use [WebMock](https://github.com/bblimke/webmock) to disable network connections when running tests, you might find that Selenium, Capybara, and Chromedriver really like to connect to each other, and WebMock isn't having any of that. I have a configuration that seems to work as expected, so here is `capybara.rb` in its entirety with the WebMock changes added in:

```ruby
Capybara.register_driver :remote_selenium do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--window-size=1400,1400")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-dev-shm-usage")

  Capybara::Selenium::Driver.new(
    app,
    browser: :chrome,
    url: "http://#{ENV["SELENIUM_HOST"]}:4444/wd/hub",
    options: options,
  )
end

Capybara.register_driver :remote_selenium_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless")
  options.add_argument("--window-size=1400,1400")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-dev-shm-usage")

  Capybara::Selenium::Driver.new(
    app,
    browser: :chrome,
    url: "http://#{ENV["SELENIUM_HOST"]}:4444/wd/hub",
    options: options,
  )
end

Capybara.register_driver :local_selenium do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--window-size=1400,1400")

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

Capybara.register_driver :local_selenium_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless")
  options.add_argument("--window-size=1400,1400")

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

selenium_app_host = ENV.fetch("SELENIUM_APP_HOST") do
  Socket.ip_address_list
        .find(&:ipv4_private?)
        .ip_address
end

Capybara.configure do |config|
  config.server = :puma, { Silent: true }
  config.server_host = selenium_app_host
  config.server_port = 4000
end

# NOTE: It's worth noting that chromedriver will pick port 9515 by default, but
# if for some reason an existing chromedriver instance is still running, it
# will pick the next largest port, which will break everything.
ALLOWED_WEBMOCK_ADDRESSES = [
  "localhost:#{Capybara.server_port}",
  "#{selenium_app_host}:#{Capybara.server_port}",
  "127.0.0.1:9515", # chromedriver
  "localhost:9515", # chromedriver
  /chromedriver/,
]

ALLOWED_WEBMOCK_ADDRESSES << ENV["SELENIUM_HOST"] if ENV["SELENIUM_HOST"].present?

WebMock.disable_net_connect!(allow: ALLOWED_WEBMOCK_ADDRESSES)

RSpec.configure do |config|
  config.before(:each, type: :system) do |example|
    # `Capybara.app_host` is reset in the RSpec before_setup callback defined
    # in `ActionDispatch::SystemTesting::TestHelpers::SetupAndTeardown`, which
    # is annoying as hell, but not easy to "fix". Just set it manually every
    # test run.
    Capybara.app_host = "http://#{Capybara.server_host}:#{Capybara.server_port}"

    # Allow Capybara and WebDrivers to access network if necessary
    driver = if example.metadata[:js]
        locality = ENV["SELENIUM_HOST"].present? ? :remote : :local
        headless = "_headless" if ENV["DISABLE_HEADLESS"].blank?

        "#{locality}_selenium#{headless}".to_sym
      else
        :rack_test
      end

    WebMock.disable!
    driven_by driver
    WebMock.enable!

    WebMock.disable_net_connect!(allow: ALLOWED_WEBMOCK_ADDRESSES)
  end
end

```

This will allow the `webdrivers` gem to fetch Chromedriver itself, and it will allow communication between Capybara and the Selenium server.

## Running our specs

With what we've set up, we now have four different ways of running our specs:

```bash
# Locally, with a headless Chrome instance
bundle exec rspec

# Locally, with a visible Chrome window that will steal focus when it opens
DISABLE_HEADLESS=true bundle exec rspec

# Remotely, with a headless (invisible) Chrome instance
SELENIUM_HOST="http://localhost" bundle exec rspec

# Remotely, with a visible Chrome window
SELENIUM_HOST="http://localhost" DISABLE_HEADLESS=true bundle exec rspec
```

The value of `SELENIUM_HOST` can be any server, though it does assume that the Selenium server is running on port `4444`. You can always tweak the driver creation part of the code above to not include the port number if that's what you want, but remember to leave the `/wd/hub` suffix on there.

And that's it! If you're using Selenium in CI, just set up the server however you see fit, make sure it's network-accessible from your Rails server, and set `SELENIUM_HOST` to the hostname or IP of the selenium server. Everything else will be handled automatically.

Happy hacking!