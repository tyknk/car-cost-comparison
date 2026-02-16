# 新車・中古車 コスト比較シミュレーター 設計書

## プロジェクト概要

トヨタディーラーの商談で使用する、新車と中古車の「トータルコスト比較」ツール。
購入金額だけでなく、所有期間中にかかる車検・整備・税金などの維持費を含めた総額を算出し、
**「1年あたりいくらかかるか」**を比較することで、新車と中古車のどちらが経済的かを可視化する。

### このツールが伝えたいこと

中古車は購入金額が安いが、所有できる期間が短い。
新車は購入金額が高いが、長く乗れるため維持費を含めた年換算コストでは安くなるケースがある。
「購入金額＋維持費の合計」を「所有年数」で割った**年間コスト**が比較の本質。

## 成果物

`index.html` 単一ファイル。サーバー不要でダブルクリックで開ける。

## 技術仕様

- 単一HTMLファイル（HTML + CSS + JavaScript を1ファイルに）
- グラフ描画: Chart.js v4（CDN: https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js）
- CSS変数でカラーテーマ管理
- レスポンシブ対応（タブレット横向き最優先、PC・スマホでも使用可能）
- 言語: 日本語

---

## 入力パラメータ

### パラメータの分類

入力パラメータは**2つのグループ**に分かれる。

**グループ1: 車両情報（上部に配置、必ず変更する項目）**

商談のたびに車種に合わせて変更する主要パラメータ。

| # | パラメータ名 | 変数名 | デフォルト | 範囲 | 刻み | 単位 | 説明 |
|---|------------|--------|-----------|------|------|------|------|
| 1 | 新車 購入金額 | newCarPrice | 4,600,000 | 500,000〜10,000,000 | 10,000 | 円 | 新車の車両本体＋諸費用 |
| 2 | 中古車 購入金額 | usedCarPrice | 3,500,000 | 500,000〜10,000,000 | 10,000 | 円 | 中古車の車両本体＋諸費用 |
| 3 | 中古車 年式 | usedCarYear | 2020 | 2005〜当年 | 1 | 年 | 中古車の年式（西暦） |
| 4 | 総所有期間 | totalOwnership | 15 | 5〜20 | 1 | 年 | 新車を何年乗るかの想定 |

※新車の年式は常に当年（自動計算）。
※中古車の残り所有年数は「総所有期間 − 中古車の経過年数」で自動算出。

**グループ2: 維持費単価（下部に配置、通常はデフォルトのまま使用）**

車種や地域によって変わる可能性がある単価。普段はデフォルトのまま使うが、変更もできる。

| # | パラメータ名 | 変数名 | デフォルト | 範囲 | 刻み | 単位 | 説明 |
|---|------------|--------|-----------|------|------|------|------|
| 5 | 車検整備費用 | shakenCost | 50,000 | 0〜200,000 | 5,000 | 円/回 | 車検時の整備工賃 |
| 6 | 車検法定費用 | shakenLegalCost | 45,000 | 0〜150,000 | 5,000 | 円/回 | 重量税＋自賠責＋印紙代 |
| 7 | 定期点検費用 | inspectionCost | 30,000 | 0〜100,000 | 5,000 | 円/回 | 車検がない年の12ヶ月点検等 |
| 8 | タイヤ交換費用 | tireCost | 130,000 | 0〜500,000 | 10,000 | 円/回 | タイヤ4本交換の費用 |
| 9 | バッテリー交換費用 | batteryCost | 25,000 | 0〜200,000 | 5,000 | 円/回 | バッテリー交換の費用 |
| 10 | 一般整備費用（年額） | annualRepairCost | 30,000 | 0〜200,000 | 5,000 | 円/年 | 保証期間終了後の年間整備費 |
| 11 | 自動車税（年額） | annualTax | 36,000 | 0〜150,000 | 1,000 | 円/年 | 排気量に応じた自動車税 |

グループ2のパラメータは**新車・中古車共通の値**として使用する。
（実際の商談でも同じ車種の新車vs中古車を比較するため、整備単価は同額になる）

### 入力方式

- グループ1: **スライダー＋数値入力の双方向連動**（燃費シミュレーターと同じ方式）
- グループ2: **数値入力欄＋スライダー**。折りたたみ可能なセクション（デフォルトは閉じた状態）にして、画面をスッキリさせる。「維持費の単価を調整」のようなボタンで開閉。

