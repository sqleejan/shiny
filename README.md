# `✨ shiny ✨`

Shiny is an alternative networking framework for Go that uses I/O multiplexing.
It makes direct [epoll](https://en.wikipedia.org/wiki/Epoll) and [kqueue](https://en.wikipedia.org/wiki/Kqueue) syscalls rather than the standard Go [net](https://golang.org/pkg/net/) package.

This is similar to the way that [libuv](https://github.com/libuv/libuv), [libevent](https://github.com/libevent/libevent), [haproxy](http://www.haproxy.org/), [nginx](http://nginx.org/), [redis](http://redis.io/), and other high performance network servers work.

The reason for this project is that I want to upgrade the networking for [Tile38](http://github.com/tidwall/tile38) so that it will perform on par with Redis, but without having to interop with Cgo. Early benchmarks are exceeding my expectations.

**This project is a (sweet) work in progress. The API will likely change between now and Tile38 v2.0 release.**


## Features

- Simple API. Only one entrypoint and four event functions
- Low memory usage
- Very fast single-threaded support
- Support for non-epoll/kqueue operating systems by simulating events with the net package.


## Getting Started

### Installing

To start using Shiny, install Go and run `go get`:

```sh
$ go get -u github.com/tidwall/shiny
```

This will retrieve the library.

### Usage

There's only the one function:

```go
func Serve(net, addr string,
    handle func(id int, data []byte, ctx interface{}) (send []byte, keepopen bool),
    accept func(id int, addr string, wake func(), ctx interface{}) (send []byte, keepopen bool),
    closed func(id int, err error, ctx interface{}),
    ticker func(ctx interface{}) (keepserving bool),
    context interface{}) error
```

## Example

Please check out the [examples](examples) subdirectory for a simplified [redis](examples/redis-server.go) clone and an [echo](examples/echo-server.go) server.

Here's a basic echo server:

```go
package main

import (
	"flag"
	"fmt"
	"log"

	"github.com/tidwall/shiny"
)

var shutdown bool
var started bool
var port int

func main() {
	flag.IntVar(&port, "port", 9999, "server port")
	flag.Parse()
	log.Fatal(shiny.Serve("tcp", fmt.Sprintf(":%d", port),
		handle, accept, closed, ticker, nil))
}

// handle - the incoming client socket data.
func handle(id int, data []byte, ctx interface{}) (send []byte, keepopen bool) {
	if shutdown {
		return nil, false
	}
	keepopen = true
	if string(data) == "shutdown\r\n" {
		shutdown = true
	} else if string(data) == "quit\r\n" {
		keepopen = false
	}
	return data, keepopen
}

// accept - a new client socket has opened.
// 'wake' is a function that when called will fire a 'handle' event
// for the specified ID, and is goroutine-safe.
func accept(id int, addr string, wake func(), ctx interface{}) (send []byte, keepopen bool) {
	if shutdown {
		return nil, false
	}
	// this is a good place to create a user-defined socket context.
	return []byte(
		"Welcome to the echo server!\n" +
			"Enter 'quit' to close your connection or " +
			"'shutdown' to close the server.\n"), true
}

// closed - a client socket has closed
func closed(id int, err error, ctx interface{}) {
	// teardown the socket context here
}

// ticker - a ticker that fires between 1 and 1/20 of a second
// depending on the traffic.
func ticker(ctx interface{}) (keepserving bool) {
	if shutdown {
		// do server teardown here
		return false
	}
	if !started {
		fmt.Printf("echo server started on port %d\n", port)
		started = true
	}
	// perform various non-socket-io related operation here
	return true
}
```

Run the example:

```
$ go run examples/echo-server.go
```

Connect to the server:

```
$ telnet localhost 9999
```


## Performance

The benchmarks below use pipelining which allows for combining multiple Redis commands into a single packet.

**Redis**

```
$ redis-server --port 6379 --appendonly no
```
```
redis-benchmark -p 6379 -t ping,set,get -q -P 128
PING_INLINE: 961538.44 requests per second
PING_BULK: 1960784.38 requests per second
SET: 943396.25 requests per second
GET: 1369863.00 requests per second
```

**Shiny**

```
$ go run examples/redis-server.go --port 6380 --appendonly no
```
```
redis-benchmark -p 6380 -t ping,set,get -q -P 128
PING_INLINE: 3846153.75 requests per second
PING_BULK: 4166666.75 requests per second
SET: 3703703.50 requests per second
GET: 3846153.75 requests per second
```

*Running on a MacBook Pro 15" 2.8 GHz Intel Core i7 using Go 1.7*

## Contact

Josh Baker [@tidwall](http://twitter.com/tidwall)

## License

Shiny source code is available under the MIT [License](/LICENSE).
