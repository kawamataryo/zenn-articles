---
title: "はじめてのelm-graphql - GraphQL Pokemonを表示する -"
emoji: "🦋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["elm", "graphql", "pokemon"]
published: false
---

この記事は [Elm アドベントカレンダー](https://qiita.com/advent-calendar/2020/elm)22 日目の記事です。

Elm & elm-graphql の入門として [GraphQL Pokemon](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) のデータをただ描画するだけのアプリを作りました。チュートリアル形式でまとめます。

# 作るもの

GraphQL Pokemon へクエリを投げて、取得結果をレンダリングする Elm アプリを作ります。

![](https://i.gyazo.com/44df3b3717a80d74e446c6d22eb98fd3.png)

技術スタックは以下の通りです。

- Elm `0.19.1`
- elm-graphql `5.0.3`
- create-elm-app `5.22.0`

今回説明するコードは全てこちらのリポジトリにあります。もし動かない等あればご確認ください。

https://github.com/kawamataryo/elm-graphql-pokemon

# 1. プロジェクトの作成

create-react-app の Elm 版的な [create-elm-app](https://github.com/halfzebra/create-elm-app) でプロジェクトを作成します。

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
elm の環境構築は完了です。

![](https://i.gyazo.com/31f30556d4e959934f51d60f55491b06.png)


あと、CSS を書きたくないので CSS フレームワークの [Bulma](https://bulma.io/) を追加します。Bulma のクラスを Elm 上で type-safe に使える `ahstro/elm-bulma-classes`も合わせて追加します。

https://github.com/ahstro/elm-bulma-classes

```
$ yarn add bulma
$ elm install ahstro/elm-bulma-classes
```

index.js に以下を追記してください。これでビルド時に Bulma の CSS が読み込まれます

```js:index.js
import 'bulma/css/bulma.min.css';
```

# 2. elm-graphqlの追加 & Code Generate
elm の GraphQL クライアントは色々あるみたいなのですが、今回は type-safe に GraphQL クエリを書きたいので、GraphQL のスキーマから型や関連関数を自動生成してくれる dillonkearns/elm-graphql を使います。

https://github.com/dillonkearns/elm-graphql

まずインストール。

```
$ elm install dillonkearns/elm-graphql
$ elm install elm/json
$ elm install krisajenkins/remotedata
$ yarn add -D @dillonkearns/elm-graphql
```

続いて elm-graphql の Code Generator を起動するスクリプト package.json に追記します。
ここで、今回使う[GraphQL Pokemon](https://graphql-pokemon2.vercel.app)の URL を指定します。

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

これが GraphQL を type-safe に使うための肝となります。

# 3. 実装

まず GraphQL のリクエストを送るためのクライアントを作ります。
GraphQL Pokemon には認証が不要なのでとても簡単です。

```elm:src/GraphQLClient.elm
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

あとは、先ほど生成されたコードを使って Main.elm を作っていけば完成です。

:::details Main.elm
```elm:Main.elm
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

ポイントのみ解説します。

まず必要な type alias 及び Model はこちらです。
GraphQL リクエストのレスポンスは`krisajenkins/remotedata`の RemoteData を使いハンドリングします。
あと、init のタイミングでポケモンを取得する GraphQL クエリ実行しています。

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

GraphQL のクエリ部分はこちらです。

`pokemonsRequiredArguments`で GraphQL クエリ時の引数。`pokemonListSelection`で取得するフィールドを組み立てています。
どのクエリを呼び出すかどうかは`fetchPokemonsQuery`の`Query.pokemons`で指定します。これは`elm-graphql`で自動的に生成される関数です。
最後に`fetchPokemons`で、先ほど作った`makeGraphQLQuery` を使った GraphQL リクエストを実行する関数を作っています。

elm-graphql から提供されいている型・関数を使うことで完全に type-safe に GraphQL クエリがかけます。すごい！

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


Update 部分はこちらです。
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

最後に View の部分です。
`renderPokemons`で pokemonData にある RemoteData の値によって処理を変更しています。
もし、RemoteData が Success の場合は結果の HTML を表示するようになっています。

:::message
ここの Maybe 型のアンラップをしながら HTML を組んでいく過程が慣れておらず、めちゃ詰まりました。結局こちらの記事を参考に`Maybe.withDefault`と`Maybe.map`を使って書いています。
https://qiita.com/aimy-07/items/76f85697f5996276f8f4
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

この状態で、以下コマンドを実行すればポケモンが表示されるはずです！
完成 🎉

```
$ npx elm-app start
```

![](https://i.gyazo.com/08c36cb31a9a9488bb20a053f1bba59b.png)

# おわりに
以上「はじめての elm-graphql」でした。
[以前 Elm の勉強会に参加](https://zenn.dev/ryo_kawamata/articles/elm-hands-on-report)したものの、結局それから Elm を触る機会を持てず Elm アドベントカレンダー期限の 4 日前まできてしまい急遽作ったのがこちらです😅
まだまだ Elm に慣れていないので、もしおかしいところあれば気軽にコメント頂けると嬉しいです。

正直最初はどうにもコンパイルエラーが直せず、めちゃくちゃ詰まったのですが、新しい言語を試行錯誤しながら学ぶ過程が面白かったです。
今回のアプリ作成で、少し Elm と仲良くなれた気がするので、今後は機会見てちょこちょこ書いていきたいと思います。
