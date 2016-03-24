# 第２回 Elasticsearch 入門 データスキーマ設計のいろは
第2回目の Elasticsearch 入門は「データスキーマ設計のいろは」です。

設計と言うほどでもないのですが、例えば RDB で検索にフォーカスした設計や、他の検索エンジンも経験していると、これまでの制限や習慣で Elasticsearch の特徴を生かせない設計をしてしまう事があるので、このテーマにしてみました。

それではインデックスするためのデータ構造を Elasticsearch でどのように設計するのか解説したいと思います。

## 設計フローまで変えてしまう画期的なドキュメント指向型検索エンジン
Elastic 社のホームページを見てみると Elasticsearch の特徴の１つとして「Document-Oriented」と言う記載があります。直訳すると「ドキュメント指向」です。

簡単に説明すると

> 現実世界の複雑なデータをJSONドキュメントにしてインデックスするだけで、デフォルトで全てのフィールドにインデックスが作られて、検索のスピード落とさずに検索できるよ。

という特徴を持っています。

言い換えれば、ビジネス要件を保ったままインデックス化するためのドキュメントが定義可能であるということです。（余計に分かりにくいw）

例えば、以下のユーザー情報を表すデータでは、「１つのユーザー情報に対して名前や年齢などの情報をそれぞれ１つ持つことができ、さらにSNSの種類とそのIDの組み合わせを持つ複数のアカウント情報を持つことができる。」というビジネス要件を持ったデータがあるとしましょう。

```
---
  name: "Kunihiko Kido"
  age: 39
  confirmed: true
  join_date: "2014-06-01"
  location:
    lat: 51.5
    lon: 0.1
  accounts:
    -
      type: "facebook"
      id: "kunihikokido"
    -
      type: "twitter"
      id: "9215"
```

このデータはそのビジネス要件を保ったまま以下のように Elasticsearch のインデックス可能なドキュメントとして定義することができるのです。

```js
{
	"name": "Kunihiko Kido",
	"age": 39,
	"confirmed": true,
	"join_date": "2014-06-01",
	"location": {
		"lat": 51.5, "lon": 0.1
	},
	"accounts": [
		{"type": "facebook", "id": "kunihikokido"},
		{"type": "twitter", "id": "9215"}
	]
}
```

すらばらしい！

他の検索エンジンや RDB でを使って、検索のパフォーマンスにフォーカスしたデータ設計をする場合、そのデータ構造は極力フラットに、RDBであれば関連テーブルを極力なくして非正規化していくのがこれまでの常識でした。

この設計方法は、今までのシステムの制限や仕組みの制約上仕方のない設計でしたが、この非正規化方法はビジネス要件（ビジネスを表現するデータの構造）を満たす検索の実現が難しかったり、検索のために非正規化したデータは、検索要件とかなり密接に関わるため、検索対象の元データは同じでも、検索ニーズが変わってしまうとそのデータ構造さえガラッと変わってしまいます。（仕様変更がアプリ側もデータベース側もすごく大変なのです。）

これらの背景から Elasticsearch のドキュメント指向はとても画期的で、これまでの検索アプリケーションの設計フローまで変えてしまいます。

データスキーマ設計フロー比較してみます。

**これまでの検索エンジン（検索指向型）では**

1. 検索ニーズを全て洗い出す（検索要件、集計要件、ソート要件）
2. 使用する検索エンジンのクエリー仕様の把握（検索仕様、集計仕様、ソート仕様）
3. 検索ニーズ、検索対象データ、クエリー仕様をもとにデータを非正規化してスキーマを設計

検索側の要件を先行して、データ設計と並行で進めるイメージ。途中で大きな変更が必要になる場合も。。また、ビジネス要件を満たすデータ構造が難しいため、例えば、フィールド数を減らすためにフラグ系のフィールドデータの値を１つのフィールドにまとめてコード化したものをインデックスしましょうとか、データ構造以外にデータの内容も工夫を繰り返し、利用者（フロントエンドの開発者など）が仕様を把握するのが難しい設計になりがち。

**Elasticsearch（ドキュメント指向型）では**

1. 検索対象データのビジネス要件を JSON 構造へ落とし込む（基本これだけ）

（後で、検索要件にあったクエリーを考える、Query DSL 調べる）

ちゃんと検索対象データのビジネス要件を理解して設計していれば、途中の変更はフィールドの追加など小さな変更ですみます。

Elasticsearch でも複雑な構造のデータの場合は、JSON に落とし込みつつ、検索、集計、ソートがビジネス要件を満たせるか実際にクエリーを書いて検証しながら進めたり、４階層とか５階層とかネストしすぎる場合は、ビジネス要件を満たしつつ階層を減らせないか考えたりしたほうが安心ですが、基本的には、様々なデータ型や、それに対応した検索・集計・ソート機能が提供されているので、データスキーマの設計では、検索対象データのビジネス要件にフォーカスして設計してしまっても途中で大きな変更はないはずです。

