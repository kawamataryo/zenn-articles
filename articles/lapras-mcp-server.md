---
title: "LAPRAS公式のMCPサーバーをリリースして得た学び"
emoji: "🐬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "lapras", "cursor", "claude"]
publication_name: "lapras_inc"
published: true
---

最近リリースした[LAPRAS公式MCPサーバー](https://github.com/lapras-inc/lapras-mcp-server)についての学びをまとめます。SaaS公式のMCPサーバーをリリースした例はまだあまり多くないと思うので、何かの参考になれば嬉しいです。

## 何を開発した？

エンジニア向けポートフォリオ兼転職サービス [LAPRAS](https://lapras.com/) の公式MCPサーバーを開発しました。

https://github.com/lapras-inc/lapras-mcp-server

このMCPサーバーを使うことで、CursorやClaude DesktopなどMCPクライアントを利用できるLLMツール経由で、[LAPRAS](https://lapras.com/)上のデータを簡単に取得、更新することができます。

**2025年4月22日現在の機能**

- 求人一覧、詳細の取得
- 職歴の取得・更新
- 職務要約の取得・更新
- 今後のキャリアでやりたいことの取得・更新

https://x.com/KawamataRyo/status/1910545512840962219

## 使い方

各LLMツールのMCPサーバーの設定（[Cursor](https://docs.cursor.com/context/model-context-protocol#configuring-mcp-servers)、[Claude Desktop](https://modelcontextprotocol.io/quickstart/user)）を参考に、`mcp.json`または`claude_desktop_config.json`に以下を追記してください。  `LAPRAS_API_KEY`はキャリア関連のtoolを使う場合のみ必要です。[設定](https://lapras.com/config/api-key)からキーの発行ができます。

**npxの場合**

```json
{
  "mcpServers": {
    "lapras": {
      "command": "npx",
      "args": [
        "-y",
        "@lapras-inc/lapras-mcp-server"
      ],
      "env": {
        "LAPRAS_API_KEY": "<YOUR_LAPRAS_API_KEY>"
      }
    }
  }
}
```

**Dockerの場合**

```json
{
  "mcpServers": {
    "lapras": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "-e",
        "LAPRAS_API_KEY",
        "laprascom/lapras-mcp-server:v0.4.0"
      ],
      "env": {
        "LAPRAS_API_KEY": "<YOUR_LAPRAS_API_KEY>"
      }
    }
  }
}
```

設定後にMCPクライアントを再起動し、以下のようなプロンプトを入力することでLAPRAS上のデータを取得、更新することができます。

**求人検索の例**

```
フルリモートワーク可能でRustが使えるバックエンドの求人を探してください。年収は800万以上で。
結果はMarkdownの表にまとめてください。
``` 

**他媒体のデータを使った職歴更新の例**

```
<自分のキャリアがわかる画像 or URL を貼り付ける> 
これが私の職歴です。LAPRASの職歴を更新してください。
```

**LAPRASの職歴を改善する例**

```
LAPRASの職歴を取得して、ブラッシュアップするための質問をしてください。
改善後、LAPRASの職歴を更新してください。
```


# 開発時に意識したこと


## MCP公式のライブラリを利用する
MCPはまだ仕様が定義されて間もなく、今後も大きく変化することが考えられます。MCPサーバーの開発には[fastmcp](https://github.com/jlowin/fastmcp)など利用しやすいサードパーティ製ライブラリの選択肢もありますが、将来的な仕様変更への追従性を考慮して、[公式のTypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)のみの採用としました。

https://github.com/modelcontextprotocol/typescript-sdk

## リモートMCPは利用しない
開発をスタートした時は、[CloudflareのリモートMCPの記事](https://blog.cloudflare.com/remote-model-context-protocol-servers-mcp/) がちょうど公開された時期でした。

リモートMCPの採用も考えたのですが、採用例が少なく、ベストプラクティスがわからない。認証周りの構築に工数がかかる。クライアント側の設定としては、リモートMCPを採用しても、結局npxで[mcp-remote](https://developers.cloudflare.com/agents/guides/test-remote-mcp-server/) を実行するので手間は変わらない。ということで、デリバリーの速さを重視して、リモートMCPは採用しませんでした。

現在であれば、Streamable HTTP Transportを使いステートレスなリモートMCPサーバーも手軽に構築できるようなので、採用するのはアリかもしれません。

https://github.com/modelcontextprotocol/typescript-sdk/releases/tag/1.10.0?ref=blog.lai.so
https://blog.lai.so/remote-mcp/ 

## 拡張性を意識してtool定義を分離する
[modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) のリポジトリでは、多くのサンプルコードが`server.py`などの単一ファイルにtool定義を列挙する形で実装されていました。
これも一つのアプローチですが、lapras-mcp-serverでは拡張性を重視し、以下のような設計を採用しました。

- tool定義のインターフェイスを実装するクラスごとに個別のファイルを作成
- index.tsでtool定義をまとめてserverに登録する

この設計により、個別toolに対するテストが書きやすく、また新しいtoolの追加も容易になり、結果としては良かったなと思います。

**階層構造**

```
lapras-mcp-server/src/
├── tools/
│   ├── createExperience.py
│   ├── updateExperience.py
│   ├── searchJobs.py
│   └── ...
└── index.ts
```

**toolの実装**

https://github.com/lapras-inc/lapras-mcp-server/blob/main/src/tools/getJobDetail.ts#L8-L66

**serverへの登録**

https://github.com/lapras-inc/lapras-mcp-server/blob/main/src/index.ts#L17-L47

## 監視体制を整える

MCPサーバーのtoolで利用しているAPIは、新規に作成したMCP専用APIです。

LAPRAS内でまだ公開APIの知見も少なく公開API用の環境整備も後追いとなってしまう状況でした。またLLM側でどのようなAPIリクエストを行うか予想できないということで、リクエストの負荷による本プロダクトへの影響が懸念されました。そのため、SREチームと協力して、監視体制を構築しました。

主に行ったのは、AWS WAFによるAPIごとのレートリミットの設定と、DatadogでのMCP用APIのリクエスト監視・アラートの設定です。
今のところWAFの発動や、異常なリクエストもなく安定しています。監視体制があることで安心してリリースできているので、初期に整えてよかったなと思います。SREチームの[@なっぱ](https://x.com/b7472)さんと[@Kumamoto](https://x.com/digitalhimiko)さんに感謝です。

**AWS WAF**

https://aws.amazon.com/jp/waf/

**Datadogのダッシュボード**


![Datadogのダッシュボード](/images/lapras-mcp-server/2025-04-22-09-59-40.png)
*数値はダミーです。*

# 開発時の失敗と学び

## コンテキスト長を考慮できていなかった

求人一覧など大量のデータを返すtoolを実行した際、コンテキスト長の制限によりリクエストが失敗するケースがありました。

この問題は、コンテキスト長に余裕がある環境（Claude Desktopの有料版やCursorのPro版など）では発生せず、コンテキスト長が制限されている環境（Cursorの無料版など）でのみ発生したため、発見が遅れました。

API側のページングのper_pageを縮小したり、MCP用APIでは不要なデータを除外するなど対応しました。

**対応commit**
https://github.com/lapras-inc/lapras-mcp-server/commit/f04dc9c41e91a60917b2ebdb822cbc3513cb328f

## LLM側に複雑な操作を要求してしまった

当初、職歴更新toolで呼び出していたAPIは、バルクでのcreate・delete・updateという一般的ではないインターフェイスでした。
これはプロダクト側のUIから呼び出されていたAPI実装を踏襲したものだったのですが、LLM側がtoolの動作を上手く把握できず、職歴の部分更新を行ってしまい、データが吹き飛ぶという問題が発生しました。

意図せぬ削除が発生するかどうかをMCPサーバーのtoolのコードで検証し、`force: true`を指定された場合のみ許可するなど、インターフェイスはそのままで実装上の工夫もやってみましたが、それでもLLMがユーザー確認なしに`force: true`を指定してしまい同じくデータが吹き飛び結局断念しました。


https://github.com/lapras-inc/lapras-mcp-server/pull/3

最終的に、一般的なAPIインターフェイスに変更して、部分更新ができるようにすることで安定した呼び出しを実現しました。LLMといえど万能ではなく、素直なAPI実装が扱いやすく安定したtool呼び出しにつながるという学びでした。


https://github.com/lapras-inc/lapras-mcp-server/pull/4




:::message
forceの仕組みを思いつき、圧倒的閃きだと思って投稿したslackのメモ😇 結局これは実現せずでした。
![圧倒的閃き](/images/lapras-mcp-server/2025-04-22-11-08-58.png)
:::

## LLM特有の出力を考慮できていなかった

LLM側でtool呼び出し時のパラメーターに改行をエスケープした文字列（`//n`）を入れてしまい、プロダクト側の表示が崩れるケースがありました。仕様的なバリデーションはAPI側で行っていたのですが、これは想定外でした。

MCPサーバー側でLLMの入力値をクレンジングすることで対応しました。

**該当Issue**

https://github.com/lapras-inc/lapras-mcp-server/issues/5

**修正PR**

https://github.com/lapras-inc/lapras-mcp-server/pull/6

## どこまでをサービスとして担保するのか問題

これは失敗と学びではないですが、チーム内で議論になったので記載しておきます。

職歴など更新系のtoolを提供するにあたって、サービスとしてどこまでの品質を担保すべきか意見が分かれました。いくらMCPサーバー側でtoolやパラメーターの説明を詳細に書いても、使われるLLMのモデルによっては説明にそぐわない使い方や入力をしてしまう可能性があります。
もちろん仕様的な値のバリデーションはAPI側で行いますが、内容自体の正当性は担保することができません。

今回提供するtoolは更新後にユーザーによって確認・修正が可能なものなので、リスクを考えつつもリリースに踏み切りましたが、やり直しの効かない操作（例えば転職サービスなら応募やメッセージの送信など）での利用は、まだ難しいと感じました。


# 使ってみての感想と今後について

色々苦労はありましたが、自分でも使ってみての感想や、リリースしてみてのユーザーの反応はかなり良いです。
まだ利用ユーザーは限定的ですが、今後さらに広がると嬉しいなと思います。

https://x.com/k0n_karin/status/1912480665318089145

https://x.com/ekusiadadus/status/1912759909516820949

今後も、改善を続けていく予定なのでぜひ使ってみてください。







