# 概要

単純にQiitaの記事を管理するためのリポジトリ

# 使い方

* プレビュー

```bash
npx qiita preview

Preview: http://127.0.0.1:8888
```

* 記事作成
  * 普通にpreviewから作る
  * あるいはCLI的にも作れる

```bash
npx qiita new hogehoge
```

* アップストリームへのプッシュ
  * masterブランチにマージされたらGHAが走って差分同期される
  * あるいはCLIで決めうちでもやれる

```bash
npx qiita publish hogehoge
```

* アップストリーム側からのプル

```bash
npx qiita pull
```

# 参考

* https://qiita.com/Qiita/items/666e190490d0af90a92b