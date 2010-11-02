# redis_ext

Ruby extension that wraps [hiredis](http://github.com/antirez/hiredis) reply
parsing code. It is targeted at speeding up parsing multi bulk replies.

It is bundled with this specific branch of hiredis (until that branch is
merged to master):
[pietern/hiredis/buffer](http://github.com/pietern/hiredis/tree/buffer).

## WARNING

This code is considered alpha. Do not use it for other purposes than
testing _yet_.

## Install

Install with Rubygems:

    gem install redis_ext --pre

## Usage

This gem cannot be used with redis-rb out of the box _yet_.

### Connection

A connection to Redis can be opened by creating an instance of
`RedisExt::Connection` and calling `#connect`:

    conn = RedisExt::Connection.new
    conn.connect("127.0.0.1", 6379)

Commands can be written to Redis by calling `#write` with an array of
arguments. You can call write more than once, resulting in a pipeline of
commands.

    conn.write ["SET", "speed", "awesome"]
    conn.write ["GET", "speed"]

After commands are written, use `#read` to receive the subsequent replies.
Make sure **not** to call `#read` more than you have replies to read, or
the connection will block indefinitely. You _can_ use this feature
to implement a subscriber (for Redis Pub/Sub).

    > conn.read
    => "OK"
    > conn.read
    => "awesome"

When the connection was closed by the server, an error of the type
`RedisExt::Connection::EOFError` will be raised. For all I/O related errors,
the Ruby built-in `Errno::*` errors will be raised. All other errors
(such as a protocol error) result in a `RuntimeError`.

### Reply parser

Only using `redis_ext` for the reply parser can be very useful in scenarios
where the I/O is already handled by another component (such as EventMachine).

Use `#feed` on an instance of `RedisExt::Reader` to feed the stream parser with
new data. Use `#read` to get the parsed replies one by one:

    > reader = RedisExt::Reader.new
    > reader.feed("*2\r\n$7\r\nawesome\r\n$5\r\narray\r\n")
    > reader.gets
    => ["awesome", "array"]

## Benchmarks

These numbers were generated by running `benchmark/compare.rb` against
`ruby 1.8.7 (2010-08-16 patchlevel 302) [i686-darwin10.4.0]`. Because this
extension is only concerned with optimizing the code for parsing the reply,
parse code, this is the only thing that is benchmarked here. This does **not**
take I/O in account.

For simple line or bulk replies, the throughput improvement is insignificant,
while the larger multi bulk responses will have a noticeable higher throughput.

                                                            user     system      total        real
    ruby:100000x bulk (10 bytes)                        0.390000   0.000000   0.390000 (  0.387764)
     ext:100000x bulk (10 bytes)                        0.090000   0.000000   0.090000 (  0.090202)
    ruby:100000x bulk (100 bytes)                       0.400000   0.000000   0.400000 (  0.399499)
     ext:100000x bulk (100 bytes)                       0.090000   0.010000   0.100000 (  0.096638)
    ruby:100000x bulk (1000 bytes)                      0.430000   0.020000   0.450000 (  0.455056)
     ext:100000x bulk (1000 bytes)                      0.130000   0.030000   0.160000 (  0.158906)
    ruby:10000x multi bulk (10 items x 10 bytes)        0.420000   0.000000   0.420000 (  0.424638)
     ext:10000x multi bulk (10 items x 10 bytes)        0.030000   0.000000   0.030000 (  0.033699)
    ruby:10000x multi bulk (100 items x 10 bytes)       3.850000   0.010000   3.860000 (  3.855463)
     ext:10000x multi bulk (100 items x 10 bytes)       0.260000   0.000000   0.260000 (  0.265152)
    ruby:10000x multi bulk (1000 items x 10 bytes)     39.100000   0.060000  39.160000 ( 39.188940)
     ext:10000x multi bulk (1000 items x 10 bytes)      2.600000   0.000000   2.600000 (  2.607604)

## License

This code is released under the BSD license, after the license of hiredis.
