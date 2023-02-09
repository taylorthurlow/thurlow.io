---
layout: post
title: OpenSSL & Ruby with Homebrew on macOS
categories: macos
date: 2019-12-12 10:46:00 -0800
---
> **Last updated 2023-02-09:** Added more information on the decision between OpenSSL 1.1 and 3, as well as running on ARM/Apple Silicon (it's as good as they say!).

Lots of applications used in development utilize OpenSSL, and macOS is shipped with its own build of OpenSSL as a result. Instead of having to deal with the version of OpenSSL shipped with your operating system, it's often much easier to download OpenSSL through Homebrew, and let Apple's version do what it likes, how it likes.

Using a separate installation of OpenSSL is certainly not without its own share of complications, so this post is a short guide on how to set up OpenSSL through Homebrew. I'll go into some detail in an effort to explain why we're doing what we're doing. We'll also talk about specific issues with Ruby development and OpenSSL, as well as how to install SSL certificates. I will make an effort to explain how this applies to both Intel and ARM-based Macs.

## Intel vs. ARM (Apple Silicon)

Homebrew functions largely the same between both architectures, but the primary installation directory of Homebrew (and where your packages will live) is different:

* ARM (Apple Silicon): `/opt/homebrew`
* Intel: `/usr/local`

The remainder of this guide will use ARM paths, but _generally_ the paths should also apply to Intel macs if you change the prefix. This is not a hard-and-fast rule, so double-check that the paths you are using on an Intel Mac exist. If they don't, typically the correct path isn't too far off, so dig around in the directory structure a tad and you should find what you are looking for.

## Getting to a clean slate

It's important that we start from a clean slate, so we need to make sure any existing versions of OpenSSL from Homebrew are uninstalled.

*Disclaimer:* I have no way of telling what uninstalling OpenSSL will do to your system. We're going to be reinstalling it (assuming you've even installed it before), so it seems unlikely it'll negatively impact anything, but please don't dive into this without being able to dedicate some time to problem-solving if something goes awry.

First, to check if it's installed, see if this command prints anything:

```bash
brew list | grep openssl
```

We're looking for the `openssl` package. If it's in the list, then remove it with:

```bash
brew uninstall --ignore-dependencies openssl
```

## Installing OpenSSL

Next we want to make sure we have the latest Xcode command line tools:

```bash
xcode-select --install
```

And now we can install OpenSSL again:

```bash
brew install openssl
```

It's going to depend on when you're following this guide, but the `openssl` package in Homebrew is likely an [alias](https://en.wikipedia.org/wiki/Aliasing_(computing)) for an OpenSSL package with a specific version number. At the time of writing, `openssl` was an alias for the `openssl@1.1` package. Currently, it is an alias for `openssl@3`. This is totally fine, but you'll need to know what the package name is (it might not be `openssl@3` if it's been updated since I wrote this).

You can check the real package name with `brew info openssl | head -n 1`. When I run this command, I get:

```
==> openssl@3: stable 3.0.8 (bottled) [keg-only]
```

## Configuration

Because Homebrew's version of OpenSSL doesn't want to step on macOS's built-in OpenSSL, installing Homebrew's OpenSSL intentionally skips adding any executables to your shell's `PATH`. We'll need to add some paths ourselves in order for things to work as expected.

Edit your `bashrc`/`bashprofile`/`zshrc`/`zprofile` and add the following lines, substituting the package name if necessary:

```bash
export PATH="/opt/homebrew/opt/openssl@3/bin:$PATH"
export LIBRARY_PATH="$LIBRARY_PATH:/opt/homebrew/opt/openssl@3/lib/"
```

The first will add the OpenSSL executables to our path, and the second will help us with some code compilation errors you might run into when programming. Remember, you need to restart your shell/terminal instances in order for these changes to take effect!

### An extra thing for Ruby developers to do

If you're doing Ruby development, you should also add this to the same file:

```bash
export RUBY_CONFIGURE_OPTS="--with-openssl-dir=$(brew --prefix openssl@3)"
```

This option changes how Ruby is built on your machine. If you use a utility like [`ruby-build`](https://github.com/rbenv/ruby-build) (or something like `rbenv` which relies on `ruby-build`), then this is important. By default `ruby-build` will download and build it's own instance of OpenSSL. We want it to use the one we've installed, and this compile option does that. It also makes the Ruby installation process a bit faster.

Because this takes place at the time you install Ruby, this means that you'll need to uninstall and reinstall all versions of Ruby installed on your machine. If you're using `rbenv`, then that would be a `rbenv uninstall 3.2.0 && rbenv install 3.2.0` for every version you have installed.

If you still have issues with OpenSSL and Ruby, also try:

```bash
export PKG_CONFIG_PATH="$(brew --prefix openssl@3)/lib/pkgconfig"
```

#### ARM/Apple Silicon and older versions of Ruby

The specifics of this problem are not yet clear to me, but versions of Ruby older than 2.7 do not appear to support ARM in any capacity. If you are looking for an older (end-of-life) version you will need to reach for some sort of virtualization technology. If there is a solution here involving Rosetta, then I am not aware of it. Feel free to send me an email if you are able to get it working!

Ruby 2.7 itself will also not install properly with the latest version of OpenSSL. In order to install 2.7 I had to also install the `openssl@1.1` package, and temporarily change the aforementioned environment variables:

```bash
export PATH="/opt/homebrew/opt/openssl@1.1/bin:$PATH"
export LIBRARY_PATH="$LIBRARY_PATH:/opt/homebrew/opt/openssl@1.1/lib/"
export RUBY_CONFIGURE_OPTS="--with-openssl-dir=/opt/homebrew/opt/openssl@1.1"
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:"/opt/homebrew/opt/openssl@1.1/lib/pkgconfig"
```

It is important to note that we need to ensure that there are no `openssl@3` paths in any of these variables (especially `PKG_CONFIG_PATH`, that caused me issues). So I would suggest manually editing your bash/zsh startup variables rather than temporarily exporting additional paths _on top of_ the existing `openssl@3` paths.

Once I ensured I had everything pointing to `openssl@1.1`, I was able to install Ruby 2.7.2 and the at-time-of-writing-latest patch version, 2.7.7. Attempting to install 2.7.0 and 2.7.1 spat out 130,000+ lines of compiler warnings and then failed, and I did not want to dig any further. Reach out if you have any info on this.

## Installing SSL certificates

Because we are now using a separate OpenSSL installation, we can't install certificates through `Keychain Access.app`. Instead, add your certificates to `/opt/homebrew/etc/openssl@3/certs`, substituting `@3` for your version number, as before. Then run `c_rehash` in any terminal. You should see a confirmation message that the aforementioned path was scanned for certificates. This process should add a new symlink to the certs directory, something like `85cf5865.0`. This means the rehash worked properly.

Happy hacking!

