# 作業手順

## 1. フォルダを作る

```bash
mkdir newcar-vs-usedcar
cd newcar-vs-usedcar
```

## 2. SPEC.md をフォルダに入れる

ダウンロードした newcar-vs-usedcar-SPEC.md を
フォルダ内に SPEC.md としてコピーする。

## 3. Claude Code を起動

```bash
claude
```

## 4. 指示を出す

```
SPEC.md を読んで、その仕様通りに index.html を作成してください。
```

## 5. ブラウザで確認

```bash
# Mac
open index.html

# Windows
start index.html
```

## 6. 確認チェックリスト

□ スライダーを動かすとKPI・グラフ・テーブルがリアルタイム更新される
□ 中古車の年式を変えると残り所有年数が自動で変わる
□ KPI「年間コスト」が新車453,000円、中古車519,333円と表示される（デフォルト値の場合）
□ 維持費内訳テーブルの数値がスプレッドシートと一致する
□ グラフが表示されている
□ 「維持費の単価を調整する」の折りたたみが動作する
□ 印刷ボタンが動作する
□ 中古車の残り所有年数が0以下のとき警告が出る
