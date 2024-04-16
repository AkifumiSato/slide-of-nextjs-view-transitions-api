---
theme: default
# https://unsplash.com/ja/%E5%86%99%E7%9C%9F/%E7%B7%91%E8%B5%A4%E9%9D%92%E3%81%AE%E5%85%89--MzXKfizmQs
background: /assets/top.png
class: text-center
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
transition: slide-left
title: App Router with View Transitions API
mdc: true
---

# App Router with<br>View Transitions API

@東京Node学園祭202404

---

# Profile

- name: 佐藤 昭文（Akifumi Sato）
  - twitter: akfm_sato
  - github: AkifumiSato
  - Web Engineer
- Activity
  - https://zenn.dev/akfm
  - [JS Conf](https://main--remarkable-figolla-a694f0.netlify.app/1)
  - [Vercel meetup](https://zesty-basbousa-04576f.netlify.app/1)

---

# 先にDEMO

App Routerのsoft navigationをいい感じにアニメーションする

- http://localhost:3000/
- https://github.com/AkifumiSato/next-view-transition

---

# View Transitions API とは

https://developer.chrome.com/docs/web-platform/view-transitions?hl=ja

- ページ遷移時のアニメーションを容易にするためのAPI
- SPAをターゲットとし、DOM更新を`document.startViewTransition(callback)`に渡す
- DOM更新前後のキャプチャ擬似要素が作成され、アニメーションされる
- アニメーションは擬似要素に対するスタイルとして定義できる
- 遷移前後で似た要素があれば紐づけることができる

---
transition: fade
---

# View Transitions APIの実装

JSで遷移処理を`document.startViewTransition(callback)`に渡す

```js
document.startViewTransition(async () => {
  // DOMの更新処理
  const data = await fetchData();
  updateTheDOMSomehow(data);
});
```

---

# View Transitions APIの実装

遷移前後の擬似要素に対してアニメーションを定義

```css
@keyframes fade-in {
  from { opacity: 0; }
}

@keyframes fade-out {
  to { opacity: 0; }
}

::view-transition-old(root) {
  animation: 90ms cubic-bezier(0.4, 0, 1, 1) both fade-out;
}

::view-transition-new(root) {
  animation: 210ms cubic-bezier(0, 0, 0.2, 1) 90ms both fade-in;
}
```

---

# App Routerで扱うのが難しい

Next.jsはView Transitions APIを考慮して設計・実装されてはいない

- soft navigationによるレンダリングを`Promise`として扱えない
  - Pages Routerならeventがあったので容易に扱えた
  - `Link`の遷移は当然関数として扱えない
  - `router.push()`は`Promise`を返さないし、suspenseが発生するので同期的にも完了しない
- discussion中
  - https://github.com/vercel/next.js/discussions/46300
  - 昔自分が立てたけどなんかすごい盛り上がってる
- Nuxtは早い段階からサポートして容易に実装ができる
  - https://nuxt.com/docs/getting-started/transitions

---

# 余談: next-view-transitionsのリリース

本発表資料作成中にNext.jsコアチームメンバーの[shuding](https://twitter.com/shuding_)氏によってリリースされた

https://github.com/shuding/next-view-transitions

- ~~App Routerで扱うのが難しい~~ -> 容易に扱うライブラリが登場
- 先を越されたので本発表は「`useTransition`すごいね」なお話しとして聞いてください

---
transition: fade
---

# レンダリングをPromiseで扱う方法が提案される

前述のdiscussionでstylexの作者より提案された方法

- `startTransition`の`Promise`版hooks
  - https://github.com/vercel/next.js/discussions/46300#discussioncomment-5894648
  - `useTransition`の復習
    - https://zenn.dev/uhyo/books/react-concurrent-handson-2
    - レンダリングを裏の世界だけで行って反映を遅延させるAPI
    - suspendを含むレンダリングが全て完了してから描画される
- `router.push()`によるレンダリングを`transition`とすることで、**遷移の開始〜終了を`Promise`として扱える**
  - `Promise`を`document.startViewTransition`に渡せる=View Transitions APIが扱える

---

# レンダリングをPromiseで扱う方法が提案される

前述のdiscussionでstylexの作者より提案された方法

```tsx
/* 通常のuseTransition */
const [isPending, startTransition] = useTransition();
// 戻り値なし
startTransition(() => {
  router.push('/new-page');
})
```

```tsx
/* カスタムhooksの利用イメージ */
const [isPending, startTransitionWithCompletion] = useTransitionWithCompletion();
// 戻り値がPromise
const transitionPromise: Promise<void> = startTransitionWithCompletion(() => {
  router.push('/new-page');
})
```

---

# soft navigationをPromiseにする

`router.push()`によるレンダリングを`transition`を通してPromiseとして扱える

```tsx
// LinkのonClickを↓してしまえば、View Transitions APIも扱えるはず
const [_isPending, startTransitionWithCompletion] =
  useTransitionWithCompletion();
const onClick = useCallback<React.MouseEventHandler<HTMLAnchorElement>>(
  (e) => {
    e.preventDefault();
    document.startViewTransition(() =>
      startTransitionWithCompletion(() => {
        router.push(href);
      }),
    );
  },
  [router, href, startTransitionWithCompletion],
);
```

---
transition: fade
---

# 実際に試してみる

https://github.com/AkifumiSato/next-view-transition

1. `useTransitionWithCompletion`を実装する
2. `AnimationLink`を作る
    - `Link`の`onClick`で`preventDefault()`する（prefetchはしたいので`Link`は使う）
    - `startTransitionWithCompletion(router.push("/new-page"))`で遷移の`Promise`を作る
    - `document.startViewTransition`がある時だけ↑を渡す
3. CSSでView Transitionsの擬似要素にアニメーションスタイルを定義する
   - `root`のアニメーションを定義
   - 要素間（ヘッダーとか画像とか）のアニメーションを定義
4. `popstate`イベントで遷移Promiseを作成してブラウザバックにも対応する

---

# 実際に試してみる

https://github.com/AkifumiSato/next-view-transition

1. `useTransitionWithCompletion`を実装する
   - https://github.com/AkifumiSato/next-view-transition/blob/main/src/app/_lib/use-transition-with-completion.ts
2. `AnimationLink`を作る
   - https://github.com/AkifumiSato/next-view-transition/blob/main/src/app/_components/animation-link.tsx
   - https://github.com/AkifumiSato/next-view-transition/blob/main/src/app/_lib/document-transition.ts
3. CSSでView Transitionsの擬似要素にアニメーションスタイルを定義する
   - https://github.com/AkifumiSato/next-view-transition/blob/main/src/app/globals.css
4. `popstate`イベントで遷移Promiseを作成してブラウザバックにも対応する
   - https://github.com/AkifumiSato/next-view-transition/blob/main/src/app/_components/back-forward-transition.tsx

---

# まとめ・感想

- `useTransition`はとても強力なAPI
  - App RouterがReactの最新機能で作られてるが故、transitionとの組み合わせの可能性はもっとありそう
- View Transitions APIのアニメーション実装は今までのCSSからすると癖が強く感じるができあがると結構楽に感じる
- SafariもView Transitions APIの実装進んでるっぽい（近々リリース？）
