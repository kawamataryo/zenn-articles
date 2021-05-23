---
title: "Elixir で GitHub Trending をスクレイピングしてみる"
emoji: "📿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["elixir", "floki", "HTTPoison", "スクレイピング"]
published: true
---

Elixir の勉強のため GitHub Trending の内容をスクレイピングするコードを書いてみたのでまとめます。

# 何を作る？

[GitHub Trending repositories](https://github.com/trending)のリポジトリ一覧の主要データを配列で返してくれる関数を作ります。
一応引数で言語指定もできるようにします。

```elixir
GithubTrendingCrawler.crawl() # 言語指定なしのトレンドリポジトリを取得
GithubTrendingCrawler.crawl("python") # Pythonのトレンドリポジトリを表示
```

実際の動作はこちらです。

![](https://i.gyazo.com/15c661e235a0843ed0737ccdb7c52a43.gif)

また、今から記載するコードは全てこちらにあります。

https://github.com/kawamataryo/github_trending_crawler

# プロジェクトの作成、依存関係の追加

最初に Elixir のパッケージマネージャー兼ビルドツールである[mix](https://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html)でプロジェクトを作成します。

```bash
$ mix new github_trending_crawler
$ cd github_trending_crawler
```

そしてスクレイピングに利用するパッケージを追加します。
今回は Web ページの情報を取得するための HTTP クライアントとして [HTTPoison](https://github.com/edgurgel/httpoison)、取得した HTML から目的のデータを抽出するための HTML パーサーとして [Floki](https://github.com/philss/floki)を利用します。

https://github.com/edgurgel/httpoison

https://github.com/philss/floki

`mix.exs`を開いてそれぞれ依存に追加します。

```elixir:mix.exs
defmodule GithubTrendingCrawler.MixProject do
  # ...
  defp deps do
    [
      {:httpoison, "~> 1.8"},
      {:floki, "~> 0.30.0"}
    ]
  end
end
```

`mix deps.get`コマンドでインストールします。

```bash
$ mix deps.get
```

これで準備は完了です！

# パーサーの実装
最初にパーサーを実装します。パーサーでは Floki を使って HTML の文字列から、特定のデータを抽出し構造化したデータで返します。

今回は GitHub Trending に掲載されている各リポジトリから以下情報を抽出することとします。

- 名前
- URL
- 概要
- 累計スター数
- 使用言語

`lib`配下に`github_trending_parser.ex`を作成します。

```elixir:github_trending_parser.ex
defmodule GithubTrendingParser do

  def parse(document) when is_binary(document) do
    document
    |> Floki.parse_document!
    |> parse_rows
    |> Enum.map(
      &(%{
        title: parse_title(&1),
        description: parse_description(&1),
        lanugage: parse_language(&1),
        link: parse_link(&1),
        star_count: parse_star_count(&1)
      })
    )
  end

  defp parse_rows(document) do
    document
    |> Floki.find(".Box-row")
  end

  defp parse_link(document) do
    document
    |> Floki.find("h1 > a")
    |> Enum.at(0)
    |> (&("#{@base_url}#{Floki.attribute(&1, "href")}")).()
  end

  defp parse_title(document) do
    document
    |> Floki.find("h1 > a")
    |> Enum.at(0)
    |> Floki.text
    |> remove_whitespace
  end

  defp parse_star_count(document) do
    document
    |> Floki.find("div.color-text-secondary > a:nth-child(2)")
    |> Enum.at(0)
    |> Floki.text
    |> remove_whitespace
    |> String.replace(",", "")
    |> String.to_integer
  end

  defp parse_description(document) do
    document
    |> Floki.find("p")
    |> Enum.at(0)
    |> Floki.text
    |> String.trim
  end

  defp parse_language(document) do
    document
    |> Floki.find("[itemprop='programmingLanguage']")
    |> Enum.at(0)
    |> Floki.text
  end

  defp remove_whitespace(text) do
    Regex.replace(~r/\s+/, text, "")
  end
end
```

関数がいくつかあるのですが、 エントリーポイントはパブリック関数となっている`parse`です。
`parse`で、HTML の文字列を受け取り、それを以下のような手順でパースしています。

1. HTML 文字列を Floki で構造化された Map データにパース
2. プライベート関数の`parse_rows` で GitHub Trending のリポジトリごとの行のデータに変換
3. 各行のデータに対して`Enum.map`で、目的のデータの key-value となるマップを作成（マップ作成時に`parse_language`等、各行のデータを取得する関数を実行）

それぞれのパース関数では `Floki.find` で目的の DOM を抽出して、`Floki.text`や`Floki.attribute`で値を取得しています。
`|>` のパイプライン演算子で流れるように処理をかけるので、読みづらくなりがちなパース処理も比較的見やすくかけてる気がします。

# クローラーの実装

次に、HTTP Client の HTTPoison を使用して、パーサーに HTML 文字列を渡すためのクローラーを実装します。
`lib`配下に`github_trending_crawler.ex`を作成します。

```elixir:github_trending_crawler.ex
defmodule GithubTrendingCrawler do
  @base_url "https://github.com"

  def crawl() do
    HTTPoison.get!("#{@base_url}/trending/").body
    |> GithubTrendingParser.parse
  end

  def crawl(language) when is_binary(language) do
    HTTPoison.get!("#{@base_url}/trending/#{language}").body
    |> GithubTrendingParser.parse
  end
end
```

今回はエラーハンドリングを省いたので、クローラーの実装はとてもシンプルです。
`HTTPoison.get!`で取得した HTTP レスポンスの body を先程作成したパーサーに渡して結果を返すだけです。

引数のあるバージョンとないバージョンで同じ`crawl`関数を定義しているのは、言語の指定がある場合とない場合でリクエストする URL を分けるためです。
Elixir の関数呼び出しはパターンマッチで行われるらしいので、このように書くと`GithubTrendingCrawler.crowl()`の呼び出しでは上の関数定義、`GithubTrendingCrawler.crowl("Python")`の呼び出しでは下の関数定義が実行されます。
普通だと引数の値を if 文で判定する処理を書きたくなりますが、Elixir だとこのようにシンプルにかけるので良いですね。

これでクローラーも完成です🎉
`iex -S mix` で REPL を起動して、クローラーを実行してみましょう。

```bash
$ iex -S mix
# Erlang/OTP 23 [erts-11.2] [source] [64-bit] [smp:16:16] [ds:16:16:10] [async-threads:1] [hipe] [dtrace]
#
# Compiling 1 file (.ex)
# Interactive Elixir (1.11.4) - press Ctrl+C to exit (type h() ENTER for help)

iex(1)> GithubTrendingCrawler.crawl()
[
  %{
    description: "High Performance, Kubernetes Native Object Storage",
    lanugage: "Go",
    link: "/minio/minio",
    star_count: 27750,
    title: "minio/minio"
  },
  %{
    description: "GoogleTest - Google Testing and Mocking Framework",
    lanugage: "C++",
    link: "/google/googletest",
    star_count: 22423,
    title: "google/googletest"
  },
  # ...
```

# 終わりに

以上「Elixir で GitHub Trending をスクレイピングしてみる」でした。
初めて Elixir でスクレイピングのコードを書いたのですが、パイプ演算子で関数をつないていく書き方が、書いていてとても楽しかったです。今後も折り見て書いていきたいです。
