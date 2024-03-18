NOTE: if you are reporting is a potential security vulnerability or a crash,
please follow our CVE process at
https://github.com/grpc/proposal/blob/master/P4-grpc-cve-process.md instead of
filing an issue here.

Please see the FAQ in our main README.md, then answer the questions below
before submitting your issue.

# Errors such as "frame too large" and "PROTOCOL_ERROR" occurred with Unix domain socket on Windows

### What version of gRPC are you using?
v1.62.1

### What version of Go are you using (`go version`)?
go version go1.19.13 windows/amd64

### What operating system (Linux, Windows, …) and version?
Windows 10 Enterprise 22H2 19045.4046


### What did you do?


### What did you expect to see?
Continue running as if using a TCP socket connection.

When using TCP connection, there are no code differences besides the address specified in 'server.go' and 'client.go'. They can be found at [this repository](https://github.com/dizecto/grpc-go-uds-test.git).

### What did you see instead?

#### Try 1
Error occurred within 15 seconds.

##### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 10:13:40 INFO: [core] [Server #1] Server created
2024/03/14 10:13:41 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 10:14:28 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 10:14:43 INFO: [transport] [server-transport 0xc0000841a0] Closing: http2: frame too large
2024/03/14 10:14:43 INFO: [transport] [server-transport 0xc0000841a0] loopyWriter exiting with error: transport closed by client
```
##### client.exe logs
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

#### Try 2
Error occurred within 1 seconds.

##### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 11:18:32 INFO: [core] [Server #1] Server created
2024/03/14 11:18:33 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 11:18:38 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 11:18:39 INFO: [transport] [server-transport 0xc0001ae000] Closing: connection error: PROTOCOL_ERROR
2024/03/14 11:18:39 INFO: [transport] [server-transport 0xc0001ae000] loopyWriter exiting with error: transport closed by client
```
##### client.exe logs
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

#### Try 3
Error occurred within 1 minute and 16 seconds.

##### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 16:04:31 INFO: [core] [Server #1] Server created
2024/03/14 16:04:31 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 16:04:34 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 16:05:50 INFO: [transport] [server-transport 0xc0001ae000] Closing: http2: frame too large
2024/03/14 16:05:50 INFO: [transport] [server-transport 0xc0001ae000] loopyWriter exiting with error: connection error: desc = "transport is closing"
```
##### client.exe logs
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

#### Try 4 -  seconds later
Error occurred within 2 minutes and 27 seconds.

##### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 16:09:59 INFO: [core] [Server #1] Server created
2024/03/14 16:09:59 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 16:10:02 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 16:12:29 INFO: [transport] [server-transport 0xc000084340] Closing: read unix D://temp/test.sock->@: wsarecv: An existing connection was forcibly closed by the remote host.
2024/03/14 16:12:29 INFO: [transport] [server-transport 0xc000084340] loopyWriter exiting with error: transport closed by client
```
##### client.exe logs
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

#### Try 5
Error occurred within 2 minutes and 3 seconds.

##### server.exe logs
```
D:\github.com\grpc-go\examples>server.exe
2024/03/14 16:40:26 INFO: [core] [Server #1] Server created
2024/03/14 16:40:26 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
2024/03/14 16:40:30 INFO: [core] CPU time info is unavailable on non-linux environments.
2024/03/14 16:42:33 INFO: [transport] [server-transport 0xc0001ae000] Closing: http2: frame too large
2024/03/14 16:42:33 INFO: [transport] [server-transport 0xc0001ae000] loopyWriter exiting with error: connection error: desc = "transport is closing"
```
##### client.exe logs
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