---

## 計算ロジック（詳細解説）

### ステップ1: 基本情報の算出

```javascript
// 当年の西暦
const currentYear = new Date().getFullYear();

// 新車
const newCarAge = 0;                                    // 新車は経過0年
const newCarOwnership = totalOwnership;                 // 新車の所有年数 = 総所有期間そのまま

// 中古車
const usedCarAge = currentYear - usedCarYear;           // 中古車の経過年数（例: 2026-2020 = 6年）
const usedCarOwnership = totalOwnership - usedCarAge;   // 中古車の残り所有年数（例: 15-6 = 9年）
```

**注意: 中古車の残り所有年数が0以下になる場合**
→ 「この中古車は想定所有期間を既に超えています」と警告を表示する。

### ステップ2: 回数の算出

**車検回数の計算（スプレッドシートの数式を忠実に再現）**

```javascript
// 新車の車検回数
// 初回車検は3年後、以降2年ごと → (所有年数 - 2.5) / 2 を切り捨て
// 2.5は「3年目の車検を1回目として数えるため」の補正値
// 例: 15年所有 → (15-2.5)/2 = 6.25 → 6回（3,5,7,9,11,13年目）
const newShakenCount = Math.floor((newCarOwnership - 2.5) / 2);

// 中古車の車検回数
// 既に初回車検済みのため2年ごと → (所有年数 - 1.5) / 2 を切り捨て
// 1.5は「2年ごとの車検タイミング」の補正値
// 例: 9年所有 → (9-1.5)/2 = 3.75 → 3回
const usedShakenCount = Math.floor((usedCarOwnership - 1.5) / 2);
```

**タイヤ交換回数（5年ごと）**

```javascript
const newTireCount = Math.floor(newCarOwnership / 5);    // 例: 15/5 = 3回
const usedTireCount = Math.floor(usedCarOwnership / 5);  // 例: 9/5 = 1回
```

**バッテリー交換回数（3年ごと）**

```javascript
const newBatteryCount = Math.floor(newCarOwnership / 3);    // 例: 15/3 = 5回
const usedBatteryCount = Math.floor(usedCarOwnership / 3);  // 例: 9/3 = 3回
```

### ステップ3: 費用の算出

```javascript
// --- 新車 ---
// 定期整備代 = 車検整備費用×車検回数 + 定期点検費用×(所有年数−車検回数)
// 理由: 所有年数のうち、車検がある年は車検整備費、ない年は定期点検費がかかる
const newMaintenance = shakenCost * newShakenCount + inspectionCost * (newCarOwnership - newShakenCount);

// 法定費用 = 車検法定費用 × 車検回数
const newLegalCost = shakenLegalCost * newShakenCount;

// タイヤ交換 = タイヤ交換費用 × 回数
const newTireCost = tireCost * newTireCount;

// バッテリー交換 = バッテリー交換費用 × 回数
const newBatteryCost = batteryCost * newBatteryCount;

// 一般整備費用（保証期間除く）
// 新車はメーカー保証5年間は大きな整備費がかからない想定
// 保証期間を超えた年数分だけ一般整備費がかかる
const newWarrantyYears = 5;
const newRepairCost = Math.max(0, newCarOwnership - newWarrantyYears) * annualRepairCost;

// 自動車税 = 年額 × 所有年数
const newTaxTotal = annualTax * newCarOwnership;

// 新車 維持費総額（購入金額込み）
const newTotalCost = newCarPrice + newMaintenance + newLegalCost + newTireCost + newBatteryCost + newRepairCost + newTaxTotal;

// --- 中古車 ---
// 計算構造は新車と同じ。保証期間が3年になる点のみ異なる。
const usedMaintenance = shakenCost * usedShakenCount + inspectionCost * (usedCarOwnership - usedShakenCount);
const usedLegalCost = shakenLegalCost * usedShakenCount;
const usedTireCost = tireCost * usedTireCount;
const usedBatteryCost = batteryCost * usedBatteryCount;

// 中古車の保証期間は3年（認定中古車等の想定）
const usedWarrantyYears = 3;
const usedRepairCost = Math.max(0, usedCarOwnership - usedWarrantyYears) * annualRepairCost;

const usedTaxTotal = annualTax * usedCarOwnership;

// 中古車 維持費総額（購入金額込み）
const usedTotalCost = usedCarPrice + usedMaintenance + usedLegalCost + usedTireCost + usedBatteryCost + usedRepairCost + usedTaxTotal;
```

