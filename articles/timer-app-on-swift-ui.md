---
title: "Swift UIで初めてのApple Watch アプリを作る"
emoji: "✌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "Xcode", "ios"]
published: false
---

せっかくApple Watchを持っているので何かアプリを作りたいと思い、Swift UI を勉強中です。
最初に修作として作ったタイマーアプリの作り方をまとめます。

本業は Webフロントエンドなので、Swift 自体が初めてです。間違い等あれば、気軽にコメントいただけると嬉しいです🙏

# 作るもの

簡単なタイマーアプリを作ります。
ピッカーで秒数を選んで`GO`を押すと、タイマーが起動して、設定された秒数が経過すると、通知音とともにタイマーが終了します。


# Xcode プロジェクトのスタート

Xcode を開き `Create a new Xcode project` で新しいプロジェクトを作成します。
最初の画面で何を作るのか選択できるので、Watch App を選択して Next を押します。

![](https://storage.googleapis.com/zenn-user-upload/gdcekhgj81z337f83pddzfbe0gxj)

適当に Product Name などを埋めていきます。

![](https://storage.googleapis.com/zenn-user-upload/pc2dnr4qvlh9y9rn0dtpnl5a1rl1)

これでプロジェクトの準備は完了です。
この状態でビルドすると、Apple Watch のシミュレーターに Hello World と表示されるはずです。

![](https://storage.googleapis.com/zenn-user-upload/jkz9mwofilak0zndamqe0g6coa2h)


# 秒数ピッカー画面の作成

最初にタイマーの起動時間を設定するための秒数のピッカーを作ります。

`contentView`を以下のように変更します。

```swift
struct ContentView: View {
    @State var timerVal = 1

    var body: some View {
        VStack {
            Text("Timer \(timerVal) seconds").font(.body)
            Picker(selection: $timerVal, label: Text("")) {
                    Text("1").tag(1).font(.title2)
                    Text("5").tag(5).font(.title2)
                    Text("10").tag(10).font(.title2)
                    Text("30").tag(30).font(.title2)
                    Text("60").tag(60).font(.title2)
                }
            Button(action: {},label: {
                Text("Start")
            })
        }
    }
}
```

この状態で、プレビューを確認すると以下のようにタイマーの秒数選択画面が表示出来ているはずです。

![](https://storage.googleapis.com/zenn-user-upload/gixnspx8yx4869iypgc94l7un7kt)

簡単にコードの説明をします。
Swift UIでは、VueやReactのように宣言的UIでアプリの画面を構築していきます。
このコードで使ったコンポーネントは以下の通りです。

- `VStack`: 内部の要素を縦方向に並べる
- `Text`: テキストを表示する
- `Picker`: 内部の要素をピッカー形式で表示する
- `Button`: ボタンを表示する

また、それぞれのコンポーネントに引数で設定値、Structの初期化のクロージャとして、別のコンポーネントを与え階層構造を作ってUIを組み立てます。

以下コードの場合は、TextというコンポーネントのStructを`()`内のテキストで初期化し、`.font`でフォントサイズを設定している感じです。

```swift
Text("Timer \(timerVal) seconds").font(.body)
```

# 次の画面への移動

次にStartボタンを押した後に、タイマーをスタートさせたいのでその部分を実装します。
まず新しいViewを作ります。

```swift
struct TimerView: View {
    @Binding var timerScreenShow:Bool
    @State var timerVal:Int

    var body: some View {
        Text("\(timerVal)")
    }
}
```

ただ、`@State` として受け取った値を表示するだけの画面です。
次に最初の画面から、先ほど作ったViewに移動するためのコードを実装します。

```swift
struct ContentView: View {
    @State var timerVal = 1
    @State var timerScreenShow:Bool = false

    var body: some View {
        VStack {
            Text("Timer \(timerVal) seconds").font(.body)
            Picker(selection: $timerVal, label: Text("")) {
                    Text("1").tag(1).font(.title2)
                    Text("5").tag(5).font(.title2)
                    Text("10").tag(10).font(.title2)
                    Text("30").tag(30).font(.title2)
                    Text("60").tag(60).font(.title2)
                }
            NavigationLink(
                destination: TimerView(timerScreenShow: $timerScreenShow, timerVal: timerVal),
                isActive: $timerScreenShow,
                label: {
                    Text("Start")
                })
        }
    }
}
```

# 分からないところ
- View のストラクトの中にローカルのオブジェクトを作る
- Timer アプリの