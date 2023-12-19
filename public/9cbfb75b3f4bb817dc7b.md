---
title: 自動的に作成される仮想NICのMACアドレスをイベントドリブンに変更する方法
tags:
  - Linux
  - NIC
  - udev
private: false
updated_at: '2021-01-11T11:31:46+09:00'
id: 9cbfb75b3f4bb817dc7b
organization_url_name: null
slide: false
ignorePublish: false
---
# 概要

Linuxでは簡単にNICのMACアドレスを変更できる。
仮にベアメタルマシンに対して直接Linuxをインストールしているような状況であっても同様。
物理的なMACアドレスは変わらなくても、OSとして認識する論理部分としてMACアドレスを上書きできる。

一方で、現代のOpenStackやK8sなどでは、仮想的なNICが自動的に作成されるシーンが非常に多い。
こういったシーンで、仮想的なNICのMACアドレスをどーーしても変更したいときの方法について書きたい。

もちろん仮想NICが作成後に、手動でそれを変更することは簡単に可能。
だが作成後に瞬時に通信開始するようなシーンの場合、それでは間に合わないこともある。何より面倒くさい。
なので自動的にやりたい。そういったときの方法を簡単ではあるが記事として残しておく。

正直、そんな状況なんてなかなかないとは思うが・・・一応誰かの役に立てれば幸い。
※私の場合、OpenStack on OpenStackな環境を作っていて、こういう状況に見舞われた

そしてもっと良い方法があれば是非教えてください！

# 方法

udevさんを使って自動的に変更してもらう。マジでudevさん便利すぎ。

以下のようなルールファイルを作成し、/etc/udev/rules.d/配下に置いておく。配置しておくだけで自動的に機能してくれる。

* `KERNEL`による条件式で対象NICを判定する。
	* ここでは`*`なども使える。
	* しかし特定NICに特定MACアドレスを付与したいはずなので、多くはNIC名を直書きだと思う。
* `RUN`で任意のコマンドを実行可能。ここでipコマンドを叩いてMACアドレスを変更させる。
* `$name`にはNIC名が代入されている。RUNと組み合わせて使用することで特定のNICを狙い撃ちすることが出来る。
* `[macaddress]`には変更したいMACアドレスを指定する。

```
ACTION=="add", SUBSYSTEM=="net", KERNEL=="[nic_name]", RUN+="ip link set dev $name address [macaddress]"
```

上記は単一ルールファイルに複数行書いておくことも可能。MACアドレス変更系は1つのファイルにまとめて書いておいたほうが便利だと思う。

* 例

```/etc/udev/rules.d/75-mac-spoof.rules
ACTION=="add", SUBSYSTEM=="net", KERNEL=="vlan1000", RUN+="ip link set dev $name address 0a:1b:2c:3d:4e:5f"
ACTION=="add", SUBSYSTEM=="net", KERNEL=="vlan2000", RUN+="ip link set dev $name address 00:11:22:33:44:55"
ACTION=="add", SUBSYSTEM=="net", KERNEL=="vlan3000", RUN+="ip link set dev $name address aa:bb:cc:dd:ee:ff"
```
