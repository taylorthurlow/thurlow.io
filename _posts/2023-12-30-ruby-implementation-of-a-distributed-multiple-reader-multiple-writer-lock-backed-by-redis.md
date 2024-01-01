---
layout: post
title: Distributed, multi-reader, multi-writer lock, with Redis & Ruby
categories: ruby redis concurrency
date: 2023-12-30 14:03:31 -08:00
---
I am in the middle of a work project which requires a distributed (multi-host) locking mechanism which supports multiple readers and multiple writers. In this case I am not literally reading and writing a specific piece of data, but rather I am in a situation where a front-end reads and writes data to an authoritative back-end system that generally "stays the same", but semi-regularly undergoes changes (think CMS-level changes) that have the potential to fundamentally change the content the front-end accesses and manipulates, or the validity of the stuff the front-end has already done.

Using a "read/write" lock seems like a good fit - the front-end can obtain a read lock whenever it wants to read from or manipulate data on the back-end, and the back-end can obtain a write lock whenever it wants to make these fundamental changes that directly impact the front-end. Then the front-end is free to check in with the back-end and reconcile any problems that may exist now that the back-end has changed. All this is to say that while this might be called a read/write lock in practice, the way this lock can be used is a little broader and more abstract.

Here's my (not yet tested in production) implementation of this lock. Assume that it doesn't work yet - but tests locally seem to suggest it's doing what we want it to do. The code is well-documented so if you want to understand how it works, give it a read.

<script src="https://gist.github.com/taylorthurlow/84d942b77adf9e1200260b7d64d7fdff.js"></script>

Happy hacking!