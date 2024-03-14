# grpc-go-uds-test

## How to reproduce

### Install and add environment variables.
1. Install [go1.19.13.windows-amd64.msi](https://go.dev/dl/go1.19.13.windows-amd64.msi).
2. Add the following environment variables.
```
GRPC_GO_LOG_SEVERITY_LEVEL: info
GRPC_GO_LOG_VERBOSITY_LEVEL: 99
GODEBUG: http2debug=2
```

### Clone the repository and compile
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

### Run server.exe and client.exe in the different cmd window
1. Run server.exe
```
github.com\grpc-go\exmaples> server.exe
2024/03/14 10:13:40 INFO: [core] [Server #1] Server created
2024/03/14 10:13:41 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created

```
2. Run client.exe
```
github.com\grpc-go\exmaples> client.exe
2024/03/14 10:14:28 INFO: [core] [Channel #1] Channel created
2024/03/14 10:14:28 INFO: [core] [Channel #1] original dial target is: "unix:///temp/test.sock"
2024/03/14 10:14:28 INFO: [core] [Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"unix", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/temp/test.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}

```
