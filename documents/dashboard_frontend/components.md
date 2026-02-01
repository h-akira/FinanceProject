# FinanceDashboardProject_Frontend コンポーネント設計書

## 概要

本ドキュメントは、FinanceDashboardProject_Frontendのコンポーネント詳細設計を記載します。
基本設計については[README.md](./README.md)を参照してください。

## 技術仕様

- **Vue.js:** Composition API（`<script setup>`構文）を使用
- **状態管理:** `ref()`、`computed()`によるリアクティブ状態
- **ライフサイクル:** `onMounted()`、`onBeforeUnmount()`
- **ルーティング:** `useRouter()`によるプログラマティックナビゲーション

## コンポーネント階層

```
App.vue（ルートコンポーネント）
└── router-view
    └── Home.vue（ダッシュボードページ）
        ├── MarketSummary.vue（市場サマリー）
        ├── Calendar.vue（経済指標カレンダー）
        ├── Exchange.vue（為替チャート）
        ├── JP225.vue（日経平均チャート）
        └── M2.vue（マネーサプライチャート）
```

---

## App.vue

### 役割

- アプリケーションのルートコンポーネント
- ナビゲーションバーの表示
- 認証状態の管理とUI表示切り替え

### リアクティブ状態

| 変数名 | 型 | 初期値 | 説明 |
|--------|-----|--------|------|
| `authStatus` | `ref<Object \| null>` | `null` | 認証状態を保持 |

### メソッド

| メソッド名 | 引数 | 戻り値 | 説明 |
|-----------|------|--------|------|
| `checkAuthStatus` | なし | `Promise<void>` | `/accounts/status`から認証状態を取得し、`authStatus`を更新 |
| `handleRedirectAfterLogin` | なし | `void` | URLの`next`パラメータを処理し、ログイン後のリダイレクトを実行 |

### ライフサイクル

| フック | 処理内容 |
|--------|---------|
| `onMounted` | `checkAuthStatus()`を実行後、`handleRedirectAfterLogin()`を実行 |

### テンプレート構造

```html
<div id="app">
  <!-- Bootstrap Navbar (dark theme) -->
  <nav class="navbar navbar-expand-lg bg-dark" data-bs-theme="dark">
    <!-- ブランドロゴ -->
    <!-- 認証状態に応じたボタン表示 -->
    <!--   認証済み: Profile, Logout -->
    <!--   未認証: Login -->
  </nav>

  <!-- Vue Routerによるページコンテンツ -->
  <router-view />
</div>
```

### 認証状態によるUI切り替え

- `authStatus.authenticated === true`: Profile、Logoutリンクを表示
- それ以外: Loginリンクを表示

---

## Home.vue

### 役割

- ダッシュボードのメインページ
- チャートコンポーネントの動的表示/非表示制御
- レスポンシブなグリッドレイアウト管理

### リアクティブ状態

| 変数名 | 型 | 初期値 | 説明 |
|--------|-----|--------|------|
| `columnsPerRow` | `ref<string>` | `'2'` | 横に並べるカラム数（1, 2, 3） |
| `availableCharts` | `ref<Array>` | 下記参照 | 利用可能なチャート一覧 |

### availableChartsの構造

```javascript
[
  { id: 'market-summary', label: 'MarketSummary', component: MarketSummary, visible: true },
  { id: 'calendar', label: 'Calendar', component: Calendar, visible: true },
  { id: 'exchange', label: 'Exchange', component: Exchange, visible: true },
  { id: 'jp225', label: 'JP225', component: JP225, visible: true },
  { id: 'm2', label: 'M2', component: M2, visible: true }
]
```

### 算出プロパティ

| プロパティ名 | 依存 | 説明 |
|-------------|------|------|
| `visibleCharts` | `availableCharts` | `visible: true`のチャートのみをフィルタリング |
| `columnClass` | `columnsPerRow` | Bootstrap grid classを返却（下表参照） |

### columnClassのマッピング

| columnsPerRow | 返却値 |
|---------------|--------|
| `'1'` | `'col-12'` |
| `'2'` | `'col-12 col-lg-6'` |
| `'3'` | `'col-12 col-lg-4'` |

### テンプレート構造

```html
<div class="container-fluid mt-4">
  <!-- コントロールパネル -->
  <div class="row mb-3">
    <!-- チャート表示/非表示チェックボックス -->
    <!-- 横配置数セレクト（lg以上で表示） -->
  </div>

  <!-- チャートグリッド -->
  <div class="row">
    <div v-for="chart in visibleCharts" :class="columnClass">
      <component :is="chart.component" />
    </div>
  </div>
</div>
```

---

## MarketSummary.vue

### 役割

- TradingView Market Quotesウィジェットの表示
- 株価指数、先物、為替のサマリー表示

### リアクティブ状態

| 変数名 | 型 | 初期値 | 説明 |
|--------|-----|--------|------|
| `container` | `ref<HTMLElement \| null>` | `null` | ウィジェットコンテナへの参照 |

### メソッド

