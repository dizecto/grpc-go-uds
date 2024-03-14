NOTE: if you are reporting is a potential security vulnerability or a crash,
please follow our CVE process at
https://github.com/grpc/proposal/blob/master/P4-grpc-cve-process.md instead of
filing an issue here.

Please see the FAQ in our main README.md, then answer the questions below
before submitting your issue.

# Weird errors like "frame too large" and "PROTOCOL_ERROR" occurrred for unix domain socket on Windows

### What version of gRPC are you using?
v1.62.1

### What version of Go are you using (`go version`)?
go version go1.19.13 windows/amd64

### What operating system (Linux, Windows, …) and version?
Windows 10 Enterprise 22H2 19045.4046

### What did you do?
I saw these errors at unix domain socket connection between daprd and its pluggable component at the first time. After I changed the RouteChat codes in grpc-go\examples\route_guide, they have been reproduced.

#### What I did

Changed server.go and client.go is at https://github.com/dizecto/grpc-go-uds-test.git.

Use unix domain socket
```
    // server.go
    os.Remove("D://temp/test.sock")
    lis, err := net.Listen("unix", "D://temp/test.sock")

    // client.go
    serverAddr = flag.String("addr", "unix:///temp/test.sock", "The server address in the format of host:port")
```

Add dummy data to increase the message size
```
    // client.go
    {Location: &pb.Point{Latitude: 0, Longitude: 1}, Message: "First message: 1234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890"}
```

Send a request infinitely not concurrently and add some logs
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

Response a received request as it is
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

#### How to reproduce

##### Install and add environment variables.
1. Install [go1.19.13.windows-amd64.msi](https://go.dev/dl/go1.19.13.windows-amd64.msi).
2. Add the following environment variables.
```
GRPC_GO_LOG_SEVERITY_LEVEL: info
GRPC_GO_LOG_VERBOSITY_LEVEL: 99
GODEBUG: http2debug=2
```

##### Clone the repositories and compile for unix domain socket
```
> mkdir github.com
> cd github.com
github.com> git clone --depth 1 --branch v1.62.1 https://github.com/grpc/grpc-go.git
github.com> git clone https://github.com/dizecto/grpc-go-uds-test.git
github.com> copy /Y .\grpc-go-uds-test\route_guide\unix\client\client.go .\grpc-go\examples\route_guide\client\client.go
github.com> copy /Y .\grpc-go-uds-test\route_guide\unix\server\server.go .\grpc-go\examples\route_guide\server\server.go
github.com> cd grpc-go\exmaples
github.com\grpc-go\exmaples> go mod vendor
github.com\grpc-go\exmaples> go clean -modcache
github.com\grpc-go\exmaples> go clean -cache
github.com\grpc-go\exmaples> go clean -testcache
github.com\grpc-go\exmaples> go build -mod vendor route_guide\server\server.go
github.com\grpc-go\exmaples> go build -mod vendor route_guide\client\client.go
```

##### Run server.exe and client.exe in the different cmd window
1. Make "temp" directory in D drive
```
github.com\grpc-go\exmaples> mkdir D:\temp
```
2. Run server.exe
```
github.com\grpc-go\exmaples> server.exe
2024/03/14 10:13:40 INFO: [core] [Server #1] Server created
2024/03/14 10:13:41 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created

```
3. Run client.exe
```
github.com\grpc-go\exmaples> client.exe
2024/03/14 10:14:28 INFO: [core] [Channel #1] Channel created
2024/03/14 10:14:28 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 10:14:28 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}

```

### What did you expect to see?
No errors and continue to run like tcp socket connection.

### What did you see instead?
Try 1

Server logs
```
github.com\grpc-go\examples>server.exe
2024/03/14 10:13:40 INFO: [core] [Server #1] Server created
2024/03/14 10:13:41 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 10:14:28 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 10:14:43 INFO: [transport] [server-transport 0xc0000841a0] Closing: http2: frame too large
2024/03/14 10:14:43 INFO: [transport] [server-transport 0xc0000841a0] loopyWriter exiting with error: transport closed by client
```
Client logs
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

Try 2

Server logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 11:18:32 INFO: [core] [Server #1] Server created
2024/03/14 11:18:33 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 11:18:38 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 11:18:39 INFO: [transport] [server-transport 0xc0001ae000] Closing: connection error: PROTOCOL_ERROR
2024/03/14 11:18:39 INFO: [transport] [server-transport 0xc0001ae000] loopyWriter exiting with error: transport closed by client
```
Client logs
```
github.com\grpc-go\examples>client.exe
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