# ![nio4r](https://raw.github.com/celluloid/nio4r/master/logo.png)

[![Gem Version](https://badge.fury.io/rb/nio4r.svg)](http://rubygems.org/gems/nio4r)
[![Build Status](https://secure.travis-ci.org/celluloid/nio4r.svg?branch=master)](http://travis-ci.org/celluloid/nio4r)
[![Code Climate](https://codeclimate.com/github/celluloid/nio4r.svg)](https://codeclimate.com/github/celluloid/nio4r)
[![Coverage Status](https://coveralls.io/repos/celluloid/nio4r/badge.svg?branch=master)](https://coveralls.io/r/celluloid/nio4r)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/celluloid/nio4r/blob/master/LICENSE.txt)

_NOTE: This is the 2.x **development** branch of nio4r.  For the 1.x **stable**
branch (used by [Rails 5]), please see:_

https://github.com/celluloid/nio4r/tree/1-x-stable

**New I/O for Ruby (nio4r)**: cross-platform asynchronous I/O primitives for
scalable network clients and servers. Modeled after the Java NIO API, but
simplified for ease-of-use.

**nio4r** provides an abstract, cross-platform stateful I/O selector API for Ruby.
I/O selectors are the heart of "reactor"-based event loops, and monitor
multiple I/O objects for various types of readiness, e.g. ready for reading or
writing.

[Rails 5]: https://rubygems.org/gems/actioncable

## Projects using nio4r

* [ActionCable]: Rails 5 WebSocket protocol, uses nio4r for a WebSocket server
* [Celluloid::IO]: Actor-based concurrency framework, uses nio4r for async I/O

[ActionCable]: https://rubygems.org/gems/actioncable
[Celluloid::IO]: https://github.com/celluloid/celluloid-io

## Goals

* Expose high-level interfaces for stateful IO selectors
* Keep the API small to maximize both portability and performance across many
  different OSes and Ruby VMs
* Provide inherently thread-safe facilities for working with IO objects

## Supported platforms

* Ruby 2.2
* Ruby 2.3
* Ruby 2.4
* JRuby 9000
* Pure Ruby using Kernel.select

## Platform notes

* MRI/YARV and Rubinius implement nio4r with a C extension based on libev,
  which provides a high performance binding to native IO APIs
* JRuby uses a Java extension based on the high performance Java NIO subsystem
* A pure Ruby implementation is also provided for Ruby implementations which
  don't implement the MRI C extension API

## Usage

If you're interested in using nio4r for a project but just getting started,
check out this blog post which provides some background and examples:

[A gentle introduction to nio4r: low-level portable asynchronous I/O for Ruby][blogpost]

[blogpost]: https://tonyarcieri.com/a-gentle-introduction-to-nio4r

### Selectors

The `NIO::Selector` class is the main API provided by nio4r. Use it where
you might otherwise use `Kernel.select`, but want to monitor the same set
of IO objects across multiple select calls rather than having to reregister
them every single time:

```ruby
require 'nio'

selector = NIO::Selector.new
```

Selectors use various platform-specific backends in order to select sockets
that are ready for I/O. You can check which backend is in use with the
`#backend` method:

```ruby
>> selector.backend
 => :epoll
 ```

To monitor IO objects, attach them to the selector with the `#register`
method, monitoring them for read readiness with the `:r` parameter, write
readiness with the `:w` parameter, or both with `:rw.`

```ruby
>> reader, writer = IO.pipe
 => [#<IO:0xf30>, #<IO:0xf34>]
>> monitor = selector.register(reader, :r)
 => #<NIO::Monitor:0xfbc>
```

After registering an IO object with the selector, you'll get a `NIO::Monitor`
object which you can use for managing how a particular IO object is being
monitored. Monitors will store an arbitrary value of your choice, which
provides an easy way to implement callbacks:

```ruby
>> monitor = selector.register(reader, :r)
 => #<NIO::Monitor:0xfbc>
>> monitor.value = proc { puts "Got some data: #{monitor.io.read_nonblock(4096)}" }
 => #<Proc:0x1000@(irb):4>
```

The main method of importance is `#select`, which monitors all
registered IO objects and returns an array of monitors that are ready.

```ruby
>> writer << "Hi there!"
 => #<IO:0x103c>
>> ready = selector.select
 => [#<NIO::Monitor:0xfbc>]
>> ready.each { |m| m.value.call }
Got some data: Hi there!
 => [#<NIO::Monitor:0xfbc>]
```

By default, `#select` will block indefinitely until one of the IO objects being
monitored becomes ready. However, you can also pass a timeout to wait in
to `#select` just like you can with `Kernel.select`:

```ruby
ready = selector.select(15) # Wait 15 seconds
```

If a timeout occurs, ready will be nil.

You can avoid allocating an array each time you call `#select` by passing a
block to select. The block will be called for each ready monitor object,
with that object passed as an argument. The number of ready monitors
is returned as a `Fixnum`:

```ruby
>> selector.select { |m| m.value.call }
Got some data: Hi there!
 => 1
```

When you're done monitoring a particular IO object, just deregister it from
the selector:

```ruby
selector.deregister(reader)
```

### Monitors

The `NIO::Monitor` class monitors a specific IO object and lets you introspect on
why that object was selected. These methods are not thread safe unless you are holding
the selector lock (i.e. if you're in a block passed to #select). Only use them
if you aren't concerned with thread safety, or you're within a #select
block:

- ***#interests***: what this monitor is interested in (:r, :w, or :rw)
- ***#interests=***: change the current interests for a monitor (to :r, :w, or :rw)
- ***#readiness***: what I/O operations the monitored object is ready for
- ***#readable?***: was the IO readable last time it was selected?
- ***#writable?***: was the IO writable last time it was selected?

Monitors also support a ***#value*** and ***#value=*** method for storing a
handle to an arbitrary object of your choice (e.g. a proc)

### Byte Buffers

*NOTE: This feature was added in nio4r 2.0 as a Google Summer of Code project
by Upekshe Jayasekera*

The `NIO::ByteBuffer` class represents a fixed-sized native buffer, and is
modeled after the corresponding Java NIO class. The closest Ruby equivalent
is a `StringIO`. However, unlike Ruby `String`/`StringIO` there are
no encoding or internal structure issues to worry about. Instead byte buffers
provide the most efficient means of performing I/O operations on in-memory data.

To create a byte buffer, construct it with a given capacity:

```ruby
>> buffer = NIO::ByteBuffer.new(16384)
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=0 @limit=16384 @capacity=16384>
```

All byte buffers have the following attributes:

- ***position***: a cursor from which all I/O operations will take place
- ***limit***: size of the current data in the buffer. Defaults to capacity
- ***capacity***: total size of the buffer

These values uphold a `position <= limit <= capacity` invariant.

To add data to the buffer directly, use the `<<` method:

```ruby
>> buffer << "Hello, world!" 
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=13 @limit=16384 @capacity=16384>
```

The intended use of a byte buffer is to first read some data into it, then
once we've done one or more reads to get the complete data, read the data
out of it. Before we read the data out, let's add some more data:

```ruby
>> buffer << " This is a byte buffer."
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=36 @limit=16384 @capacity=16384>
```

Now to begin reading data out, we'll use the `#flip` method. Pay attention to
`#flip` because it's the secret sauce in the API:

```ruby
>> buffer.flip
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=0 @limit=36 @capacity=16384>
```

Calling `#flip` changed the *limit* value to be the previous *position* cursor
value, and set *position* to be 0. Now we can read data out of the buffer by
using the `#get` method:

```ruby
>> buffer.get
 => "Hello, world! This is a byte buffer."
>> buffer
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=36 @limit=36 @capacity=16384>
```

Calling `#get` returned all of the data up to the limit as a string, and also
moved the *position* cursor to match the limit. We can also call `#get` with
a length:

```ruby
>> buffer.flip
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=0 @limit=36 @capacity=16384>
>> buffer.get(13)
 => "Hello, world!"
```

We can set the limit back to its original value using the `#limit=` method:

```ruby
>> buffer.flip
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=0 @limit=36 @capacity=16384>
>> buffer.limit = 16384
 => 16384
```

To perform I/O operations using the buffer, use the `#read_from` and
`#write_to` methods. These methods perform non-blocking I/O on the
remaining space in the buffer after the *position* cursor:

```ruby
>> buffer << "GET / HTTP/1.0\r\n\r\n"
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=18 @limit=16384 @capacity=16384>
>> buffer.flip
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=0 @limit=18 @capacity=16384>
>> socket = TCPSocket.new("github.com", 80)
 => #<TCPSocket:fd 11>
>> buffer.write_to(socket)
 => 18
>> buffer.clear
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=0 @limit=16384 @capacity=16384>
>> buffer.read_from(socket)
 => 93
>> buffer.flip
 => #<NIO::ByteBuffer:0x007fc60fa41528 @position=0 @limit=93 @capacity=16384>
>> buffer.get
 => "HTTP/1.1 301 Moved Permanently\r\nContent-length: 0\r\nLocation: https:///\r\nConnection: close\r\n\r\n"
```

Also note the `#clear` method, which returns a buffer to its original state.

### Flow Control

For information on how to compose nio4r selectors inside of event loops,
please read the [Flow Control Guide on the
Wiki](https://github.com/celluloid/nio4r/wiki/Basic-Flow-Control)

## Concurrency

**nio4r** provides internal locking to ensure that it's safe to use from multiple
concurrent threads. Only one thread can select on a NIO::Selector at a given
time, and while a thread is selecting other threads are blocked from
registering or deregistering IO objects. Once a pending select has completed,
requests to register/unregister IO objects will be processed.

NIO::Selector#wakeup allows one thread to unblock another thread that's in the
middle of an NIO::Selector#select operation. This lets other threads that need
to communicate immediately with the selector unblock it so it can process
other events that it's not presently selecting on.

## Non-goals

**nio4r** is not a full-featured event framework like [EventMachine] or [Cool.io].
Instead, nio4r is the sort of thing you might write a library like that on
top of. nio4r provides a minimal API such that individual Ruby implementers
may choose to produce optimized versions for their platform, without having
to maintain a large codebase.

[EventMachine]: https://github.com/eventmachine/eventmachine
[Cool.io]: https://coolio.github.io/

## License

Copyright (c) 2011-2016 Tony Arcieri. Distributed under the MIT License.
See LICENSE.txt for further details.

Includes libev 4.23. Copyright (c) 2007-2016 Marc Alexander Lehmann.
Distributed under the BSD license. See ext/libev/LICENSE for details.
