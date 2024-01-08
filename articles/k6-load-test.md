---
title: "負荷テストツール k6 の基本的な使い方"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["負荷テスト", "k6"]
published: true
publication_name: "yumemi_inc"
---

## はじめに

Web アプリケーションや API の負荷テストに使えるツール、[k6](https://k6.io/docs/) を試してみました。この記事では、k6 の基本的な使い方についてご紹介します。

## 環境構築

k6 を使い始めるにあたり、まずは環境構築が必要です。以下の環境で進めていきます。

- macOS: `Sonoma 14.2.1`
- Homebrew: `4.2.2`

### インストール

```
$ brew install k6
$ k6 --version
k6 v0.48.0 (go1.21.5, darwin/arm64)
```

https://k6.io/docs/get-started/installation/#macos

## 実行方法

インストールが完了すると、k6 をローカルで実行することができるようになりますので、実際に実行してみます。

### 初期化

以下のコマンドを実行することで、新規で実行ファイルが作成されます。なお、デフォルトのファイル名は `script.js` となります。

```
$ k6 new
```

```js:script.js
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '30s',
};

export default function() {
  http.get('https://test.k6.io');
  sleep(1);
}
```

### 実行

実行する際は、以下のコマンドを実行します。上記で作成したサンプルの場合は 30 秒テストを実行する設定になっているので、テストの実行は 30 秒ほどかかり、テスト結果が表示されます。

```
$ k6 run script.js
...
data_received..................: 2.9 MB 94 kB/s
data_sent......................: 28 kB  899 B/s
http_req_blocked...............: avg=19.3ms   min=2µs      med=9µs      max=484.18ms p(90)=19.3µs   p(95)=24.94µs 
...
```

### 実行結果

実行すると実行結果のメトリクスが表示されます。各メトリクスの解説は以下の公式ドキュメントを確認してください。

https://k6.io/docs/using-k6/metrics/reference/

## 基本的な使い方

### テストスクリプトのライフサイクル

新規テンプレートを使ったテストスクリプトだと、初期化処理(オプション設定)、テスト内容の処理(`export default`している function)のみ使ってテストを実行しました。それに加えてテスト前処理(`setup`)、テスト後処理(`teardown`)のような処理を実装することができます。

```js
// 1. init code

export function setup() {
  // 2. setup code
}

export default function (data) {
  // 3. VU code
}

export function teardown(data) {
  // 4. teardown code
}
```

https://k6.io/docs/using-k6/test-lifecycle/

### オプション

どのようなテストを実行したいか？に応じて、テスト実行時の仮想ユーザー数（並列数）、実行時間、実行回数など様々なオプションを指定することができます。利用できるオプションについては、以下の公式ドキュメントを確認してください。

https://k6.io/docs/using-k6/k6-options/reference/

これらのオプションを指定する方法については、スクリプトの初期化処理で指定する、CLI のオプションで指定するなどいくつかあります。また、指定する方法に応じて適用の優先度もありますので、特定のオプションを上書きしたい場合などは優先度を考慮する必要があります。

![](/images/k6-load-test/option-precedence.png)
（[How to use options - Order of precedence]([b.com/aws/aws-cdk/wiki/Security-And-Safety-Dev-Guide](https://k6.io/docs/using-k6/k6-options/how-to/#order-of-precedence)) より）

https://k6.io/docs/using-k6/k6-options/how-to/

### HTTP Request

テスト対象のエンドポイントに応じて、リクエストのメソッドを指定することができます。

https://k6.io/docs/using-k6/http-requests/#available-methods

#### GET

[http.get](https://k6.io/docs/javascript-api/k6-http/get/)( url: string | [HTTP URL](https://k6.io/docs/javascript-api/k6-http/urlurl/#returns), params?: [object](https://k6.io/docs/javascript-api/k6-http/params/) ): [Response](https://k6.io/docs/javascript-api/k6-http/response/)

```js
import http from 'k6/http';

export default function () {
  const res = http.get('https://test.k6.io');
  console.log(JSON.stringify(res.headers));
}
```

#### POST

[http.post](https://k6.io/docs/javascript-api/k6-http/post/)( url: string | [HTTP URL](https://k6.io/docs/javascript-api/k6-http/urlurl/#returns), body?: string | object | ArrayBuffer, params?: [object](https://k6.io/docs/javascript-api/k6-http/params/) ): [Response](https://k6.io/docs/javascript-api/k6-http/response/)

```js
import http from 'k6/http';

const url = 'https://httpbin.test.k6.io/post';
const logoBin = open('./logo.png', 'b');

export default function () {
  let data = { name: 'Bert' };

  // Using a JSON string as body
  let res = http.post(url, JSON.stringify(data), {
    headers: { 'Content-Type': 'application/json' },
  });
  console.log(res.json().json.name); // Bert

  // Using an object as body, the headers will automatically include
  // 'Content-Type: application/x-www-form-urlencoded'.
  res = http.post(url, data);
  console.log(res.json().form.name); // Bert

  // Using a binary array as body. Make sure to open() the file as binary
  // (with the 'b' argument).
  http.post(url, logoBin, { headers: { 'Content-Type': 'image/png' } });

  // Using an ArrayBuffer as body. Make sure to pass the underlying ArrayBuffer
  // instance to http.post(), and not the TypedArray view.
  data = new Uint8Array([104, 101, 108, 108, 111]);
  http.post(url, data.buffer, { headers: { 'Content-Type': 'image/png' } });
}
```

#### PUT

[http.put](https://k6.io/docs/javascript-api/k6-http/put/)( url: string | [HTTP URL](https://k6.io/docs/javascript-api/k6-http/urlurl/#returns), body?: string | object | ArrayBuffer, params?: [object](https://k6.io/docs/javascript-api/k6-http/params/) ): [Response](https://k6.io/docs/javascript-api/k6-http/response/)

```js
import http from 'k6/http';

const url = 'https://httpbin.test.k6.io/put';

export default function () {
  const headers = { 'Content-Type': 'application/json' };
  const data = { name: 'Bert' };

  const res = http.put(url, JSON.stringify(data), { headers: headers });

  console.log(JSON.parse(res.body).json.name);
}
```

#### DELETE

[http.del](https://k6.io/docs/javascript-api/k6-http/del/)( url: string | [HTTP URL](https://k6.io/docs/javascript-api/k6-http/urlurl/#returns), body?: string | object | ArrayBuffer, params?: [object](https://k6.io/docs/javascript-api/k6-http/params/) ): [Response](https://k6.io/docs/javascript-api/k6-http/response/)
※ `delete` ではなく `del`

```js
import http from 'k6/http';

const url = 'https://httpbin.test.k6.io/delete';

export default function () {
  const params = { headers: { 'X-MyHeader': 'k6test' } };
  http.del(url, null, params);
}
```

## テストの実装例

基本的な使い方をベースにして、いくつかのテストパターンを試してみます。

### 並列度 1 のリクエストを 100 回実行

```js:script.js
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 1,
  iterations: 100,
};

export default function() {
  http.get('https://test.k6.io');
  sleep(1);
}
```

### 並列度 1 のリクエストを 30 秒実行

```js:script.js
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 1,
  duration: '30s',
};

export default function() {
  http.get('https://test.k6.io');
  sleep(1);
}
```

### レスポンスのステータスコードをチェック

```js:script.js
import { check } from 'k6';
import http from 'k6/http';

export const options = {
  vus: 1,
  iterations: 10,
};

export default function() {
  const res = http.get('https://test.k6.io');
  check(res, {
    'is status 200': (r) => r.status === 200,
  })
}
```

[check](https://k6.io/docs/using-k6/checks/) を利用してテストをすると、対象のテストの `check` が成功した、失敗したという結果が追加されます。

```
...
 ✓ is status 200

     checks.........................: 100.00% ✓ 10       ✗ 0 
...
```

複数項目を検証したい場合は以下の通り。

```js
check(res, {
  'is status 200': (r) => r.status === 200,
  'body size is 11,105 bytes': (r) => r.body.length == 11105,
});
```

```
...
✓ is status 200
✗ body size is 11,105 bytes
 ↳  0% — ✓ 0 / ✗ 10

checks.........................: 50.00% ✓ 10       ✗ 10 
...
```

https://k6.io/docs/using-k6/checks/#check-for-http-response-code

### Cookies を指定したリクエスト

```js:script.js
import http from 'k6/http';

export default function() {
  const params = {
    cookies: {
      my_cookie: 'hello world',
    }
  };
  // Cookie を付与してリクエスト
  const res = http.get('https://httpbin.test.k6.io/cookies', params);
}
```

https://k6.io/docs/using-k6/cookies/

### Authorization ヘッダーを指定したリクエスト

```js:script.js
import http from 'k6/http';

export default function() {
  const token = "xxxx";
  const params = {
    headers: {
      Authorization: `Bearer ${token}`,
    }
  };
  http.get('https://test.k6.io', params);
}
```