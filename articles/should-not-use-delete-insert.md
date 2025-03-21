---
title: "delete-insertで困った話"
emoji: "🍣"
type: "tech"
topics: ["db"]
published: true
---

一つのリソースに対して一部のデータを複数持つような構造をRDBで扱う際、各データを個別のレコードとして管理することが一般的です。
このようなデータ構造の更新処理では、単純な1レコードの更新処理よりも考えることが若干増えます。

たとえば、以下のようなテーブル構造があるとします。

#### `products` テーブル
| id  | name          | created_at         |
|-----|---------------|--------------------|
| 1   | Laptop        | 2023-01-01 12:00  |
| 2   | Smartphone    | 2023-01-02 15:00  |

#### `product_tags` テーブル
| id  | product_id  | tag       | created_at         |
|-----|-------------|-----------|--------------------|
| 1   | 1           | Electronics | 2023-01-01 12:10  |
| 2   | 1           | Office      | 2023-01-01 12:15  |
| 3   | 2           | Mobile      | 2023-01-02 15:10  |

更新する際は主に2つの方法があります。

1. 前後を比較して差分のみを追加、削除、更新する
2. 全てを削除してから、必要なデータを全て追加する (いわゆる delete-insert)

1 に比べて 2 の方が処理は簡単なため、使われる場面がよくあると思います。
が、その処理の簡単さのトレードオフとして、データの方に問題が出るケースがあります。

## ログ用のデータが壊れる
レコードの調査等に使うカラムとして `created_at` や `updated_at` といったデータを持つ場合が多いと思いますが、delete-insert すると関連する全ての timestamp が最新に書き換わってしまいます。
それぞれのレコードがいつ作られたのかと言う情報が失われてしまい、`updated_at` を持っている場合はそのカラムの意味が全くなくなってしまいます。

## 関連データの意図しない更新・削除
他のテーブルからの外部キー制約で `on delete cascade` が設定されている場合、意図せず別のリソースが削除されてしまう危険性があります。

例えば、追加実装として以下のようなテーブルが作られたとします。
`product_tags` テーブルに外部キーを貼っており、delete cascade の制約もつけているとします。

#### `tag_analytics` テーブル
| id  | tag_id  | views   | last_viewed       |
|-----|---------|---------|-------------------|
| 1   | 1       | 100     | 2023-01-05 14:00 |
| 2   | 2       | 50      | 2023-01-06 16:00 |
| 3   | 3       | 75      | 2023-01-07 10:00 |

この場合、`product_tags` テーブルの更新が delete-insert で実装されていると以下の問題が起こります。

- product のタグが更新された場合に tag が一度全部消えてから再度作られるため、このテーブルにあるレコードも削除されてしまいます。
- 結果的に views や last_viewed もリセットされ、データが失われてしまいます。

# まとめ
個別で見るとそんなことしなくない？と思いがちですが、トレードオフを認識できていないと思わぬ不具合を生むことがあります。
このテーブルはあまり重要じゃないから楽な実装にしとこう、という気持ちで実装していたが、事業の方向転換などでそのテーブルを参照するテーブルがたくさん作られたり、といったケースもあると思います。

差分更新にすると更新のロジックが若干複雑になるため、一概に delete-insert は全部やめようとは思っていないのですが、なるべくデータに優しい実装を心がけたいですね。