### ステップ4: 年換算コスト（最重要KPI）

```javascript
// 年間コスト = 維持費総額 ÷ 所有年数
const newAnnualCost = newTotalCost / newCarOwnership;
const usedAnnualCost = usedTotalCost / usedCarOwnership;

// 差額（マイナスなら新車が安い）
const annualDifference = newAnnualCost - usedAnnualCost;
```

### 検算用データ

以下の入力条件で計算結果が合うことを確認すること:

入力: 新車購入金額=4,600,000 / 中古車購入金額=3,500,000 / 中古車年式=2020 / 総所有期間=15
維持費単価: 全てデフォルト値

| 項目 | 新車 | 中古車 |
|------|------|--------|
| 所有年数 | 15年 | 9年 |
| 車検回数 | 6回 | 3回 |
| タイヤ交換 | 3回 | 1回 |
| バッテリー交換 | 5回 | 3回 |
| 定期整備代 | 570,000 | 330,000 |
| 法定費用 | 270,000 | 135,000 |
| タイヤ交換費 | 390,000 | 130,000 |
| バッテリー交換費 | 125,000 | 75,000 |
| 一般整備費用 | 300,000 | 180,000 |
| 自動車税 | 540,000 | 324,000 |
| **維持費総額** | **6,795,000** | **4,674,000** |
| **年間コスト** | **453,000** | **519,333** |

この条件では新車の方が年間約66,333円安い。

---

## 画面構成

### 1. ヘッダー
- タイトル: 「新車・中古車 コスト比較シミュレーター」
- サブタイトル: 「購入金額＋維持費の年間コストで比較します」

### 2. 車両情報入力エリア（グループ1）
- 2列レイアウト: 左列「新車」、右列「中古車」
- 新車側: 購入金額のスライダー＋数値入力
- 中古車側: 購入金額、年式のスライダー＋数値入力
- 下部に「総所有期間」のスライダー（新車・中古車共通で1つ）
- 中古車の「経過年数」と「残り所有年数」は自動計算して表示（入力不要、表示のみ）
- 残り所有年数が0以下の場合は赤字で警告表示

### 3. KPIカード（メイン結果表示）

**上段: 年間コスト比較（最も大きく表示）**

3枚のカードを横並び:
| カード | 内容 | デザイン |
|-------|------|---------|
| 新車 年間コスト | 金額 | 左側、ブルー系 |
| 中古車 年間コスト | 金額 | 右側、オレンジ系 |
| 年間差額 | 金額＋「新車がお得」or「中古車がお得」 | 中央、最も大きく目立つ。お得な方の色で表示 |

**下段: 内訳サマリー（やや小さく表示）**

2枚のカードを横並び:
| カード | 内容 |
|-------|------|
| 新車 | 総額 ○○万円 ÷ ○○年 |
| 中古車 | 総額 ○○万円 ÷ ○○年 |

### 4. グラフ（Chart.js）

**棒グラフ（横並び比較）**を推奨。新車と中古車の維持費内訳を積み上げ棒グラフで並べる。

- X軸: 「新車」「中古車」の2本
- 積み上げ内訳: 購入金額 / 定期整備代 / 法定費用 / タイヤ交換 / バッテリー交換 / 一般整備 / 自動車税
- 各セグメントに異なる色を割り当て
- 棒の上に総額を表示

**もう1つ: 年間コスト比較の棒グラフ**
- 新車の年間コストと中古車の年間コストを2本の棒で比較
- こちらの方がシンプルで商談向き

→ 2つのグラフを並べるか、タブで切り替えられる形式にする。

### 5. 維持費内訳テーブル

スプレッドシートの左側（A〜C列）を再現する比較表。

