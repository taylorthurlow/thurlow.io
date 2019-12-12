---
layout: post
title:  "OpenSSL with Homebrew on MacOS"
date:   2019-12-12 10:46:00 -0800
categories: macos
---
Intro here

## Header

```bash
export PATH="$PATH:/usr/local/opt/openssl@1.1/bin"
export LIBRARY_PATH="$LIBRARY_PATH:/usr/local/opt/openssl@1.1/lib/"
```

basic steps for doing the install properly
* `brew uninstall openssl`
* ensure `brew list | grep openssl` is empty
* get the latest xcode command line tools with `xcode-select --install` after updating to the latest xcode
* `brew install openssl` (`openssl@1.1` is aliased to `openssl` at the time of writing)
* add the above lines to your bashrc/zshrc/bashprofile/zprofile
  * the first will fix most compilation issues, I use the 2nd one to fix `mysql2` gem installation
* if you build ruby from scratch or use any utility which does so (like `ruby-build`) then you'll need to uninstall and rebuild all ruby versions
  * Might need `export RUBY_CONFIGURE_OPTS="--with-openssl-dir=$(brew --prefix openssl)"`, if rbenv doesn't automatically use homebrew installed version (not sure why it couldn't find it? `Downloading openssl-1.1.1d.tar.gz...`
  * with rbenv, do `rbenv uninstall VERSION` for all versions, and re-run `rbenv install VERSION` for all versions

