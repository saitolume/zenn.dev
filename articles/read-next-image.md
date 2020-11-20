---
title: "Next.js 10 で追加された画像最適化システム next/image を読んで理解を深める"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['javascript', 'nextjs']
published: true
---

Next.js 10 では `next/image` から提供されるコンポーネントを使用することで、開発者が特別に意識することなく画像を最適化することができるようになりました。

TODO: もう少し良い感じの導入を書く

## 何を調べるのか

目的を持たずに読んでいると露頭に迷いそうなので、最初に何を調べるのか決めます。
今回は最適化の仕組みを紐解くことを目的として、コードを読みながら次のようにステップごとに調べることにしました。

1. 最適化された画像の出し分け
2. 画像最適化処理

## どのように調べるのか

Next.js はリポジトリの `text/integration/**` の中にインテグレーションテストが大量にあります。ログを出力しながら動作確認をしてコードを読みたいときは、次の 2 つのコマンドを実行します。

```bash
$ yarn dev
$ yarn next ./test/integration/basic
```

`yarn dev` を実行すると、packages の変更を監視してリアルタイムで反映します。
`yarn next path-to-integration-test` を実行すると、開発環境の Next.js でアプリケーションのサーバーが立ち上がって動作確認ができるようになります。

## 最適化した画像の出し分け

`next/image` では画像の出し分けをローダーを用いて行います。

```ts
// packages/next/client/image.tsx
const loaders = new Map<LoaderValue, (props: LoaderProps) => string>([
  ['imgix', imgixLoader],
  ['cloudinary', cloudinaryLoader],
  ['akamai', akamaiLoader],
  ['default', defaultLoader],
])
```

標準では `defaultLoader` が使用されます。
`imgixLoader`、`cloudinaryLoader`、`akamaiLoader` は外部の画像最適化サービスを利用するときに使用します。
`next.config.js` の設定を拡張することで、画像最適化処理のみ外部サービスに移譲できます。

https://nextjs.org/docs/basic-features/image-optimization#loader

今回は [imgix](https://www.imgix.com/) のローダーを取り上げます。

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

imgix のような画像最適化サービスは、URL で指定した形式で画像を最適化します。
サービスによって指定する形式が微妙に異なるため、その差分を吸収するのがローダーの役割です。

ちなみに引数の `root` のデフォルトはビルド先である `/_next/image` です。
このときの出力結果は、`/_next/image?url=%2Fsample.jpg&w=1200&q=75"` のようになります。

## 画像最適化処理

ローダーで `default` が設定されているときは、Next.js 標準の画像最適化処理を行います。
それではどのように処理を行うのか追うために、`default` のローダーが出力するパス `/_next/image` に対応するルーティングを見てみます。

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

`/next/image` が叩かれると、`imageOptimizer` という明らかに画像を最適化していそうな関数が実行されることがわかります。
この関数は `packages/next/next-server/server/image-optimizer.ts` に定義されています。

ザックリとまとめると次のことを行います。

1. リクエストの解釈
2. キャッシュの確認
3. 最適化された画像の生成

### リクエストの解釈

URL やそれに含まれるクエリなどが正しい値か検証して、画像最適化に必要なパラメータを解釈します。
例えば、画像の width や quality、形式 (jpeg や webp) などがこれに当てはまります。

リリース時に話題になった webp 対応もここで行われます。
TODO: 調べてまとめる

### キャッシュの確認

画像最適化に必要なパラメータが確定したら、それに合致するキャッシュが存在するのか確認して、存在する場合は最適化処理をスキップします。最適化された画像は `.next/cache/images/**` にキャッシュされます。

```ts
// packages/next/next-server/server/next-server.ts
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

### 最適化された画像の生成

読み進めていると、Next.js は最適化処理自体は行わずに `sharp` という npm に公開されている画像最適化ライブラリを使用していることがわかりました。

https://github.com/lovell/sharp
