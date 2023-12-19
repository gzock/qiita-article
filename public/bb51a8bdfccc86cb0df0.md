---
title: OpenIG 対httpsサイトについて
tags:
  - Java
  - SSL
  - SSO
  - シングルサインオン
  - OpenIG
private: false
updated_at: '2018-05-06T19:02:39+09:00'
id: bb51a8bdfccc86cb0df0
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
#### 本記事は、[OpenAM/OpenIGを使用したシングルサインオン環境の構築](http://qiita.com/gzock/items/af6f820b5e872366e853)という記事から続く関連記事の一部です

本記事では、OpenIGからリバプロ先のバックエンドサーバーがhttpsサイトだった場合に対する設定の説明を行う。

OpenIGでリバプロ先のバックエンドサーバーがSSLを使用していたとき、そのサーバーに対してOpenIGでは簡単には接続できない。
何故かと言えば、対象バックエンドサーバーに仕込まれているSSL証明書のことをOpenIGは知らないため、不正なSSL証明書に見えてしまい、結果接続が失敗する。
Javaに詳しい人ならよく知っていると思うが、Javaでは`keytool`という証明書や鍵の生成と管理を行うツールを使って、`keystore`というファイルに鍵と証明書を保存しておく。
そしてJavaのアプリケーションは基本的にこの`keystore`を読み込み、中の証明書などを使ってSSLの通信を行う。
そのため、OpenIGにおいても、httpsサイトに対してリバプロを行いたいのであれば、keytoolを使って接続先バックエンドサーバーが使用している証明書がインポートされたkeystoreを作成し、そのkeystoreをOpenIGのルートファイル上で読み込ませる必要がある。

ちなみに本記事の内容は、OpenIGのリファレンスガイドの1つである"openig-4-gateway-guide.pdf"に書かれている内容を、ほぼそのまま、日本語で少し説明をつけながらトレースしているだけなので、詳しいことは原文を読んでもらったほうが良いと思う。
また、後述するClientHandlerのSSL関連設定やTrustManager Objectなどルートファイル内の具体的な内容については、"openig-4-reference.pdf"が詳しいので、凝ったことをする場合は必ずそれらのドキュメントを確認して欲しい。

# keystore作成

keytoolやkeystoreの話は別にOpenIGに限った話ではなくJava全体の話であり、私が説明するよりずっと前から詳しく説明してくれている人がググればたくさん見つかるので、ざっと簡単に・・・
※デフォルトのkeystoreなど、既存のkeystoreに証明書をインポートして使う方法もあるが、本記事では新規にkeystoreを作成する方法を説明する

keytoolは非常にオプションが多彩で様々なことをやれるのだが、とりあえず手っ取り早く任意の証明書がインポートされたkeystoreを作成するには以下のコマンドを叩く。
JDKがインストールされていればkeytoolは至って普通に実行できるはずだが、もし実行できない場合はfindコマンドで場所を探して、そのパスに移動するか絶対パスで実行しよう。

```
keytool -import -trustcacerts -keystore [/path/to/keystore_filename] -file [/path/to/certificate_filename] -alias [alias_name] -storepass [password]
```

上記のコマンドで、実行するときに変更しないといけないのは、-keystoreに続くkeystoreのパス(任意のパスかつファイル名でok)、-fileに続くインポートする証明書のパス、-aliasに続くエイリアス名、-storepassに続くkeystoreのパスワードになる。

エイリアス名とはなんぞ？と思うかもしれないが、keystoreとは証明書管理のデータベースみたいなもので、中に入っている証明書を簡単に表すための別名が必要になる。
適切なエイリアス名さえ名付けておけば、中身的には似たような、同じような証明書であっても、ひと目で何に対するどんな証明書なのかわかる。
まぁ正直、OpenIG上ではそんなにたくさんの証明書を管理するわけではないし、適当な名前で良いと思う。
storepassは、keystoreは証明書の管理に使うものなので、当然のことではあるが、第三者の手で証明書の追加や削除が勝手に行われてしまったら、非常にまずいことになる。
そのため、それを保護するために必ずパスワードを必要とする。

というわけで、実環境で実行してみると以下のような感じになる。
インポートする証明書としては、OpenSSLで自前で署名した所謂オレオレ証明書を使用する。(Apacheで既に使用しているものと仮定)

```
[root@openig ~]# keytool -import -trustcacerts -keystore added_keystore -file /etc/httpd/certs/server.crt -alias httpd-cert -storepass hogehoge
Owner: CN=openig.test.local, O=Default Company Ltd, L=Default City, ST=Tokyo, C=JP
Issuer: CN=openig.test.local, O=Default Company Ltd, L=Default City, ST=Tokyo, C=JP
Serial number: 3ef3ff945930a1f2
Valid from: Wed Dec 28 11:14:53 JST 2016 until: Sat Dec 26 11:14:53 JST 2026
Certificate fingerprints:
         MD5:  41:E4:14:E2:0A:1D:6B:A9:95:03:19:D3:AA:BB:CC:DD
         SHA1: AA:BB:CC:DD:EE:FF:94:6E:96:FE:B5:B0:72:83:8F:15:3F:97:6F:2C
         SHA256: 22:DB:4E:97:89:CA:A1:5E:2D:05:AA:BB:CC:DD:EE:FF:2E:C1:64:37:B4:B2:C1:91:AF:9E:2D:41:2E:AF:9B:34
         Signature algorithm name: SHA256withRSA
         Version: 1
Trust this certificate? [no]:  yes
Certificate was added to keystore
```

