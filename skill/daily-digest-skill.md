# 毎朝の情勢・金融ダイジェスト — 実行手順書 (v4: メモリ機能追加)

このドキュメントは、毎朝6:30 JSTに自動起動するスケジュールタスクが実行する手順です。
デバイスへの接続は不要。GitHub APIで全ての読み書きを行います。

## 設定情報
スケジュールタスクのプロンプトから以下の値が渡される：
- GITHUB_PAT: GitHub Personal Access Token
- GITHUB_OWNER: osa-01
- GITHUB_CONFIG_REPO: financial-services（private）
- GITHUB_WEB_REPO: financial-services-pub（public）
- DASHBOARD_URL: https://osa-01.github.io/financial-services-pub/latest.html
- RESEND_API_KEY: Resend APIキー
- MAIL_TO: 送信先メールアドレス

---

## 0. 前提データの読み込み

**重要: GitHub APIの呼び出しはすべてBash curlで行う（WebFetchは使わない）**

以下を取得する（レスポンスの `content` フィールドをbase64デコード）：

```bash
# watchlist.json
curl -s -H "Authorization: Bearer {GITHUB_PAT}" \
  https://api.github.com/repos/osa-01/financial-services/contents/config/watchlist.json

# artifact_state.json（SHAを記録）
curl -s -H "Authorization: Bearer {GITHUB_PAT}" \
  https://api.github.com/repos/osa-01/financial-services/contents/config/artifact_state.json

# agent_memory.json（SHAを記録）
curl -s -H "Authorization: Bearer {GITHUB_PAT}" \
  https://api.github.com/repos/osa-01/financial-services/contents/config/agent_memory.json
```

`artifact_state.json` と `agent_memory.json` の `sha` をそれぞれ記録しておく（手順4・5の更新時に必要）。

JSTの今日の日付と曜日を確認する：
```bash
TZ=Asia/Tokyo date "+%Y-%m-%d %A"
```

---

## 0.5. エージェントメモリの参照

`agent_memory.json` の内容を読み込み、今日の実行に活かす：

- **effective_queries**: 過去に有効だった検索クエリを優先的に使用する
- **blocked_sources**: ここに記載されたサイトはWebFetchでの本文取得をスキップ
- **recurring_themes**: 継続して追っているテーマは重点的に検索する
- **last_notes**: 前回の自己評価メモを参考に、今日の検索・分析方針を調整する

`history`（過去の実行サマリー）があれば、週次トレンドコメントの参考にする。

---

## 1. 情報収集（WebSearchツール + WebFetch）

**基本方針**:
- 各カテゴリで5〜8件のWeb検索を実施する。より具体的なクエリ（日付・固有名詞・指標名を含む）を優先する
- 重要度が高いと判断した記事は WebFetch で本文を取得し、詳細を確認する（ただし robots.txt でブロックされるサイトは無理に深掘りしない）
- 検索結果のタイトル・URL・スニペットの日時を必ず記録する

### 1-1. 世界情勢（5〜7クエリ）

以下のテーマを網羅するよう検索する：
- 米中関係・外交・貿易（例: "US China trade tariffs latest 2026", "米中 外交 最新"）
- 欧州の政治・経済（例: "Europe EU economy politics news today"）
- 中東・ロシア・地政学リスク（例: "Middle East conflict latest", "Russia Ukraine news today"）
- 米国政治・要人発言（例: "White House statement today", "Fed chairman speech latest"）
- 国際機関・多国間外交（例: "G7 G20 UN Security Council latest"）
- 新興国・アジア（例: "India Southeast Asia economy news today"）
- 貿易・制裁・サプライチェーン（例: "global trade sanctions supply chain news"）

### 1-2. 日本情勢（5〜7クエリ）

- 日銀・金融政策（例: "日銀 金融政策 利上げ 最新", "BOJ monetary policy latest"）
- 政治・内閣・政策（例: "日本 内閣 政策 最新ニュース"）
- 主要経済指標（例: "日本 CPI インフレ 最新", "日本 GDP 速報", "日本 貿易収支 最新"）
- 為替・経常収支・財政（例: "日本 財政 国債 最新"）
- 雇用・消費（例: "日本 失業率 消費 最新"）
- 産業・企業政策（例: "日本 産業政策 半導体 経済安保 最新"）

