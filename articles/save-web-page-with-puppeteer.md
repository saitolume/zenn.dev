---
title: "puppeteer を用いて Web ページをまるごと保存する"
emoji: "💾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['JavaScript', 'puppeteer', 'CLI']
published: true
---

`puppeteer` は Headless Chrome 使用することができる Node.js のライブラリです。今回は、その `pupeteer` を用いて Web ページをのリソースをまるごと保存する方法を紹介します。

記事内で紹介するものは CLI として npm で公開しています。すぐに試したい方は次のコマンドから試すことができます。

```bash
$ npx ankipan https://zenn.dev
$ npx serve zenn.dev // 保存した Web ページは `serve` ですぐに確認できます
```

全体のコードを読みたい方は、以下のリンクからリポジトリを参照してください。

[![saitoeku3/ankipan - GitHub](https://gh-card.dev/repos/saitoeku3/ankipan.svg?fullname=)](https://github.com/saitoeku3/ankipan)

## 💬 動機

きっかけは Web ページのパフォーマンスニューニングをする環境を自作したくなったことでした。これを満たすためには、**任意の Web ページを再現できる形でそのまま保存する**必要がありました。

Web ページの保存は Chrome 等のブラウザに標準の機能として既に実装されていますが、これには遅延読み込みされるリソースを保存できないという問題があって、用途には合いませんでした。次に先行事例を調査すると、[`vanilla-clipper`](https://github.com/yarnaimo/vanilla-clipper) というツールがありました。しかし、標準で保存時に CSS が最適化される、保存形式が独特、最新リリースが正しく動作しないという理由で採用を見送りました。

以上の理由により、`puppeteer` を用いて要件を満たすスクリプトを自作することにしました。

## 🛠️ 使用技術

- Node.js: v10.12.0 以上
- TypeScript: v4.0.3
- puppeteer: v5.3.1

## 🔧 実装

### 1. Web ページを開く

`puppeteer` の API は非常にシンプルです。

まずは、`launch()` 関数を実行して、`Browser` のインスタンスを生成します。このときに Headless Chrome が起動します。次に `browser.pages()` メソッドを実行して、ページを 1 つ取得します。ここで取得したページに対して、任意の処理を行って Headless Chrome を操作するのが一般的な使い方です。

```ts
import { launch } from 'puppeteer'

const browser = await launch() // Headless Chrome を起動
const [page] = await browser.pages() // ページを取得
```

ページを開くときは `page.goto()` メソッドに任意の URL を引数として渡すだけです。
次の例では、Zenn のトップページを開きます。

```ts
await page.goto('https://zenn.dev') // Zenn のトップページを開く
```

### 2. リソースを取得する

Web ページのリソースを把握するには、**HTML や CSS などを解析して URL を抽出する**か**Web ページのリクエスト (またはレスポンス) を監視する**必要があります。前者は実装コストが高いため、今回は `puppeteer` の `response` イベントを利用して後者で実装します。

[`page.on('response')`](https://pptr.dev/#?product=Puppeteer&version=v5.3.1&show=api-event-response) は、リクエストのレスポンスが返ってきたときにコードバック関数を実行するメソッドです。コードバック関数の第 1 引数からレスポンスの情報を取得することができます。

参考: https://pptr.dev/#?product=Puppeteer&version=v5.3.1&show=api-class-httpresponse

今回はレスポンスの内容をファイルとして保存するために、レスポンスの**ボディ**、 **URL**、**Content-Type** を取得します。

```ts
// 省略
page.on('response', async (res) => {
  const buffer = await res.buffer() // ボディ
  const url = new URL(res.url()) // URL
  const contentType = res.headers()['content-type'] // Content-Type
})
```

取得したデータを元にファイルに書き込みます。

```ts
import { promises as fs } from 'fs'
import { dirname } from 'path'

// 省略
page.on('response', async (res) => {
  const buffer = await res.buffer() // ボディ
  const url = new URL(res.url()) // URL
  const contentType = res.headers()['content-type'] // Content-Type

  // パスに対応したディレクトリを再帰的に作成して、ファイルを保存する
  await fs.mkdir(`${directory}${dirname(url.pathname)}`, { recursive: true })
  await fs.writeFile(`${directory}${url.pathname}`, buffer)
})
```

### 3. 遅延読み込みのリソースにも対応する

`puppeteer` を採用した理由は、任意の Web ページを再現可能な状態で保存することでした。ここでいう再現可能とは、現実で起こりうるイレギュラーなリソースの取得にも対応することです。

よくあるのは、ある要素が画面に入ったときに [IntersectionObserver API](https://developer.mozilla.org/ja/docs/Web/API/Intersection_Observer_API) を用いてリソースを取得する、いわゆる遅延読み込みです。これには Headless Chrome を起動後に最下部までスクロールすることで対応します。

```ts
// https://github.com/puppeteer/puppeteer/issues/305#issuecomment-385145048 から引用
await page.evaluate(async () => {
  await new Promise((resolve) => {
    let totalHeight = 0
    const distance = 100
    const timer = setInterval(() => {
      const scrollHeight = document.body.scrollHeight
      scrollBy(0, distance)
      totalHeight += distance
      if (totalHeight >= scrollHeight) {
        clearInterval(timer)
        resolve()
      }
    }, 100)
  })
})
```

注目するポイントは [`page.evaluate()`](https://pptr.dev/#?product=Puppeteer&version=v5.3.1&show=api-pageevaluatepagefunction-args) メソッドです。このメソッド内に書かれる処理はブラウザ上で実行されるため、`window` オブジェクトに生えている `scrollBy` を使用することができます。

これによってスクロール中に発生するレスポンスもハンドリングされるので、遅延読み込みのリソースも保存することができます。

## 📝 まとめ

`puppeteer` を用いると、Headless Chrome 経由で以下のことが簡単にできます。

- `launch()`: Headless Chrome を起動
- `browser.pages()`: ページを取得
- `page.goto()`: 任意のページを開く
- `page.on('response')`: レスポンスをハンドリングしてリソースを取得
- `page.evaluate()`: 任意の JavaScript をブラウザ上で実行

また、これらを用いて Web ページをのリソースをまるごと保存する CLI を作成しました。
実際に使用してバグや機能の要望があれば、リポジトリの [issues](https://github.com/saitoeku3/ankipan/issues) にてご報告いただけると幸いです。

[![saitoeku3/ankipan - GitHub](https://gh-card.dev/repos/saitoeku3/ankipan.svg?fullname=)](https://github.com/saitoeku3/ankipan)
