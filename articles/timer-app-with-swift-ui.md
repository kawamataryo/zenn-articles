---
title: "はじめての Swift UI × watchOS 〜タイマーアプリを作る〜"
emoji: "⌚"
type: "tech"
topics: ["Swift", "SwiftUI", "watchOS", "AppleWatch"]
published: true
---

せっかく Apple Watch を持っているので何かアプリを作りたいと思い Swift UI を勉強中です。 Swift UI の入門として作ったタイマーアプリについてチュートリアル形式でまとめます。

※ 本業は Web フロントエンドなので、Swift 自体が初めてです。間違い等あれば、気軽にコメントいただけると嬉しいです🙏

# 作るもの

簡単なタイマーアプリを作ります。
ピッカーで秒数を選んで`Start`を押すとタイマーが起動して、設定された秒数が経過すると、通知音とともにタイマーが終了するという簡単なものです。

![](https://storage.googleapis.com/zenn-user-upload/a7e94acmr5dezah6fy4oyf3ql7zf)

コードは全て以下リポジトリにあります。

https://github.com/kawamataryo/swift-ui-sample-timer

# 1. Xcode プロジェクトのスタート

Xcode を開き `Create a new Xcode project` で新しいプロジェクトを作成します。
最初の画面で何を作るのか選択できるので、Watch App を選択して Next を押します。
![](https://storage.googleapis.com/zenn-user-upload/gdcekhgj81z337f83pddzfbe0gxj)

適当に Product Name などを埋めていきます。

![](https://storage.googleapis.com/zenn-user-upload/pc2dnr4qvlh9y9rn0dtpnl5a1rl1)

これでプロジェクトの準備は完了です。
この状態でビルドすると、Apple Watch のプレビューに Hello World と表示されるはずです。

![](https://storage.googleapis.com/zenn-user-upload/jkz9mwofilak0zndamqe0g6coa2h)


# 2. 秒数ピッカー画面の作成

準備が整ったので画面を作っていきます。

最初にタイマーの起動時間を設定するための秒数ピッカーを作ります。
`contentView`を以下のように変更します。

```swift
struct ContentView: View {
    @State var timeVal = 1

    var body: some View {
        VStack {
            Text("Timer \(self.timeVal) seconds").font(.body)
            Picker(selection: self.$timeVal, label: Text("")) {
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

また、それぞれのコンポーネントに引数やクロージャーで、設定値や別のコンポーネントを与え階層構造を作って UI を組み立てます。

以下コードの場合は、Text というコンポーネントの Struct を`()`内のテキストで初期化し、`.font`でフォントサイズを設定しています。
とても直感的で分かりやすいですね。

```swift
Text("Timer \(timeVal) seconds").font(.body)
```

# 3. Timer画面への移動

次に Start ボタンを押した後に、タイマー画面への画面遷移をさせたいのでその部分を実装します。

まず新しい View を作ります。
`@State` として受け取った値を表示するだけの画面です。

```swift
struct TimerView: View {
    @Binding var timerScreenShow:Bool
    @State var timeVal:Int

    var body: some View {
        Text("\(self.timeVal)")
    }
}
```

次に最初の画面から、先ほど作った View に移動するためのコードを実装します。

```swift
struct ContentView: View {
    @State var timeVal = 1
    @State var timerScreenShow:Bool = false

    var body: some View {
        VStack {
            Text("Timer \(self.timeVal) seconds").font(.body)
            Picker(selection: self.$timeVal, label: Text("")) {
                    Text("1").tag(1).font(.title2)
                    Text("5").tag(5).font(.title2)
                    Text("10").tag(10).font(.title2)
                    Text("30").tag(30).font(.title2)
                    Text("60").tag(60).font(.title2)
                }
            NavigationLink(
                destination: TimerView(timerScreenShow: self.$timerScreenShow, timeVal: self.timeVal),
                isActive: self.$timerScreenShow,
                label: {
                    Text("Start")
                })
        }
    }
}
```

ポイントは、`Button`を`NavigationLink`に変更して、ボタン押下で TimerView への遷移を実行している点です。
そのために`@State`として、タイマー画面の表示状態を制御する`timerScreenShow`を宣言しています。

`timerScreenShow`と、`timeVal`は`NavigationLink`にて、`TimerView`を初期化する際に、引数として渡しています。

この状態でプレビューを起動すると以下のような画面になります。
ちゃんと`TimerView`に遷移していますね。

![](https://storage.googleapis.com/zenn-user-upload/quxomuee1vnua7bvlq27shpfmcrj)

# 4. Timer機能の完成

次に Timer 機能を、完成させます。

`TimerView`を以下のように変更します。

```swift
struct TimerView: View {
    @Binding var timerScreenShow:Bool
    @State var timeVal:Int

    var body: some View {
        if timeVal > -1 {
        VStack {
            Text("\(self.timeVal)").font(.system(size: 40))
                .onAppear() {
                    Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
                        if self.timeVal > -1 {
                            self.timeVal -= 1
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

いきなりコードが増えたのですが、1 つづつ説明します。

最初の if 文の分岐では`ContentView`から受け取った`timeVal`の値によって View を切り分けています。
これは、タイマー終了時の画面を表示するためです。

```swift
if timeVal > -1 {
  // ...
} else {
  //...
}
```

次にタイマー部分のコードです。
タイマー部分では、`timeVal`の値を表示しながら、`onAppear()`（要素の表示にフックして処理を実行するもの）で、`Timer.scheduledTimer`を実行して、1 秒ごとに`timeVal`の値をディクリメントしていきます。

Swift UI では、`@State` や `@Binding` のアノテーションをつけて宣言した変数が変化すると、View が再描画されるのでこれだけで、タイマーが実装できます。


```swift
Text("\(self.timeVal)").font(.system(size: 40))
    .onAppear() {
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
            if self.timeVal > -1 {
                self.timeVal -= 1
            }
        }
    }
```

:::message
ここの実装は若干難があり、このままだとキャンセル時などにでも、タイマーがバックグラウンドで動いたままになってしまいます。
たぶん`Timer.scheduledTimer`の戻り値を変数で保持して、画面遷移維持に`invalidate()`でタイマーの破棄を行う必要がありそうです。
:::

最後に、完了部分のコードです。
`Button`コンポーネントに`onAppear()`で、`WKInterfaceDevice.current().play(.notification)`を実行しています。これは完了を音で知らせるための通知音の起動です。

また、`action`として`self.timerScreenShow`に`false`を代入しています。
これで「Done!」のボタンを押すと最初の画面に遷移する動きが実現できています。

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

タイマー機能が完成ですね。


# 5. UIの改善

今のままでも、タイマーとして機能するのですが、ちょっとまだ画面がシンプルすぎますよね。
プラスアルファとして、Apple Watch でよくみる進捗バーを追加してみます。

最初にバー部分を別 View で実装します。
要素を重なりで配置する`ZStack`で、円を作る`Circle`を 2 つ重ねて実装しています。
引数として受け取るのは、初期値の`initial`と、進捗の`progress`です。

```swift
struct ProgressBar: View {
    let progress: Int
    let initial: Int

    var body: some View {
        ZStack {
            Circle()
                .stroke(lineWidth: 15.0)
                .opacity(0.3)
                .foregroundColor(Color.red)
            Circle()
                .trim(from: 0.0, to: CGFloat(min(Float(self.progress) / Float(self.initial), 1.0)))
                .stroke(style: StrokeStyle(lineWidth: 15.0, lineCap: .round, lineJoin: .round))
                .foregroundColor(Color.red)
                .rotationEffect(Angle(degrees: 270.0))
                .animation(.linear)
        }
    }
}
```

これを`TimerView`に組み込みます。
Timer の値を`Text`で表示している部分を`ZStack`で囲み、先ほどの ProgressBar を表示しています。
また、初期値設定用の`initialTime`というプロパティを新たに追加しています。

```swift
struct TimerView: View {
    @Binding var timerScreenShow:Bool
    @State var timeVal:Int
    let initialTime:Int

    var body: some View {
        if timeVal > -1 {
            VStack {
                ZStack {
                    Text("\(self.timeVal)").font(.system(size: 40))
                        .onAppear() {
                            Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
                                if self.timeVal > -1 {
                                    self.timeVal -= 1
                                }
                            }
                        }
                    ProgressBar(progress: self.timeVal, initial: self.initialTime).frame(width: 90.0, height: 90.0)
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

`initialTime`は、`ContentView`で`TimerView`を描画するときに渡します。

```swift
destination: TimerView(timerScreenShow: self.$timerScreenShow, timeVal: self.timeVal, initialTime: self.timeVal),
```


この状態でプレビューをみると、タイマーの数値に合わせて良い感じにバーの状態も変わるはずです。

![](https://storage.googleapis.com/zenn-user-upload/vxe63y5c3tomq20x6ow416mmjphc)

これでタイマーアプリの完成です 🎉

# おわりに
以上「はじめての Swift UI × watchOS 〜タイマーアプリを作る〜」でした。

View の構築については、Vue, React で慣れ親しんだ宣言的 UI なので あまり抵抗なく実装出来そうです。ただ、他のビルドやメモリ処理、データの永続化、ネイティブ API の使い方はまだ何もわかりません😅

これから色々アプリを作っていく段階で勉強していきたいです。


# 参考

- [(2020) SwiftUI - watchOS - Timer App - 13 Minutes - YouTube](https://www.youtube.com/watch?v=PnAvYJGNTPQ)
- [Making a WatchOS app with SwiftUI from scratch, with data fetching | by Fernando Cervantes Alvarez | Mac O’Clock | Medium](https://medium.com/macoclock/making-a-watchos-app-with-swiftui-from-scratch-with-data-fetching-68981acf615f)