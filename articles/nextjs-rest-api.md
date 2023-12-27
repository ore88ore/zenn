---
title: "Next.js で API を実装してみた"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
publication_name: "yumemi_inc"
---

## はじめに

Next.js はフロントエンドの開発で利用されることが多いフレームワークだと思いますが、[Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) を使うことで、API の実装もできますので、実際に簡単な REST API を実装して試してみました。
実装したソース一式は、以下にて確認することができます。

https://github.com/ore88ore/next-rest-api-sample

## 開発環境の設定

### 実行環境

今回の実装では、以下のバージョンを利用しています。

- Node.js: `v20.10.0`
- TypeScript: `5.3.3`
- Next.js: `14.0.4`

### プロジェクトの構築

Next.js のプロジェクトを [create-next-app](https://nextjs.org/docs/app/api-reference/create-next-app) を使って作成します。
今回は最低限必要になるオプションのみ選択して作成しました。

```
npx create-next-app@latest next-rest-api-sample
✔ Would you like to use TypeScript? … Yes
✔ Would you like to use ESLint? … No
✔ Would you like to use Tailwind CSS? … No
✔ Would you like to use `src/` directory? … No
✔ Would you like to use App Router? (recommended) … Yes
✔ Would you like to customize the default import alias (@/*)? … No
```

## APIの実装

リクエストを受けた時に、パス、メソッドに応じた処理を実行する必要があります。Next.js では、`app/api` 配下にルートファイル( [route.ts](https://nextjs.org/docs/app/api-reference/file-conventions/route) )を作成して定義していきます。
今回のサンプルでは、user というリソースに対して、簡単な CRUD ができるようなエンドポイントを作成してみます。

```
GET /api/users
POST /api/users
GET /api/users/[id]
PUT /api/users/[id]
DELETE /api/users/[id]
```

### route.ts ファイルを配置

リクエストのパス毎に route.ts ファイルを配していきます。

```
project-root/
├── app/
│   ├── api/
│   │   ├── users/
│   │   │   ├── [id]/
│   │   │   │   └── route.ts
│   │   │   └── route.ts
```

ファイル内で必要なメソッドを定義します。

```ts:app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";

export function GET(request: NextRequest): NextResponse {
    // GET /api/users リクエストの処理
}

export function POST(request: NextRequest): NextResponse {
    // POST /api/users リクエストの処理
}
```

```ts:app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";

export function GET(request: NextRequest, { params }: { params: { id: string } }): NextResponse {
    // GET /api/users/[id] リクエストの処理
}

export function PUT(request: NextRequest, { params }: { params: { id: string } }): NextResponse {
    // PUT /api/users/[id] リクエストの処理
}

export function DELETE(request: NextRequest, { params }: { params: { id: string } }): NextResponse {
    // DELETE /api/users/[id] リクエストの処理
}
```

これで、パス、メソッドに応じた処理をハンドリングすることができます。

### クエリパラメータ

クエリパラメータを含むリクエストの場合、以下の方法で使用できます。

```ts
// ...
// 例）GET /api/users?query=hoge のようなリクエストの場合
export function GET(request: NextRequest): NextResponse {
  const params = request.nextUrl.searchParams;
  const query = params.get("query");
  // query = "hoge"
// ...
```

https://nextjs.org/docs/app/building-your-application/routing/route-handlers#url-query-parameters

### リクエストボディ

リクエストボディを含むリクエストの場合、以下の方法で使用できます。

```ts
// ...
// 例）POST /api/users (request body: {"key": "hoge"}) のようなリクエストの場合
export async function POST(request: NextRequest): Promise<NextResponse> {
  const params = await request.json();
  // params = {key: "hoge"}
// ...
```

https://nextjs.org/docs/app/building-your-application/routing/route-handlers#request-body

### パスパラメータ

URLにパラメータを含むリクエストの場合、以下の方法で使用できます。

```ts
// ...
// 例）GET /api/users/hoge のようなリクエストの場合
export function GET(request: NextRequest, { params }: { params: { id: string } }): NextResponse {
    // params = "hoge"
}
// ...
```

https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-route-segments

URL に複数のパスパラメータが含まれている場合は、`params` の型に追加することで利用することができます。

```ts
// ...
// 例）GET /api/users/hoge/items/1 のようなリクエストの場合
export function GET(request: NextRequest, { params }: { params: { id: string, itemId: number } }): NextResponse {
    // params.id = "hoge"
    // params.itemId = 1
}
// ...
```

https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes#typescript

### CORS

異なるドメインからのAPIアクセスには、CORS を設定してリソースを共有できます。

```ts
export function GET(request: NextRequest): NextResponse {
    return NextResponse.json(
        { response: "Test response." },
        {
          status: 200,  // ステータスコード
          headers: {    // レスポンスヘッダー
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
            "Access-Control-Allow-Headers": "Content-Type, Authorization",
          },
        },
  );
}
```

https://nextjs.org/docs/app/building-your-application/routing/route-handlers#cors

レスポンスオブジェクトにヘッダーを指定する時に、CORS のヘッダーを指定することで設定できそうです。エンドポイントが多い場合、ヘッダーの実装が冗長になりそうなので、[Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) でヘッダーを設定する方法も良さそうです。

https://nextjs.org/docs/app/building-your-application/routing/middleware#setting-headers

## ローカルでの動作確認

API のローカルでの動作確認は、`npm run dev` コマンドを使用して行います。このコマンドを実行すると、Next.js が開発サーバーを起動し、ホットリロード機能が有効化されます。ホットリロードのおかげで、コードの変更がリアルタイムで反映され、開発体験が向上し、効率的に実装作業を進めることができます。
サーバーはデフォルトだと `localhost:3000` で起動されるので、クライアントツールなどを利用して、アクセスすることができます。

```
$ npm run dev

> next-rest-api-sample@0.1.0 dev
> next dev

   ▲ Next.js 14.0.4
   - Local:        http://localhost:3000

 ✓ Ready in 2.3s

  ✓ Compiled /api/users in 66ms (38 modules)
GET /users query=hoge   // アプリの実装でログ出力した内容

```

標準出力に出力したログは、サーバーを起動したターミナルで確認することができます。

## アプリケーションのコンテナ化

ローカルでの実装が完了したら、いずれかの環境へデプロイして稼働させるかと思います。デプロイする際はコンテナ化しておくと、メリットが大きいので、公式のサンプルを参考にコンテナ化してみます。

https://github.com/vercel/next.js/tree/canary/examples/with-docker

```docker:Dockerfile
FROM node:20-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public

# Set the correct permission for prerender cache
RUN mkdir .next
RUN chown nextjs:nodejs .next

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

# server.js is created by next build from the standalone output
# https://nextjs.org/docs/pages/api-reference/next-config-js/output
CMD ["node", "server.js"]
```

上記の Dockerfile でコンテナをビルドします。

```
docker build -t next-rest-api-sample .
```

コンテナをビルドできたら、実際に起動して、動作確認してみます。

```
docker run -p 3000:3000 next-rest-api-sample
```

`npm run dev` コマンドで実行した時と同様に `localhos:3000` でアクセスするとリクエストを実行することができました。

## さいごに

Next.js で REST API を実装できることを知ってから、少し時間が空いてしまいましたが、今回実際に手を動かして試してみることができました。
今回はバックエンドの API のみを実装しましたが、フロントエンドと組み合わせて使用することで、同じ技術スタックを用いるメリットがより大きくなると感じました。バックエンド API として利用することは十分に可能だと思いますが、実装量が多くなる傾向があること、低レベルのコーディングが必要になると感じました。（大きな問題ではないですが）
今後のアップデートに期待しつつ、サーバーサイドの API 実装する際の選択肢の一つとしておきたいと思いました。

## 参照

https://nextjs.org/docs
