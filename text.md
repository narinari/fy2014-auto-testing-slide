## TL;DR

e2eの結合テストの外部仕様テストケースを自動生成したよ

---

## プロジェクトの背景
* 開発が遅れていた <!-- .element: class="fragment" -->

* 単体テストで手一杯 <!-- .element: class="fragment" -->

* e2eの結合テストは手薄 <!-- .element: class="fragment" -->

* クライアントはできていない <!-- .element: class="fragment" -->

* ドキュメントのメンテが追いついていない <!-- .element: class="fragment" -->

  →正しい仕様は開発者が把握している <!-- .element: class="fragment" -->

* マンパワーが圧倒的に足りない <!-- .element: class="fragment" -->

---

## 考えたこと
* クライアントはSingle-Page Application で、サーバは JSON を返すだけのアーキテクチャ <!-- .element: class="fragment" -->

* HTMLを返すのに比べてレスポンスの構造が安定しているので、テストが壊れにくい <!-- .element: class="fragment" -->

* リクエストとレスポンスを形式的に定義できればテストケースを
  自動的に生成できる！
  <!-- .element: class="fragment" -->

* いまどきのアーキテクチャは、SOA的に使えるようにサーバをAPIボックスにするようなアーキテクチャになってきたから、応用範囲が広いかも！ <!-- .element: class="fragment" -->

---

## インスパイア元

#### QuickCheck
Haskellの自動テストライブラリ。引数の型を元にして入力データを生成して
異常終了しないかチェックしてくれる。 <!-- .element: class="fragment" -->

→型がわかれば、値の表現や、境界値などがわかるので、データを作れる <!-- .element: class="fragment" -->

#### DSL
APIの仕様を表現するのにDSLを使えば、人間にわかりやすく、機械的に処理し
やすい定義ができる。 <!-- .element: class="fragment" -->

#### JSON Schema
JSONの構造を定義するライブラリ。スキーマがあれば機械的に検証できる。 <!-- .element: class="fragment" -->

---

## やっていること
### リクエスト
認証
 : 認証が必要なAPIで、認証しないでアクセスしたらエラーを返す。 <!-- .element: class="fragment" -->

XSRF
 : ワンタイム・トークンが必要なAPIで、トークンなしアクセスしたらエラーを返す。 <!-- .element: class="fragment" -->

必須パラメータ
 : 指定されていなかったらエラーレスポンスを返す <!-- .element: class="fragment" -->

型
  : 範囲外の値だったら（境界値チェック）、定義外の値だったら、異なる型の値だったら、エラーレスポンスを返す <!-- .element: class="fragment" -->

---

## やっていること
### レスポンス
必須エレメント
 : レスポンスに含まれていなかったらバグ <!-- .element: class="fragment" -->

値の型
 : 範囲外の値だったら、定義外の値だったら、異なる型の値だったらバグ <!-- .element: class="fragment" -->

構造
 : 知らない属性があったらバグ <!-- .element: class="fragment" -->

* * *

上記の内容を JSON Schema でチェック <!-- .element: class="fragment" -->

---

## 成果
運用保守で役に立つ。もちろん開発中でも役に立つ。

* リグレッションテストが容易になっている <!-- .element: class="fragment" -->
* 正常性確認テストがやりやすくになっている <!-- .element: class="fragment" -->
* JUnit で動いてくれるので、テストの追加や編集が容易 <!-- .element: class="fragment" -->

---

<!-- .slide: data-transition="zoom" -->

## 成果:現場の声
### リグレッションテストが容易になっている。

![dodo image](img/dodo.jpg)

> 小規模改修プロジェクトで、デグレードやバグを検知できています。
> 中には画面上からの通常のテストからは検知できないようなものも、検知できているので、運用・保守スタッフとしては非常に助かっています。

<div class="notice">個人の感想であり、商品の効能を保証するものではありません</div>

---

<!-- .slide: data-transition="zoom" -->

## 成果:現場の声
### 正常性確認テストがやりやすくになっている
![dodo image](img/dodo.jpg)

> テストは全APIに対して、全てのリクエストパターンを網羅するよう作られています。
> そのため、正常性確認テストではAPIを意識すること無く、画面レイアウトやアプリケーションの挙動についてのテストに注力できています。
> モバイルアプリ版もAPIを使います。今までと違ったAPIの使い方をするけど、
> 既存バグが顕在化する心配が少ないです。

<div class="notice">個人の感想であり、商品の効能を保証するものではありません</div>

---

<!-- .slide: data-transition="zoom" -->

## 成果:現場の声
### JUnitで動いてくれるので、テストの追加や編集が容易
![dodo image](img/dodo.jpg)

> ローカルでもJenkins上でも複雑な設定なしに動いてくれることも機能単体テストの追加や編集が容易にしている利点の1つです。

<div class="notice">個人の感想であり、商品の効能を保証するものではありません</div>

---

## しくみ

* JUnitを利用
* 個々のテストケースクラスに、API定義をDSLで記述
* テストケースのベースクラスに、自動で適用するテストを記述
* テストの冒頭で ```assumeThat``` を使ってAPI定義を見てテストするか判断
  
  ```Assume#assumeThat``` は ```Assert#assertThat``` とほぼ同じだが、
  検証を失敗した場合、テストをスキップするメソッド。
  JUnit 4.4 から追加された機能で、テストの前提を表す。
  
---

## API定義DSL

ダブルブレス記法。ApiSpecのサブクラスを作り、```{{...}}``` 内のコードがインスタンス初期化のスコープで実行される。

