---
XMPIP: 3
Layer: Applications
Title: Persistent Key-Value Storage for PartyScript.
Author: Cryptcoin Junkey <cryptcoin.junkey@gmail.com>
Status: RFC
Type: Standards Track
Created: 2018-03-27
---

# 概要

PartyScript に、処理結果の永続性を与えるために Key-Value Storage (本文書では以下 KVS と略記)を追加する。

# 動機

PartyScript はステートレスな言語仕様/処理系であるため、トランザクション処理の演算過程を参照するための手段がない。
しかしながら、Counterparty メッセージの validation 以外の目的で PartyScript を用いる場合には、処理の演算過程や結果で生じるデータを閲覧したいという需要が考えられる。

# Counterparty asset への拡張

KVS は、既存の Counterparty asset に対して、下記のデータ構造を追加する。

* Key-Value Storage 領域

これらは、Counterparty assets の残高情報と同じく、ブロックチェーンではなく状態管理用のデータベースに保存される。
しかしながら変化の契機はブロックチェーンに記録された Counterparty メッセージを起点とするもののみであるため、(reorg によって起こり得る短期間の不一致を除き)全てのノードで一貫した値およびその変化となる。

# オブジェクトおよびデータ構造

KVS は PartyScript が管理するオブジェクトならびにそれらのデータ構造に対し、下記の拡張を行う。

* Scanner
  * Key-Value ストアを更新する。
* Key-Value ストア
  * Bitcoin Script の任意のデータ型からなる Key および Value の組を付加情報と共に保持する。


# PartyScript オペレータ

Scriptlet の中から Key-Value ストレージにアクセス可能とするため、PartyScript オペレータが定義される。

```
EOP_KEY_VALUE_GET: {key} => {value}
```

```
EOP_KEY_VALUE_SET: {value} {key} => {value}
```

```
EOP_KEY_VALUE_KEYS: empty => {key1} {key2} ... {key_n}
```

```
EOP_KEY_VALUE_DELETE: {key} => {value}
```

```
EOP_KEY_VALUE_DELETEALL: empty => empty
```

# Key-Value ストレージへのアクセス

Script の実行結果を確認するため、Counterparty API にメソッドが追加される。API 経由での Key-Value ストレージの取得には利用コストがかからない。(理由: コスト徴収の手段がない)

```
get_kvs()
```

ここで、Counterblock API には読み取り機能しか実現されず、Key-Value ストレージへの書き込みは Scriptlet のみが行えることに注意が必要である。
(理由: API から書き込めてしまうと状態の再現性が得られない。)

API 経由で Key-Value ストレージの内容を変更したい場合には、変更するための Scriptlet をデプロイし実行メッセージを送信する必要がある。

# 利用上の留意事項

## プライバシー

Counterparty ノードは、key, value を含め、ストレージに保存される全ての情報について、暗号化されていることを保証しない。

ノードが独自に暗号化を行うことは妨げないが、その場合でも、Counterparty API `get_key_value` はストレージの内容を平文で返す。

記録するデータの暗号化が求められる場合は、Scriptlet に暗号化処理を記述する必要がある。

## 利用コスト

Scriptlet からの Key-Value ストレージへのアクセスは、`EOP_KEY_VALUE_DELETE` および `EOP_KEY_VALUE_DELETEALL` を除き、燃料として XMP を消費する。
コストは未決定だが、Asset が保持している Key-Value の組の数に対して指数的に増加するよう設計される。たとえば、Key-Value の組の数を n としたとき `4^(n * 2) / 100` など。

## 更新のタイムラグ

Key-Value ストレージの更新は、スクリプト起動依頼イベントの発火が必要である。そして、スクリプト起動依頼イベントは、対応する execute メッセージを含むトランザクションが counterblock により処理されないと、発火されない。
このような時差が発生するため、アプリケーションプログラムが Key-Value ストレージの値を取得する前には、取得時点での block height を考慮する余地がある。