### 1-3. 株式市場 — 日本（5〜7クエリ）

- 日経平均・TOPIX（例: "日経平均 本日 終値 材料 解説", "TOPIX 本日"）
- 業種別動向（例: "東証 業種別 騰落 本日"）
- 主要銘柄（例: "トヨタ ソニー 任天堂 株価 本日"）
- 売買代金・市場全体（例: "東証プライム 売買代金 本日"）
- テーマ・物色動向（例: "日本株 テーマ 本日 材料"）
- 外国人投資家動向（例: "外国人 日本株 売買 最新"）

### 1-4. 株式市場 — 米国（5〜7クエリ）

- 主要指数（例: "S&P 500 Dow Jones NASDAQ close today", "US stock market today summary"）
- セクター動向（例: "US stock sector performance today tech energy financials"）
- 主要銘柄（例: "Apple Microsoft Nvidia Tesla stock today"）
- 市場センチメント（例: "VIX fear greed index today", "US market breadth today"）
- 米国経済指標・FRB（例: "US economic data today Fed policy"）
- 決算（例: "US earnings results today after hours"）

### 1-5. 為替・金利・債券（5〜7クエリ）

- ドル円・主要通貨ペア（例: "ドル円 本日 レート 要因", "USD/JPY EUR/USD GBP/USD today"）
- クロス円（例: "ユーロ円 ポンド円 豪ドル円 本日"）
- 米国債利回り（例: "US Treasury 10-year yield today", "US bond market today"）
- 日本国債・各国金利（例: "日本国債 10年 利回り 本日", "Germany bund yield today"）
- 各国政策金利（例: "central bank interest rate latest ECB BOE"）
- 為替介入・通貨政策（例: "FX intervention currency policy latest"）

### 1-6. 商品市況（4〜6クエリ）

- 原油（例: "WTI Brent crude oil price today", "原油 価格 本日 要因"）
- 金・貴金属（例: "gold silver price today", "金 価格 本日"）
- 産業用金属（例: "copper aluminum steel price today"）
- 農産物・エネルギー（例: "natural gas LNG price today", "wheat corn soybean price"）
- 商品市場全体（例: "commodity market outlook today CRB index"）

### 1-7. 仮想通貨（3〜5クエリ）

- BTC・ETH（例: "Bitcoin BTC price today", "Ethereum ETH price today"）
- 市場全体（例: "crypto market today total market cap", "altcoin news today"）
- 規制・ニュース（例: "crypto regulation news today", "crypto institutional news"）

### 1-8. 企業・決算ニュース（4〜6クエリ）

- 日本企業決算（例: "日本 企業決算 本日 サプライズ"）
- 米国企業決算（例: "US earnings today beat miss guidance"）
- M&A・再編（例: "M&A merger acquisition news today Japan"）
- 大型企業ニュース（例: "major corporate announcement today Japan US"）

### 1-9. ウォッチリスト（登録銘柄がある場合のみ）

`watchlist.json` に登録されている銘柄・通貨ペアそれぞれについて個別に検索する。

### 1-10. 経済指標カレンダー（2〜3クエリ）

- 本日・翌日の経済指標発表予定（例: "economic calendar today tomorrow major events"）
- 日本の経済指標発表（例: "日本 経済指標 発表予定 今週"）
- 中央銀行イベント（例: "central bank meeting schedule this week"）

---

## 2. 数値データの確認（重要・注意事項あり）

複数ソースを突き合わせ、各数値には「◯月◯日時点」または「速報値」を必ず明記する。
矛盾または古い値は "--" とし、無理に確定させない。

収集する数値（取得できた範囲で記載）：

**株価指数:**
- 日経平均、TOPIX、JPXプライム150
- S&P 500、NASDAQ総合、Dow Jones、Russell 2000

**為替（対円）:**
- USD/JPY、EUR/JPY、GBP/JPY、AUD/JPY、CNY/JPY

**為替（クロス）:**
- EUR/USD、GBP/USD

