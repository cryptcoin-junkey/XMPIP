---
XMPIP: 2
Layer: Applications
Title: Script language that provides smart contructs.
Author: Cryptcoin Junkey <cryptcoin.junkey@gmail.com>
Status: RFC
Type: Standards Track
Created: 2018-02-19
---

# 概要

PartyScript は、いわゆる「スマートコントラクト」を実現するための、簡易言語の仕様である。
また、便宜上、PartyScript 言語仕様に基づくプログラムの実行環境もまた、PartyScript と呼ぶ場合がある。

# 動機

PartyScript 仕様を規定する動機は、アプリケーション・プログラマに対し、広く契約(コントラクト)成立にを示す証憑として応用可能なデータ・ストアを提供することである。

契約成立を示す証憑たり得るためには、その出力データが改ざん困難であることに加え、入力が改ざん困難であり、その演算過程が再現可能である必要がある。

PartyScript は、入出力の改ざん困難性を得る手段として、また、再現可能な演算過程を保持する手段として、Counterparty ブロックチェーンを用いる。この決定により、PartyScript の実装は、Counterparty の拡張として定義される。

Counterparty の立場から見ると、PartyScript は、Counterparty に、組み込みのコントラクトによって生成されるデータ構造以外の任意のデータを埋め込む手段を、アプリケーションプログラマに提供できることになる。

PartyScript 保持するデータの操作は、ブロックチェーンに記録された情報のみに依存して行われる。
この性質は、Counterparty を始めとする既存のブロックチェーン利用システムらを、踏襲している。

# 特徴

PartyScript は、下記のような特徴を持つ


## チューリング不完全性

PartyScript は、Bitcoin Script 言語の拡張として定義される。よってループ構文は存在しない。
PartyScript では、コールスタックを用いた複数環境を持つことができる。しかし、コールスタックでの循環呼び出しを禁止している。
よってアプリケーション・プログラマは、PartyScript のみでのループ処理は不可能である。
言い換えると、PartyScript はチューリング完全ではない。

この設計は意図的なものである。PartyScript は、ブロックチェーン内のトランザクションに含まれるメッセージにより駆動される。一種のイベント駆動アーキテクチャであり、またブロック生成間隔を上限とする時間制約が存在する。
よって、処理時間の予測可能性が重要となる。

もしアプリケーションがループ構文を望む場合には、PartyScript 外に、値の変更監視と再度のスクリプト起動メッセージ送信を繰り返すループ構造が構築されても構わない。

## セミ・ステートレス

PartyScript のスタックは、ステートレスである。スクリプト起動メッセージによって生成された実行環境は、実行完了から、次のスクリプト起動メッセージが受理されるままでに、完全に破棄される。
しかしブロックチェーンをメッセージのコンテナとして利用している Counterparty では、完全にステートレスにしてしまうと、実行結果を得る手段が無い。

なお、実行結果を得たい場合には(おそらくほとんどのケースで実行結果は得たいはず)、Key-Value ストレージ領域が利用できる。

## コードとアセットとのバインディング、およびアクセス制御

PartyScript のプログラム片(Scriptlet と呼ぶ)は、Counterparty に対して束縛(bind)される。
言い換えると、任意のアセット A に対して束縛された Scriptlet の所有権は、アセット A の所有権をもつアドレスが有する。

Scriptlet の開発者は、この性質を利用して、スクリプト起動メッセージに対する柔軟なアクセス制御を実装できる。

# Counterparty asset への拡張

PartyScript は、既存の Counterparty asset に対して、下記のデータ構造を追加する。

* Scriptlet を格納するバイナリデータ領域
* Key-Value ストレージ領域

これらは、Counterparty assets の残高情報と同じく、ブロックチェーンではなく状態管理用のデータベースに保存される。
しかしながら変化の契機はブロックチェーンに記録された Counterparty メッセージを起点とするもののみであるため、(reorg によって起こり得る短期間の不一致を除き)全てのノードで一貫した値およびその変化となる。

# オブジェクトおよびデータ構造

PartyScript が管理するオブジェクトならびにそれらのデータ構造は、下記のとおりである。

* Asset
 * いわゆる "カウンターパーティ・トークン" と呼ばれるものと等価である。
