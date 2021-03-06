# Elasticsearch Marvel 2.x はプロダクションでも無料で使えるので入れておこう
入社３日目の木戸です。入りみだれるチャットコミュニケーションにあたふたしつつも、社内のドキュメントなどを読みあさりながら、少しづつ会社にも慣れてきたかな？といった感じです。

そのうち「 Elasticsearch 入門シリーズ」でも連載しようかと考えているのですが、今回記念すべき１本目のブログは、

なぜか、Marvel です！ニッチです。

## Marvel
Marvel 2.x は Kibana 4 をベースに UI が再設計され、Elasticsearch 2.x を効率的に管理するための主要なメトリクスにフォーカスされていて、より監視しやすい UI になっています。

Marvel 1.x では有償の製品として提供されていました。Marvel 2.x は、Basic License を申請することで、開発でもプロダクションでも無料で使い続けることができますので是非導入しておきましょう。

**注意** マルチクラスタ 対応などの無料範囲外の機能が使いたい場合は、有償のサポートが必要になります。

今回ご紹介するのは、Basic License の導入方法です。主にシングルクラスタ構成の開発環境やプロダクション環境で、Marvel を使用し続けるための手順となります。

**参考** 無料範囲はシングルクラスタなので、１つのクラスタで複数ノード構成は OK なのです！

## 導入手順

### 1. Marvel agent plugin のインストール
以下の手順で、Elasticsearch の ノード毎に Marvel agent をインストールします。

```bash
# 1. License plugin のインストール
bin/plugin install license

# 2. Marvel agent のインストール
bin/plugin install marvel-agent

# 3. Elasticsearch の起動
bin/elasticsearch
```

### 2.  Marvel app のインストール
以下の手順で、Marvel app を Kibana のプラグインとしてインストールします。

```bash
# 1. Marvel app のインストール
bin/kibana plugin —install elasticsearch/marvel/latest

# 2. Kibana の起動
bin/kibana
```

### 3.  Marvel のインストールの確認
ブラウザで、http://0.0.0.0:5601/app/marvel へアクセスして、Marvel が正しくインストールされているか確認してください。正しくインストールされていれば、Elasticsearch の Cluster が認識され、以下のような画面が表示されます。

![marvel](https://raw.githubusercontent.com/KunihikoKido/docs/master/images/elasticsearch-marvel-2-x-basic-license-1.png)

よく見ると、Cluster 一覧の License が **Trial** になっています！

![marvel](https://raw.githubusercontent.com/KunihikoKido/docs/master/images/elasticsearch-marvel-2-x-basic-license-2.png)

このままでは、３０日で使えなくなってしまいますので、次の手順でライセンスの申請と更新をします。

### 4. Basic license の入手
Elastic 社のホームページの以下のページのフォームから必要事項を入力し、申請するとライセンスをダウンロードするための URL がメールで送られてきますので、手順に従ってライセンスファイルをダウンロードしてください。

Marvel » [Free License Registration](https://register.elastic.co/marvel_register)

### 5. ライセンスの更新
入手したライセンスファイルを使って、以下の手順で Marvel のライセンスを更新します。

```bash
# 1. ライセンスの更新
curl -XPUT 'http://localhost:9200/_license?acknowledge=true' -d @license.json
```

以下のようにレスポンスが帰って来れば成功です。

```js
{
  "acknowledged": true,
  "license_status": "valid"
}
```

### 6. ライセンス更新の確認
ライセンスの更新が完了したら、ブラウザで http://0.0.0.0:5601/app/marvel へアクセスして、ライセンスが更新されたか確認してください。

Basic License では、マルチクラスタ をサポートしていないため、Cluster 一覧ではなく 以下のように Overview のページが表示されます。

![marvel](https://raw.githubusercontent.com/KunihikoKido/docs/master/images/elasticsearch-marvel-2-x-basic-license-3.png)

これでめでたく Marvel を使い続けられます。また、API を使用してライセンスの情報を取得するには、以下のようにリクエストします。

#### ライセンスリクエスト方法
```bash
# 1. インストールされているライセンスの表示
curl -XGET 'http://localhost:9200/_license?pretty'
```

#### ライセンス結果例
ライセンスの情報は、こんな感じです。``license.type`` が、``basic`` になっていますね。

```js
{
  "license" : {
    "status" : "active",
    "uid" : "9fa6a1d9-ddb1-49f9-824d-e2eee5840d49",
    "type" : "basic",
    "issue_date" : "2016-01-28T00:00:00.000Z",
    "issue_date_in_millis" : 1453939200000,
    "expiry_date" : "2017-02-03T23:59:59.999Z",
    "expiry_date_in_millis" : 1486166399999,
    "max_nodes" : 100,
    "issued_to" : "Kunihiko Kido (Classmethod, Inc.)",
    "issuer" : "Web Form"
  }
}
```

これを見ると、最大ノード数は100で制限されているのかな？

## まとめ
Marvel は Elasticsearch の状態を把握するためには必須の製品です。
CPU やメモリ、Disk の使用率以外にも Search や Indexing などのパフォーマンスの状態／ JVM Heap の状態／ Shard の状態などを把握することができます。
シングルクラスタであれば、無料で使い続けることができますので、ぜひ導入しましょう！
