# wasmws: Web-Assembly Websocket (for Go)

## What is wasmws? Why would I want to use this?

[wasmws](https://github.com/gbl08ma/wasmws) was written primarily to allow [Go](https://golang.org/) applications targeting [WASM](https://en.wikipedia.org/wiki/WebAssembly) to communicate with a [gRPC](https://grpc.io/) server. This is normally challenging for two reasons: 

1. In general, many internet ingress paths are not [HTTP/2](https://en.wikipedia.org/wiki/HTTP/2) end to end (gRPC uses HTTP/2). In particular, most CDN vendors do not support HTTP/2 back-haul to origin (ex. [Cloudflare](https://support.cloudflare.com/hc/en-us/articles/214534978-Are-the-HTTP-2-or-SPDY-protocols-supported-between-Cloudflare-and-the-origin-server-)).
2. Browser WASM applications cannot use [grpc-go](https://github.com/grpc/grpc-go) due to the low level networking that go-grpc requires for native operation not being available. (ex. ``dial tcp: Protocol not available`` fun...)

This library allows you to use a websocket as [net.Conn](https://golang.org/pkg/net/#Conn) for arbitrary traffic, this includes running protocols like HTTP, gRPC or any other TCP protocol over it). This is most useful for protocols that are not normally exposed to client side web applications. In our examples we will focus on gRPC since that was my use-case. 

### Aproach taken

#### Client-side (Go WASM application running in browser)
wasmws provides Go WASM specific [net.Conn](https://golang.org/pkg/net/#Conn) implementation that is backed by [a browser native websocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API):
```go
myConn, err := wasmws.Dial("websocket", "ws://demos.kaazing.com/echo")
```
It is fairly straight forward to use this package to set up a gRPC connection:
```go
conn, err := grpc.DialContext(dialCtx, "passthrough:///"+websocketURL, grpc.WithContextDialer(wasmws.GRPCDialer), grpc.WithTransportCredentials(creds))
```
See the [demo client](https://github.com/gbl08ma/wasmws/blob/master/demo/client/main.go) for an extended example.

#### Server-side
wasmws includes websocket [net.Listener](https://golang.org/pkg/net/#Listener) that provides a [HTTP handler method](https://golang.org/pkg/net/http/#HandlerFunc) to accept HTTP websocket connections...
```go
wsl := wasmws.NewWebSocketListener(appCtx)
router.HandleFunc("/grpc-proxy", wsl.HTTPAccept)
```
And a network listener to provide net.Conns to network oriented servers:
```go
err := grpcServer.Serve(wsl)
```
See the [demo server](https://github.com/gbl08ma/wasmws/blob/master/demo/server/main.go) for an extended example. If you need more server-side helpers checkout [nhooyr.io/websocket](https://github.com/nhooyr/websocket) which these helpers use themselves.

#### Security

If you use a secure websocket and gRPC or HTTPS this means you get double TLS (once using the browser's TLS stack and once again using Go's). Unless the extra defense in depth is desirable, you may want to consider using an unsecured websocket.

## Performance

Due to challenges around having the sever running as a native application and the client running in a browser coordinate, I have not yet added unit benches... :( yet. However, running the demo which performs 8192 gRPC hello world calls provides an idea of the library's performance:

Median of 6 runs:

 * 	FireFox 71.0 (64-bit) on Linux:
     * ``SUCCESS running 8192 transactions! (average 452.88µs per operation)``
 * 	Chrome Version 79.0.3945.88 (Official Build) (64-bit) on Linux:
     * ``SUCCESS running 8192 transactions! (average 475.485µs per operation)``

This implementation tries to be intelligent about managing buffers (via [pooling](https://golang.org/pkg/sync/#Pool)) and switches on the fly between JavaScript [ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) and streaming [Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob) based websocket read interfaces based the size of the chunks/messages being received. Web browsers which do not support [Blob stream](https://developer.mozilla.org/en-US/docs/Web/API/Blob/stream) and [Blob arrayBuffer](https://developer.mozilla.org/en-US/docs/Web/API/Blob/arrayBuffer) methods, such as Microsoft Edge, always use ArrayBuffer-based message consumption.

The test results above are from tests run on a local development workstation:

 * CPU: AMD Ryzen 9 3900X 12-Core Processor
 * OS: Ubuntu 18.04 LTS (Linux 4.15.X)

## Running the demo

If you do not have Go installed, you will of course need to [install it](https://golang.org/doc/install).

1. Checkout repo: ``git clone https://github.com/gbl08ma/wasmws.git``
2. Change to the demo directory: ``cd ./wasmws/demo/server``
3. Build the client, server and run the server (which serves the client): ``./build.bash && ./server``
4. Open [http://localhost:8080/](http://localhost:8080/) in your web browser
5. Open the web console (Ctrl+Shift+K in Firefox, Ctrl+Shift+I in Chrome) and observe the output!
		
## Alternatives

1. Use [gRPC-Web](https://github.com/grpc/grpc-web) as a HTTP to gRPC gateway/proxy. (If you don't mind a TCP connection per request, running extra middleware which are also extra points of failure...)
2. Use "[nhooyr.io/websocket](https://github.com/nhooyr/websocket)"'s implementation which unlike "wasmws" does not use the browser provided websocket functionality. Test and bench your own use-case!
		
## Future

wasmws is actively being maintained, but that does not mean there are not things to do:

* [Testing on browsers besides Firefox and Chrome](https://github.com/gbl08ma/wasmws/issues/5)
* [Further optimization/profiling](https://github.com/gbl08ma/wasmws/issues/4)
		
## Contributing

[Issues](https://github.com/gbl08ma/wasmws/issues), and especially issues with [pull requests](https://github.com/gbl08ma/wasmws/pulls) are welcome!

This code is licensed under [MPL 2.0](https://en.wikipedia.org/wiki/Mozilla_Public_License).
