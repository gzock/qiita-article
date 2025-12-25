---
title: Cloudflare Workers にホスティングした Next.js アプリのセキュリティを多少なりとも向上させる
tags:
  - cloudflare
  - Next.js
private: false
updated_at: '2025-12-22T22:45:56+09:00'
id: be4c48f0bd2a3f4a7ead
organization_url_name: null
slide: false
ignorePublish: false
---
# 概要

今までAWSおよびGoogle Cloudをめちゃくちゃ使ってきたけど、令和7年になってCloudflareに手を出し始めた。めっちゃ楽。Google Cloud使い始めた時にも、これめっちゃ楽だな！って思ったけど、それ以上に楽。

何やってるかっていうと、Workersにブログをホストしている。自分好みのブログを作れるCMSがなくて自作した。

全然知らんかったのだが、普通にWorkersにのっけるだけだと、TLSまわりが全然弱い。こんだけマネージドで楽ちんにやれるサービスだと勝手に色々やってくれるのかな？と思っていたのだが案外そうでもなかった。

自前でどうにかせにゃならん！ということでやってみた。

# チェッカー

[Security Headers](https://securityheaders.com/) を使う。

# 結論

いきなりだが、Security Headersの中の人がこうすりゃ良い評価とれるぜっていう設定をブログで公開してくれている。本当に本当に超大感謝！
- [The brand new Security Headers Cloudflare Worker](https://scotthelme.co.uk/security-headers-cloudflare-worker/)

こちらを見てやればいいだけ。

# やってみる

私はNext.jsで作っていたのでそれベースで説明。
昔、チュートリアルをやって以来、まともにはじめて作るNext.jsなので、ぶっちゃけ全然詳しくない。

`next.config.mjs` にこんな感じで突っ込む。場所としてここが良いのかどうか自信はない。
元ネタとして [上述の中の人のブログ](https://scotthelme.co.uk/security-headers-cloudflare-worker/) を参考にしているのだが、 `Strict-Transport-Security` はちょっとチューニング。どうせただのブログだし...ってことで強めにpreloadをかけている。

```react:next.config.mjs
const nextConfig = {
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          {
            key: "Strict-Transport-Security",
            value: "max-age=63072000; includeSubDomains; preload",
          },
          {
            key: "X-XSS-Protection",
            value: "1; mode=block",
          },
          {
            key: "X-Frame-Options",
            value: "DENY",
          },
          {
            key: "X-Content-Type-Options",
            value: "nosniff",
          },
          {
            key: "Referrer-Policy",
            value: "strict-origin-when-cross-origin",
          },
          {
            key: "Content-Security-Policy",
            value: "upgrade-insecure-requests",
          },
          {
            key: "Permissions-Policy",
            value:
              "camera=(), microphone=(), geolocation=(), browsing-topics=()",
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```


# 確認

さくっと成功して `A+` ゲット！ マジで中の人ありがとう！超助かる！

![貼り付けた画像_2025_12_22_22_37.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/80163/6dab18e4-fc21-4cbc-bc21-3543bbf7d9ca.png)

# ちなみに...

昔からお馴染みの `X-XSS-Protection` は最近だと無効化しておいたほうがむしろ良いっていう意見もあるらしい。そのうち変えるかも？