最後に、"Trust this certificate?"と聞かれるので、証明書が間違ったものでないことを確認し、"yes"と答えよう。
これで、/etc/httpd/certs/server.crtがインポートされた状態のkeystoreである`added_keystore`が指定したパス、上記の例でいえばホームディレクトリ配下に作成される。

試しに中身を確認したいときは以下のコマンドを実行する。
パスワードを求められたら、keytoolコマンドを実行したときのstorepassを入力する。

```
[root@openig ~]#  keytool -v -list -keystore ./added_keystore
Enter keystore password: hogehoge

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 1 entry

Alias name: httpd-cert
Creation date: Feb 7, 2017
Entry type: trustedCertEntry

Owner: CN=openig.test.local, O=Default Company Ltd, L=Default City, ST=Tokyo, C=JP
Issuer: CN=openig.test.local, O=Default Company Ltd, L=Default City, ST=Tokyo, C=JP
Serial number: 3ef3ff945930a1f2
Valid from: Wed Dec 28 11:14:53 JST 2016 until: Sat Dec 26 11:14:53 JST 2026
Certificate fingerprints:
         MD5:  41:E4:14:E2:0A:1D:6B:A9:95:03:19:D3:AA:BB:CC:DD
         SHA1: AA:BB:CC:DD:EE:FF:94:6E:96:FE:B5:B0:72:83:8F:15:3F:97:6F:2C
         SHA256: 22:DB:4E:97:89:CA:A1:5E:2D:05:AA:BB:CC:DD:EE:FF:2E:C1:64:37:B4:B2:C1:91:AF:9E:2D:41:2E:AF:9B:34
         Signature algorithm name: SHA256withRSA
         Version: 1


*******************************************
*******************************************
```

# ルートファイル設定変更
次に作成したkeystoreをルートファイルに読み込ませる。
これもそれ用のObjectが用意されているため、さほど難しくはない。

まずはKeyStoreというObjectを作り、その中で作成したkeystoreとstorepassを指定する。
またそのKeyStore ObjectをTrustManager Objectで使用するように定義し、作成したTrustManager ObjectをClient Handler内で使用するように定義する。
これでこのClientHandlerを使用するルートファイルは指定したkeystoreにインポートされた証明書を使っているhttpsサイトに対して正常なSSL通信が出来るようになる。

```
{
    "name": "MyKeyStore",
    "type": "KeyStore",
    "config": {
        "url": "file:///home/jetty/added_keystore",
        "password": "hogehoge"
    }
},
{
    "name": "MyTrustManager",
    "type": "TrustManager",
    "config": {
       "keystore": "MyKeyStore"
    }
},
{
    "name": "MyClientHandler",
    "type": "ClientHandler",
    "comment": "Testing only: blindly trust the server cert for HTTPS.",
    "config": {
        "trustManager": "MyTrustManager"
    }
}
```

ちなみに接続だけなら上記の設定で十分可能なはずだが、それ以外にもClientHandlerにはいくつかSSL関連のプロパティとして以下のようなものが存在する。

* hostnameVerifier
* sslCipherSuites
* sslContextAlgorithm
* sslEnabledProtocols

説明しなくても恐らくプロパティ名だけでどんなものかイメージがつくと思う。
"openig-4-reference.pdf"のClientHandlerの項で詳しく説明されているので、目的のhttpsサイトにうまく接続できない、あるいは、もっとセキュアなSSL設定を行いたい、という場合は是非ドキュメントを確認して設定を施してみて欲しい。

# 備考
ベリサインとかグローバルサインなどに署名してもらった証明書を使ってる場合はどうなの？それでも上記のことやらないといけないの？と思うかもしれない。
非常に申し訳ないが、私はそういったきちんとしたhttpsサイトに対してOpenIGを使ったことがないので、何とも言えない・・・
だが、"openig-4-reference.pdf"によれば下記のように書かれており、

> If you do not configure a trust manager, then the client uses only the default
> Java truststore. The default Java truststore depends on the Java environment.
> For example, $JAVA_HOME/lib/security/cacerts.

実際に中身を確認してみると・・・

```
[root@openig ~]# keytool -v -list -keystore /usr/java/jdk1.8.0_111/jre/lib/security/cacerts
Enter keystore password: changeit

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 104 entries

※以下とても長いので省略
```

恐らく上記の104の中に入っているCAから署名された証明書なら、わざわざkeystore作ったりしないでもOKなはず。理論的には。
