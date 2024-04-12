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

- `Link`による遷移時のレンダリングを`Promise`として扱えない
  - Pages Routerならeventがあったので容易に扱えた
- `Link`での遷移は関数として`document.startViewTransition`に提供できない
- Nuxtは早い段階からサポートして容易に実装ができる
  - https://nuxt.com/docs/getting-started/transitions
- Next.jsでdiscussion中
  - https://github.com/vercel/next.js/discussions/46300

---

# レンダリングをPromiseで扱う方法が提案される

前述のdiscussionでstylexの作者より提案された方法

- `startTransition`の`Promise`版hooks
  - https://github.com/vercel/next.js/discussions/46300#discussioncomment-5894648
- `router.push()`と組み合わせれば**遷移の開始〜終了を`Promise`として扱える**
  - `Promise`を`document.startViewTransition`に渡せばView Transitions APIが扱えるということ

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

# 実際に試してみる

TBW

---

# 構成

- 実際の実装
  - `AnimationLink`を作る
  - `router.push()`をtransitionとして扱う
- View Transition APIのcss実装
- Demo
- まとめ