## フィールド名はわかりやすくシンプルに統一性を重視
インデックスのために設計した JSON は検索結果でも使用されます。Elasticsearch に限ったことではないですが、フィールド名はわかりやすくシンプルに統一性を重視して命名しましょう。

### キャメルケース or スネークケースに統一する
例えば表記方法では、キャメルケースまたはスネークケースに統一する。設計者の趣味もあると思うので、どちらがオススメというのはありませんが、どちらかに統一することをお勧めします。

_キャメルケース:_

```js
{"userId": 1, "createDate": "2016-03-15"}
```
or

_スネークケース:_

```js
{"user_id": 1, "create_date": "2016-03-15"}
```

ちなみに、Elasticsearch のすべての REST APIs には、``case`` パラメータが許可されていて、``camelCase`` をセットすると、そのレスポンスのJSONをキャメルケースに統一することもできます。（インデックスするドキュメントのフィールドは自分で統一）

_``case=camelCase`` オプション無し:_

```js
# curl 'localhost:9200/?pretty=true'
{
  "name" : "Gloom",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.2.0",
    "build_hash" : "8ff36d139e16f8720f2947ef62c8167a888992fe",
    "build_timestamp" : "2016-01-27T13:32:39Z",
    "build_snapshot" : false,
    "lucene_version" : "5.4.1"
  },
  "tagline" : "You Know, for Search"
}
```


_``case=camelCase`` オプション有り:_

```js
# curl 'localhost:9200/?pretty=true&case=camelCase'
{
  "name" : "Gloom",
  "clusterName" : "elasticsearch",
  "version" : {
    "number" : "2.2.0",
    "buildHash" : "8ff36d139e16f8720f2947ef62c8167a888992fe",
    "buildTimestamp" : "2016-01-27T13:32:39Z",
    "buildSnapshot" : false,
    "luceneVersion" : "5.4.1"
  },
  "tagline" : "You Know, for Search"
}
```


### データの内容が想像できるフィールド名称にする
検索アプリケーションを実装するにしても、Kibana で分析するにしてもデータスキーマ設計書を見なくてもフィールド名でその値が想像できるような名前をつけましょう。

_悪い例:_

以下の例では、ユーザーの名前なのかメールアドレスなのかフィールド名だけではわかりません。

```js
{
  "user": "Kunihiko Kido"
}
```

_良い例その１:_

``_name`` をサフィックスにつけることで、ユーザーの名前がストアされていることがわかります。

```js
{
  "user_name": "Kunihiko Kido"
}
```

_良い例その２:_

複数の情報がある場合は、オブジェクトにまとめるのも良い方法です。

```js
{
  "user": {
    "name": "Kunihiko Kido",
    "twitter_id": "@9215",
    "site_url": "https://medium.com/hello-elasticsearch"
  }
}
```
#### フィールド命名規約を作成することで将来的な設計コストも削減できる
データの内容が想像できるフィールド名称にすることで、わかりやすくなるだけでなく、ルール化もされていくため、Elasticsearch のインデックス・テンプレートを使って、アナライザーやマルチフィールド、適用するデータ型など、マッピング定義をルール化することも可能です。そうすると新しいフィールドが追加された時の設計コストが削減できます。

（極端な話、言語処理など検索エンジン特有の設計を知らないエンジニアでもフィールド命名規約さえ守れば、適切な型や言語処理が適用されたフィールドを追加できる仕組みを作ることができる。）

## Nested データ型と Object データ型の違いを理解しておくと幸せ
Elasticsearch は オブジェクトの配列構造をインデックスする方法として、Nested データ型をサポートしています。これは Object データ型の特殊な型です。今回はスキーマ設計ということなので詳細な型の設計は含まれていませんが、もしデータ構造の中にオブジェクトの配列構造を持った属性が複数あって、Nested データ型と Object データ型を使い分ける場合は、フィールド名称でその違いがわかるように設計するという考え方もあるので、説明しておきます。

以下の例では、``users`` というフィールドに ``first_name`` と ``last_name`` という対になっている情報があります。このデータが Nested データ型または Object データ型の対象データです。

```js
PUT my_index/my_type/1
{
  "group": "fans",
  "users": [
    {
      "first_name": "John",
      "last_name": "Smith"
    },
    {
      "first_name": "Alice",
      "last_name": "White"
    }
  ]
}
```

