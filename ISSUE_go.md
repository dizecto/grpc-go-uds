# It seems like frames are getting corrupted in HTTP/2 when using a Unix domain socket on Windows

### What version of Go are you using (`go version`)?
```
go version go1.21.8 windows/amd64
```

### Output of `go env` in your module/workspace
```
set GO111MODULE=
set GOARCH=amd64
set GOBIN=C:\Users\dze\go\bin
set GOCACHE=C:\Users\dze\AppData\Local\go-build
set GOENV=C:\Users\dze\AppData\Roaming\go\env  
set GOEXE=.exe
set GOEXPERIMENT=
set GOFLAGS=
set GOHOSTARCH=amd64
set GOHOSTOS=windows
set GOINSECURE=git.ncsoft.net/*
set GOMODCACHE=C:\Users\dze\go\pkg\mod
set GONOPROXY=git.ncsoft.net/*
set GONOSUMDB=git.ncsoft.net/*
set GOOS=windows
set GOPATH=C:\Users\dze\go
set GOPRIVATE=git.ncsoft.net/*
set GOPROXY=https://proxy.golang.org,direct
set GOROOT=C:\Program Files\Go
set GOSUMDB=sum.golang.org
set GOTMPDIR=
set GOTOOLCHAIN=auto
set GOTOOLDIR=C:\Program Files\Go\pkg\tool\windows_amd64
set GOVCS=
set GOVERSION=go1.21.8
set GCCGO=gccgo
set GOAMD64=v1
set AR=ar
set CC=gcc
set CXX=g++
set CGO_ENABLED=0
set GOMOD=D:\github.com\grpc-go\examples\go.mod
set GOWORK=
set CGO_CFLAGS=-O2 -g
set CGO_CPPFLAGS=
set CGO_CXXFLAGS=-O2 -g
set CGO_FFLAGS=-O2 -g
set CGO_LDFLAGS=-O2 -g
set PKG_CONFIG=pkg-config
set GOGCCFLAGS=-m64 -fno-caret-diagnostics -Qunused-arguments -Wl,--no-gc-sections -fmessage-length=0 -ffile-prefix-map=C:\Users\dze\AppData\Local\Temp\go-build297662695=/tmp/go-build -gno-record-gcc-switches
```

### What operating system (Linux, Windows, …) and version?

Windows 10 Enterprise 22H2 19045.4046


## What did you do?

I initially observed these errors in Unix domain socket connection between daprd and its pluggable component. After modifying the RouteChat code in grpc-go/examples/route_guide, I was able to reproduce them. 
The issue is reproducible on other machines as well, and it occurs consistently across different versions like go1.19 and go1.22. Additionally, it persists on Windows 11.

## What I did

