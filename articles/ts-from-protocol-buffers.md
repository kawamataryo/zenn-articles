---
title: "Protocol Buffers から TypeScript の型定義を作る"
emoji: "🥞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "grpc", "ProtocolBuffers"]
published: true
---

gRPC の Protocol Buffers から TypeScript の型定義を作る方法のメモです。

# 前提

## Protocol Buffers とは？

グーグルによって開発されたインターフィス定義言語で、gRPC 利用時に、クライアント／サーバーコードの生成に使われます。

以下のように message と service でインタフェースを定義します。

```protobuf
syntax = "proto3";

package Posts;

import "google/protobuf/empty.proto";

message Post {
  int32 id = 1;
  string title = 2;
  string content = 3;
}

service Posts {
  rpc GetPosts(google.protobuf.Empty) returns (GetPostsResponse) {};
}
```

# Protocol Buffers からの TypeScriptの型定義生成
[ts-protoc-gen](https://github.com/improbable-eng/ts-protoc-gen)を使って、クライアント／サーバーコード生成時に TypeScript の型定義も合わせて生成する手順を記します。

https://github.com/improbable-eng/ts-protoc-gen

## 1. プロジェクトの作成・依存ライブラリのインストール

npm プロジェクトを作成し、依存ライブラリをインストールします。
追加するライブラリは以下のとおりです。

- [grpc-tools](https://github.com/grpc/grpc-node/tree/master/packages/grpc-tools): Protocol Buffers のコンパイラ。クライアント／サーバーコードの生成を行う
- [@grpc/grpc-js](https://github.com/grpc/grpc-node/tree/master/packages/grpc-js): Node.js 製の gRPC クライアント。生成されたクライアント／サーバーコードで利用される
- [google-protobuf](https://github.com/protocolbuffers/protobuf/tree/master/js): Protocol Buffers のランタイムライブラリ。生成されたクライアント／サーバーコードで利用される
- [ts-protoc-gen](https://github.com/improbable-eng/ts-protoc-gen): Proctocol Buffers から TypeScript の型定義を生成するプラグイン

``` bash
# プロジェクト作成
$ yarn init -y
# 依存ツールのインストール
$ yarn add @grpc/grpc-js grpc-tools google-protobuf ts-protoc-gen
```

## 2. protocファイルの追加

gRPC の API の設計書となる Protocol Buffers `.proto`ファイルを追加します。

```
$ mkdir proto
$ touch proto/message.proto
```

message.proto の内容は以下とします。

```protobuf:proto/message.proto
syntax = "proto3";

package Posts;

import "google/protobuf/empty.proto";

message Post {
  int32 id = 1;
  string title = 2;
  string content = 3;
}

message GetPostsResponse {
  repeated Post posts = 1;
}

message AddPostRequest {
  Post post = 1;
}

message AddPostResponse {
  Post post = 1;
}

service Posts {
  rpc GetPosts(google.protobuf.Empty) returns (GetPostsResponse) {};
  rpc AddPost(AddPostRequest) returns (AddPostResponse) {};
}
```

## 3 コード生成

Protocol Buffers からクライント・サーバーコード、TypeScript の型定義を生成します。
`package.json`に`codegen`スクリプトを追記します。

```json:package.json
{
  "scripts" : {
    "codegen": "grpc_tools_node_protoc -I ./proto --plugin=protoc-gen-ts=./node_modules/.bin/protoc-gen-ts --js_out=import_style=commonjs,binary:./generated --grpc_out=grpc_js:./generated --ts_out=service=grpc-node,mode=grpc-js:./generated ./proto/*.proto"
  },
  "dependencies": {
    "@grpc/grpc-js": "^1.2.3",
    "google-protobuf": "^3.14.0",
    "grpc-tools": "^1.10.0",
    "ts-protoc-gen": "^0.14.0"
  }
}
```

`codegen`スクリプトでは、`grpc-tools`による Protocol Buffers からのクライアント／サーバーコードの生成を定義しています。
その際に、プラグインとして`protoc-gen-ts`を指定することで、TypeScript の型定義も合わせて生成しています。

スクリプトの詳細はこちらです。

```
grpc_tools_node_protoc -I <.protoファイルのディレクトリ> --plugin=<プラグイン名>=<プラグインのパス> --js_out=<オプション>:<クライアントコード生成先パス> --grpc_out=<オプション>:<サーバーコード生成先パス>  --ts_out=<オプション>:<TypeScript型定義生成先パス> <生成元のProtocol Buffersのパス>
```

実行してみます。

```bash
# 生成先ディレクトリの作成
$ mkdir generated
# コード生成スクリプトの実行
$ yarn codegen
```

`./generated`配下にサーバー・クライアントコード、TypeScript の型定義が生成されるはずです。

```bash
~/s/z/proto ❯❯❯ tree -I node_modules
.
├── generated
│   ├── message_grpc_pb.d.ts
│   ├── message_grpc_pb.js
│   ├── message_pb.d.ts
│   └── message_pb.js
├── package.json
├── proto
│   └── message.proto
└── yarn.lock
```

# 参考

- [OK Google, Protocol Buffers から生成したコードを使って Node.js で gRPC 通信して | メルカリエンジニアリング](https://engineering.mercari.com/blog/entry/20201216-53796c2494/)
- [gRPC on Node.js with Buf and TypeScript — Part 1 | by Slavo Vojacek | Medium](https://slavovojacek.medium.com/grpc-on-node-js-with-buf-and-typescript-part-1-5aad61bab03b)
- [サービス間通信のための新技術「gRPC」入門 | さくらのナレッジ](https://knowledge.sakura.ad.jp/24059/)