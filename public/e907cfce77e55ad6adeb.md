---
title: dockerのlog-driverでjournaldを使ったときにコンテナ名を表示させる
tags:
  - Docker
  - systemd
  - syslog
  - journalctl
  - systemd-journald
private: false
updated_at: '2019-12-29T14:06:38+09:00'
id: e907cfce77e55ad6adeb
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

docker [create|run]させるときにlog-driverオプションにてロギング方法を色々と切り替えることができる。
この辺は他のQiitaの記事でも散々既出だと思うので詳細は割愛。

私はlog-driver journaldを多用しているのだが、普通にドライバーを切り替えただけだと、ログの中にCONTAINER_IDしか入っておらず、どのコンテナから出力されたものなのかがイマイチよくわからない。やはりコンテナ名がログに載って欲しい。そっちの方が絶対わかりやすい。

まぁ特定コンテナのログを見たい、という観点で言うならば、`docker logs`使うとか、`journalctl CONTAINER_NAME`を使うとか方法はあるのだけど。

個人的に困ったのは、journaldからsyslog転送されて/var/log/messagesに流れてきた場合。この場合、ログの中身でしか"どのコンテナがどういったログを出力したのか"を判断できない。そうなったときに、やはりログの中にコンテナ名が記載されていると非常に便利なわけだ。

ということで、それを実現するための設定についての記事。


# デフォルトの挙動
`--log-driver jounarld`を指定し、単に"hogehoge"と出力させるコンテナを動かしてみる。

```console
$ docker run --log-driver journald --name test-container centos:7 echo hogehoge
hogehoge
```

journalctlを使って特定コンテナのログを確認してみる。この場合、明示的にコンテナ名でフィルタリングしているわけだから、まぁログの中にコンテナ名が記載されていなくても特に問題はないだろう。なぜならコマンド実行時点で特定のコンテナを指定しているのだから、表示されるログは常にそのコンテナからのものであると言い切れる。

```
$ journalctl CONTAINER_NAME=test-container
-- Logs begin at Sun 2019-12-29 13:30:27 JST, end at Sun 2019-12-29 13:46:13 JST. --
Dec 29 13:41:06 test-svr 0c642bc6d137[5193]: hogehoge
```

RHEL/CentOSだと、特に何も設定しなくてもjournaldからsyslog転送されて、自動的に/var/log/messagesにまで流れてくる。私の場合、全てのコンテナのlog-driverをjournaldにすることで、全てのログを/var/log/messagesに集約し、そこに対してエラーログ監視をしていたりする。
この場合、どのコンテナが出力したものかがわからない。ログの中にある"0c642bc6d137"はコンテナIDだが、いちいち自分が作ったコンテナのIDを記憶している人間はいないだろう。

```console
$ grep hogehoge /var/log/messages
Dec 29 13:41:06 test-svr 0c642bc6d137: hogehoge
```


# 解決策

`--log-opt tag="{{.Name}}"`を指定する。--log-optは本来ログドライバーに対して様々なオプション設定を行うものである。その--log-optの`tag`に対して、任意文字列を与えると、ログの中でその文字列が表示される。もちろん、スタティックな文字列でも構わないのだが、今回の場合、正しいコンテナ名を動的に入れて欲しい、ということでコンテナ名を表すテンプレート要素である`{{.Name}}`を指定している。つまり、`tag="test-container"`としても、同じ結果になる。

```console
$ docker run --log-driver journald --log-opt tag="{{.Name}}" --name test-container centos:7 echo hogehoge
hogehoge
```

# 動作確認

journalctlで確認してみる。確かに以前はコンテナIDであった部分がコンテナ名に置き換わっている。

```console
$ journalctl CONTAINER_NAME=test-container
-- Logs begin at Sun 2019-12-29 13:30:27 JST, end at Sun 2019-12-29 13:54:18 JST. --
Dec 29 13:41:06 dev-harvest 0c642bc6d137[5193]: hogehoge
Dec 29 13:54:18 dev-harvest test-container[5193]: hogehoge
```

肝心の/var/log/messagesでも確認してみる。当然、こちらもコンテナ名に置き換わったログで流れてきている。

```console
$ grep hogehoge /var/log/messages
Dec 29 13:41:06 dev-harvest 0c642bc6d137: hogehoge
Dec 29 13:54:18 dev-harvest test-container: hogehoge
```

# docker-composeの場合

こんな感じ。

```text:docker-compose.yml
version: "3"
services:
  test-container-service:
    image: centos:7
    container_name: test-container
    logging:
      driver: journald
      options:
        tag: "{{.Name}}"
```

# 参考

* [https://docs.docker.com/config/containers/logging/journald/](https://docs.docker.com/config/containers/logging/journald/)
