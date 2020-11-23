---
title: "Swift UIで初めてのApple Watch アプリを作る"
emoji: "✌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "Xcode", "ios"]
published: false
---

せっかく Apple Watch を持っているので何かアプリを作りたいと思い、Swift UI を勉強中です。
最初に修作として作ったタイマーアプリの作り方をまとめます。

本業は Web フロントエンドなので、Swift 自体が初めてです。間違い等あれば、気軽にコメントいただけると嬉しいです🙏

# 作るもの

簡単なタイマーアプリを作ります。
ピッカーで秒数を選んで`GO`を押すと、タイマーが起動して、設定された秒数が経過すると、通知音とともにタイマーが終了します。


# 1. Xcode プロジェクトのスタート

Xcode を開き `Create a new Xcode project` で新しいプロジェクトを作成します。
最初の画面で何を作るのか選択できるので、Watch App を選択して Next を押します。 
![](https://storage.googleapis.com/zenn-user-upload/gdcekhgj81z337f83pddzfbe0gxj)

適当に Product Name などを埋めていきます。

![](https://storage.googleapis.com/zenn-user-upload/pc2dnr4qvlh9y9rn0dtpnl5a1rl1)

これでプロジェクトの準備は完了です。
この状態でビルドすると、Apple Watch のシミュレーターに Hello World と表示されるはずです。

![](https://storage.googleapis.com/zenn-user-upload/jkz9mwofilak0zndamqe0g6coa2h)


# 2. 秒数ピッカー画面の作成

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
Swift UI では、Vue や React のように宣言的 UI でアプリの画面を構築していきます。
このコードで使ったコンポーネントは以下の通りです。

- `VStack`: 内部の要素を縦方向に並べる
- `Text`: テキストを表示する
- `Picker`: 内部の要素をピッカー形式で表示する
- `Button`: ボタンを表示する

また、それぞれのコンポーネントに引数で設定値、Struct の初期化のクロージャとして、別のコンポーネントを与え階層構造を作って UI を組み立てます。

以下コードの場合は、Text というコンポーネントの Struct を`()`内のテキストで初期化し、`.font`でフォントサイズを設定している感じです。

```swift
Text("Timer \(timerVal) seconds").font(.body)
```

# 3. Timer画面への移動

次に Start ボタンを押した後に、タイマーをスタートさせたいのでその部分を実装します。
まず新しい View を作ります。

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
次に最初の画面から、先ほど作った View に移動するためのコードを実装します。

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

ポイントは、`Button`を`NavigationLink`に変更して、TimerView への遷移を実行している点です。
また、そのために`@State`として、`timerScreenShow`を宣言しています。

`timerScreenShow`と、`timerVal`は`NavigationLink`にて、`TimerView`を初期化する際に、引数として渡しています。

この状態でプレビューを起動すると以下のような画面になります。
ちゃんと`TimerView`に遷移していますね。

![](https://storage.googleapis.com/zenn-user-upload/quxomuee1vnua7bvlq27shpfmcrj)

# Timer機能の完成

次に Timer 機能を、完成させていきます。

`TimerView`を以下のように変更します。

```swift
struct TimerView: View {
    @Binding var timerScreenShow:Bool
    @State var timerVal:Int

    var body: some View {
        if timerVal > -1 {
        VStack {
            Text("\(timerVal)").font(.system(size: 40))
                .onAppear() {
                    Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
                        if self.timerVal > -1 {
                            self.timerVal -= 1
                        }
                    }
                }
            Button(action: {
                self.timerScreenShow = false
            }, label: {
                Text("Cancel")
                    .foregroundColor(Color.red)
            })
            .padding(.top)
        }
        } else {
            Button(action: {
                self.timerScreenShow = false
            }, label: {
                Text("Done!")
                    .font(.title)
                    .foregroundColor(Color.green)
            }).onAppear() {
                WKInterfaceDevice.current().play(.notification)
            }
        }
    }
}
```

いきなりコードが増えたのですが、一つづつ説明します。

最初のif文の分岐では`ContentView`から受け取った`timerVal`の値によってViewを切り分けています。
これは、タイマー終了時の画面を表示するためです。

```swift
if timerVal > -1 {
  // ...
} else {
  //...
}
```

次にタイマー部分のコードです。
タイマー部分では、`timerVal`の値を表示しながら、`onApper()`で、表示時に`Timer.scheduledTimer`を実行して、1秒ごとに`timerVal`の値をディクリメントしていきます。

Swift UIでは、`@State` や `@Binding` で宣言した変数が変化すると、Viewが再描画されるのでこれだけで、タイマーが実装できます。


```swift
Text("\(timerVal)").font(.system(size: 40))
    .onAppear() {
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
            if self.timerVal > -1 {
                self.timerVal -= 1
            }
        }
    }
```

最後に、完了部分のコードです。
`Button`コンポーネントに`onAppear()`で、`WKInterfaceDevice.current().play(.notification)`を実行しています。これは完了を音で知らせるための通知音です。

また、`action`として`self.timerScreenShow`に`false`を代入しています。
これでDone!のボタンを押すと最初の画面に遷移する動きが実現できています。

```swift
Button(action: {
    self.timerScreenShow = false
}, label: {
    Text("Done!")
        .font(.title)
        .foregroundColor(Color.green)
}).onAppear() {
    WKInterfaceDevice.current().play(.notification)
}
```

この状態でプレビューを起動すると以下のような画面になります。

![](https://storage.googleapis.com/zenn-user-upload/dq723lu2h7bwek3ruykvyph47zjd)

タイマー機能の完成です🎉


# UIの改善

今のままでも、タイマーとして機能するのですが、もうちょっと見栄えをよくしたいですよね。
プラスアルファとして、Apple Watchでよくみる進捗バーを追加してみます。

最初にバー部分を別Viewで実装します。

```swift
struct ProgressBar: View {
    var progress: Int
    var initialTime: Int

    var body: some View {
        ZStack {
            Circle()
                .stroke(lineWidth: 15.0)
                .opacity(0.3)
                .foregroundColor(Color.red)
            Circle()
                .trim(from: 0.0, to: CGFloat(min(Float(self.progress) / Float(self.initialTime), 1.0)))
                .stroke(style: StrokeStyle(lineWidth: 15.0, lineCap: .round, lineJoin: .round))
                .foregroundColor(Color.red)
                .rotationEffect(Angle(degrees: 270.0))
                .animation(.linear)
        }
    }
}
```

# 分からないところ
- View のストラクトの中にローカルのオブジェクトを作る
- Timer アプリの