| 項目 | 新車 | 中古車 | 説明 |
|------|------|--------|------|
| 購入金額 | ○○円 | ○○円 | |
| 定期整備代 | ○○円 | ○○円 | 車検○回 + 点検○回分 |
| 法定費用 | ○○円 | ○○円 | 車検○回分 |
| タイヤ交換 | ○○円 | ○○円 | ○回分 |
| バッテリー交換 | ○○円 | ○○円 | ○回分 |
| 一般整備費用 | ○○円 | ○○円 | 保証○年除く○年分 |
| 自動車税 | ○○円 | ○○円 | ○年分 |
| **維持費総額** | **○○円** | **○○円** | **太字で強調** |
| 所有年数 | ○年 | ○年 | |
| **年間コスト** | **○○円** | **○○円** | **最重要行。背景ハイライト** |

- 「説明」列で回数や年数を補足表示（「車検6回＋点検9回分」など）
- 年間コストが安い方の列をハイライト
- 金額はすべて3桁カンマ区切り、整数（Math.round）

### 6. 維持費単価調整セクション（グループ2、折りたたみ）
- デフォルトは**折りたたんだ状態**（閉じている）
- 「維持費の単価を調整する ▼」ボタンで開閉
- 開くと7つのパラメータがスライダー＋数値入力で表示される
- 変更するとリアルタイムに全体が再計算される

### 7. フッター
- 「※入力された条件に基づく概算です。実際の費用は車種・使用状況・地域により異なります。」
- 「※保証期間は新車5年、中古車3年として計算しています。」

### 8. 印刷ボタン
- 画面右上に配置
- @media print: 印刷ボタン非表示、スライダー非表示、折りたたみは展開状態で印刷

---

## デザイン仕様

### カラー（CSS変数）
```css
:root {
  --new-car-color: #2563EB;      /* 新車: ブルー */
  --new-car-light: #DBEAFE;
  --used-car-color: #EA580C;     /* 中古車: オレンジ */
  --used-car-light: #FFF7ED;
  --result-positive: #16A34A;    /* お得（結果表示用）: グリーン */
  --result-positive-light: #DCFCE7;
  --bg-main: #F9FAFB;
  --bg-card: #FFFFFF;
  --text-primary: #111827;
  --text-secondary: #6B7280;
  --border-color: #E5E7EB;
}
```

### グラフ配色（維持費内訳の積み上げ棒グラフ用）
```javascript
const chartColors = {
  purchase:    '#374151', // 購入金額: ダークグレー
  maintenance: '#2563EB', // 定期整備: ブルー
  legal:       '#7C3AED', // 法定費用: パープル
  tire:        '#059669', // タイヤ: グリーン
  battery:     '#D97706', // バッテリー: アンバー
  repair:      '#DC2626', // 一般整備: レッド
  tax:         '#6B7280', // 自動車税: グレー
};
```

### フォント・サイズ
- 年間コスト（KPI最重要）: 2.5rem、font-weight: bold
- 差額表示: 2rem、font-weight: bold
- テーブル内金額: 1rem、tabular-nums（桁揃え）
- 説明文: 0.875rem、text-secondary色

---

## コード構造の指針

```
index.html
├── <style> ... </style>
├── <div id="app">
│   ├── ヘッダー（タイトル＋サブタイトル）
│   ├── 車両情報入力エリア（グループ1）
│   ├── KPIカード（年間コスト3枚 + 内訳サマリー2枚）
│   ├── グラフエリア（canvas × 2 or タブ切り替え）
│   ├── 維持費内訳テーブル
│   ├── 維持費単価調整（折りたたみ、グループ2）
│   ├── フッター
│   └── 印刷ボタン
├── <script src="Chart.js CDN">
└── <script>
    ├── getParams()          // 全入力パラメータをオブジェクトで取得
    ├── calculate(params)    // 全計算を実行、結果オブジェクトを返す
    ├── updateKPI(result)    // KPIカード更新
    ├── updateChart(result)  // Chart.js更新（chart.update()を使用）
    ├── updateTable(result)  // テーブル再描画
    ├── formatNumber(n)      // 3桁カンマ区切り
    ├── toggleSettings()     // 折りたたみ開閉
    ├── イベントリスナー登録  // input イベントで双方向同期＋全体再計算
    └── 初期化処理           // ページ読込時に初回計算実行
</script>
```

---

## 作業の進め方

1. この設計書（SPEC.md）を読んで内容を把握する
2. index.html を作成する
3. 検算用データ（上記の表）で計算結果を検証する
4. 完成したら報告する