The changes to server.go and client.go can be found at [this repository](https://github.com/dizecto/grpc-go-uds-test).

### Use Unix domain socket
```
    // server.go
    os.Remove("D://temp/test.sock")
    lis, err := net.Listen("unix", "D://temp/test.sock")

    // client.go
    serverAddr = flag.String("addr", "unix:///temp/test.sock", "The server address in the format of host:port")
```

### Include a dummy string in the Message for sizing purposes
```
    // client.go
    {Location: &pb.Point{Latitude: 0, Longitude: 1}, Message: "First message: 1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890"}
```

### Continuously send requests without concurrency and log status
```
    // client.go
    for {
        for _, note := range notes {
            if err := stream.Send(note); err != nil {
                log.Fatalf("client.RouteChat: stream.Send(%v) failed: %v, outstanding: %d", note, err, outstanding)
            }

            atomic.AddInt64(&sent, 1)
            // current := atomic.AddInt64(&outstanding, 1)
            atomic.AddInt64(&outstanding, 1)
            if true {
                if sent%10000 == 1 {
                    log.Printf("client.RouteChat: a lot of outstandings:  %4d, sent: %d, recv: %d", outstanding, sent, recv)
                }
                // time.Sleep(time.Microsecond)
                if sent%10000 == 1 {
                    log.Printf("client.RouteChat: decreased outstandings: %4d, sent: %d, recv: %d", outstanding, sent, recv)
                }
            }

            //<-recvc
        }
    }

```

### Respond to received requests as they are
```
// server.go
func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        if err := stream.Send(in); err != nil {
            return err
        }
    }
}
```

### The line where the error occurred during debugging

D:\github.com\grpc-go\examples\vendor\golang.org\x\net\http2\frame.go
```
func (fr *Framer) ReadFrame() (Frame, error) {
	fr.errDetail = nil
	if fr.lastFrame != nil {
		fr.lastFrame.invalidate()
	}
	fh, err := readFrameHeader(fr.headerBuf[:], fr.r)
	if err != nil {
		return nil, err
	}
	if fh.Length > fr.maxReadSize {
		return nil, ErrFrameTooLarge    // <-----
	}
    ...
```
or
D:\github.com\grpc-go\examples\vendor\google.golang.org\grpc\internal\transport\http2_client.go
```
func (t *http2Client) reader(errCh chan<- error) {
	defer close(t.readerDone)

	if err := t.readServerPreface(); err != nil {
		errCh <- err
		return
	}
	close(errCh)
	if t.keepaliveEnabled {
		atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
	}

	// loop to keep reading incoming messages on this transport.
	for {
		t.controlBuf.throttle()
		frame, err := t.framer.fr.ReadFrame()
		if t.keepaliveEnabled {
			atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
		}
		if err != nil {
			// Abort an active stream if the http2.Framer returns a
			// http2.StreamError. This can happen only if the server's response
			// is malformed http2.
			if se, ok := err.(http2.StreamError); ok {
				t.mu.Lock()
				s := t.activeStreams[se.StreamID]
				t.mu.Unlock()
				if s != nil {
					// use error detail to provide better err message
					code := http2ErrConvTab[se.Code]
					errorDetail := t.framer.fr.ErrorDetail()
					var msg string
					if errorDetail != nil {
						msg = errorDetail.Error()
					} else {
						msg = "received invalid frame"
					}
					t.closeStream(s, status.Error(code, msg), true, http2.ErrCodeProtocol, status.New(code, msg), nil, false)
				}
				continue
			} else {
				// Transport error.
				t.Close(connectionErrorf(true, err, "error reading from server: %v", err))
				return
			}
		}
		switch frame := frame.(type) {
		case *http2.MetaHeadersFrame:
			t.operateHeaders(frame)
		case *http2.DataFrame:
			t.handleData(frame)
		case *http2.RSTStreamFrame:
			t.handleRSTStream(frame)
		case *http2.SettingsFrame:
			t.handleSettings(frame, false)
		case *http2.PingFrame:
			t.handlePing(frame)
		case *http2.GoAwayFrame:
			t.handleGoAway(frame)
		case *http2.WindowUpdateFrame:
			t.handleWindowUpdate(frame)
		default:
			if logger.V(logLevel) {
				logger.Errorf("transport: http2Client.reader got unhandled frame type %v.", frame) // <-----
			}
		}
	}
}
```

## How to reproduce

### Install go and set up environment variables
1. Install go1.19.13.windows-amd64.msi
2. Set the following environment variables.
```
GRPC_GO_LOG_SEVERITY_LEVEL: info
GRPC_GO_LOG_VERBOSITY_LEVEL: 99
```

And if 'GODEBUG: http2debug=2' is present, delete it. When printing this debug log, no errors were observed to occur.

### Clone the repositories and compile for Unix domain socket
```
> mkdir github.com
> cd github.com
github.com> git clone --depth 1 --branch v1.62.1 https://github.com/grpc/grpc-go.git
github.com> git clone https://github.com/dizecto/grpc-go-uds-test.git
github.com> copy /Y .\grpc-go-uds-test\route_guide\unix\client\client.go .\grpc-go\examples\route_guide\client\client.go
github.com> copy /Y .\grpc-go-uds-test\route_guide\unix\server\server.go .\grpc-go\examples\route_guide\server\server.go
github.com> cd grpc-go\exmaples
github.com\grpc-go\exmaples> go mod vendor
github.com\grpc-go\exmaples> go build -mod vendor route_guide\server\server.go
github.com\grpc-go\exmaples> go build -mod vendor route_guide\client\client.go
```

### Execute 'server.exe' and 'client.exe' in separate command windows.
1. Create a "temp" directory in the D drive
```
github.com\grpc-go\exmaples> mkdir D:\temp
```
2. Launch 'server.exe'
```
github.com\grpc-go\exmaples> server.exe
2024/03/14 10:13:40 INFO: [core] [Server #1] Server created
2024/03/14 10:13:41 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created

```
3. Launch 'client.exe'
```
github.com\grpc-go\exmaples> client.exe
2024/03/14 10:14:28 INFO: [core] [Channel #1] Channel created
2024/03/14 10:14:28 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 10:14:28 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}

```
4. Wait for errors

Be prepared to wait for errors; they may occur after several minutes or even tens of minutes.



### What did you expect to see?
Continue running as if using a TCP socket connection.

When using TCP connection, there are no code differences besides the address specified in 'server.go' and 'client.go'. They can be found at [this repository](https://github.com/dizecto/grpc-go-uds-test.git).


## What did you see instead?

## Try 1
Error occurred within 15 seconds.

## server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 10:13:40 INFO: [core] [Server #1] Server created
2024/03/14 10:13:41 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 10:14:28 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 10:14:43 INFO: [transport] [server-transport 0xc0000841a0] Closing: http2: frame too large
2024/03/14 10:14:43 INFO: [transport] [server-transport 0xc0000841a0] loopyWriter exiting with error: transport closed by client
```
### client.exe logs
```
D:\github.com\grpc-go\examples>client.exe
2024/03/14 10:14:28 INFO: [core] [Channel #1] Channel created
2024/03/14 10:14:28 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 10:14:28 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}
...
read unix @->/temp/test.sock: wsarecv: An existing connection was forcibly closed by the remote host."
2024/03/14 10:14:43 INFO: [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE
2024/03/14 10:14:43 INFO: [transport] [client-transport 0xc0000e6000] loopyWriter exiting with error: transport closed by client
2024/03/14 10:14:43 client.RouteChat failed: rpc error: code = Unavailable desc = error reading from server: read unix @->/temp/test.sock: wsarecv: An existing connection was forcibly closed by the remote host.
```

## Try 2
Error occurred within 1 seconds.

### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 11:18:32 INFO: [core] [Server #1] Server created
2024/03/14 11:18:33 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 11:18:38 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 11:18:39 INFO: [transport] [server-transport 0xc0001ae000] Closing: connection error: PROTOCOL_ERROR
2024/03/14 11:18:39 INFO: [transport] [server-transport 0xc0001ae000] loopyWriter exiting with error: transport closed by client
```
### client.exe logs
```
D:\github.com\grpc-go\examples>client.exe
2024/03/14 11:18:38 INFO: [core] [Channel #1] Channel created
2024/03/14 11:18:38 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 11:18:38 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}
...
2024/03/14 11:18:39 client.RouteChat: decreased outstandings:  115, sent: 430001, recv: 429886
2024/03/14 11:18:39 INFO: [transport] [client-transport 0xc00012c480] Closing: connection error: desc = "error reading from server: read unix @->/temp/test.sock: wsarecv: An existing connection was forcibly closed by the remote host."
2024/03/14 11:18:39 INFO: [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE
2024/03/14 11:18:39 INFO: [transport] [client-transport 0xc00012c480] loopyWriter exiting with error: transport closed by client
2024/03/14 11:18:39 client.RouteChat: stream.Send(location:{longitude:1} message:"Fourth message: 1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890") failed: EOF, outstanding: 182
```

## Try 3
Error occurred within 1 minute and 16 seconds.

### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 16:04:31 INFO: [core] [Server #1] Server created
2024/03/14 16:04:31 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 16:04:34 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 16:05:50 INFO: [transport] [server-transport 0xc0001ae000] Closing: http2: frame too large
2024/03/14 16:05:50 INFO: [transport] [server-transport 0xc0001ae000] loopyWriter exiting with error: connection error: desc = "transport is closing"
```
### client.exe logs
```
D:\github.com\grpc-go\examples>client.exe
2024/03/14 16:04:34 INFO: [core] [Channel #1] Channel created
2024/03/14 16:04:34 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 16:04:34 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}
...
2024/03/14 16:05:50 client.RouteChat: decreased outstandings:  126, sent: 19350001, recv: 19349875
2024/03/14 16:05:50 INFO: [transport] [client-transport 0xc0000d2000] Closing: connection error: desc = "error reading from server: read unix @->/temp/test.sock: wsarecv: An existing connection was forcibly closed by the remote host."
2024/03/14 16:05:50 INFO: [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE
2024/03/14 16:05:50 INFO: [transport] [client-transport 0xc0000d2000] loopyWriter exiting with error: transport closed by client
2024/03/14 16:05:50 INFO: [core] [pick-first-lb 0xc00019be60] Received SubConn state update: 0xc00019bf80, {ConnectivityState:IDLE ConnectionError:<nil>}
2024/03/14 16:05:50 INFO: [core] [Channel #1] Channel Connectivity change to IDLE
2024/03/14 16:05:50 client.RouteChat failed: rpc error: code = Unavailable desc = error reading from server: read unix @->/temp/test.sock: wsarecv: An existing connection was forcibly closed by the remote host.
```

## Try 4 -  seconds later
Error occurred within 2 minutes and 27 seconds.

### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 16:09:59 INFO: [core] [Server #1] Server created
2024/03/14 16:09:59 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 16:10:02 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 16:12:29 INFO: [transport] [server-transport 0xc000084340] Closing: read unix D://temp/test.sock->@: wsarecv: An existing connection was forcibly closed by the remote host.
2024/03/14 16:12:29 INFO: [transport] [server-transport 0xc000084340] loopyWriter exiting with error: transport closed by client
```
### client.exe logs
```
D:\github.com\grpc-go\examples>client.exe
2024/03/14 16:10:02 INFO: [core] [Channel #1] Channel created
2024/03/14 16:10:02 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 16:10:02 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}
...
2024/03/14 16:12:29 client.RouteChat: decreased outstandings:  136, sent: 37900001, recv: 37899865
2024/03/14 16:12:29 INFO: [transport] [client-transport 0xc0000c1200] Closing: connection error: desc = "error reading from server: http2: frame too large"
2024/03/14 16:12:29 INFO: [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE
2024/03/14 16:12:29 INFO: [transport] [client-transport 0xc0000c1200] loopyWriter exiting with error: transport closed by client
2024/03/14 16:12:29 INFO: [core] [pick-first-lb 0xc00021a030] Received SubConn state update: 0xc00021a180, {ConnectivityState:IDLE ConnectionError:<nil>}
2024/03/14 16:12:29 client.RouteChat: stream.Send(location:{longitude:3} message:"Third message: 1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890") failed: EOF, outstanding: 181
```

## Try 5
Error occurred within 2 minutes and 3 seconds.

### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 16:40:26 INFO: [core] [Server #1] Server created
2024/03/14 16:40:26 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 16:40:30 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 16:42:33 INFO: [transport] [server-transport 0xc0001ae000] Closing: http2: frame too large
2024/03/14 16:42:33 INFO: [transport] [server-transport 0xc0001ae000] loopyWriter exiting with error: connection error: desc = "transport is closing"
```
### client.exe logs
```
D:\github.com\grpc-go\examples>client.exe
2024/03/14 16:40:30 INFO: [core] [Channel #1] Channel created
2024/03/14 16:40:30 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 16:40:30 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}
...
2024/03/14 16:42:33 client.RouteChat: decreased outstandings:  137, sent: 31430001, recv: 31429864
2024/03/14 16:42:33 INFO: [transport] [client-transport 0xc000141680] Closing: connection error: desc = "error reading from server: read unix @->/temp/test.sock: wsarecv: An existing connection was forcibly closed by the remote host."
2024/03/14 16:42:33 INFO: [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE
2024/03/14 16:42:33 INFO: [transport] [client-transport 0xc000141680] loopyWriter exiting with error: transport closed by client
2024/03/14 16:42:33 client.RouteChat: stream.Send(location:{longitude:2} message:"Second message: 1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890") failed: EOF, outstanding: 169
```

## Try 6
Error occurred within 2 minutes and 47 seconds.
This was debugging mode.

### server.exe logs
```
Starting: C:\Users\dze\go\bin\dlv.exe dap --listen=127.0.0.1:63676 from d:\github.com\grpc-go\examples\route_guide\server
DAP server listening at: 127.0.0.1:63676
Type 'dlv help' for list of commands.
2024/03/18 13:47:27 INFO: [core] [Server #1] Server created
2024/03/18 13:47:27 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/18 13:47:46 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/18 13:50:33 INFO: [transport] [server-transport 0xc000057ba0] Closing: read unix D://temp/test.sock->@: wsarecv: An existing connection was forcibly closed by the remote host.
2024/03/18 13:50:33 INFO: [transport] [server-transport 0xc000057ba0] loopyWriter exiting with error: write unix D://temp/test.sock->@: wsasend: An existing connection was forcibly closed by the remote host.
```
### client.exe logs
```
Starting: C:\Users\dze\go\bin\dlv.exe dap --listen=127.0.0.1:63687 from d:\github.com\grpc-go\examples\route_guide\client
DAP server listening at: 127.0.0.1:63687
Type 'dlv help' for list of commands.
2024/03/18 13:47:46 INFO: [core] [Channel #1] Channel created
2024/03/18 13:47:46 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/18 13:47:46 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}
2024/03/18 13:47:46 INFO: [core] [Channel #1] Channel authority set to "localhost"
...
2024/03/18 13:50:33 client.RouteChat: decreased outstandings:  168, sent: 16110001, recv: 16109833
2024/03/18 13:50:33 ERROR: [transport] transport: http2Client.reader got unhandled frame type [FrameHeader UNKNOWN_FRAME_TYPE_215 len=2].
2024/03/18 13:50:33 ERROR: [transport] transport: http2Client.reader got unhandled frame type [FrameHeader UNKNOWN_FRAME_TYPE_210 flags=0x2|0x8 stream=34603794 len=2].
2024/03/18 13:50:33 INFO: [transport] [client-transport 0xc0000d6000] Closing: connection error: desc = "error reading from server: http2: frame too large"
2024/03/18 13:50:33 INFO: [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE
2024/03/18 13:50:33 INFO: [transport] [client-transport 0xc0000d6000] loopyWriter exiting with error: transport closed by client
2024/03/18 13:50:33 client.RouteChat: stream.Send(location:{longitude:3}  message:"Sixth message: 1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890") failed: EOF, outstanding: 162
Process 63280 has exited with status 1
Detaching
```