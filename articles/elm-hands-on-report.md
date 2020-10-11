---
title: "Elm初心者が感じたElmの魅力〜知識0から始めるElm入門ハンズオン参加レポート〜"
emoji: "📝"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["勉強会", "Elm"]
published: false
---

昨日（2020/10/10）、 (@ababupdownba)[https://Twitter.com/ababupdownba]さんが開催した Elm 初心者向けの勉強会（[知識0から始めるElm入門ハンズオン](https://elm-jp.connpass.com/event/188305/) )に参加して感じた Elm の魅力を、勉強会の振り返りもかねて書いてみました。

もし理解に間違いがあれば、コメント頂けると嬉しいです。

# Elmとは？
2012 年にエバン・チャプリッキ（Evan Czaplicki）氏によって発表された比較的若いプログラミング言語で JavaScript にトランスパイルされる AltJS の一種。
また、Haskell の影響を受けた関数型言語である。

https://elm-lang.org/

# Elmの魅力
## ほぼ全てが関数である
最初に Elm をみたときに「う、、全然分からん」ってなったのですが、「Elm ではほぼ全てが関数である」というのを聞いた後からは少し読みやすくなりました。
以下、The Elm Architecture での View コードの抜粋です。ここで`div`や`button`など HTML 要素ぽいものが出ててくるのですが、これらも関数です。属性の配列と、子要素の配列を受け取ります。
関数なので型チェックが効き HTML の閉じタグ忘れでのレイアウト崩れなど起こり用がありません。また、View のテストも用意にかけるそうです。

```elm
view : Model -> Html Msg
view model =
    div []
        [ button [ onClick Increment ] [ text "+1" ]
        , div [] [ text <| String.fromInt model.count ]
        , button [ onClick Decrement ] [ text "-1" ]
        ]
```


## パイプライン演算子がある
式の評価結果を次の式に渡す`|>`と`<|`のパイプライン演算子がとても使いやすそうに思えました。
例えば値`x`を`funcA`関数で処理して、その結果を`funcB`関数に、さらにその結果を`funcC`で処理する必要がある場合。
JS で書くと、、

```js
const result = funcC(funcB(funcA(x)))
```

もしく丁寧に書くと、

```js
const aResult = funcA(x)
const bResult = funcB(aResult)
const cResult = funcC(bResult)
```

となると思います。

これを Elm のパイプ演算子を使うとかなり直感的に書くことができます。

```elm
result = x |> funcA |> funcB |> funcC

-- もしくは

result = funcC <| funcB <| funcA <| 1
```

ちなみにパイプライン演算子も関数で型定義があります。

https://package.elm-lang.org/packages/elm/core/latest/Basics#(|%3E)

## フレームワークが言語に内包されている
JS で Web アプリを作ろうとすると、フレームワークの選定は React or Vue or Angular？　、React を選んでも Store の設計は Recoil を使う？　Redux を使う？　と迷うと思います。
Elm では、言語に The Architecture Architecture というフレームワークが内包されています（この表現があっているのかちょっと自信がないです）。全てはこのフレームワークの設計にそって行うので、迷いようがありません。
一見すると、柔軟性がないように感じますが、1 つの設計を覚えれば、他 Elm を採用しているどの会社、どのプロジェクトでも使えるというのは魅力的だなと思いました。

[The Elm Architecture · An Introduction to Elm](https://guide.elm-lang.jp/architecture/)

## 実行時例外が発生しない
Elm では基本的に実行時例外が発生しないという話も驚きました。型によってパターンの網羅性のチェックをしていて、null や undefined が発生しないようになっているようです。
例えばパターンマッチでの網羅性のチェックなどが強力です。

以下 The Elm Architecture の update のコードなのですが、update 関数のパターンマッチで`Foo`バリアントが考慮されていません。

```elm
type Msg
    = Bar
    | Baz
    | Foo -- これがcase句で考慮されていない


update : Msg -> Model -> Model
update msg model =
    case msg of
        Bar ->
          { model|message = 'bar' }
        Baz ->
          { model|message = 'baz' }
```

この場合は、コンパイル時に以下のような警告がでます。

> This `case` does not have branches for all possibilities:Missing. possibilities include: Foo

コンパイル時に考慮漏れを未然に防ぐことが出来るのはよいですね。
また、副作用のある処理は明示的に別に管理するようで、万が一何かエラーが発生した際も原因の特定が容易との話でした（ここら辺曖昧な理解です）。

## 覚える文法が少ない
今回 4 時間の勉強会だったのですが、その時間でほぼ全ての Elm の構文を学べました。
実際に後半、OSS で公開されている Elm パッケージのコードをみたのですが、何となく何が書いてあるのか理解できたのが凄かったです。
実際に、Elm はバージョンアップの度に機能が増えるのではなく、機能を減らすこともあるそうです。他の言語とは真逆でなかなか珍しいですよね。

覚える文法が少ない=学習コストが少ないとなるので、今後 Elm を学びたいと思っている自分のような人にも良いです。

# 勉強会の感想

今回の勉強会はなかなか変わった形式で、Elm の知識 0 の状態で、Elm の Playground である[Ellie](https://ellie-app.com/new)にあるサンプルコードをみながら、参加者が疑問点を質問していくという形式でした。
全くの初心者向けの勉強会ということで、Elm のデータ型や、構文を一通り講義形式で受けられるものかな？　と参加前は思っていたので、最初は驚いたのですが、この参加者質問形式で進める方式は結果として理解が進みました。というのも、知識 0 の状態からコードみるのでそもそも疑問点しかなく、なんの抵抗もなく質問できます。質問のハードルがとても下がります。
また、所々で（@ababupdownba)[https://Twitter.com/ababupdownba]さんから理解を確認するために、「このコードの説明をしてみてください」と逆に質問されるので、そこで自分の言葉で、説明することで理解の確認もできました（ここでも、そもそも分からないこと前提なので、なんの恥ずかしげもなく答えられました）。

この方式でやるには、参加者をかなり制限する必要がありますが、初めての言語を学ぶ方法として、とても優れているのではないかと思いました。今まで参加した勉強会のなかで、一番勉強になった勉強会だと思います。

本当にありがとうございました。

Elm についても、今回の勉強会で色々魅力を感じたので今後も継続して勉強していきたいです。
まず、公式ドキュメントを読んで、その後こちらのサンプルでもやってみようかなと思います。

https://employment.en-japan.com/engineerhub/entry/2020/03/05/103000
