---
title: "はじめての elm-graphql 〜 Elm でポケモンを描画するまで 〜"
emoji: "🦋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["elm", "graphql", "pokemon"]
published: false
---

この記事は [Elm アドベントカレンダー](https://qiita.com/advent-calendar/2020/elm)22 日目の記事です。

elm-graphql のお試しとして [GraphQL Pokemon](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) のデータをただ描画するだけのアプリを作りました。Elm x GraphQL の入門に良いと思うので、チュートリアル形式でまとめます。

# 作るもの

[GraphQL Pokemon](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) へクエリを投げて、レスポンスをレンダリングする Elm アプリを作ります。

![](https://i.gyazo.com/44df3b3717a80d74e446c6d22eb98fd3.png)

技術スタックは以下の通りです。

- Elm `0.19.1`
- elm-graphql `5.0.3`
- create-elm-app `5.22.0`

今回説明するコードは全てこちらのリポジトリにあります。もし動かない等あればご確認ください。

https://github.com/kawamataryo/elm-graphql-pokemon

# 1. プロジェクトの作成

create-react-app の Elm 版的な [create-elm-app](https://github.com/halfzebra/create-elm-app) でプロジェクトを作成します。
主要ファイルとビルド環境を一度に作ってくれるので便利です。

```bash
$ npx create-elm-app my-app
```

my-app に移動すると以下のようにディレクトリが作られているはずです。

```
my-app/
├── .gitignore
├── README.md
├── elm.json
├── elm-stuff
├── public
│   ├── favicon.ico
│   ├── index.html
│   ├── logo.svg
│   └── manifest.json
├── src
│   ├── Main.elm
│   ├── index.js
│   ├── main.css
│   └── serviceWorker.js
└── tests
    └── Tests.elm
```

これでアプリを起動します。

```
npx elm-app start
```

しばらく待つと localhost でアプリが起動します。
Elm の環境構築は完了です。

![](https://i.gyazo.com/31f30556d4e959934f51d60f55491b06.png)


あと、CSS を書きたくないので CSS フレームワークの [Bulma](https://bulma.io/) を追加します。Bulma のクラスを Elm 上で型安全に使える `ahstro/elm-bulma-classes`も合わせて追加します。

https://github.com/ahstro/elm-bulma-classes

```
$ yarn add bulma
$ elm install ahstro/elm-bulma-classes
```

src/index.js に以下を追記してください。これでビルド時に Bulma の CSS が読み込まれます

```js:index.js
import 'bulma/css/bulma.min.css';
```

# 2. elm-graphqlの追加 & Code Generate
Elm の GraphQL クライアントは色々あるみたいなのですが、今回は 型安全 に GraphQL クエリを書きたいので、GraphQL のスキーマから型や関連関数を自動生成してくれる `dillonkearns/elm-graphql` を使います。

https://github.com/dillonkearns/elm-graphql

まずインストール。
elm-graphql は elm のパッケージだけでなく、npm パッケージも追加するのがポイントです。

```
$ elm install dillonkearns/elm-graphql
$ elm install elm/json
$ elm install krisajenkins/remotedata
$ yarn add -D @dillonkearns/elm-graphql
```

続いて elm-graphql の Code Generator を起動するスクリプトを package.json に追記します。
ここで、[GraphQL Pokemon](https://graphql-pokemon2.vercel.app)の URL を指定します。

```json:package.json
  "scripts": {
    "code:generate": "elm-graphql https://graphql-pokemon2.vercel.app/ --base Pokemon"
  }
```

この状態で`yarn code:generate`を実行すると以下のように`src/Pokemons`配下にファイルが自動生成されます。

```
src/Pokemon/
├── Enum
├── InputObject
├── InputObject.elm
├── Interface
├── Interface.elm
├── Object
│   ├── Attack.elm
│   ├── Pokemon.elm
│   ├── PokemonAttack.elm
│   ├── PokemonDimension.elm
│   └── PokemonEvolutionRequirement.elm
├── Object.elm
├── Query.elm
├── Scalar.elm
├── ScalarCodecs.elm
├── Union
├── Union.elm
├── VerifyScalarCodecs.elm
└── elm-graphql-metadata.json
```

これが Elm 上で GraphQL を型安全に使うための肝となります。

# 3. 実装

続いて本体を実装していきます。
コードが長いのでポイントのみ抜粋します。

全体のコードは以下をご覧ください。

:::details GraphQLClient.elm
```elm:src/GraphqlClient.elm
module GraphQLClient exposing (makeGraphQLQuery)

import Graphql.Http
import Graphql.Operation exposing (RootQuery)
import Graphql.SelectionSet as SelectionSet exposing (SelectionSet)


graphql_url : String
graphql_url =
    "https://graphql-pokemon2.vercel.app/"


makeGraphQLQuery : SelectionSet decodesTo RootQuery -> (Result (Graphql.Http.Error decodesTo) decodesTo -> msg) -> Cmd msg
makeGraphQLQuery query decodesTo =
    query
        |> Graphql.Http.queryRequest graphql_url
        |> Graphql.Http.send decodesTo
```
:::

:::details Main.elm
```elm:src/Main.elm
module Main exposing (..)

import Browser
import Bulma.Classes as Bulma
import GraphQLClient exposing (makeGraphQLQuery)
import Graphql.Http
import Graphql.Operation exposing (RootQuery)
import Graphql.SelectionSet as SelectionSet exposing (SelectionSet)
import Html exposing (Html, div, figure, h1, img, p, text)
import Html.Attributes exposing (class, src)
import List
import Pokemon.Object
import Pokemon.Object.Pokemon as Pokemon
import Pokemon.Query as Query exposing (PokemonsRequiredArguments)
import Pokemon.ScalarCodecs
import RemoteData exposing (RemoteData)



---- MODEL ----


type alias Pokemon =
    { id : Pokemon.ScalarCodecs.Id
    , name : Maybe String
    , image : Maybe String
    }


type alias Pokemons = Maybe (List (Maybe Pokemon)) 

type alias PokemonData =
    RemoteData (Graphql.Http.Error Pokemons) Pokemons


type alias Model =
    { pokemons : PokemonData }


init : ( Model, Cmd Msg )
init =
    ( { pokemons = RemoteData.Loading
      }
    , fetchPokemons 151
    )



---- UPDATE ----


type Msg
    = FetchDataSuccess PokemonData


update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
    case msg of
        FetchDataSuccess response ->
            updatePokemonsData response model Cmd.none


updatePokemonsData : PokemonData -> Model -> Cmd Msg -> ( Model, Cmd Msg )
updatePokemonsData data model cmd =
    ( { model | pokemons = data }, cmd )



---- VIEW ----


view : Model -> Html Msg
view model =
    div [ class Bulma.container ]
        [ img [ src "https://i.gyazo.com/480551bded5134ddacf08616b2595717.png" ] []
        , h1 [ class Bulma.title, class Bulma.is4, class Bulma.mb6 ] [ text "Pokemons with elm-graphql" ]
        , renderPokemons model.pokemons
        ]


renderPokemons : PokemonData -> Html Msg
renderPokemons pokemonData =
    case pokemonData of
        RemoteData.NotAsked ->
            p [ class Bulma.isSize4, class Bulma.hasTextCentered ] [ text "not" ]

        RemoteData.Success maybePokemonsList ->
            maybePokemonsList
                |> Maybe.withDefault []
                |> List.map renderPokemon
                |> div [ class Bulma.columns, class Bulma.isMultiline ]

        RemoteData.Loading ->
            p [ class Bulma.isSize4, class Bulma.hasTextCentered ] [ text "loading..." ]

        RemoteData.Failure err ->
            p [ class Bulma.isSize4, class Bulma.hasTextCentered ] [ text "Error" ]


renderPokemon : Maybe Pokemon -> Html Msg
renderPokemon maybePokemon =
    div [ class Bulma.column, class Bulma.is3 ]
        [ maybePokemon
            |> Maybe.map
                (\pokemon ->
                    div [ class Bulma.card ]
                        [ div [ class Bulma.cardImage ]
                            [ figure [ class Bulma.image, class Bulma.is16by9, class Bulma.mx5, class Bulma.mt5 ]
                                [ img [ src (pokemon.image |> Maybe.withDefault "") ] []
                                ]
                            ]
                        , div [ class Bulma.cardContent ]
                            [ p [ class Bulma.isSize4 ] [ text (pokemon.name |> Maybe.withDefault "") ]
                            ]
                        ]
                )
            |> Maybe.withDefault (text "")
        ]



---- GraphQL API ----


pokemonsRequiredArguments : Int -> PokemonsRequiredArguments
pokemonsRequiredArguments num =
    { first = num }


pokemonListSelection : SelectionSet Pokemon Pokemon.Object.Pokemon
pokemonListSelection =
    SelectionSet.map3 Pokemon
        Pokemon.id
        Pokemon.name
        Pokemon.image


fetchPokemonsQuery : Int -> SelectionSet Pokemons RootQuery
fetchPokemonsQuery num =
    Query.pokemons (pokemonsRequiredArguments num) pokemonListSelection


fetchPokemons : Int -> Cmd Msg
fetchPokemons num =
    makeGraphQLQuery (fetchPokemonsQuery num) (RemoteData.fromResult >> FetchDataSuccess)



---- PROGRAM ----


main : Program () Model Msg
main =
    Browser.element
        { view = view
        , init = \_ -> init
        , update = update
        , subscriptions = always Sub.none
        }
```
:::


### GraphQL Client

GraphQL のリクエストを送るためのクライアントです。
GraphQL Pokemon には認証が不要なので、elm-graphql で提供されている関数にエンドポイントの URL を渡すだけで OK です。

```elm:src/GraphQLClient.elm
graphql_url : String
graphql_url =
    "https://graphql-pokemon2.vercel.app/"


makeGraphQLQuery : SelectionSet decodesTo RootQuery -> (Result (Graphql.Http.Error decodesTo) decodesTo -> msg) -> Cmd msg
makeGraphQLQuery query decodesTo =
    query
        |> Graphql.Http.queryRequest graphql_url
        |> Graphql.Http.send decodesTo
```

### Type alias & Model

Main.elm の type alias 及び Model です。
GraphQL リクエストのレスポンスは`krisajenkins/remotedata`の [RemoteData](https://package.elm-lang.org/packages/krisajenkins/remotedata/latest/) を使いハンドリングします。init のタイミングでポケモンを取得する GraphQL クエリ実行しています。

```elm:main.elm
type alias Pokemon =
    { id : Pokemon.ScalarCodecs.Id
    , name : Maybe String
    , image : Maybe String
    }


type alias Pokemons =
    Maybe (List (Maybe Pokemon))


type alias PokemonData =
    RemoteData (Graphql.Http.Error Pokemons) Pokemons


type alias Model =
    { pokemons : PokemonData }

init : ( Model, Cmd Msg )
init =
    ( { pokemons = RemoteData.Loading
      }
    , fetchPokemons 151
    )
```

### GraphQL Query

GraphQL のクエリ部分です。
`pokemonsRequiredArguments`で GraphQL クエリ時の引数。`pokemonListSelection`で取得するフィールドを組み立てています。
どのクエリを呼び出すかどうかは`fetchPokemonsQuery`の`Query.pokemons`で指定します。これは`elm-graphql`で自動的に生成される関数です。
最後に`fetchPokemons`で、先ほど作った`makeGraphQLQuery` を使った GraphQL リクエストを実行する関数を作っています。

elm-graphql から提供されいている型・関数を使うことで、完全に型で守られた状態で GraphQL クエリがかけます。すごい！

```elm:Main.elm
pokemonsRequiredArguments : Int -> PokemonsRequiredArguments
pokemonsRequiredArguments num =
    { first = num }


pokemonListSelection : SelectionSet Pokemon Pokemon.Object.Pokemon
pokemonListSelection =
    SelectionSet.map3 Pokemon
        Pokemon.id
        Pokemon.name
        Pokemon.image


fetchPokemonsQuery : Int -> SelectionSet Pokemons RootQuery
fetchPokemonsQuery num =
    Query.pokemons (pokemonsRequiredArguments num) pokemonListSelection


fetchPokemons : Int -> Cmd Msg
fetchPokemons num =
    makeGraphQLQuery (fetchPokemonsQuery num) (RemoteData.fromResult >> FetchDataSuccess)
```

### Update

状態変更は Update で行います。
GraphQL クエリの実行後に呼ばれる FetchDataSuccess の Msg でレスポンスから Model の更新を行っています。

```elm:Main.elm
type Msg
    = FetchDataSuccess PokemonData


update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
    case msg of
        FetchDataSuccess response ->
            updatePokemonsData response model Cmd.none


updatePokemonsData : PokemonData -> Model -> Cmd Msg -> ( Model, Cmd Msg )
updatePokemonsData data model cmd =
    ( { model | pokemons = data }, cmd )
```

### View

最後に View です。
`renderPokemons`で pokemonData にある RemoteData の値によって処理を変更しています。
RemoteData が Success の場合のみ結果の HTML を表示するようになっています。
RemoteData を使うとリクエスト中は、ローディング文字列を出すなども簡単に出来るので良いですね。

:::message
GraphQL Pokemon のレスポンスは階層的な Maybe 型なのですが Maybe 型のアンラップをしながら HTML を組んでいく過程が慣れておらず、とても詰まりました。結局[こちら](https://qiita.com/aimy-07/items/76f85697f5996276f8f4)の記事を参考に`Maybe.withDefault`と`Maybe.map`を使って書いています。
もしもっとスマートに Maybe 型を扱える方法があれば知りたいです🙏
:::

```elm
view : Model -> Html Msg
view model =
    div [ class Bulma.container, class Bulma.mt5 ]
        [ h1 [ class Bulma.title, class Bulma.is1, class Bulma.mb6 ] [ text "Pokemon with elm-graphql" ]
        , renderPokemons model.pokemons
        ]


renderPokemons : PokemonData -> Html Msg
renderPokemons pokemonData =
    case pokemonData of
        RemoteData.NotAsked ->
            p [ class Bulma.isSize4, class Bulma.hasTextCentered ] [ text "not" ]

        RemoteData.Success maybePokemonsList ->
            maybePokemonsList
                |> Maybe.withDefault []
                |> List.map renderPokemon
                |> div [ class Bulma.columns, class Bulma.isMultiline ]

        RemoteData.Loading ->
            p [ class Bulma.isSize4, class Bulma.hasTextCentered ] [ text "loading..." ]

        RemoteData.Failure err ->
            p [ class Bulma.isSize4, class Bulma.hasTextCentered ] [ text "Error" ]


renderPokemon : Maybe Pokemon -> Html Msg
renderPokemon maybePokemon =
    div [ class Bulma.column, class Bulma.is3 ]
        [ maybePokemon
            |> Maybe.map
                (\pokemon ->
                    div [ class Bulma.card ]
                        [ div [ class Bulma.cardImage ]
                            [ figure [ class Bulma.image, class Bulma.is16by9 ]
                                [ img [ src (pokemon.image |> Maybe.withDefault "") ] []
                                ]
                            ]
                        , div [ class Bulma.cardContent ]
                            [ p [ class Bulma.isSize4 ] [ text (pokemon.name |> Maybe.withDefault "") ]
                            ]
                        ]
                )
            |> Maybe.withDefault (text "")
        ]
```

以上で終わりです。
以下コマンドを実行すればポケモンが表示されるはずです！
完成 🎉

```
$ npx elm-app start
```

![](https://i.gyazo.com/08c36cb31a9a9488bb20a053f1bba59b.png)
※ トップ画像と背景色が違うのは main.css で body 要素の background-color を指定していないからです。

# おわりに
以上「はじめての elm-graphql」でした。

[以前 Elm の勉強会に参加](https://zenn.dev/ryo_kawamata/articles/elm-hands-on-report)したものの、結局それから Elm を触る機会を持てず Elm アドベントカレンダー期限の 4 日前まできてしまい急遽作ったのがこちらです😅
まだまだ Elm に慣れていないので、もしおかしいところあれば気軽にコメント頂けると嬉しいです。

正直最初はどうにもコンパイルエラーが直せず、めちゃくちゃ詰まったのですが、新しい言語を試行錯誤しながら学ぶ過程が面白かったです。
今回のアプリ作成で、少し Elm と仲良くなれた気がするので、機会見てちょこちょこ書いていきたいなと思っています。

# 参考

- [[Elm] Maybeを活用する - Qiita](https://qiita.com/aimy-07/items/76f85697f5996276f8f4)
- [halfzebra/create-elm-app: 🍃 Create Elm apps with zero configuration](https://github.com/halfzebra/create-elm-app)
- [Client Side Elm Setup | GraphQL Elm Tutorial](https://hasura.io/learn/graphql/elm-graphql/elm-graphql/)
- [dillonkearns/elm-graphql: Autogenerate type-safe GraphQL queries in Elm.](https://github.com/dillonkearns/elm-graphql)