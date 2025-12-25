---
title: Meraki Scanning API を使って位置情報を可視化する -前編-
tags:
  - Cisco
  - NW
  - meraki
private: false
updated_at: '2025-12-18T23:12:25+09:00'
id: c6a5ebecb47b5b5137ae
organization_url_name: null
slide: false
ignorePublish: false
---
# なにそれ？

かなりマイナー機能だと思うのだが、Cisco Meraki には Scanning API なるものが存在する。
https://documentation.meraki.com/Wireless/Operate_and_Maintain/FAQs/Scanning_API_for_Location_Analytics_Solution

Merakiの各APとそれにぶらさがる端末たちとの位置関係を三角測量で推測する。その集計データをWebhoook的に通知してくれる機能。XY座標情報まで自動的に計算してくれた結果を教えてくれるので、ものすごく簡単にオフィス内での位置検知が可能になる。

APIと書かれてはいるが、こちらから叩きにいくような形ではなく、どちらかというとパッシブに待ち構えてデータを受け取るようなプッシュアーキテクチャになっている。ポーリング的なめんどうくさいことをする必要がないのでめちゃくちゃ楽。

社内で誰がどのへんにいるっていうのを可視化するソリューションは世の中に数多あるのだが、Merakiが導入されている環境であるならば、この手法を使うことで内製開発することも十分に可能なはず。

私が試した限り、確かにズレはあるのだが、思ったよりも誤差が少なく十分に実用できそうだな、と感じた。

もちろん技術的にいって場合によっては数mズレるのは当たり前にある話のはずなので、そういったレベルでの位置情報が欲しいならこの手法は向かない。が、大体どのへんにいるっていうのが知りたいだけなら全然使えると思う。