| メソッド名 | 引数 | 戻り値 | 説明 |
|-----------|------|--------|------|
| `loadWidget` | なし | `void` | TradingViewスクリプトを動的に読み込み |

### ライフサイクル

| フック | 処理内容 |
|--------|---------|
| `onMounted` | `loadWidget()`を実行 |
| `onBeforeUnmount` | コンテナ内のscriptタグを削除 |

### ウィジェット設定

```javascript
{
  width: '100%',
  height: '600',
  symbolsGroups: [
    { name: '指数', symbols: ['S&P 500', 'US 100', 'Dow 30', 'Nikkei 225', 'DAX', 'UK 100'] },
    { name: '先物', symbols: ['Gold', 'Oil', 'Gas', 'Corn', 'US10Y'] },
    { name: 'FX', symbols: ['USD/JPY', 'EUR/JPY', 'EUR/USD', 'GBP/JPY', 'AUD/JPY', 'GBP/USD', 'AUD/USD'] }
  ],
  colorTheme: 'light',
  locale: 'ja'
}
```

---

## Calendar.vue

### 役割

- TradingView Economic Calendarウィジェットの表示
- 経済指標の発表予定表示

### リアクティブ状態

| 変数名 | 型 | 初期値 | 説明 |
|--------|-----|--------|------|
| `container` | `ref<HTMLElement \| null>` | `null` | ウィジェットコンテナへの参照 |

### メソッド

| メソッド名 | 引数 | 戻り値 | 説明 |
|-----------|------|--------|------|
| `loadWidget` | なし | `void` | TradingViewカレンダースクリプトを動的に読み込み |

### ライフサイクル

| フック | 処理内容 |
|--------|---------|
| `onMounted` | `loadWidget()`を実行 |
| `onBeforeUnmount` | コンテナ内のscriptタグを削除 |

### ウィジェット設定

```javascript
{
  width: '100%',
  height: '600',
  colorTheme: 'light',
  locale: 'ja',
  importanceFilter: '0,1',  // 重要度フィルタ
  currencyFilter: 'AUD,USD,CAD,EUR,FRF,DEM,JPY,MXN,CHF,TRL,GBP'  // 通貨フィルタ
}
```

---

## Exchange.vue

### 役割

- TradingView Advanced Chartウィジェットによる為替チャート表示
- 通貨ペア選択機能
- インジケーター切り替え機能
- 高さリサイズ機能

### リアクティブ状態

| 変数名 | 型 | 初期値 | 説明 |
|--------|-----|--------|------|
| `selectedPair` | `ref<string>` | `'OANDA:USDJPY'` | 選択中の通貨ペア |
| `selectedIndicator` | `ref<string>` | `'Ichimoku'` | 選択中のインジケーター |
| `currentHeight` | `ref<number>` | `700` | チャートの高さ（px） |

### 非リアクティブ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `widget` | `TradingView.widget \| null` | ウィジェットインスタンス |
| `isResizing` | `boolean` | リサイズ中フラグ |

### 通貨ペアオプション

| 値 | 表示名 |
|----|--------|
| `'OANDA:USDJPY'` | USD/JPY |
| `'OANDA:EURJPY'` | EUR/JPY |
| `'OANDA:GBPJPY'` | GBP/JPY |
| `'OANDA:AUDJPY'` | AUD/JPY |
| `'OANDA:EURUSD'` | EUR/USD |

### インジケーターオプション

| 値 | TradingView Studies |
|----|---------------------|
| `'Ichimoku'` | `['STD;Ichimoku%1Cloud']` |
| `'Bollinger_SMA'` | `['STD;Bollinger_Bands', 'STD;SMA']` |

### メソッド

| メソッド名 | 引数 | 戻り値 | 説明 |
|-----------|------|--------|------|
| `getStudiesArray` | `indicator: string` | `string[]` | インジケーターに対応するStudies配列を返却 |
| `createWidget` | なし | `void` | TradingViewウィジェットを生成（既存があれば破棄） |
| `loadTradingViewScript` | なし | `void` | TradingView tv.jsを読み込み、完了後にウィジェット生成 |
| `updateWidget` | なし | `void` | ウィジェットを再生成 |
| `startResize` | なし | `void` | リサイズ開始（mousedownイベント） |
| `resize` | `event: MouseEvent` | `void` | リサイズ処理（mousemoveイベント） |
| `stopResize` | なし | `void` | リサイズ終了（mouseupイベント） |

### ライフサイクル

| フック | 処理内容 |
|--------|---------|
| `onMounted` | `loadTradingViewScript()`を実行 |
| `onBeforeUnmount` | ウィジェットの破棄、イベントリスナーの削除 |

### リサイズ制約

- 最小高さ: 300px
- 最大高さ: 1200px

---

## JP225.vue

### 役割

- TradingView Advanced Chartウィジェットによる日経平均チャート表示
- インジケーター切り替え機能
- 高さリサイズ機能

### リアクティブ状態

| 変数名 | 型 | 初期値 | 説明 |
|--------|-----|--------|------|
| `selectedIndicator` | `ref<string>` | `'Ichimoku'` | 選択中のインジケーター |
| `currentHeight` | `ref<number>` | `700` | チャートの高さ（px） |

