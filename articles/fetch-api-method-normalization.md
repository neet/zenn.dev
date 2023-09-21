---
title: "Fetch API は「PATCH」だけ大文字と小文字の挙動が異なる"
emoji: "⛰️"
type: "tech"
topics:
  - "javascript"
published: false
---

こんにちは。皆様はウェブブラウザーの JavaScript から利用できる Fetch API で、メソッド名として `PATCH` と書くべきところを `patch` のように書くとエラーになる場合があることをご存知でしょうか。^[表題について、正確には「RFC 2016 および RFC 5789 で定義されている *common methods* の中で PATCH だけ」です。]

**_「え、大文字で書いても小文字で書いても一緒じゃないの？」_**

確かに、GET や POST などは大文字・小文字のどちらで書いても問題ありません。試しに次のコードをブラウザのコンソールで実行してみましょう。

```js
const BASE_URL = "https://huge-crow-49.deno.dev";

await fetch(BASE_URL, { method: "GET" });
await fetch(BASE_URL, { method: "get" });
```

結果は次のようになるはずです。どちらのリクエストも正常に終了し、メソッドは GET として処理されています。

> 画像

しかし、PATCH を利用する場合は同じようにはいきません。次のコードをお手元のブラウザーで実行してみてください。

```js
const BASE_URL = "https://huge-crow-49.deno.dev";

await fetch(BASE_URL, { method: "PATCH" });
await fetch(BASE_URL, { method: "patch" });
```

これを実行すると、次のようなエラーを得るでしょう。

> 画像

果たして、これはなぜ起こっているのでしょうか。以下でその原因を探ってみましょう。

## エラーが発生する条件

このエラーは、 `PATCH` と `patch` の一方にしかリソースを提供していない URL に対して Fetch API を利用する場合に発生します。

また、クロスオリジンでのリクエストの場合は CORS ヘッダー `Access-Control-Allow-Methods` に両方を設定していないときも、別種のエラーが発生し同様に失敗します。

一方、`GET`・`POST`・`PUT`・`DELETE` などのメソッドでは上記に当てはまる場合でも大文字・小文字の両方でリクエストを送信できます。

## どうして起こるのか

どうしてこのようなエラーが発生するのでしょうか。ウェブにまつわる仕様は通常 [W3C](https://www.w3.org/) や [WHATWG](https://whatwg.org/) といった組織によって文書化されているため、 HTML や JavaScript といった技術について疑問を持った際は、その挙動を定義する仕様書を当たると解決することがあります。

ここで、 Fetch API は WHATWG によって公開されている [Fetch Standard](https://fetch.spec.whatwg.org/) という仕様の一部です。中でも `fetch()` などのメソッドを提供する部分の仕様は [5. Fetch API](https://fetch.spec.whatwg.org/#fetch-api) に書かれています。

> 画像

これによると `fetch(url, init)` の第二引数は `RequestInit` と呼ばれ、その振る舞いについては [5.4 Request class](https://fetch.spec.whatwg.org/#request-class) で言及されています。ここで `RequestInit` のインターフェイスは WebIDL というウェブの仕様を記述するための言語で定義されています。

> 画像

<!-- ![](https://res.craft.do/user/full/baa05f35-02e0-6418-31f2-e667e2909b6b/doc/1E0EA07D-FEB3-4DF1-B1D6-DEFCE48BC62F/ada659fc-b402-4d7e-a025-461f375fd4b0) -->

[5.4 Request class](https://fetch.spec.whatwg.org/#request-class) をさらに読み進めると、後半部分で `Request` クラスをインスタンス化する際の手順が 42 件にも及ぶステップで定義されているのを見つけられます。

> 画像

<!-- ![](https://res.craft.do/user/full/baa05f35-02e0-6418-31f2-e667e2909b6b/doc/1E0EA07D-FEB3-4DF1-B1D6-DEFCE48BC62F/1f1847a4-b0b8-4f11-b77b-6461359e2378) -->

このステップの 12 番目の記述によると、 `Request` コンストラクタは `RequestInit` で受け取った `method` を _method_ として解釈し `Request` をインスタンス化することがわかります。ここで、 _method_ は同仕様の [2.1.1 Methods](https://fetch.spec.whatwg.org/#methods) にハイパーリンクされています。

> 画像

<!-- ![](https://res.craft.do/user/full/baa05f35-02e0-6418-31f2-e667e2909b6b/doc/1E0EA07D-FEB3-4DF1-B1D6-DEFCE48BC62F/e3abf769-ebca-41e2-a443-cd16d85dec2a) -->

ここまで辿るとようやく `fetch` の第二引数のオブジェクトに渡した `method` がどのように利用されるのかを知ることができます。ここでは、下記のように述べられています

> メソッド は、 method トークン生成規則に合致するバイト列である。 HTTP における要請メソッドに対応する。（中略）
> メソッドを正規化する ときは、 所与の ( メソッド M ) に対し，［ M が次のいずれかにバイト大小無視で合致するならば バイト大文字化する( M ) ／ 他の場合は M ］を返す ： DELETE, GET, HEAD, OPTIONS, POST, PUT
> Fetch Standard（日本語訳） より CC 4.0 BY に基づき引用

この記述を言い換えると **「大文字・小文字を無視して `DELETE`・`GET`・`HEAD`・`OPTIONS`・`POST`・`PUT` に合致する文字列が渡された場合は、すべて大文字にするよ」** と言っています。

いま注目したいのは、 **`PATCH` が対象に含まれていないことです**。実際に、Chrome Devtools の Network パネルを開くと、小文字で `patch` と指定した時は小文字のままリクエストが行われているのを確認できます。

> 画像

なお、HTTP/1.1 の仕様を定義している [RFC 2616](https://datatracker.ietf.org/doc/html/rfc2616) によれば、本来 HTTP メソッドは大文字・小文字を区別します。^[The Method token indicates the method to be performed on the resource identified by the Request-URI. The method is case-sensitive.] したがって `patch` としたときは `PATCH` としては解釈されず、全く別の「カスタムメソッド」として扱われることになります。

## どうしてこんな仕様なのか

この仕様の意図を理解するには、 Fetch API およびその仕様である Fetch Standard が作成された目的を理解する必要があるでしょう。
