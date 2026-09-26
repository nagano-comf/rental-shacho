# rental-shacho.jp

レンタル社長 LP。静的HTML 1枚。GitHub Pages で公開。

## ファイル

| ファイル | 役割 |
|---|---|
| `index.html` | LP本体。CSS・計測・フォーム送信のJSをすべて内包 |
| `CNAME` | GitHub Pages に独自ドメインを教えるファイル。中身は `rental-shacho.jp` の1行 |
| `Code.gs` | フォームの受け口（Google Apps Script）。**リポジトリには置かず**、スプレッドシート側の Apps Script に貼る |

## 公開までの順番

### 1. GAS を先に動かす
1. 新しいスプレッドシートを作る → 拡張機能 → Apps Script → `Code.gs` を貼る
2. プロジェクトの設定 → スクリプト プロパティ に `NOTIFY_TO`、`SLACK_WEBHOOK_URL` を登録。Apps Script をシートから開かず単体で作った場合は `SPREADSHEET_ID`（シートURLの `/d/` と `/edit` の間）も必須
3. エディタから `testNotify()` を実行 → シート・メール・Slack の3経路が通ることを確認
4. デプロイ → 新しいデプロイ → ウェブアプリ → 実行ユーザー「自分」／アクセス「全員」 → URL をメモ

### 2. GA4 を作る
1. GA4 プロパティを新規作成（ウェブ、URL は `https://rental-shacho.jp`）
2. データストリーム → 拡張計測 → **フォームの操作をオフ**（自前の `form_start` / `generate_lead` と二重になるため）
3. 測定ID `G-…` をメモ
4. 公開後、イベントが流れ始めてから：管理 → イベント → `generate_lead` を「キーイベントとしてマーク」／カスタム定義 → イベントスコープのカスタムディメンション `plan`、`section` を登録

### 3. index.html の差し替え箇所（2か所）
- `G-XXXXXXXXXX`（2か所、同じ値）→ GA4 の測定ID
- `https://script.google.com/macros/s/XXXXXXXXXXXXXXXX/exec` → GAS のウェブアプリURL

### 4. GitHub
1. リポジトリを作る（public でも private でも可。Pages は private でも Free プランで使えるようになっている）
2. `index.html`、`CNAME`、`README.md` を push（`main` ブランチ直下）
3. Settings → Pages → Source: Deploy from a branch → `main` / `/ (root)` → Save
4. 同じ画面の Custom domain に `rental-shacho.jp` が入っていることを確認（CNAME ファイルがあれば自動で入る）

### 5. DNS（ドメインを取ったレジストラ側）
apex ドメインなので A レコードで GitHub Pages の IP を4つ。

```
A     @    185.199.108.153
A     @    185.199.109.153
A     @    185.199.110.153
A     @    185.199.111.153
CNAME www  <GitHubユーザー名>.github.io
```

IPv6 も足すなら AAAA で `2606:50c0:8000::153` 〜 `2606:50c0:8003::153` の4つ。

DNS が通ると GitHub 側で「DNS check successful」になり、少し待つと証明書が発行される。そうしたら Settings → Pages → **Enforce HTTPS** にチェック。

### 6. 公開後の確認
- `https://rental-shacho.jp/` が開く（`http://` が `https://` に飛ぶ）
- GA4 のリアルタイムで `page_view` が出る
- スマホ幅でフォームまでスクロールして崩れていない
- フォームをテスト送信 → シート・メール・Slack に届く → 画面が「受け付けました」に変わる → GA4 リアルタイムに `generate_lead` が出る
- ハニーポットの確認：DevTools で `#f-website` に何か入れて送信 → 「受け付けました」は出るが、シートには増えない

## 更新のしかた
`index.html` を直して push するだけ。1〜2分で反映。
GAS を直したときは「デプロイを管理 → 編集 → 新バージョン」（新しいデプロイにしない。URLが変わる）。

## フォームが「送信に失敗しました」になるとき
- GAS が `{"ok":false,"error":"server_error"}` を返している＝シート追記で落ちている。Apps Script の「実行数」で `doPost` のエラー内容を見る
- 単体プロジェクトで作っていて `SPREADSHEET_ID` 未設定 → スクリプト プロパティに追加して新バージョンで再デプロイ
- `testNotify()` を一度も実行していない → 実行して権限を承認してから再デプロイ
- 生存確認：ブラウザで GAS の exec URL を開くと `{"ok":true,"service":"rental-shacho-form"}` が出る