### 非リアクティブ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `widget` | `TradingView.widget \| null` | ウィジェットインスタンス |
| `isResizing` | `boolean` | リサイズ中フラグ |

### 固定設定

- **シンボル:** `'OANDA:JP225USD'`

### インジケーターオプション

Exchange.vueと同一（Ichimoku / Bollinger_SMA）

### メソッド

Exchange.vueと同一構造（`selectedPair`がない点のみ異なる）

### ライフサイクル

Exchange.vueと同一

---

## M2.vue

### 役割

- Trading Economics埋め込みiframeによるマネーサプライM2チャート表示
- 日付範囲選択機能

### リアクティブ状態

| 変数名 | 型 | 初期値 | 説明 |
|--------|-----|--------|------|
| `startDate` | `ref<string>` | 10年前の日付（YYYY-MM-DD） | 開始日 |
| `endDate` | `ref<string>` | 今日の日付（YYYY-MM-DD） | 終了日 |

### 算出プロパティ

| プロパティ名 | 依存 | 説明 |
|-------------|------|------|
| `chartUrl` | `startDate`, `endDate` | iframeのsrc URLを生成 |

### URL生成

```javascript
`https://tradingeconomics.com/embed/?s=japanmonsupm2&v=...&d1=${startDate}&d2=${endDate}`
```

### メソッド

| メソッド名 | 引数 | 戻り値 | 説明 |
|-----------|------|--------|------|
| `updateChart` | なし | `void` | 日付変更時のイベントハンドラ（算出プロパティが自動更新） |

### テンプレート構造

```html
<div class="m2-component">
  <!-- 日付選択フォーム -->
  <div class="d-flex flex-wrap gap-2 mb-3">
    <input type="date" v-model="startDate" @change="updateChart">
    <input type="date" v-model="endDate" @change="updateChart">
  </div>

  <!-- Trading Economics iframe -->
  <iframe :src="chartUrl" width="600" height="300" />

  <!-- ソースリンク -->
</div>
```

---

## main.js（エントリーポイント）

### 役割

- Vueアプリケーションの初期化
- Vue Routerの設定
- 認証ガードの実装
- グローバルスタイルのインポート

### ルート定義

```javascript
const routes = [
  { path: '/', name: 'Home', component: Home, meta: { requiresAuth: true } }
]
```

### 認証ガード（beforeEach）

```
1. meta.requiresAuth === true のルートに対して実行
2. /accounts/status をfetchして認証状態を確認
3. 認証済み → next() で遷移許可
4. 未認証 → /accounts/login?next=<元のパス> にリダイレクト
```

### インポート順序

1. Vue core（createApp）
2. Vue Router
3. コンポーネント（App, Home）
4. Bootstrap CSS
5. Bootstrap JavaScript
6. カスタムCSS（index.css）

---

## 共通パターン

### TradingViewウィジェットの読み込みパターン

```javascript
const loadWidget = () => {
  const script = document.createElement('script')
  script.src = 'https://s3.tradingview.com/...'
  script.async = true
  script.type = 'text/javascript'
  script.innerHTML = JSON.stringify({ /* config */ })
  container.value.appendChild(script)
}

onMounted(() => loadWidget())

onBeforeUnmount(() => {
  // cleanup scripts
})
```

### TradingView Advanced Chartパターン

```javascript
let widget = null

const createWidget = () => {
  if (widget) widget.remove()

  widget = new TradingView.widget({
    width: '100%',
    height: currentHeight.value,
    symbol: '...',
    // ...config
    container_id: 'tradingview_xxx'
  })
}

const loadTradingViewScript = () => {
  if (window.TradingView) {
    createWidget()
    return
  }

  const script = document.createElement('script')
  script.src = 'https://s3.tradingview.com/tv.js'
  script.onload = createWidget
  document.head.appendChild(script)
}
```

### リサイズパターン

```javascript
let isResizing = false

const startResize = () => {
  isResizing = true
  document.addEventListener('mousemove', resize)
  document.addEventListener('mouseup', stopResize)
}

const resize = (event) => {
  if (!isResizing) return
  // calculate new height
  // update widget
}

const stopResize = () => {
  isResizing = false
  document.removeEventListener('mousemove', resize)
  document.removeEventListener('mouseup', stopResize)
}
```

---

## スタイル規約

### scoped CSS

- 各コンポーネントは`<style scoped>`を使用
- グローバルスタイルは`index.css`に記載

### Bootstrap利用

- グリッドシステム: `container-fluid`, `row`, `col-*`
- フォーム: `form-check`, `form-select`, `form-control`
- Navbar: `navbar`, `navbar-expand-lg`, `bg-dark`
- ボタン: `btn`, `btn-outline-light`
- ユーティリティ: `d-flex`, `gap-*`, `mt-*`, `mb-*`

### レスポンシブブレークポイント

- `d-none d-lg-flex`: lgサイズ以上で表示
- `col-12 col-lg-6`: lgサイズ以上で2カラム
- `col-12 col-lg-4`: lgサイズ以上で3カラム