**金利・国債利回り:**
- 米国10年債、米国2年債（逆イールド確認）
- 日本10年国債、ドイツ10年Bund
- FF金利（現在の政策金利）、日銀政策金利

**商品:**
- WTI原油、Brent原油
- 金（XAU/USD）、銀（XAG/USD）
- 銅（LME）

**仮想通貨:**
- BTC/USD、ETH/USD

---

## 3. ダッシュボードHTMLの生成

自己完結的な1ファイルのHTML（インラインCSS/JS、外部ライブラリ不使用）として生成する。
デザインベースは既存のドライラン版（ダーク背景、タブ切り替え方式）を踏襲する。

### 3-1. 全体構成

- **ヘッダー**: タイトル「情勢・市場デイリーダイジェスト」、生成日時（JST）、曜日
- **サマリーバナー**（ページ上部、全タブ共通）: 本日の最重要ニュース1〜3点を太字で表示
- **タブナビゲーション**（横スクロール対応）: 以下の8タブ
  1. 世界情勢
  2. 日本情勢
  3. 株式（日本）
  4. 株式（米国）
  5. 為替・金利
  6. 商品・コモディティ
  7. 仮想通貨
  8. ウォッチリスト
- **経済指標カレンダー**（固定セクション、タブ外・全タブの下部に常時表示）
- **免責事項フッター**（固定）

### 3-2. 各タブの構成

**ニュースタブ（世界情勢・日本情勢・企業ニュースを含む全タブ）:**
- 🔴 重要 バッジ付きの最重要ニュース（1〜3件）
- 通常の箇条書き（7〜10行）
- 各項目に出典リンクと日時
- タブ末尾に「週次トレンドコメント」（直近1週間の大きな流れを1〜2行で要約）

**数値タイルを持つタブ（株式・為替・金利・商品・仮想通貨）:**
- 数値タイルカードを2行グリッドで配置
  - カード内: 名称、値、前日比（%）、取得日時
  - 上昇: 赤系（`--up`）、下落: 緑系（`--down`）、不明: グレー
- 数値の下にニュース箇条書き（5〜8行）
- 週次トレンドコメント

**ウォッチリストタブ:**
- 登録銘柄ごとに個別セクション（値動き + 関連ニュース）
- 空の場合は登録方法の案内を表示

### 3-3. 経済指標カレンダーウィジェット

タブ外・ページ下部（フッターの直前）に常時表示するセクション：

```
📅 経済指標カレンダー
[本日] 15:00 日本 消費者物価指数 (前月比) 予想: +0.2%
[本日] 23:00 米国 ISM製造業景況指数 予想: 48.5
[明日] 08:50 日本 貿易収支
[明日] 21:30 米国 雇用統計
```

取得できた情報の範囲で記載。不明な場合は省略してよい。

### 3-4. CSS・デザイン仕様

```css
:root {
  --bg: #0f1420;
  --card-bg: #1a2032;
  --card-border: #2a3348;
  --text-main: #e8ecf4;
  --text-sub: #9aa5b8;
  --accent: #4f8cff;
  --up: #ef5350;
  --down: #26a69a;
  --warn: #f4b942;
  --important: #ff6b6b;
}
```

- タブボタンは横スクロール可能（モバイル対応）
- 数値タイルは `display: grid; grid-template-columns: repeat(auto-fill, minmax(160px, 1fr))`
- 重要バッジ: `background: var(--important); color: white; padding: 2px 6px; border-radius: 4px; font-size: 11px`
- 週次トレンドは薄い背景ボックス（`background: rgba(79,140,255,0.08); border-left: 3px solid var(--accent)`）

---

## 4. GitHub APIでの保存

全APIリクエストに共通ヘッダー: `Authorization: Bearer {GITHUB_PAT}`, `Content-Type: application/json`

HTMLのbase64エンコードにはPythonを使用する（シェルのechoは文字化けが起きやすい）：
```python
import base64
content_b64 = base64.b64encode(html_content.encode('utf-8')).decode('ascii')
```

### 4-1. latest.html をパブリックリポジトリに保存/更新

