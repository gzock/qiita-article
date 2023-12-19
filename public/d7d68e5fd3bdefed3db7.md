---
title: OpenAM v13 on CentOS7
tags:
  - CentOS
  - SSO
  - OpenAM
  - シングルサインオン
private: false
updated_at: '2017-06-15T09:19:21+09:00'
id: d7d68e5fd3bdefed3db7
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
本記事は、[OpenAM/OpenIGを使用したシングルサインオン環境の構築](http://qiita.com/gzock/items/af6f820b5e872366e853)という記事から続く関連記事の一部です。

既にTomcatが構築してあることを前提に、OpenAMのインストールから初期セットアップをし、OpenAMが動作するまでを目指します。

# 前提
* OpenAMをインストールするサーバはCentOS7.3を想定
* OpenAMは無料で手に入る中では最新のv13.0を使用
* 既にtomcatはインストール済であること
    * 理論上はyumで入れるtomcatでも大丈夫なはずだが、私が検証した限りでは正常に動作しなかった
    * そのため、本記事内で使用するtomcatは公式から落とした手動でインストールしたものを使用している
    * ちなみにjettyはサポート対象外
* 検証目的の構成を行う (冗長構成を想定しない)

# 環境
* サーバOS: CentOS7.3
* ホスト名: sso.test.local
* IPアドレス: 172.16.1.130
* JDK Ver: 1.8.0_111
* Tomcat Ver: 8.0.39

# OpenAMサポートプラットフォームについて
**2017/06/15 terukizm様からのコメントにより追記しました。ありがとうございます。**
OpenAMはサポートしているOSやJava,Tomcatなどの制限が厳しく、既にインストール済のもの流用して作ろうとすると意外とサポートしていなくて途中でエラーになってしまったりすることがある。
詳しくは、[Release Notes](https://backstage.forgerock.com/docs/openam/13/release-notes)を読んでほしいのだが、いくつか記載する。
※下記は上記Release Notesからの転載です。

#### サポートOS
>Operating System|Version
>|:-:|:-:|
>Red Hat Enterprise Linux, Centos|6, 7
>SuSE|11
>Ubuntu|12.04 LTS, 14.04 LTS
>Solaris x64|10, 11
>Solaris Sparc|10, 11
>Windows Server|2008, 2008 R2, 2012, 2012 R2

#### サポートJDK
>Vendor|Version
>|:-:|:-:|
>Oracle JDK|7, 8
>IBM SDK, Java Technology Edition (Websphere only)|7

#### サポートWebアプリケーションサーバー
>Web Container|Version
>|:-:|:-:|
>Apache Tomcat|7, 8
>Oracle WebLogic Server|12c
>JBoss Enterprise Application Platform|6.1+
>JBoss Application Server|7.2+
>WildFly AS|9
>IBM WebSphere|8.0, 8.5

特にTomcatに関しては、v8といっても厳密にはv8.0.x と v8.5.xが存在している。
これに関しては、v8.5.xを使うと、後述するセットアップウィザードの途中で処理が停止してしまい、先に進めないとのこと。
事前にTomcatを用意する場合はv8.0.xを使用するように注意してほしい。
[https://forum.forgerock.com/topic/openam-13-with-tomcat-8-5-9/](https://forum.forgerock.com/topic/openam-13-with-tomcat-8-5-9/)

# OpenAMの入手
OpenAMはForgerockという会社が開発・公開しているので、以下のURLからOpenAM => OpenAM OEM => 13.0.0 と辿って、.warを落としてくる。
[https://backstage.forgerock.com/downloads/OpenAM](https://backstage.forgerock.com/downloads/OpenAM
)

恐らく`subscription only`というラベルが付いていてDLできないようになっていると思うが、これはサインインしていないため。
面倒くさいけど、まずは右上の`Register`から、アカウントを作ってサインインする必要がある。
ちなみに、サインインしたとしても`subscription only`になっているものに関しては、それはお金払わないと落とせないやつら。
一部、そういうのがある。

# 構築作業

## インストール
### OpenAM配置と所有者および権限変更
公式から頂戴した.warを対象サーバにアップロード。
※私は自PCに落としてsftpで持っていった
tomcat配置パス配下に`webapps`というフォルダがあるので、その中に.warファイルを配置。
あとはtomcatが起動時にこの.warを自動的に読み込んでくれる。

```
[root@sso ]# mv ~/OpenAM-13.0.0.war /opt/tomcat/webapps/openam.war
[root@sso ]# chown tomcat:tomcat /opt/tomcat/webapps/openam.war
[root@sso ]# chmod 755 /opt/tomcat/webapps/openam.war
[root@sso ]# 
[root@sso ]# systemctl restart tomcat
[root@sso ]# ss -nat | grep 8080
LISTEN     0      100                      :::8080                    :::*
```

### ホスト名変更とhosts修正
なぜかと言うと、OpenAMはFQDNでの利用が必須であるため。
IPアドレスでの利用は不可、最初の初期セットアップウィザードすら完了できない。
※本記事内では、OpenAMサーバに対して、`sso.test.local`というFQDNでアクセスするものとする

```
[root@sso ]# hostnamectl set-hostname sso.test.local
[root@sso ]# vim /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 sso.test.local
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.16.1.130    sso.test.local

[root@localhost ]# shutdown -r now
※再起動
```

## 初期セットアップ
* Webブラウザでhttp://[fqdn]:8080/openam にアクセス
    *   もし対象サーバとは別の例えばWindows PCなどからアクセスする場合はDNSで解決させるか、これもhostsで解決させる必要あり
* ウィザードに従ってセットアップを進めていくが、最初は必ず`カスタム設定`を選択すること
* 大体の設定は後から変更できるのであんまり神経質にならなくて良い
* ユーザデータストアはOpenAM組み込みを選択すること

1. トップページ、ここでは必ず`カスタム設定`の`新しい設定の作成`から次へ進む 
![2017-03-14-15-26-52.png](https://qiita-image-store.s3.amazonaws.com/0/80163/a104aa8a-ceb1-b544-6836-8e6d29d4076f.png)

1. よくあるライセンス承認なので気にせずcontinue (読める人はきちんと読みましょう) 
![2017-03-14-15-29-22.png](https://qiita-image-store.s3.amazonaws.com/0/80163/83203197-345b-31eb-1ffd-68a036523e73.png)

1. デフォルトの管理者アカウントとして、`amadmin`というアカウントが作成されるので、そのパスワードを設定する
![2017-03-14-15-29-57.png](https://qiita-image-store.s3.amazonaws.com/0/80163/cbb55ff1-fb41-7c6d-0441-3941ca6c3fd3.png)

1. OpenAMへアクセスする際のURLやcookieドメインを設定する。
ここはかなり重要で、後から変更できることはできるが、過去試したときはOpenAMがぶっ壊れて最初から作り直しになった経験がある。
cookieドメインは単純にアクセスするURLからホスト名を抜いたもので良い。つまり今回の場合、`.test.local`になる。
この辺は本来、自動的に設定され最初から入力されている部分だが、何の理由かわからないが、たまに値が間違っている(微妙に文字が足りない)ことがあるので、その場合は手動で入れ直してあげよう。
![2017-03-14-15-30-42.png](https://qiita-image-store.s3.amazonaws.com/0/80163/986b0b9d-9751-7927-978e-749c1f2c23eb.png)


1. 設定データストアの設定を行う。設定データストアの他には後述するユーザデータストアが存在するが、設定データストアの場合、その名の如く、OpenAM本体の様々な設定がここに保存される。
今回はOpenAM内部のデータストアに保存するようにするが、冗長構成であったり大規模構成である場合は、別途Directory Server(例えばForgerockのOpenDJ)を用意してあげて、そこに保存するようにしてあげないといけない。
![2017-03-14-15-31-17.png](https://qiita-image-store.s3.amazonaws.com/0/80163/db38d914-2526-0b35-6594-438becd8f607.png)


1. ユーザーデータストアは、OpenAMにログインするユーザーの在り処であり、結局はSSOをするユーザーである。
例えば、ActiveDirectoryであったり、OpenLDAPなど。
今はOpenAM内部の組み込みデータストアを指定する。注意書きの通り、検証環境のみで使おう。この辺は別記事でAD連携をさせるネタを書きます。
![2017-03-14-15-33-30.png](https://qiita-image-store.s3.amazonaws.com/0/80163/96f9c142-7219-2ee7-1dc3-8d528e252583.png)


1. 原則的に`いいえ`で良い。ただし冗長構成を組むときは、この辺の設定をしないといけない。
![2017-03-14-15-33-52.png](https://qiita-image-store.s3.amazonaws.com/0/80163/14008ef5-94f3-53a9-27f4-ae450b05ef33.png)

1. ポリシーエージェントのパスワードを決める。amadminのパスワードとは異なるものにしなければならない。
え？ポリシーエージェントってなんだって？
私も実はよくわかっていないが、別記事で書くWebエージェントを使用したSSOなどをするときに、エージェントとOpenAMが連携する際に裏側で動いているエージェントのよう。
OpenAMは様々なエージェントと連携することで、多様なユースケースのSSOを実現できるアプリケーションなので、エージェント云々というのは今後もたくさん出てくる。
![2017-03-14-15-34-24.png](https://qiita-image-store.s3.amazonaws.com/0/80163/7f02c5d2-dcb7-b46d-5412-b661a171d03d.png)

1. ただの確認なので問題なかったら、`設定の作成`に進む
![2017-03-14-15-34-49.png](https://qiita-image-store.s3.amazonaws.com/0/80163/8fa9cd9a-f565-2e56-1f80-f58460578982.png)

1. 色々がんばってくれている。祈ろう。私は原因不明のエラーで何度か失敗したことがある。その場合はもう一回やり直しです。
![2017-03-14-15-35-20.png](https://qiita-image-store.s3.amazonaws.com/0/80163/af6b0eab-43e8-e445-2248-8a3e6aca2ff5.png)

1. 問題なく完了すればこの画面になる。ログインに進もう。
![2017-03-14-15-39-32.png](https://qiita-image-store.s3.amazonaws.com/0/80163/803ca16f-9262-0dfb-ee3f-8deef13f7b38.png)

1. Webブラウザで、`http://[fqdn]:8080/openam`に移動すると、このログイン画面が表示されるはず。
ログインすると、OpenAMの管理画面が表示される。
セットアップウィザードの中で指定した`amadmin`とパスワードでログインを行う。
ちなみに、管理画面用途だけでなく、SSOをする際のユーザは全てこの画面を経由してログインを行うことになる。
ということは、見た目を変えたくなるわけだが、その辺は別記事に書きます。
※例えばロゴを会社のロゴに変えたりとか
![2017-03-14-15-47-27.png](https://qiita-image-store.s3.amazonaws.com/0/80163/37f88568-b3f0-5767-0f2d-feba4c9bdaba.png)

1. ログイン後画面。以降、全てこの画面から様々なOpenAMの設定変更を行う。
![2017-03-14-15-48-02.png](https://qiita-image-store.s3.amazonaws.com/0/80163/be7c3b83-f939-bce9-b053-c9a1c9d5eb03.png)

## 小ネタ
### 設定を間違ってしまった、もう1回やり直したい
初期セットアップ項の4.でパスを指定している部分があるが、こいつを削除かリネームしてtomcatを再起動すれば、もう1回初期セットアップを行える。
今回の場合は、`/home/tomcat/openam`
tomcat実行ユーザのホームディレクトリ配下/openamというのがデフォルトのパスであるらしい。
