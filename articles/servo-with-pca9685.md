---
title: "Raspberry Pi Zero + PCA9685で複数サーボモーターを動かす"
emoji: "🏍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['raspberypi', 'python', '電子工作']
published: true
---

Raspberry Pi Zero + PCA9685 で複数サーボモーターを動かすまでのメモです。

# 作るもの

この動画のように Raspberry Pi Zero で複数サーボモーターを動かします。

https://twitter.com/KawamataRyo/status/1388768862116618242

# 準備物

- [Raspberry Pi Zero](https://www.switch-science.com/catalog/3200/)
- [PCA9685](https://www.switch-science.com/catalog/961/)
- [MG996R](https://akizukidenshi.com/catalog/g/gM-12534/) * 3
- [電池ボックス](https://akizukidenshi.com/catalog/g/gP-04103/)

[PCA9685](https://www.switch-science.com/catalog/961/) は 16 チャンネルの PWM 出力ドライバー搭載基板です。I2C でラズパイと接続し、複数のサーボモーターや LED を操作出来ます。

https://www.switch-science.com/catalog/961/

# 回路図

![](https://i.gyazo.com/286aec57fbdb0addc6bccfbfd2946f63.jpg)

# 準備

## I2Cの有効化

ラズパイでは I2C 通信がデフォルトで OFF となっているので、`raspi-config`で I2C を有効化します。

```bash
$ sudo raspi-config
```

![](https://i.gyazo.com/7383e1c4681d4d5ffd137270ac337b3f.png =400x)

![](https://i.gyazo.com/5f3a1765260b3fc7398fd4a2887cda1c.png =400x)

Interfacing Options > P5 I2C > Enabled で OK です。

これで有効化されたので、回路図の通り配線して I2C の番号を確認します。

```bash
$ sudo i2cdetect -y 1
```

以下のような出力が出れば OK です。

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: 40 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: 70 -- -- -- -- -- -- --
```

## /boot/config.txtに通信速度の設定を追加
通信速度が早すぎるとエラーが出るのでその対策だそうです（あまり分かってない。.）

```bash
$ sudo vi /boot/config.txt
```

```bash
# 末尾に追加
dtparam=i2c_baudrate=10000
```

## モジュールのダウンロード

pip で PCA9685 のライブラリを追加します。

```bash
$ sudo pip install adafruit-pca9685
```



# コーディング

任意のディレクトリに`servo.py`を作り以下を記述します。

```python:servo.py
#!/usr/bin/python

import Adafruit_PCA9685
import time

def move(pwm, channel):
  time.sleep(1)
  pwm.set_pwm(channel, 0, 350)
  time.sleep(1)
  pwm.set_pwm(channel, 0, 600)

pwm = Adafruit_PCA9685.PCA9685()

pwm.set_pwm_freq(60)

move(pwm, 0)
move(pwm, 4)
move(pwm, 11)
```

実行します。

```bash
$ python servo.py
```

無事それぞれのサーボモーターが動けば完成です 🎉
