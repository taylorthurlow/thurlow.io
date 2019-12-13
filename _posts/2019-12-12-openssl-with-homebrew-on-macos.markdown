---
layout: post
title:  "OpenSSL & Ruby with Homebrew on macOS"
date:   2019-12-12 10:46:00 -0800
categories: macos
---
Lots of applications used in development utilize OpenSSL, and macOS is shipped with its own build of OpenSSL as a result. Instead of having to deal with the version of OpenSSL shipped with your operating system, it's often much easier to download OpenSSL through Homebrew, and let Apple's version do what it likes, how it likes.

Using a separate installation of OpenSSL is certainly not without its own share of complications, so this post is a short guide on how to set up OpenSSL through Homebrew. I'll go into some detail in an effort to explain why we're doing what we're doing. We'll also talk about specific issues with Ruby development and OpenSSL, as well as how to install SSL certificates.

At the time of writing, the current version of OpenSSL is `1.1.1d  10 Sep 2019`. You can find this version string by running `openssl version`.

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

It's going to depend on when you're following this guide, but the `openssl` package in Homebrew is likely an [alias](https://en.wikipedia.org/wiki/Aliasing_(computing)) for an OpenSSL package with a specific version number. Currently, `openssl` is a package alias for the `openssl@1.1` package. This is totally fine, but you'll need to know what the package name is (it might not be `openssl@1.1` if it's been updated since I wrote this).

You can check the real package name with `brew info openssl | head -n 1`. When I run this command, I get:

```
openssl@1.1: stable 1.1.1d (bottled) [keg-only]
```

## Configuration

Because Homebrew's version of OpenSSL doesn't want to step on macOS's built-in OpenSSL, installing Homebrew's OpenSSL intentionally skips adding any executables to your shell's `PATH`. We'll need to add some paths ourselves in order for things to work as expected.

Edit your `bashrc`/`bashprofile`/`zshrc`/`zprofile` and add the following lines, substituting the package name if necessary:

```bash
export PATH="/usr/local/opt/openssl@1.1/bin:$PATH"
export LIBRARY_PATH="$LIBRARY_PATH:/usr/local/opt/openssl@1.1/lib/"
```

The first will add the OpenSSL executables to our path, and the second will help us with some code compilation errors you might run into when programming.

Remember, you need to restart your shell/terminal instances in order for these changes to take effect!

### An extra thing for Ruby developers to do

If you're doing Ruby development, you should also add this to the same file:

```bash
export RUBY_CONFIGURE_OPTS="--with-openssl-dir=$(brew --prefix openssl)"
```

This option changes how Ruby is built on your machine. If you use a utility like [`ruby-build`](https://github.com/rbenv/ruby-build) (or something like `rbenv` which relies on `ruby-build`), then this is important. By default `ruby-build` will download and build it's own instance of OpenSSL. We want it to use the one we've installed, and this compile option does that. It also makes the Ruby installation process a bit faster.

Because this takes place at the time you install Ruby, this means that you'll need to uninstall and reinstall all versions of Ruby installed on your machine. If you're using `rbenv`, then that would be a `rbenv uninstall 2.6.5 && rbenv install 2.6.5` for every version you have installed.

## Installing SSL certificates

Because we are now using a separate OpenSSL installation, we can't install certificates through `Keychain Access.app`. Instead, add your certificates to `/usr/local/etc/openssl@1.1/certs`, substituting `@1.1` for your version number, as before. Then run `c_rehash` in any terminal. You should see a confirmation message that the aforementioned path was scanned for certificates. This process should add a new symlink to the certs directory, something like `85cf5865.0`. This means the rehash worked properly.

---

Happy hacking!
