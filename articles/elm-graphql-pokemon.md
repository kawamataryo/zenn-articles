---
title: "はじめてのelm-graphql 〜Pokemonと共に〜"
emoji: "🧶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["elm", "graphql", "pokemon"]
published: false
---

この記事は [Elm アドベントカレンダー](https://qiita.com/advent-calendar/2020/elm)22 日目の記事です。

Elm & elm-graphql の入門として GraphQL Pokemon のデータをただ描画するだけのアプリを作りました。チュートリアル形式でまとめます。

# 作るもの

GraphQL Pokemon へクエリーを投げて、取得結果をレンダリングする Elm アプリを作ります。

技術スタックは以下の通りです

- Elm
- elm-graphql
- create-elm-app

今回説明するコードは全てこちらのリポジトリにあります。もし動かない等あればご確認ください。

https://github.com/kawamataryo/elm-graphql-pokemon

# 1. プロジェクトの作成

create-react-app の Elm 版的な [create-elm-app](https://github.com/halfzebra/create-elm-app) でプロジェクトを作成します。

```bash
$ npm install -g create-elm-app
$ create-elm-app my-app
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

# 2. elm-graphql
型安全に GraphQL クエリーを書きたいので elm-graphql を使います。

# 3. GraphQL クエリーの設定

# 4. 取得結果の描画

Maybe型のアンラップをしながらHTMLを組んでいく過程が慣れておらず、ここはとても詰まりました。結局こちらの記事を参考にP`Maybe.withDefault`と`Maybe.map`を使って書いています。

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
# おわりに
以上「」でした。
以前 [Elm の勉強会に参加](https://zenn.dev/ryo_kawamata/articles/elm-hands-on-report)したものの、結局それから Elm を触る機会を持てず Elm アドベントカレンダー期限の 3 日前まできてしまい、急遽作ったのがこちらです。
正直最初はどうにもコンパイルエラーが直せず、めちゃくちゃ詰まったのですが、新しい言語を試行錯誤しながら学ぶ過程が面白かったです。

今回のアプリ作成で、少し Elm と仲良くなれた気がするので、今後は機会見てちょこちょこ書いていきたいと思います。