通常 ``users`` は単に Object データ型としてインデックスされます。インデックスされたデータは以下のように変換されるため、``first_name`` と ``last_name`` の組み合わせは維持されません。

```js
{
  "group": "fans",
  "users.first_name": ["alice", "john"],
  "users.last_name": ["smith", "white"]
}
```

そのため、以下のような組み合わせ違いの検索条件でも上記のドキュメントはマッチします。

```js
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"users.first_name": "Alice"}},
        {"match": {"users.last_name":  "Smith"}}
      ]
    }
  }
}
```

また、以下のような Aggregation を使った多段の集計では、``alice`` の集計結果に、``smith`` も ``white`` も含まれます。

```js
GET my_index/_search
{
  "aggs": {
    "group_by_first_name": {
      "terms": {
        "field": "users.first_name"
      },
      "aggs": {
        "group_by_last_name": {
          "terms": {
            "field": "users.last_name"
          }
        }
      }
    }
  }
}
```

一方、Nested データ型は以下の例のように、マッピング定義時にその型を明示的に設定します。（データ構造は変える必要はありません）

```js
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "users": {
          "type": "nested"
        }
      }
    }
  }
}
```

Nested データ型の検索は先ほどのクエリーを少し変更して、``nested`` クエリーを使って検索可能になります。``path`` にはそのオブジェクトのパスを指定して以下のようにリクエストします。

```js
GET my_index/_search
{
  "query": {
    "nested": {
      "path": "users",
      "query": {
        "bool": {
          "must": [
            {"match": {"users.first_name": "Alice"}},
            {"match": {"users.last_name":  "Smith"}}
          ]
        }
      }
    }
  }
}
```

Nested データ型では、``first_name`` と ``last_name`` の値の組み合わせが正しい条件ではマッチしますが、上記のクエリーはマッチしなくなります。

また、Aggregation を使った多段の集計では、``alice`` の集計結果には ``white`` のみが含まれるようになります。（Aggregation 時も Nested Aggregation を使ってリクエストを投げるように変更する必要があるので注意）

### Nested データ型もフィールド名で識別できるようにすると理解しやすい
このように Nested データ型は、単なる Object データ型とは区別され、クエリーの書き方だけでなくその検索結果も変わるので、もしこれらのオブジェクト型を使い分ける場合は、名称で区別できるようにすると（例えば ``nested_`` 名称に頭につけて、``nested_users`` とする）使う側が理解しやすくなります。


```js
PUT my_index/my_type/1
{
  "group": "fans",
  "nested_users": [
    {
      "first_name": "John",
      "last_name": "Smith"
    },
    {
      "first_name": "Alice",
      "last_name": "White"
    }
  ]
}
```

## Nested Type vs Parent-Child
Nested オブジェクトと類似したものに Parent-Child Relationship があるのでこちらも覚えておきましょう。

Parent-Child は、１対多の関連性を持ったデータを同じインデックス内に親子それぞれドキュメントタイプを分けて別々にインデックスすることができます（正しくは同じ Shard 内にインデックスする制限がある）。この場合、データスキーマ設計は親と子それぞれ別に JSON 構造に落とし込む設計イメージになります。（RDB 的なリレーションはこちらに近い）

Nested オブジェクトと比較したメリット・デメリットを以下にまとめました。

_Parent-Child のメリット:_

* 子のドキュメントを再インデックスなしに親のデータを更新できる
* 子のドキュメントは親や多の子ドキュメントに影響を与えることなく、追加、更新、削除することができる。
* 子のドキュメントは検索要求の結果として返すことができる。

_Parent-Child のデメリット:_

* クエリーのパフォーマンスが落ちる
* メモリを多く消費する

デメリットも少しありますが、正直どれくらいパフォーマンスに違いがあるのかわかりません。。関連する子のドキュメント数が大量にあるとか、さらに更新頻度も高い場合は、Nested ではなく Parent-Child を選択しましょう。

### Nested Type と Parent-Child の使い分け
具体的な Nested Type と Parent-Child の使い分けは悩ましいところですが、感覚的に例えば商品情報に対する SKU の組み合わせのようにメインの情報の属性データは Nested Type で設計し、商品情報に対する口コミ情報のように、それそのものの情報でも独立していて、データ量も多く頻繁に追加更新があるようなデータは Parent-Child で設計すると良い感じでしょうか？

## まとめ
いかがでしたでしょうか？今回は「データスキーマ設計のいろは」というテーマで解説しました。
Elasticsearch の「Document-Oriented」という特徴はあまり語られないような気もしますが、この特徴だけでも検索ドリブンな古い仕様の検索エンジンから Elasticsearch へ乗り換えるだけのメリットがあります。これらの特徴を理解して、最適な設計に役立ててください。