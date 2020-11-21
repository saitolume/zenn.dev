---
title: "Next.js 10 の画像最適化システム next/image を読んで理解を深める"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['javascript', 'nextjs']
published: true
---

**※ ソースコードは 2020/11/20 時点の canary ブランチを参照しています。**

Next.js 10 では `next/image` から提供されるコンポーネントを使用することで、開発者が特別に意識することなく画像を最適化することができるようになりました。リリースのタイミングで Next.js Conf が開催されていたこともあり、この機能は大きく話題になりました。
今回はコードを読みながら最適化の裏側を紐解いて `next/image` の理解を深めようと思います。

## 何を調べるのか

目的を持たずに読んでいると露頭に迷いそうなので、最初に何を調べるのか決めました。
今回は最適化の仕組みを紐解くことを目的として、コードを読みながら次の 2 つについて調べます。

1. 最適化された画像の出し分け
2. 画像最適化処理

## 結論

1. 最適化された画像の出し分け
   - `img` 要素の `srcset` 属性を利用して画面サイズに合う画像を表示している
   - コンポーネントでは props の値から画像最適化に使用する URL を生成する
   - `next.config.js` でローダーを指定することで外部サービスを用いた画像最適化ができる
2. 画像最適化処理
   - 画像の最適化や形式の変更はクライアントのリクエストを元にバックエンドで行う
   - サーバーとクライアント両方で最適化された画像をキャッシュしている
   - 最適化処理には `sharp` というライブラリが使用されている

## どのように調べるのか

Next.js はリポジトリの `text/integration/**` の中にインテグレーションテストが大量にあります。ログを出力しながら動作確認をしてコードを読みたいときは、次の 2 つのコマンドを実行します。

```bash
$ yarn dev
$ yarn next ./test/integration/basic
```

`yarn dev` を実行すると、packages の変更を監視してリアルタイムで反映します。
`yarn next path/to/integration/test` を実行すると、開発環境の Next.js でアプリケーションのサーバーが立ち上がって動作確認ができるようになります。

## 最適化した画像の出し分け

`next/image` では画像の出し分けを `img` 要素の `srcset` 属性を用いて行います。