* Scriptlet
 * 実行エンジンが処理するオペレータおよびデータを含むバイト列で、Asset が保持している。
* Engine
 * Environment を保持する環境スタックと、commit/rollback 前の key-value ストアの値を保持している。
* Environment
 * アセットが保持している Scriptlet を保持する実行スタックと、Scriptlet に含まれるオペレータの操作対象であるデータスタックを保持している。
* Key-Value ストア
 * Bitcoin Script の任意のデータ型を Key および Value を保持している。

# Scriptlet

## 定義

PartyScript の実行環境を制御するためのデータとオペレータを並べたものである。プログラミング言語の一種と捉えられる。

## 言語仕様

PartyScript の文法、オペレータの作用、ならびにオペコードは、限られた例外を除き Bitcoin Script に準拠する。

### 例外事項

限られた例外は、以下のとおりである。

* OP_INVALID_OPCODE を、PartyScript 独自のオペレータを呼び出す OP_PARTYSCRIPT の alias とする

### PartyScript オペレータと EOP_ prefix

PartyScript では、下記の通り OP_PARTYSCRIPT が定義される。

```
OP_PARTYSCRIPT: n => {result of sub-operator}
```

全ての PartyScript オペレータは、1バイトの数値と OP_PARTY_SCRIPT の連続により表現される。
本仕様では、可読性のため、PartyScript のオペレータのマクロ表記として EOP_ プレフィクスで表す。

### send_execute メッセージの直接呼び出し

PartyScript には、OP_SEND_EXECUTE オペレータが定義され、下記のように作用する。

```
EOP_SEND_EXECUTE : obj1:any obj2:any ... objn:any n:int asset_name:string => {{result of scriptlet on `asset_name`}}
```

OP_SEND_EXECUTE オペレータは、Engine に対し、Counterparty が send_execute メッセージを受信したときと同じように作用する。
結果として、指定した `asset_name` に相当する scriptlet の実行が開始され、メッセージ送信元の scriptlet の実行は遅延される。

また、Engine は、スタック最上位にある scriptlet の実行が終了した時、未受理のフラグが立っていなければ、データスタックの内容を新たにスタック最上位になる Environment のデータスタックに push する。

OP_SEND_EXECUTE オペレータは、他言語におけるサブルーチンや関数と呼ばれるものと同等の機能を実現するために利用されうる。


# アクセス制御

PartyScript の言語仕様には、Scriptlet の実行を行うための権限に関する機能は含まれない。
また PartyScript の処理系でも、スクリプト実行依頼メッセージに対して、暗黙的な権限チェックは行わない。

しかしながら、Scriptlet 内でメッセージ送信元に関する情報取得するための、拡張オペレータが、いくつか提供される。

## 所有権の取得

asset の所有権を得るための拡張オペレータとして、`EOP_ROOT_OWNER` および `EOP_ASSET_OWNER` が提供される。

```
EOP_ROOT_OWNER : empty => {Address}
```

```
EOP_ASSET_OWNER: empty => {Address}
```

`EOP_ROOT_OWNER` は、コントラクト開始イベントの発火契機となったトークンの所有権を有するアドレスをデータスタックに push する。
`EOP_ASSET_OWNER` は、現在実行中の scriptlet に紐付いているトークンの所有権を有するアドレスをデータスタックに push する。

たとえば、`EOP_ROOT_OWNER` と `EOP_TOKEN_OWNER` とを用いて、特定のアドレス以外が所有しているトークンからの scriptlet 実行を拒否させるには、
下記のようにすればよい。

```
EOP_ROOT_OWNER EOP_TOKEN_OWNER OP_EQUAL OP_VERIF  {{処理本体}}
```

## コールツリーにあるアセット名を取得

```
EOP_ASSET_NAME: {number} => {String}
```

```
EOP_ASSET_BASENAME: {string} => {string}
```

`EOP_ASSET_NAME` は、実行中の scriptlet を 0 として、コールスタックを n 回遡った実行環境の scriptlet を束縛している、アセット名を返す。

`EOP_ASSET_BASENAME` は、データスタックの top にある文字列がサブアセット名だった時、その文字列を pop してベースアセット名を data stack に push する。data stack の top がサブアセットでなかった場合、何もしない。