このMeraki Scanning APIについて 前編ではまずは設定からデータの連携まで。
[後編](https://qiita.com/gzock/items/b801f4fa916d0756102e)でそのデータを使って可視化するところまでやりたいと思う。

# 参考資料

ガチでこれらしか見てない。日本語情報はあんまりないので期待しないほうがいい。

- [公式ガイド](https://documentation.meraki.com/Wireless/Operate_and_Maintain/FAQs/Scanning_API_for_Location_Analytics_Solutions#Scanning_API_FAQ)
    - Scanning APIがなんのか、どんなことができるのか？はここ読めばわかる
- [Scanning APIについての詳細仕様](https://developer.cisco.com/meraki/scanning-api/introduction/)
    - 実際にScanning APIを使って何かやりたいよ、作りたいよっていうときにはここ読めばわかる
- webhook的に投げられてくるときの[RequestBodyのスキーマ](https://developer.cisco.com/meraki/scanning-api/3-0/#observation-payloads)

## ざっくりとしたMeraki側の挙動

### 1. Validate（GET）

Meraki Dashboard で Post URL を設定して **Validate** を押すと、Meraki がこちらで用意したAPIに **GET** してきて、**レスポンス本文が validator 文字列と一致**することを確認する

重要なポイントとして、GET と POST は **同じ URL パス** である必要がある。
- 例：`https://example.com/meraki/scanning` なら GET も POST もそこ

### 2. 本送信（POST）

Validate が通ると、Meraki クラウドが集計結果を約1分に1回毎に**JSON でまとめて POST** してくる。

ちなみに、APIにはバージョンがあってざっくり以下のような違いがある。まぁよほどのことがない限りV3を使えばOK.

- **v3**:
    - 三角測量
    - NW単位でまとまって届く
- **v2**:
    - 単純なRSSIベースの距離測定
    - AP 単位で届く (AP 台数が多いほど POSTトラフィック が増える)


# 設定

Network-wide > Configure > General > Location and scanning を設定する。

`Analytics enabled` と `Scanning API enabled` を設定。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/80163/90840d4a-2894-4417-92f9-c51ac8a4ec2c.png)

ここでぶん投げ先のAPIを指定する。これは自分たちで作らないとダメ。後述する。各種設定はぶっちゃけそのままデフォでOK. 

Secretはお好みの任意文字列を指定しておく。このSecretの値とValidatorの値をメモっておく。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/80163/40e951ae-f0b4-43fa-86ee-1fb58a91f106.png)


# 待ち構えるAPIの実装

さくっとAWS Lambdaを使って作ってみる。上述の挙動の話をサポートできるように作ればOK. 

あとは[関数URL](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/urls-configuration.html)を発行しちゃえばお手軽にScanning APIを試せる環境がいったん出来上がる。

例えばこんな感じ。メモっておいたvalidatorとsecretはいったん環境変数に突っ込んでおく。

ただし、これらは秘匿情報であるため、本来は環境変数ではなく、SecretManagerなどから取ってくる実装にすべき。今は単なる検証およびサンプルの説明であるため、そのへんは割愛。

```python
import os
import json
import base64

MERAKI_VALIDATOR = os.environ["MERAKI_VALIDATOR"]
MERAKI_SECRET = os.environ["MERAKI_SECRET"]
ALLOWED_PATH = "/meraki/scanning"

def _get_method(event) -> str:
    # HTTP API / Lambda Function URL
    m = (event.get("requestContext", {}).get("http", {}) or {}).get("method")
    if m:
        return m.upper()
    # REST API (old)
    m = event.get("httpMethod")
    return (m or "GET").upper()

def _get_path(event) -> str:
    # HTTP API / Lambda Function URL (typically requestContext.http.path or rawPath)
    p = (event.get("requestContext", {}).get("http", {}) or {}).get("path")
    if p:
        return p
    p = event.get("rawPath")
    if p:
        return p
    # REST API (old)
    return event.get("path") or "/"

def _get_query(event) -> dict:
    return event.get("queryStringParameters") or {}

def _get_body_text(event) -> str:
    body = event.get("body") or ""
    if event.get("isBase64Encoded"):
        try:
            return base64.b64decode(body).decode("utf-8", errors="replace")
        except Exception:
            return ""
    return body

def _resp(status: int, body: str, content_type: str = "text/plain; charset=utf-8"):
    return {
        "statusCode": status,
        "headers": {"Content-Type": content_type},
        "body": body,
    }

def lambda_handler(event, context):
    method = _get_method(event)
    path = _get_path(event)

    # 1) パスを固定（/meraki/scanning 以外は拒否）
    if path != ALLOWED_PATH:
        return _resp(404, "not found")

    # 2) GET: Meraki Validate
    #    - 本文に validator を"そのまま"返す
    if method == "GET":
        return _resp(200, MERAKI_VALIDATOR)

    # 3) POST: Scanning payload受信
    if method == "POST":
        raw = _get_body_text(event)
        if not raw.strip():
            return _resp(400, "empty body")

        try:
            payload = json.loads(raw)
        except json.JSONDecodeError:
            return _resp(400, "invalid json")

        # v3想定：root.secret を照合（合わなければ拒否）
        incoming_secret = payload.get("secret")
        if incoming_secret != MERAKI_SECRET:
            return _resp(403, "forbidden")

        # 必須っぽい要素を軽くチェック（必要なら強化してOK）
        # type/version/data などは環境により揺れるので、厳密にしすぎない
        if "data" not in payload:
            return _resp(400, "missing data")

        # ここまできたらあとはに煮るなり焼くなりお好きにどうぞ
        # 例えば加工して保存するとか/キュー投入など（重い処理は非同期のほうが良いと思う）
        # このサンプルでは単純にCloudWatch Logsに吐くだけ
        print(json.dumps(payload, ensure_ascii=False))

        return _resp(200, "ok")

    # 4) その他メソッドは拒否
    return _resp(405, "method not allowed")

```

この時点では単にログとして吐くだけなので、結果自体はCloudWatch Logsなどで確認することになる。中身を見てもらえれば、ばっちり各クライアントのXY座標が含まれていることがわかるはず。

ただし、Merakiの環境によってはあまりにも数が多すぎてCloudWatch Logsで表示しきれなかったりするので注意。 (私が試した環境がまさにそれだった)

参考情報に記載した通り、ここに対して投げられてくるリクエストボディは[こちら](https://developer.cisco.com/meraki/scanning-api/3-0/#wifi-properies)に書かれているので、必要に応じて煮るなり焼くなり加工して後続処理につなげられると良いかと思う。

# 注意点

APが何台あるか？どれだけのクライアントがぶらさがっているか？次第になるのだが、下手するとJSONの中に何千件って含まれていてものすごく重くなる。普通にリクエストボディサイズが数百KBから数MBとかになったりする。

どうせ1分に1回起動で大したAPI実行回数じゃないし、MEMは256MBか512MBぐらいには最低でもしておいたほうがいい。

まぁ大丈夫だとは思うが、1分毎にどデカいのが飛んでくるので24365で考えたときの従量課金分も注意。

もしかしたら、Meraki側でStatusが黄色くなってるかもしれない。おそらく、200返るまでおっせーよ！ っていう警告のはず。

上述の通り、デカすぎて時間がかかってる。特にこのスクリーンショットでの話は、後編で言及するCSVパースなどの処理も含めた時間なのでちょっと時間がかかりがち。

本ページでのLambdaであれば、さすがにこんなにはかからないのだが、まぁそれでもミリ秒範囲は無理で最低でも1secぐらいはかかっちゃっていた。

まぁ警告が出るだけで、実際の運用には影響はない。別に止められるとかもないので。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/80163/9dad8f5d-2c47-46dd-bea2-b237a109799c.png)

# さいごに

ここまでの内容で少なくとも Scannin API を使い始めることはできると思う。
多少なりともコードを書けるのなら、紹介したLambdaをちょちょっと改造して、様々な処理につなげて利用していけると思う。
ただ、言うてもここまでの内容だと、はーんって感じになってしまうかもなので、[後編](https://qiita.com/gzock/items/b801f4fa916d0756102e)にて実際にオフィスマップ上にプロットして可視化する部分までを紹介する。
