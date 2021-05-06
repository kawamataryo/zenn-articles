---
title: "ラズパイで 8BitDo Zero2 (Bluetoothコントローラー)を使うためのPythonパッケージを作ってみた"
emoji: "🎮"
type: "tech"
topics: ["Raspberrypi", "python", "8bitdo", "電子工作"]
published: true
---

Raspberry Pi で 8BitDo Zero2 を簡単に使うための Python パッケージを作ってみたので紹介です。

# 8BitDo Zero2とは？
[8BitDo](https:www.8bitdo.com/) という ゲームハードウェアの開発会社が販売している格安・小型の Bluetooth ゲームコントローラーです。Amazon で 2,000 円程度で購入出来ます。

![](https://i.gyazo.com/690ad6041167c81311d74eaa8e432359.png)
https://www.amazon.co.jp/-/en/6922621501121/dp/B081HML6MP

Switch などのゲームでも使えますし、Raspberry Pi と Bluetooth 接続し、ボタン押下のイベントで GPIO を操作することで、電子工作のインタフェースとして使うことも出来ます。

# 作ったもの

8BitDo Zero のボタン操作イベントに対して、簡単に処理を割り当てられる[eitghtbitdo-zero2](https://pypi.org/project/eightbitdo-zero2/)という Python パッケージを作りました。

https://github.com/kawamataryo/8bitdo_zero2

使い方はとても簡単です。以下のようにクラスの初期化時、割り当てたいボタンに関数を設定し、`listen()`を呼ぶだけで動作します。

```python
# コントローラーの初期化
controller = EightBitDoZero2(
  on_a=<任意の関数>,
  on_b=<任意の関数>,
  # ...
)

# ボタン押下イベント購読スタート
controller.listen()
```

※ 取り敢えず公開したものの、まだエラーハンドリング周りとか細かいところを全然作り込めてないです・・

# 仕組み

仕組みもとても単純です。8BitDo Zero2 のボタン押下時にファイルストリームに追記されるバイトコードを見て、初期化時に設定した特定の処理を実行しているだけです。

```python
class BtnCode:
    ON_UP = [-32767, 2, 1] #事前に検出した上ボタン押下時のコード
    # ...

class EightBitDoZero2:
    EVENT_FORMAT = "LhBB"
    EVENT_SIZE = struct.calcsize(EVENT_FORMAT)

    def __init__(self,
                 device_path="/dev/input/js0",
                 debug=False,
                 on_up=lambda: None,
                 # ...
                 ):
        self.device_path = device_path
        self.debug = debug
        self.on_up = on_up
        # ...

    def listen(self):
        with open(self.device_path, "rb") as device:
            while True:
                event = device.read(self.EVENT_SIZE)
                (_time, val, type, num) = struct.unpack(self.EVENT_FORMAT, event) #バイトコードの変換
                code = [val, type, num]

                if code == BtnCode.ON_UP:
                    if self.debug: # debug Trueの場合は、標準出力にどのボタンを押したかが出力される
                        print("UP_ON")
                    self.on_up() # 上ボタンが押された時用に登録した処理を実行
                # ...
```

https://github.com/kawamataryo/8bitdo_zero2/blob/master/eightbitdo_zero2/eightbitdo_zero2.py

# DEMO（8BitDo Zero2 + Lピカ）
8BitDo で LED をピカピカさせる例です。

## 1. Raspberry Piとの接続

まず、8BitDo Zero2 を Raspberry Pi と接続します。画面上の Bluetooth 設定もしくは`bluetoothctl`コマンドで接続してください。`bluetoothctl`での接続はこちらの記事が分かりやすいです。

https://qiita.com/takashi53/items/f6a866f0081609dbb8d6

接続できれば、`/dev/input/`配下に`js<number>`でファイルとして表示されるはずです。以下は`js0`で接続された場合の例です。

`cat`コマンドで接続されたファイルを開き、コントローラーを操作するとイベントが流れているのがわかります。これで接続は完了です。

![](https://i.gyazo.com/41911f7443658247146c7af793d25642.gif)


## 2. パッケージのインストール

PyPI からパッケージをインストールします。

```bash
$ pip3 install eightbitdo-zero2
```

https://github.com/kawamataryo/8bitdo_zero2

## 3. Lチカを実装

8BitDo Zero2 を`/dev/input/js0`に接続し、 17 番ピンに LED の回路を接続したと想定した場合のコード例です。
`EightBitDoZero2`の初期化時にコントローラーがつながっているファイルのパスと、A ボタンの on, off イベントの処理を渡しています。

※ もし 8BitDo Zero2 を`/dev/input/js0`以外に接続している場合は、初期化時に`device_path=<接続先>`の引数も追加してください。

```python:demo.py
from eightbitdo_zero2 import EightBitDoZero2
import RPi.GPIO as GPIO

GPIO.setmode(RPi.GPIO.BCM)
GPIO.setup(17, GPIO.OUT)

def main():
  def led_on():
    GPIO.output(17, True)
  def led_off():
    GPIO.output(17, False)

    controller = EightBitDoZero2(
      on_a=led_on,
      off_a=led_off,
    )

    # Start listen
    controller.listen()

if __name__ == "__main__":
  main()
```

コードを実行すれば、コントローラの A ボタンを押している間は LED が点灯し、離すと消灯するはずです。

```bash
$ python3 demo.py
````

少し拡張すれば、ボタン別に LED を操作することも簡単に出来ます。

https://twitter.com/KawamataRyo/status/1378168650029441024

他 [examples](https://github.com/kawamataryo/8bitdo_zero2/tree/master/examples) にサーボモーターや DC モーターの操作例もあります。

https://github.com/kawamataryo/8bitdo_zero2/tree/master/examples


# 終わりに

以上、「ラズパイで 8BitDo Zero2(Bluetooth コントローラー)を使うための Python パッケージを作った」でした。
コントローラーで操作できるようにするだけで、何だか面白いですし、子供ウケも抜群です。
電子工作のネタの 1 つとして使ってもらえると嬉しいです。