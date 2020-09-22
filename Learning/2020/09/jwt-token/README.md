---
title: 'JWT token'
parent: 學習心得
---

# Protobuf and GPRC
**Protobuf 筆記**

Protobuf是Protocol Buffers的簡稱，用於描述一種輕便高效的結構化數據存儲格式，簡單說就是能以一份文件定義格式，然後能方便直接轉成各類語言的資料型態，言外之意，只要Protobuf有支援的語言，就能透過這個協定處理client和server之間的資料傳遞。

書寫Protobuf的重點:
* `syntax = "proto3"`:代表使用的協定版本
* `service serveice_name`:定義伺服器的名稱，裡面也定義相關的API
* rpc API_name (Input_message) returns (Return_message)`:API定義
* `message Message_name`:定義API需要的輸入或輸出格式
* `string greeting = 1`:後面的數字必須唯一，且從1開始，可以想像成一個不能改變的key，一旦編碼後就不能修改這個數字。
* 當protobuf寫好後，依照你的語言找對應的CMD來編譯他，會自動生城鄉貴硬的檔案給client和service參考使用，這也是他們對街參考的標準

* [詳細參數和特殊符號的使用](https://colobu.com/2015/01/07/Protobuf-language-guide/)
* [詳細參數和特殊符號的使用_英文](https://developers.google.com/protocol-buffers/docs/proto3)


**GPRC筆記**
當完Protobuf之後，就可以寫client code和service code，在這裡會依照語言不同寫法略微不同，但大致方向一樣的，以下用python為範例。

前情提要:python編譯完Protobuf後會產生XXX_pb2.py和XXX_pb2_grpc.py

Service:實現Protobuf定義的API，和啟動service
1. Import
    * `import grpc`:引入grpc，在啟動伺服器時使用
    * `from concurrent import futures`:相關引用
    * `import XXX_pb2_grpc, XXX_pb2`:將上述生成的檔案引入
2. API
    * `def API_name(self, request, context)`:對應API的實現，依照不同語言寫法不同，request就是當初API定義的輸入，用法和用物件一樣Object.variable
    * `return XXX_pb2.Return_message(...)`:這裡會依照語法有所不同
3. 啟動伺服器:
    * `grpcServer = grpc.server(futures.ThreadPoolExecutor(max_workers=10))`:確定max_worker
    * `XXX_pb2_grpc.[grpc裡面對應service function](model(), grpcServer)`:
    * `grpcServer.add_insecure_port(HOST + ':' + PORT)`:設定IP和port
    * `grpcServer.start()`:啟動service

Client:使用一樣的Protobuf設定
1. Import
    * `import grpc`:引入grpc，在啟動伺服器時使用
    * `import XXX_pb2_grpc, XXX_pb2`:將上述生成的檔案引入
2. 和伺服器取資料
    * `conn = grpc.insecure_channel(_HOST + ':' + _PORT)`:設定要連到哪個service
    * `client = XXX_pb2_grpc.[grpc裡面對應client function](channel=conn)`:
    * `client.API_name(XXX_pb2.Message_name(...))`:使用AP取得資料等

* [Python範例1](https://www.itread01.com/content/1549577363.html)
* [Python範例2](https://zwindr.blogspot.com/2019/08/go-python-grpc.html)
* [Golang範例1](https://ithelp.ithome.com.tw/articles/10207405)
* [Golang範例2](https://medium.com/hobo-engineer/%E7%AD%86%E8%A8%98-%E5%AF%A6%E4%BD%9C%E5%88%86%E6%95%A3%E5%BC%8F%E8%A8%88%E5%88%86%E7%B3%BB%E7%B5%B1-%E4%B8%89-grpc-and-protobuf-f8edd3d1e981)
* [Golang範例3](https://myapollo.com.tw/zh-tw/golang-grpc-tutorial-part-1/?fbclid=IwAR2IRRv5p_swYwJaMgZ6eyXPC8l0JPVT5mX5R4Oea-OL7FHH_XTJS68_BGs)
* [Java範例](https://blog.csdn.net/Axela30W/article/details/84569498)
    
**Token筆記**
Token驗證:
在客戶端可以使用GPRC內建提供的方法PerRPCCredentials，裡面有兩個function要實作
```go
type PerRPCCredentials interface {
    GetRequestMetadata(ctx context.Context, uri ...string) 
        (map[string]string, error)
    RequireTransportSecurity() bool
}
```
* GetRequestMetadata: 放入client端的相關資訊(例如:token, account and password)
```go
type tokenAuth struct {
    token string
}

func (t *tokenAuth) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    return map[string]string{
        "authorization": "Bearer " + t.token,
    }, nil
}
```
* equireTransportSecurity: 是否使用 TLS 安全加密
```go
func (t *tokenAuth) RequireTransportSecurity() bool {
    return true
}
```
Client server:
將上述相關的function加入送往service的資料中
```go
opts := []grpc.DialOption{
    grpc.WithTransportCredentials(creds),
    grpc.WithPerRPCCredentials(&tokenAuth{
        token: "some-secret-token",
    }),
}

// Set up a connection to the server.
// To call service methods, we first need to create a gRPC channel to communicate with the server. We create this by passing the server address and port number to grpc.Dial()
conn, err := grpc.Dial(*addr, opts...) 
if err != nil {
    log.Fatalf("did not connect: %v", err)
}
defer conn.Close()

// Once the gRPC channel is setup, we need a client stub to perform RPCs. We get this using the NewEchoClient method provided in the pb package we generated from our .proto.
c := pb.NewEchoClient(conn) 

// Contact the server and print out its response.
msg := "madmalls.com"
// Now let’s look at how we call our service methods. Note that in gRPC-Go, RPCs operate in a blocking/synchronous mode, which means that the RPC call waits for the server to respond, and will either return a response or an error.
resp, err := c.UnaryEcho(context.Background(), &pb.EchoRequest{Message: msg}) 
if err != nil {
    log.Fatalf("failed to call UnaryEcho: %v", err)
}
fmt.Printf("response:\n")
fmt.Printf(" - %q\n", resp.GetMessage())
```
Service:
* 用FromIncomingContext來皆來自client端傳來的資料
* token 就保存在 metadata.MD 中

```go
func FromIncomingContext(ctx context.Context) (md MD, ok bool) {
    ...
}

// MD is a mapping from metadata keys to values.
type MD map[string][]string
```
由FromIncomingContext取出的變數就可以使用了，使用方式如下
```go
func (s *server) UnaryEcho(ctx context.Context, req *pb.EchoRequest) (*pb.EchoResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    
    md["token"]...
}
```
* [Token範例1](https://madmalls.com/blog/post/grpc-authentication/)
* [Token範例2](https://juejin.im/post/6844903982393982989)
* [Token範例3](https://chai2010.cn/advanced-go-programming-book/ch4-rpc/ch4-05-grpc-hack.html)
* [英文文件](https://grpc.io/docs/guides/auth/)
* [metadata](http://ralphbupt.github.io/2017/05/27/gRPC%E4%B9%8Bmetadata/)