アセット名を用いて、特定のベースアセット名を持つサブアセットの scriptlet からの send だけ実行可能にするためには、下記のようにすればよい。

0 EOP_ASSET_NAME EOP_ASSET_BASENAME 1 EOP_ASSET_NAME EOP_ASSET_BASENAME OP_EQUAL OP_VERNOTIF {{処理の本体}}

このように保護されたサブアセットらに束縛された scriptlet 群は、共通のベースアセット名を持つオブジェクトの private メソッドと見立てて活用できる。


# デプロイ

Scriptlet で実行するスクリプトは、アセットに紐付けられたストレージからロードされる。そのため、Scriptlet は、実行前にストレージへ登録しアセットに束縛されなければならない。この作業をデプロイと呼ぶ。

## デプロイ・メッセージ

デプロイを行うためのメッセージ作成を目的として Counterparty API に send_deploy メソッドが追加される。

# Engine 起動メッセージ

Engine の起動契機をブロックチェーンに記録するため、Counterparty API に send_script メソッドが追加される。

# Key-Value ストレージ

## Key-Value ストレージへのアクセス

Script の実行結果を確認するため、Counterblock API にメソッドが追加される。

ここで、Counterblock API には読み取り機能しか実現されず、Key-Value ストレージへの書き込みは Scriptlet のみが行えることに注意が必要である。
(理由: API から書き込めてしまうと状態の再現性が得られない。)

API 経由で Key-Value ストレージの内容を変更したい場合には、変更するための Scriptlet をデプロイし実行メッセージを送信する必要がある。

## 更新の時差

Key-Value ストレージの更新は、スクリプト起動依頼イベントの発火が必要である。そして、スクリプト起動依頼イベントは、対応する execute メッセージを含むトランザクションが counterblock により処理されないと、発火されない。
このような時差が発生するため、Key-Value ストレージの値を取得する前には、取得時点での block height を考慮する余地がある。

# オブジェクトのライフサイクル

## Engine

Engine  のライフサイクルは、下記のとおりである。

0. イベントを受信する。
0. コールスタックを構築する。
0. Scriptlet にイベントに含まれるデータを引き渡し、生成された Scriptlet の環境をコールスタックに積む
0. コールスタックの最上部にある Scriptlet から他アセットにバインドされた Scriptlet の実行依頼があったとき、下記の処理を行う。
 0. 新規に作成予定の Scriptlet を用いた環境がコールスタックに存在する場合、循環呼び出し違反のフラグを立て即時にすべてのコールスタックを pop する。
 0. 実行依頼のあったメッセージとともに 3 に戻る。
0. コールスタックの最上部にある Script が終了した場合、下記の処理を行う。
 0. なんらかの実行違反フラグが立っていた場合、即時にコールスタックにある全てを pop する。
 0. 終了した環境に実行違反フラグが立っていない場合、その環境をコールスタックから pop し、結果をコールスタックの top にある環境のデータスタックに push する。
0. コールスタックが空になった場合には、実行に要した XMP を環境利用料として徴収し、また今回のライフサイクルで発生した Key-Value ストレージへについて下記の処理を行った上で終了する。
 0. 未受理フラグが立っていた場合、変更を rollback する。
 0. 未受理フラグが立っていない場合、変更を commit する。

ここで、実行違反が発生していた場合でも、環境利用料として XMP が徴収されることに注意が必要である。

## Scriptlet 

他言語において関数やサブルーチンに相当する概念に相当する、Party Script 言語の概念として、Scriptlet が定義される。

Scriptlet の実行コードは、各アセットが高々1つ保持している。

環境は外部からのイベントにより構築され、下記の通りのライフサイクルとなる。

0. データスタックと実行スタックが構築される。
0. アセットが保持している実行コードが実行スタック上にコピーされる。
0. イベント発火の契機となるメッセージに含まれるデータがデータスタック上にコピーされる。
0. インタプリタが実行スタックの内容を処理し、以下の状態変化が起こる
 * 実行スタックから命令を pop する。
 * データスタックの状態が変わる。
 * キーバリューストレージの状態が変わる。
 * 別の scriptlet を実行するイベントを送信する。
0. 終了状態へ遷移する。

終了状態への遷移でインタプリタが終了した時点で、データスタックと実行スタックは破棄される。