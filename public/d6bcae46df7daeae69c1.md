---
title: WisGate Developer Base RAK7371 を使ってLoRaWANゲートウェイを自作する -失敗編-
tags:
  - IoT
  - LoRaWan
private: false
updated_at: '2025-12-22T09:55:41+09:00'
id: d6bcae46df7daeae69c1
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

こちらは失敗編です。失敗してんなら記事にすんなって思うかもしれないが、まぁこういう試行錯誤もまとめておくことで、同じことにハマる人を減らしたい気持ちがある。ググればわかるが、これ系の情報は日本語もそうだが英語で調べてもあまり情報が出てこない。あるいは、この方法を応用した何かがいつの日か役にたつかもしれないので...

[成功編](https://qiita.com/gzock/items/a7e44e2bc6a521d26990)もあるので、お急ぎの方はそちらをどうぞ。

# まずはRAKとは

RAKとは、中国・深センのIoTメーカーである。

https://www.rakwireless.com/en-us/company

LoRa関連の製品を多数リリースしている。

ここがLoRaWAN GWを構築するためのデバイスをいくつか持っており、そちらを使って、自前のLoRaWANゲートウェイを作ってみる。

# ゴール

Raspberry Pi 5 + RAK7371を使ってオレオレLoRaWANゲートウェイを構築し、AWS IoT と接続する。

# 使用デバイス

- [WisGate Developer Base RAK7371](https://store.rakwireless.com/products/wisgate-developer-base?index=313&variant=39942858703046)
    - Semtech SX1303 を内蔵
- Raspberry Pi 5
    - ぶっちゃけLinuxマシンならなんでもよいのだが手元にあったRPI 5を使う

# デバイス選定理由

- RPIのHATも売ってたりするのだが、RPI に限らず汎用的に色々なデバイスで使いたかった
    - もしRPIしか使わんよ！っていう場合には別のやつを選んだ方が良いかもしれない
- こちらのDeveloper Base はUSB接続が可能であるためRPIはもちろん他の何に対しても使いまわせるのでこちらを敢えて選んだ

# 資料

- [**RAK7271/RAK7371 WisGate Developer Base Datasheet**](https://docs.rakwireless.com/product-categories/wisgate/rak7271-rak7371/datasheet/#overview)
- [**RAK7271/RAK7371 Quick Start Guide**](https://docs.rakwireless.com/product-categories/wisgate/rak7271-rak7371/quickstart/)

# 注意

RAKのLoRaWANデバイスは技適を取ってないケースが多い。必ず技適の有無を確認し、もし未認証の場合には、シールドルームの中で作業するとか、特例申請を出すなどして違法状態にならないよう事前に準備しておくこと。

# デバイスの存在確認

- dmesg
    - /dev/ttyACM0 というのがそれ

```text
[1289466.583158] cdc_acm 3-1:1.0: ttyACM0: USB ACM device
[1289466.583282] usbcore: registered new interface driver cdc_acm
[1289466.583287] cdc_acm: USB Abstract Control Model driver for USB modems and ISDN adapters
```

# まずは公式ガイドの通りやってみる

上述の [**RAK7271/RAK7371 Quick Start Guide**](https://docs.rakwireless.com/product-categories/wisgate/rak7271-rak7371/quickstart/) の通りやってみる。このあたりはそのままの話なので詳しく説明しない。

途中でモデルを選定するところでは`10`を選択。

``` console
$ git clone https://github.com/RAKWireless/rak_common_for_gateway.git
$ cd ~/rak_common_for_gateway
$ sudo ./install.sh
```

- だいたい2分ぐらいでインストール完了

```console
Updating hostname to 'rak-gateway'...

Copy sys_config file success!
~/workspace/rak_common_for_gateway
*********************************************************
*  The RAKwireless gateway is successfully installed!   *
*********************************************************
```

ここから先は [Connect to The Things Network V3 (TTNv3)](https://docs.rakwireless.com/product-categories/wisgate/rak7271-rak7371/quickstart/#connect-to-the-things-network-v3-ttnv3) に書かれてる内容をやる。

まずは`gateway-config` を叩いて設定を変える。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/80163/cc6f396f-1056-4631-be91-c235364eee23.png)

日本リージョンなので`1`を選択。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/80163/b2424ef1-f3c8-4403-98df-c62ffd08d16d.png)

ここまででインストールは完了。めっちゃ簡単！

# 問題

ここまでやって何となくわかったと思う。めっちゃ簡単なのはいいとして、あれ？ LNSとかCUPSの設定どうすんの？？？

公式ガイドでは The things Networkに接続する方法は書いてるけど、いやいや俺はAWSにつなぎたいんだよ。

ていうか、ものすごく嫌な予感がする。さっきgateway-config叩いたときにpacket_forwarderって書いてあった。

あと、インストール先を見てみるとやっぱり…

```console
$ ls -l

total 8
drwxr-xr-x 6 root root 4096 Dec 19 06:18 lora_gateway
drwxr-xr-x 6 root root 4096 Dec 19 06:18 packet_forwarder
```

あれ、これさ、CUPS対応してないよね。Basic Station じゃないもんね。

全文検索してみるが一切それっぽい単語が見当たらない… ちなみにこれは最初に持ってきたGitHubリポジトリの中でも同じ。

```console
$ grep -ir -e cups -e lns /opt/ttn-gateway
```

詰んだね。

# 公式ガイドを無視してBasic Station を使ってみる

どうせSemtechのSX130xを使ってる以上、そのままリファレンス実装のBasic Station使っていけないかな？？ USB接続のデバイスを参照することぐらいできそうな気もしないでもない。

公式はこちら。

https://github.com/lorabasics/basicstation

一応、exampleの中で以下のようにUSBデバイスを指定しているケースもあるので、いけそう？？ 試してみよう。

```console
$ RADIODEV=/dev/ttyACM0 ../../build-linuxpico-std/bin/station
```

READMEに書いてある通り、普通にmakeしてみる。自分はRaspberryPiを使ってるのでplatformとしてrpiを選択。variantはよくわからん。

```console
$ git clone git@github.com:lorabasics/basicstation.git
$ cd basicstation
$ make platform=rpi variant=std
```

はい。普通にエラーになる。

```console
$ make platform=rpi variant=std
setup.gmk:61: No toolchain for platform 'rpi' and local arch is not 'arm-linux-gnueabihf'
platform=rpi variant=std make -C deps/mbedtls
make[1]: Entering directory 'basicstation/deps/mbedtls'
../../setup.gmk:61: No toolchain for platform 'rpi' and local arch is not 'arm-linux-gnueabihf'
make[1]: Nothing to be done for 'all'.
make[1]: Leaving directory 'basicstation/deps/mbedtls'
platform=rpi variant=std make -C deps/lgw
make[1]: Entering directory 'basicstation/deps/lgw'
../../setup.gmk:61: No toolchain for platform 'rpi' and local arch is not 'arm-linux-gnueabihf'
make[1]: Nothing to be done for 'all'.
make[1]: Leaving directory 'basicstation/deps/lgw'
make -C build-rpi-std/s2core all
make[1]: Entering directory 'basicstation/build-rpi-std/s2core'
../../setup.gmk:61: No toolchain for platform 'rpi' and local arch is not 'arm-linux-gnueabihf'
 [arm-linux-gnueabihf] CC   aio.o
make[1]: NO-TOOLCHAIN-FOUND-gcc: No such file or directory
make[1]: *** [../../makefile.s2core:67: aio.o] Error 127
make[1]: Leaving directory 'basicstation/build-rpi-std/s2core'
make: *** [makefile:39: s-all] Error 2
```

どうもこいつは ARMv7ベース x 32bit OS用になっているようだ。手持ちはARMv8および64bit OSの状態なのでマッチしない。

`setup.gmk` を見ると、確かに確定で `arm-linux-gnueabihf` になるようになってる。

```text
ARCH.linux   = x86_64-linux-gnu
ARCH.linuxV2 = x86_64-linux-gnu
ARCH.linuxpico = x86_64-linux-gnu
ARCH.corecell  = arm-linux-gnueabihf
ARCH.rpi     = arm-linux-gnueabihf
ARCH.kerlink = arm-klk-linux-gnueabi
ARCH=${ARCH.${platform}}
```

ひとまずちょっと書き換えて先に進めるようにする。

```diff
+ ARCH.rpi     = aarch64-linux-gnu
- ARCH.rpi     = arm-linux-gnueabihf
```

こんな感じで成功！

```console
$ make platform=rpi variant=std
platform=rpi variant=std make -C deps/mbedtls
make[1]: Entering directory 'basicstation/deps/mbedtls'
make[1]: Nothing to be done for 'all'.
make[1]: Leaving directory 'basicstation/deps/mbedtls'
platform=rpi variant=std make -C deps/lgw
make[1]: Entering directory 'basicstation/deps/lgw'
make[1]: Nothing to be done for 'all'.
...
...
 [aarch64-linux-gnu] CC   web_linux.o
 [aarch64-linux-gnu] AR   ../lib/libs2core.a
 [aarch64-linux-gnu] CC   ../bin/station
 platform=rpi variant=std STATION EXE built
make[1]: Leaving directory 'basicstation/build-rpi-std/s2core'
```

試しにexampleを使って動かしてみる。

examples/live-s2.sm.tc に移動する。他のでもよいのだが、ここが一番それっぽかった。

詳細は割愛するが、AWS IoT で Gatewayを登録して証明書たちを落としておく。

それらを以下のように配置。

```console
$ ls -l cups.*
-rw-r--r-- 1 root root 1220 Dec 21 07:33 cups.crt
-rw-r--r-- 1 root root 1679 Dec 21 07:34 cups.key
-rw-r--r-- 1 root root 1606 Dec 21 07:32 cups.trust
-rw-r--r-- 1 root root   68 Dec 21 07:34 cups.uri
```

あとはREADMEに書かれてる通りに起動。

```console
$ RADIODEV=/dev/ttyACM0 ../../build-rpi-std/bin/station
```

ちなみに Gatewayを登録する時には↑で先に起動すると、起動ログの最初に `Station EUI` が表示されてるのでそれを使う。

あるいは、station.conf に以下のように記載する形でもOK. これで任意の値に強制できる。

```json
"station_conf": {
	"routerid": "your_gateway_id",
}
```

さてさて、これで起動すると…  アホほどログが流れるので最後だけ記載する。

```console
00000002025-12-21 14:42:59.840 [RAL:INFO] SX130x LBT not enabled
2025-12-21 14:42:59.840 [RAL:INFO] Station device: /dev/ttyACM0 (PPS capture disabled)
2025-12-21 14:42:59.841 [HAL:XDEB] [lgw_spi_open:94] ERROR: SPI PORT FAIL TO SET IN MODE 0
2025-12-21 14:42:59.845 [HAL:XDEB] [lgw_connect:520] ERROR CONNECTING CONCENTRATOR
2025-12-21 14:42:59.845 [HAL:ERRO] [lgw_start:742] FAIL TO CONNECT BOARD
2025-12-21 14:42:59.845 [RAL:ERRO] Concentrator start failed: lgw_start
2025-12-21 14:42:59.845 [RAL:ERRO] ral_config failed with status 0x08
2025-12-21 14:42:59.845 [any:ERRO] Closing connection to muxs - error in s2e_onMsg
2025-12-21 14:42:59.845 [AIO:DEBU] [3] ws_close reason=1000
2025-12-21 14:42:59.845 [AIO:XDEB] [3] ws_closing_w state=5
2025-12-21 14:42:59.845 [AIO:DEBU] Echoing close - reason=1000
2025-12-21 14:42:59.845 [AIO:XDEB] [3] socket write bytes=8
2025-12-21 14:42:59.849 [AIO:XDEB] [3] socket read  bytes=4
2025-12-21 14:42:59.849 [AIO:DEBU] [3|WS] Server sent close: reason=1000
2025-12-21 14:42:59.849 [AIO:DEBU] [3] WS connection shutdown...
2025-12-21 14:42:59.849 [TCE:VERB] Connection to MUXS closed in state -1
2025-12-21 14:42:59.849 [TCE:INFO] INFOS reconnect backoff 10s (retry 1)
```

こんな感じでエラーになって落ちる。どうもデバイスが開けてないらしい。

ていうかSPIで開こうとしている。なんでやねん！ そりゃダメに決まってるだろう。

ちなみに証明書系は大丈夫らしく、ログ的には無事にアップリンク側はひらけているように見えている。

よくREADMEを読む。USB系を使っているのは linuxpico ってやつだな… rpi じゃダメなんかな？

さっきと同じようなアプローチでmakeし直す。普通に成功。

```console
$ make platform=linuxpico variant=std
```

同じコンフィグを使って起動してみる。ビルドした成果物の出力パスが異なる点に注意。

```console
$ RADIODEV=/dev/ttyACM0 ../../build-linuxpico-std/bin/station 
```

再度起動してみる。先ほどとは違ってUSB接続できてる！ やはりmakeするときのターゲット次第でSPIかUSBかが決まるようだ。

が・・・ チップのバージョンが違うと怒られる。

```console
2025-12-21 14:57:04.052 [TCE:VERB] Connecting to MUXS...
2025-12-21 14:57:04.138 [TCE:VERB] Connected to MUXS.
2025-12-21 14:57:04.173 [S2E:WARN] Unrecognized region: AS923 - ignored
2025-12-21 14:57:04.173 [S2E:WARN] Unknown field in router_config - ignored: protocol (0xFD309030)
2025-12-21 14:57:04.173 [S2E:WARN] Unknown field in router_config - ignored: regionid (0xE6FFB211)
2025-12-21 14:57:04.173 [HAL:ERRO] [lgw_soft_reset:573] CONCENTRATOR UNCONNECTED
2025-12-21 14:57:04.173 [RAL:INFO] Lora gateway library version: Version: 0.2.2;
2025-12-21 14:57:04.177 [RAL:VERB] Connecting to smtcpico device: /dev/ttyACM0
2025-12-21 14:57:04.430 [HAL:ERRO] [lgw_connect:527] NOT EXPECTED CHIP VERSION (v130)
2025-12-21 14:57:04.680 [RAL:INFO] [LGW smtcpico] clksrc=1 lorawan_public=1
2025-12-21 14:57:04.681 [HAL:ERRO] [lgw_mcu_board_setconf:95] failed to configure board, ACK failed
2025-12-21 14:57:04.681 [RAL:ERRO] Concentrator start failed: lgw_board_setconf
2025-12-21 14:57:04.681 [RAL:ERRO] ral_config failed with status 0x08
2025-12-21 14:57:04.681 [any:ERRO] Closing connection to muxs - error in s2e_onMsg
2025-12-21 14:57:04.681 [AIO:DEBU] [3] ws_close reason=1000
2025-12-21 14:57:04.681 [AIO:DEBU] Echoing close - reason=1000
2025-12-21 14:57:04.687 [AIO:DEBU] [3|WS] Server sent close: reason=1000
2025-12-21 14:57:04.687 [AIO:DEBU] [3] WS connection shutdown...
2025-12-21 14:57:04.687 [TCE:VERB] Connection to MUXS closed in state -1
2025-12-21 14:57:04.687 [TCE:INFO] INFOS reconnect backoff 10s (retry 1)
```

ちなみにこのlinuxpicoというのはSemtechがリファレンス実装の1つとして出してる超小型のUSB接続可能なゲートウェイデバイスである。公式ではPicoCellと呼ばれている。
https://www.semtech.com/products/wireless-rf/lora-core/sx1308p868gw
※ 心の声: これ先に買えばよかったな... 失敗した, 手に入れよう

そのためUSB接続できる構造であるという点では合っているのだが、そりゃもちろんSemtechのリファレンスとRAKのデバイスではSX13シリーズであるのは合致するにせよ、それを制御するためのその他は変わるわけで、そこで怒られてるんだと思われる。まぁそりゃそうだとは思うのだが…

ちなみに 上述のエラーは `deps/smtcpico/platform-linuxpico/libloragw/src/loragw_reg.c` に実装されている。詳細は面倒なので割愛するが、こちらのソースコードをイジって当該処理をバイパスするようにしてみたが… まぁ当然の如くダメだった。そりゃそうだ、結局フロントのチップが違うから、イニシャライズのコマンドが通るわけもない、というこだ。

# 万事休す - 再検討

うーーーーん、まずい。これもしかして買うモノ間違えたのか？？ 間違えたっていうか、自分のやりたいこととは合致してないものを勘違いして調達してしまったというか。

ちなみに、[Issues](https://github.com/lorabasics/basicstation/issues)を見ていたら、まぁ似たようなことで困ってる人がちらほら。

そこで紹介があったのだが、xoseperezという方がフォークしたこのリポジトリだといくつか修正が加えられている。

https://github.com/xoseperez/basicstation

とはいえ、HEADからの修正点をみると、そこまで大きく変わるとは思えない… 少なくとも私がやりたいUSB接続のパターンで大きく変わるとは思えない。

件のsetup.gmkに手が加えられていて、こちらだとめっちゃ簡単に64bit用にビルドできる。すばらしい。

こちらは特に platform=corecell のときのパターンに手が加えられているので、オリジナルおよびこちらのリポジトリのコードを使って corecell でビルドして試してみたが、結果は変わらずであった。

うーーーん… とはいえさ、RAKも別に弱小メーカーではなく、ある程度は名の知れたメーカーである。旧来のPacket Forwarderのやり方しかサポートしてないとはあまり思えない。

RAKが作ったBasic Station相当のものがあれば解決するんじゃないか？ 実際、Semtechリファレンス実装のBasic Stationをフォークして独自のものを作っているメーカーがいくつかあるのは知っている。

探してみよう。そして、あった。上述のxoseperezさんが救世主であった。

[後編へ続く…](https://qiita.com/gzock/items/a7e44e2bc6a521d26990)