```
# 既存のSHAを取得
GET https://api.github.com/repos/osa-01/financial-services-pub/contents/latest.html

# 更新
PUT https://api.github.com/repos/osa-01/financial-services-pub/contents/latest.html
Body: {"message": "Update digest YYYY-MM-DD", "content": "<base64>", "sha": "<sha>"}
```

### 4-2. archive/YYYY-MM-DD.html をパブリックリポジトリに保存

```
PUT https://api.github.com/repos/osa-01/financial-services-pub/contents/archive/YYYY-MM-DD.html
Body: {"message": "Add archive YYYY-MM-DD", "content": "<base64>"}
```

### 4-3. artifact_state.json をプライベートリポジトリで更新

```
PUT https://api.github.com/repos/osa-01/financial-services/contents/config/artifact_state.json
Body: {"message": "Update state YYYY-MM-DD", "content": "<base64>", "sha": "<手順0で取得したsha>"}
```

更新するJSONの内容:
```json
{
  "last_generated_at": "<ISO 8601形式 JST>",
  "last_generated_date_jst": "YYYY-MM-DD"
}
```

---

## 5. 完了メッセージ＆メール送信

以下を含む日本語サマリーを作成する：

- **本日の最重要ニュース**（世界・日本・市場から各1〜2点、計3〜5行）
- **主要数値サマリー**（日経平均・ドル円・米10年債利回り・BTC/USDを1行で）
- **ウォッチリスト特記事項**（登録銘柄がある場合のみ）
- **ダッシュボードURL**: https://osa-01.github.io/financial-services-pub/latest.html

サマリーを作成したら、Resend APIでメールを送信する：

```bash
SUBJECT="情勢・市場ダイジェスト $(TZ=Asia/Tokyo date +%Y-%m-%d)"
# BODYには上記サマリー全文を代入する

curl -s -X POST https://api.resend.com/emails \
  -H "Authorization: Bearer {RESEND_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"from\":\"onboarding@resend.dev\",\"to\":\"{MAIL_TO}\",\"subject\":\"${SUBJECT}\",\"text\":\"${BODY}\"}"
```

---

## 5.5. エージェントメモリの更新

セクション5完了後、今日の実行を振り返り `agent_memory.json` を更新する。

更新内容を評価・整理する：

1. **effective_queries**: 今日使ったクエリのうち、有益な結果が得られたものを追加（既存リストと合わせて最大20件に絞る）
2. **blocked_sources**: 今日アクセスできなかったサイトを追加
3. **recurring_themes**: 今日のニュースで継続的に重要と判断したテーマを更新
4. **last_notes**: 今日の実行の自己評価を100字以内で記録（検索精度・改善点・気づき）
5. **history**: 今日のサマリーを追加（最大30件、古いものから削除）

historyの1エントリ形式：
```json
{
  "date": "YYYY-MM-DD",
  "top_news": ["ニュース1", "ニュース2"],
  "key_figures": "日経XXXXX / ドル円XXX / 米10年X.XX% / BTC XXXXX"
}
```

更新したJSONをGitHub APIで保存する（手順0で取得したSHAを使用）：

```bash
# Python でbase64エンコード
python3 -c "
import base64, json
data = <更新後のagent_memory dictをここに>
content = json.dumps(data, ensure_ascii=False, indent=2)
print(base64.b64encode(content.encode('utf-8')).decode('ascii'))
"

curl -s -X PUT \
  -H "Authorization: Bearer {GITHUB_PAT}" \
  -H "Content-Type: application/json" \
  https://api.github.com/repos/osa-01/financial-services/contents/config/agent_memory.json \
  -d "{\"message\":\"Update memory YYYY-MM-DD\",\"content\":\"<base64>\",\"sha\":\"<手順0で取得したsha>\"}"
```

---

## 6. ウォッチリストの更新（通常のチャットセッションで随時）

ユーザーがチャットで「◯◯をウォッチリストに追加して」等と伝えた場合：

```
# 現在の内容とSHAを取得
GET https://api.github.com/repos/osa-01/financial-services/contents/config/watchlist.json

# 編集後に更新（SHA指定が必要）
PUT https://api.github.com/repos/osa-01/financial-services/contents/config/watchlist.json
Body: {"message": "Update watchlist", "content": "<base64>", "sha": "<sha>"}
```
