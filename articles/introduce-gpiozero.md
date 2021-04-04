---
title: "[電子工作] GPIO Zeroで直感的にデバイスを制御する"
emoji: "🛵"
type: "tech"
topics: ["raspberrypi", "電子工作", "python"]
published: false
---

最近電子工作で使った [GPIO Zero](https://github.com/gpiozero/gpiozero) がとても良かったので紹介です。

# GPIO Zeroとは？

Raspberry Pi の GPIO（デバイス制御に使う汎用 I/O ポート）を直感的なインタフェースで制御するライブラリです。

https://github.com/gpiozero/gpiozero

例えば電子工作の Hello World である「L チカ」を行うコードを、通常の GPIO 制御ライブラリ[RPi.GPIO](https://pypi.org/project/RPi.GPIO/)と比べると以下のような違いがあります。

**RPi.GPIO**

```py
import RPi.GPIO as GPIO
import time

PNO = 17

GPIO.setmode(GPIO.BCM)
GPIO.setup(PNO, GPIO.OUT)

while True:
    GPIO.output(PNO, GPIO.HIGH) # 点灯
    time.sleep(1)
    GPIO.output(PNO, GPIO.LOW) # 消灯
    time.sleep(1)
```

**GPIO Zero**

```py
from gpiozero import LED

led = LED(17)

while True:
  led.blink(on_time=1, off_time=1) # 1秒間隔点滅
```

GPIO Zero の場合は`LED`モジュールを使うことで、わずかなコードで GPIO を制御していることを感じさせることなく直感的にデバイスを扱うことが出来ています。
今回の例では LED ですが、その他のデバイスにももちろん対応しています。

# インストール
[raspberrypi.org](raspberrypi.org)の通常の Raspberry Pi OS には標準でインストールされています。
なのでその OS を使っている場合は追加のインストール作業は不要です。

もし、Raspberry Pi Light やその他の OS を使っている場合は pip でインストールしましょう。

```bash
# for Python3
$ sudo pip3 install gpiozero

# for Python2
$ sudo pip install gpiozero
```
# サンプル

実際に GPIO Zero を使って動かしてみたコードを紹介します。
公式のレシピ集にも大量のコードと回路図があるのでそちらもおすすめです。

https://gpiozero.readthedocs.io/en/stable/recipes.html

## LEDの操作

[8BitDo](https://www.8bitdo.com/zero2/) のコントローラーで LED を操作した例です。

@ツイート[https://Twitter.com/KawamataRyo/status/1378168650029441024]


回路図
![](https://i.gyazo.com/1ec0bc14ce46ecd1e54827975605328c.png =500x)


```py:led.py
import struct
from gpiozero import LED, LEDBarGraph

# 8BitDo path
device_path = "/dev/input/js0"

# event
EVENT_FORMAT = "LhBB"
EVENT_SIZE = struct.calcsize(EVENT_FORMAT)

# btn
class BtnCode:
  Y_ON = [1, 1, 2]
  Y_OFF =  [0, 1, 2]
  X_ON = [1, 1, 3]
  X_OFF = [0, 1, 3]
  A_ON = [1, 1, 1]
  A_OFF = [0, 1, 1]
  B_ON = [1, 1, 0]
  B_OFF = [0, 1, 0]

# led
red = LED(17)
yellow = LED(22)
blue = LED(10)
white = LED(11)

with open(device_path, "rb") as device:
  while True:
    event = device.read(EVENT_SIZE)
    (_time, val, type, num) = struct.unpack(EVENT_FORMAT, event)
    code = [val, type, num]

    if code == BtnCode.Y_ON:
        print("Y ON")
        red.on()
    if code == BtnCode.Y_OFF:
        print("Y ON")
        red.off()
    if code == BtnCode.X_ON:
        print("X ON")
        yellow.on()
    if code == BtnCode.X_OFF:
        print("X OFF")
        yellow.off()
    if code == BtnCode.A_ON:
        print("A ON")
        blue.on()
    if code == BtnCode.A_OFF:
        print("A OFF")
        blue.off()
    if code == BtnCode.B_ON:
        print("B ON")
        white.on()
    if code == BtnCode.B_OFF:
        print("B ON")
        white.off()
```

## モーターの操作

8BitDo のコントローラーで DC モーターを操作した例です。

@tweet[https://Twitter.com/KawamataRyo/status/1378226047737454600]

回路図

![](https://i.gyazo.com/57a9c1a8680956c926492eb84f12d15c.png)

```py:motor.py
import struct
from gpiozero import Motor

# 8BitDo btn code
class BtnCode:
  UP_ON = [-32767, 2, 1]
  DOWN_ON = [32767, 2, 1]
  UP_OR_DOWN_OFF = [0, 2, 1]
  RIGHT_ON = [32767, 2, 0]
  LEFT_ON = [-32767, 2, 0]
  RIGHT_OR_LEFT_OFF = [0, 2, 0]

# 8BitDo path
device_path = "/dev/input/js0"

# event
EVENT_FORMAT = "LhBB"
EVENT_SIZE = struct.calcsize(EVENT_FORMAT)

# motor
motorA = Motor(forward=17, backward=22)
motorB = Motor(forward=10, backward=11)


with open(device_path, "rb") as device:
  while True:
    event = device.read(EVENT_SIZE)
    (_time, val, type, num) = struct.unpack(EVENT_FORMAT, event)
    code = [val, type, num]

    if code == BtnCode.UP_ON:
      print("UP_ON")
      motorA.forward()
      motorB.forward()
    if code == BtnCode.DOWN_ON:
      print("DOWN_ON")
      motorA.backward()
      motorB.backward()
    if code == BtnCode.UP_OR_DOWN_OFF:
      print("UP_OR_DOWN_OFF")
      motorA.stop()
      motorB.stop()
    if code == BtnCode.LEFT_ON:
      print("LEFT_ON")
      motorA.backward()
      motorB.forward()
    if code == BtnCode.RIGHT_ON:
      print("RIGHT_ON")
      motorA.forward()
      motorB.backward()
    if code == BtnCode.RIGHT_OR_LEFT_OFF:
      print("RIGHT_OR_LEFT_OFF")
      motorA.stop()
      motorB.stop()
```

# おわりに

以上「GPIO Zero で直感的にデバイスを制御する」でした。
GPIO Zero を使ってコードを書くことで、電子工作に対する苦手意識がかなり減った気がします。
今後も色々電子工作していきたです。
