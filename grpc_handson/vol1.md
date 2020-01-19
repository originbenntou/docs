# gRPCハンズオン #1

## 目的

- gRCPが何か概要をざっくり理解する
- echoプログラムを実装して動きを理解する

## 概要

- さまざまなOSで動作し複数の言語で扱えるRPC（遠隔手続き呼び出し）フレームワーク
  - さまざまな言語のクライアントアプリケーションからインターフェースを介してサーバーアプリケーションを呼び出すことができる
    - Lambda

## 特徴

- HTTP/2+バイナリプロトコルによる高速通信
  - RESTはHTTP/1.1プロトコル
    - HTTP/2は並列ストリーミング通信
    - HTTP/1.1は通信の並び順を担保
    - 詳しいことは https://milestone-of-se.nesuke.com/l7protocol/http/http3-over-quic/
- インターフェースを介した強い型付けとAPI定義・設計ができる
  - OpenAPIはスキーマレベルな型付け
  - GraphQLはひとつのエンドポイントに対して様々なクエリーを送信してデータ照会できる点で柔軟

## 利用シーン

- 様々な言語での実装を受け入れる
  - gRPCはバックエンド間でしか通信できない
    - フロントから呼び出すにはgRPC-Gatewayが必要
      - Lambdaもフロントから呼び出すときはAPIGatewayが必要
    - プロジェクトではフロントからバックエンドAPIを呼び出すのにはRESTを利用
- インターフェース定義ファイルにより仕様書・設計書が不要

## 実装

### ■セッティング

#### Goインストール

```
$ apt update
$ apt install golang
$ go version

# GOPATH設定
## お好みで
$ mkdir ~/go
$ export $GOPATH ~/go
$ export PATH=$PATH:$GOPATH/bin
```

go versionが1.6とかだったら以下
https://github.com/golang/go/wiki/Ubuntu
https://kazuhira-r.hatenablog.com/entry/20160116/1452933387

#### Go Module

- 依存モジュール管理ツール
- phpでいうcomposer
- 昔はdepとかglideとかあったけど古い
- Go1.12からvgoが導入された
  - つまりGo Moduleはもう古い←

```
$ export GO111MODULE=on
```

#### gRPCインストール

```
$ go get -u google.golang.org/grpc

# 証明書エラーがでたら
$ apt-get install -y ca-certificates
```

#### protocolbufferインストール

```
$ curl -OL https://github.com/google/protobuf/releases/download/v3.9.0/protoc-3.9.0-linux-x86_64.zip
$ mv protoc3/bin/* /usr/local/bin/
$ mv protoc3/include/* /usr/local/include/
$ protoc --version
$ rm -rf protoc-3.9.0-linux-x86_64.zip protoc3

$ go get -u github.com/golang/protobuf/protoc-gen-go
```

### サンプルプログラム

#### 準備

```
$ mkdir $GOPATH/src/grpc_go_sample
$ cd $GOPATH/src/grpc_go_sample

$ go mod init

$ mkdir -p echo/proto/
$ mkdir -p echo/client/
$ mkdir -p echo/server/
```

#### protoファイル作成

```
root@c8e09f6e3de4:/go/src/grpc_go_sample# vim echo/proto/echo.proto
```

```
syntax = "proto3";

package echo;

message EchoRequest {
    string message = 1;
}

message EchoResponse {
    string message = 1;
}

service EchoService {
    rpc Echo (EchoRequest)
    returns (EchoResponse);
}
```

#### go用のインターフェース生成

- `I`はprotoファイルが存在するディレクトリを指定
- `go_out`はgrpcプラグインで`./echo/proto`配下に生成
- 最後にファイル名を指定

```
$ protoc -I ./echo/proto --go_out=plugins=grpc:./echo/proto/ ./echo/proto/echo.proto
```

```
$ ls -la echo/proto/
```

#### サーバー実装

```
$ vim ./echo/server/service.go
```

- 先程生成したprotocolbufferをimport
- インターフェースの処理部分を実装
  - echoService構造体がレシーバ
  - `EchoRequest`を受け取って`EchoResponse`を返却
  - contextはheaderやbodyの情報が入る
  - ポインタとか多値返却とかはGoの仕様

```
package main

import (
  "context"

  pb "grpc_go_sample/echo/proto"
)

type echoService struct{}

func (s *echoService) Echo(ctx context.Context,
  req *pb.EchoRequest) (*pb.EchoResponse, error) {
  return &pb.EchoResponse{
    Message: req.GetMessage()}, nil
}
```

```
$ vim ./echo/server/main.go
```

- `grpc.NewServer()`でgrpcサーバーを登録
  - 引数でタイムアウト等のオプション設定可能
- `pb.RegisterEchoServiceServer(srv,&echoService{})`で先ほど実装したサーバー処理をサービスとして登録
- `srv.Serve(lis)`でサーバー起動

```
package main

import (
  "log"
  "net"

  pb "grpc_go_sample/echo/proto"
  "google.golang.org/grpc"
)

func init() {
  log.SetFlags(0)
  log.SetPrefix("[echo] ")
}

func main() {
  port := ":50051"
  lis, err := net.Listen("tcp", port)
  if err != nil {
    log.Fatalf("failed to listen: %v\n", err)
  }
  srv := grpc.NewServer()
  pb.RegisterEchoServiceServer(srv,
    &echoService{})
  log.Printf("start server on port%s\n", port)
  if err := srv.Serve(lis); err != nil {
    log.Printf("failed to serve: %v\n", err)
  }
}
```

#### クライアントスタブ実装

```
$ vim ./echo/client/main.go
```

- `grpc.Dial`でサーバーに接続
- `NewEchoServiceClient`でgRPCクライアント生成
  - インターフェース上のサービスを呼び出せる
- `Echo`で実行

```
package main

import (
  "context"
  "log"
  "os"
  "time"

  pb "grpc_go_sample/echo/proto"
  "google.golang.org/grpc"
)

func init() {
  log.SetFlags(0)
  log.SetPrefix("[echo] ")
}

func main() {
  target := "localhost:50051"
  conn, err := grpc.Dial(target, grpc.WithInsecure())
  if err != nil {
    log.Fatalf("did not connect: %s", err)
  }
  defer conn.Close()
  client := pb.NewEchoServiceClient(conn)
  msg := os.Args[1]
  ctx, cancel := context.WithTimeout(context.Background(),
    time.Second)
  defer cancel()
  r, err := client.Echo(ctx,
    &pb.EchoRequest{Message: msg})
  if err != nil {
    log.Println(err)
  }
  log.Println(r.GetMessage())
}
```

#### 実行

```
$ go run ./echo/server
$ go run ./echo/client hello
```

という内容が以下に記載されてある
https://github.com/vvatanabe/go-grpc-basics/tree/master/echo