![Chrome Devtools の element タブに表示されている img 要素](https://storage.googleapis.com/zenn-user-upload/m7f7zn2de76raovij1j4kqwipx5g)

`srcset` 属性には複数の画像の URL が指定されています。ここに画面サイズごとに適した URL を指定することで、大きい画面のときは大きい画像を、小さい画面のときは小さい画像を出し分けることができます。

しかし、これを全て開発者が定義するのは面倒なので `next/image` は指定された props に応じて `srcset` 属性の値を自動で生成します。コードを読みながら動作を追っていきます。

まず明らかに怪しいのが `srcSet` という変数です。ここでは `callLoader` 関数を使用して `srcSet` を算出していることがわかります。

```ts
// packages/next/client/image.tsx
const { widths, kind } = getWidths(width, layout)
const last = widths.length - 1

const srcSet = widths
  .map(
    (w, i) =>
      `${callLoader({ src, quality, width: w })} ${
        kind === 'w' ? w : i + 1
      }${kind}`
  )
  .join(', ')
```

次にローダーがどんなものか見てみます。`loaders` という複数のローダーが定義されている箇所が見つかりました。

```ts
// packages/next/client/image.tsx
const loaders = new Map<LoaderValue, (props: LoaderProps) => string>([
  ['imgix', imgixLoader],
  ['cloudinary', cloudinaryLoader],
  ['akamai', akamaiLoader],
  ['default', defaultLoader],
])
```
`imgixLoader`、`cloudinaryLoader`、`akamaiLoader` は外部の画像最適化サービスの名前が使用されています。Next.js のドキュメントによると、`next.config.js` でこれらのローダーを指定することで `next/image` の画像最適化処理を外部サービスに置き換えられるようです。

https://nextjs.org/docs/basic-features/image-optimization#loader

今回は Next.js 標準の実装を見たいので `defaultLoader` を調べてみたいところですが、どのローダーも実装はほぼ同じだったのでシンプルな `imgixLoader` を取り上げます。

```ts
// packages/next/client/image.tsx
function imgixLoader({ root, src, width, quality }: LoaderProps): string {
  // Demo: https://static.imgix.net/daisy.png?format=auto&fit=max&w=300
  const params = ['auto=format', 'fit=max', 'w=' + width]
  let paramsString = ''
  if (quality) {
    params.push('q=' + quality)
  }

  if (params.length) {
    paramsString = '?' + params.join('&')
  }
  return `${root}${normalizeSrc(src)}${paramsString}`
}
```

ソースコード内の Demo の URL を開いて試したところ、imgix や cloudinary のような画像最適化サービスは URL のクエリに渡したパラメータを参照して画像を最適化していることがわかりました。しかし、サービスによって指定する形式が微妙に異なるため、その差分を吸収するのがローダーの役割です。

ちなみに `defautLoader` を使用した場合は、引数の `root` にはビルド先である `/_next/image` が渡されます。
このときの出力結果は、`/_next/image?url=%2Fsample.jpg&w=1200&q=75"` のようになり、正しく `srcset` 属性に指定されていた URL になります。

これで `next/image` はクライアントの画面サイズに合わせて画像の URL (パラメータを指定したクエリ付き) を生成しているということがわかりました。

## 画像最適化処理

次に Next.js 標準の画像最適化処理を調べるために `defaultLoader` が出力するパス `/_next/image` に対応するルーティングを見てみました。

```ts
// packages/next/next-server/server/next-server.ts
{
   match: route('/_next/image'),
   type: 'route',
   name: '_next/image catchall',
   fn: (req, res, _params, parsedUrl) =>
    imageOptimizer(server, req, res, parsedUrl),
}
```

`/next/image` が叩かれると、`imageOptimizer` という明らかに画像を最適化していそうな関数が実行されることがわかります。この関数はそこそこ大きいので、主な役割を 3 つに絞って紹介します。

### リクエストの解釈

まずはじめに URL やそれに含まれるクエリなどが正しい値か検証して、画像最適化に必要なパラメータを解釈します。例えば、画像の width や quality、形式 (JPEG や WebP) などがこれに当てはまります。リリース時に話題になった WebP 対応ブラウザへの WebP 対応もここで行われます。

```ts
// packages/next/next-server/server/image-optimizer.ts
import { mediaType } from '@hapi/accept'

const WEBP = 'image/webp'
const MODERN_TYPES = [/* AVIF, */ WEBP]

// 色々省略
const mimeType = getSupportedMimeType(MODERN_TYPES, headers.accept)
const width = parseInt(w, 10)
const quality = parseInt(q)

function getSupportedMimeType(options: string[], accept = ''): string {
  const mimeType = mediaType(accept, options)
  return accept.includes(mimeType) ? mimeType : ''
}
```

WebP 対応の流れを追ってみましょう。まず、`@hapi/accept` の `mediaType` 関数に WebP の MIME タイプと Accept ヘッダーを渡しています。この関数は 2 つの引数に含まれる MIME タイプからレスポンスに適切な MIME タイプを算出してくれます。

ちなみに Accept ヘッダーには、クライアントが理解できるコンテンツの MIME タイプが記述されています。これを使用してクライアントごとに WebP に対応しているか確認していました。

https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Accept

### キャッシュの確認

画像最適化に必要なパラメータが確定したら、それに合致するキャッシュが存在するのか確認して、存在する場合は最適化処理をスキップします。最適化された画像は `.next/cache/images/**` にキャッシュされます。

```ts
// packages/next/next-server/server/image-optimizer.ts
const hash = getHash([CACHE_VERSION, href, width, quality, mimeType]) // パラメータからハッシュを作成
const imagesDir = join(distDir, 'cache', 'images')
const hashDir = join(imagesDir, hash) // ここにキャッシュが保存される
const now = Date.now()

if (await fileExists(hashDir, 'directory')) {
 const files = await promises.readdir(hashDir)
 for (let file of files) {
   const [prefix, etag, extension] = file.split('.')
   const expireAt = Number(prefix)
   const contentType = getContentType(extension)
   const fsPath = join(hashDir, file)
   if (now < expireAt) {
     res.setHeader('Cache-Control', 'public, max-age=0, must-revalidate')
     if (sendEtagResponse(req, res, etag)) {
       return { finished: true }
     }
     if (contentType) {
       res.setHeader('Content-Type', contentType)
     }
     createReadStream(fsPath).pipe(res)
     return { finished: true }
   } else {
     await promises.unlink(fsPath)
   }
 }
}
```

キャッシュについてもう少し深堀りしてみます。コードから Cache-Control ヘッダーに `public, max-age=0, must-revalidate` が指定されていることが読み取れます。つまり、**キャッシュは残すけど使用する前に再検証しなければならない**ということです。これに対して、Next.js は `sendEtagResponse` 関数経由で etag を返却することによって、キャッシュを使用しても良いことをブラウザに伝えています。

```ts
// packages/next/next-server/server/send-payload.ts
export function sendEtagResponse(
  req: IncomingMessage,
  res: ServerResponse,
  etag: string | undefined
): boolean {
  if (etag) {
    /**
     * The server generating a 304 response MUST generate any of the
     * following header fields that would have been sent in a 200 (OK)
     * response to the same request: Cache-Control, Content-Location, Date,
     * ETag, Expires, and Vary. https://tools.ietf.org/html/rfc7232#section-4.1
     */
    res.setHeader('ETag', etag)
  }

  if (fresh(req.headers, { etag })) {
    res.statusCode = 304
    res.end()
    return true
  }

  return false
}
```

実際に動作確認をすると、304 Not Modified になってブラウザに保存されているキャッシュを使用していることがわかります。

![Chrome Devtools の Network タブ](https://storage.googleapis.com/zenn-user-upload/pv9ol64d8yac49400f8eu0p3d8wt)

キャッシュ周りの用語については MDN の Cache-Control を参照すると理解しやすいです。

https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Cache-Control

### 最適化された画像の生成

最適化の処理を読み進めていると、Next.js 自体は最適化処理は行わずに `sharp` という npm に公開されている画像最適化ライブラリを使用していることがわかりました。

https://github.com/lovell/sharp

```ts
 const transformer = sharp(upstreamBuffer)
 transformer.rotate() // auto rotate based on EXIF data

 const { width: metaWidth } = await transformer.metadata()

 if (metaWidth && metaWidth > width) {
   transformer.resize(width)
 }

 if (contentType === WEBP) {
   transformer.webp({ quality })
 } else if (contentType === PNG) {
   transformer.png({ quality })
 } else if (contentType === JPEG) {
   transformer.jpeg({ quality })
 }

 const optimizedBuffer = await transformer.toBuffer()
 sendResponse(req, res, contentType, optimizedBuffer)
```

最適化処理の部分だけ抜き出すと非常にシンプルなことがわかります。やっていることは画像の「回転」「サイズ変更」「任意の形式への変換」「最適化されたバッファの出力」です。`contentType` は**リクエストの解釈**で算出した MIME タイプなので、WebP 対応のクライアントからのリクエストの場合は WebP に変換します。

## 結論 (再掲)

1. 最適化された画像の出し分け
   - `img` 要素の `srcset` 属性を利用して画面サイズに合う画像を表示している
   - コンポーネントでは props の値から画像最適化に使用する URL を生成する
   - `next.config.js` でローダーを指定することで外部サービスを用いた画像最適化ができる
2. 画像最適化処理
   - 画像の最適化や形式の変更はクライアントのリクエストを元にバックエンドで行う
   - サーバーとクライアント両方で最適化された画像をキャッシュしている
   - 最適化処理には `sharp` というライブラリが使用されている

## おまけ

React のフレームワークで画像最適化と言えば、Gatsby の `gatsby-image` を想像する方も多いでしょう。
少しだけ違いに触れると、`gatsby-image` は SSG (Static Site Generation) 専用のフレームワークなので、画像最適化はランタイムではなくてビルド時に行います。一方 Next.js は、SSR (Server Side Rendering) や SPA (Single Page Application) としても振る舞うため、前述のようにランタイムで画像を最適化を行います。

そこで Next.js で SSG する場合はどうなるのか実際に試してみました。

```
Error: Image Optimization using Next.js' default loader is not compatible with `next export`.
Possible solutions:
  - Use `next start`, which starts the Image Optimization API.
  - Use Vercel to deploy, which supports Image Optimization.
  - Configure a third-party loader in `next.config.js`.
Read more: https://err.sh/next.js/export-image-api
```

エラーに書かれているように、`defaultLoader` を使用している場合は画像最適化が行われませんでした。SSG で画像最適化をする場合は、「`next start` を使用する」「Vercel にデプロイする」「サードパーティのローダーを設定する」のどれかをする必要があるようです。