```java
class AnyApiTest extends ApiTestCase {
  public getApiPath() {
    return new ApiSpec("/api/any")
      {{
        // API利用時のメソッド定義(デフォルトは「GET」)
        usageMethod("GET");
        // 要認証APIは以下のいずれかを指定します。
        requireLogin();
        requireAccessToken(); // ワンタイムトークンが必要な場合は、自動的にログインも必要とみなします。
        // 必須パラメータの定義
        requireParam(cursorParamSpec().define()); // よく使われるパラメータはすでに定義済み
        requireParam(paramSpec("param1", Integer.class).define());
        requireParam(paramSpecString("param2") // パラメータ型毎にファクトリメソッドがあります
                .validValues("CATEGORY0001")   // 正常なレスポンスを返す値を設定してください
                .maxLength(100).define());
        // 任意パラメータの定義
        optionalParam(paramSpecInt("optionalParam").positiveOnly().define());
        // 区分値型のパラメータには区分に応じた列挙型を渡す
        optionalParam(paramSpecEnum("hogeKubun", HogeKubunType.class).define());
      }};
  }
}
```

---

## 自動テストの実装

```java
class ApiTestCase extends TestCase {
  @Test
  public void ApiSpec_必須パラメータなし() {
    // API外部仕様検証
    ApiSpec apiSpec = apiSpec();
    Assume.assumeFalse("必須パラメータがあるAPIだけテストする。", apiSpec.getRequireParams().isEmpty());

    RequestSpecification request = createNormalRequestSpecification();
    Response actual = call(request);

    // ステータスコード検証
    assertThat("HTTPステータスは200で返ってくる。", actual.statusCode(), is(HttpServletResponse.SC_OK));

    // エラーコード検証
    JsonPath json = parse(actual);
    for (ParameterSpec<?> paramSpec : apiSpec.getRequireParams()) {
      assertErrorCode(json, paramSpec.getName(), "010");
    }
  }
}
```

---

## 実装したテストの例
```
ApiSpec_認証済み
ApiSpec_未認証
ApiSpec_必須パラメータなし
ApiSpec_共通パラメータ_カーソル_不正文字列
ApiSpec_共通パラメータ_カーソル_マイナス値
ApiSpec_共通パラメータ_カーソル_上限値超
ApiSpec_文字列型パラメータ_必須項目に半角スペースのみ
ApiSpec_文字列型パラメータ_項目長上限超
ApiSpec_数値型パラメータ_項目長上限超
ApiSpec_数値型パラメータ_上限値超
ApiSpec_数値型パラメータ_下限値超
ApiSpec_数値型パラメータ_Long_項目長上限超
ApiSpec_数値型パラメータ_数字以外
ApiSpec_数値型パラメータ_Long_上限値超
ApiSpec_数値型パラメータ_Long_下限値超
ApiSpec_数値型パラメータ_Long_数字以外
ApiSpec_列挙型パラメータ_不正文字列
```
<!-- .element: style="font-size: 60%;" -->

---

## こんなAPIだと適用しやすい
* エラーレスポンスのメッセージ型が統一されている

* リクエストメッセージの型が統一されている

* エラーコードが統一されている

* それぞれの属性についてエラーコードが返される

---

## できていないところ
### 正常系は難しい

→次ページで説明 <!-- .element: class="fragment" -->

### パラメータ間の依存関係

パラメータAがxだったら、パラメータXが必須など。 <!-- .element: class="fragment" -->

### GETメソッドしかできていない

RESTful APIなら、POST、PUT、DELETEもできたほうがいい。 <!-- .element: class="fragment" -->

GET を想定しているところに POST でリクエストしたり。 <!-- .element: class="fragment" -->

---

## 正常系は難しい

異常系のテストケースは、状態が集約するので検証しやすい。

一方で、正常系はサーバー内で例外が発生しなかったことを確認できるけど、
分岐網羅や状態網羅までするのは仕様を詳細に記述しないと難しい。<!-- .element: class="fragment" -->

<p class="fragment">詳細な記述になるほど、実際の実装と乖離しやすくなるので *** やらなかった *** 。</p>

→その代わり、他のテストケースで担保 <!-- .element: class="fragment" -->

JSON Schema でバリデーションできる便利メソッドを作って呼び出してもらった。 <!-- .element: class="fragment" -->

---

## 今回の開発の流れ

クライアント開発者に配布 ← ドキュメント　テストコード

　　　　　　　　　　　　　↓　　　↙

　　　　　　　　　　　　API仕様DSL

　　　　　　　　　　　　↓

　　　　　　　　　　　　自動テストで確認

---

## これからの開発の流れ

API定義DSLを記述

↓　　　　　　　↓

ドキュメントを生成　　　　テストケースを生成

↓　　　　　　　↓

クライアント開発者に配布　　自動テストで確認

* * *

[RESTful API Modeling Language (RAML)](http://raml.org/index.html)など、
汎用的なAPI仕様記述言語が出てきているので、これらを活用するほうがいい
かもしれない。


---

<!-- .slide: style="font-size: 70%;" -->

## 参考資料

### REST Assured

テストケースからAPI呼び出しに使っていたライブラリ

rest-assured - Java DSL for easy testing of REST services
https://code.google.com/p/rest-assured/

### JSON Schema

JSON Schemaの詳細と検証に使っていたライブラリ

JSON Schema and Hyper-Schema
http://json-schema.org/

fge/json-schema-validator
https://github.com/fge/json-schema-validator

### RAML

API仕様をYAMLで記述する言語 RAML に関する情報

RESTful API Modeling Language (RAML)
http://raml.org/index.html

MuleSoftがRESTful APIを設計するRAMLツールをリリース
http://www.infoq.com/jp/news/2013/10/raml-rest-api-tools
