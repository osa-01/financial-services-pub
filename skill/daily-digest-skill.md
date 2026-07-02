# 毎朝の情勢・金融ダイジェスト — 実行手順書 (v2: GitHub API方式)

このドキュメントは、毎朝6:30 JSTに自動起動するスケジュールタスクが実行する手順です。
デバイス（PC）への接続は不要。GitHub APIで全ての読み書きを行います。

## 設定情報
スケジュールタスクのプロンプトから以下の値が渡される：
- GITHUB_PAT: GitHub Personal Access Token
- GITHUB_OWNER: osa-01
- GITHUB_CONFIG_REPO: financial-services（private）
- GITHUB_WEB_REPO: financial-services-pub（public）
- DASHBOARD_URL: https://osa-01.github.io/financial-services-pub/latest.html

---

## 0. 前提データの読み込み

GitHub APIで以下のファイルを取得する。レスポンスの `content` フィールドはbase64エンコードされているのでデコードして使用する。

**ウォッチリスト（プライベートリポジトリ）:**
```
GET https://api.github.com/repos/osa-01/financial-services/contents/config/watchlist.json
Authorization: Bearer {GITHUB_PAT}
```

**アーティファクト状態（プライベートリポジトリ）:**
```
GET https://api.github.com/repos/osa-01/financial-services/contents/config/artifact_state.json
Authorization: Bearer {GITHUB_PAT}
```
※ レスポンスの `sha` フィールドを必ず記録しておくこと（手順4の更新時に必要）。

今日の日付（JST）を確認する：
```bash
TZ=Asia/Tokyo date +%Y-%m-%d
```

---

## 1. 情報収集（WebSearchツールを使用）

**検索のコツ（動作検証で判明した注意点）**: 汎用的なキーワードだけで検索すると、
個別記事ではなくポータルサイトのトップページやテーマ解説ページばかりが返ってくる
ことがある。日付・「本日」「速報」等の語や、固有名詞（企業名・要人名・指標名）を
組み合わせるなど、より具体的なクエリを心がけること。また、日本経済新聞など
一部サイトはrobots.txtによりWebFetchでの本文取得がブロックされるため、無理に
個別ページを深掘りせず、WebSearchのスニペット（タイトル・要約文）を情報源として
活用してよい。

以下の4カテゴリについて、それぞれ2〜4件のWeb検索を行い、直近24時間以内を中心に
重要なニュースを収集する。検索結果のタイトル・URL・スニペットの日時を必ず記録すること。

- **世界情勢**: 地政学リスク、米国・中国・欧州など主要国の政治・外交・要人発言
  （例: "geopolitics news today", "米国 中国 外交 最新"）
- **日本情勢**: 政治動向、日銀金融政策、政府の経済対策、主要経済指標
  （例: "日銀 金融政策 最新", "日本 政治 経済対策 ニュース"）
- **株式市場**: 日経平均・TOPIX・主要日本株、S&P500・NASDAQ・主要米国株の値動きと材料
  （例: "日経平均 本日 終値 理由", "S&P500 米国株 本日"）
- **為替・金利・債券**: ドル円などの為替、各国政策金利、国債利回り
  （例: "ドル円 本日 レート", "米国債 利回り 最新"）

さらに、`watchlist.json` に登録されている銘柄・通貨ペアがあれば、それぞれについて
個別に検索し、「注目トピック」として別枠にまとめる。ウォッチリストが空の場合はこの手順を
スキップしてよい。

---

## 2. 数値データの確認（重要・注意事項あり）

**過去の検証で、WebFetchで金融情報ページを1件だけ取得すると、実際とは異なる古い日付の
数値を「最新値」として返す事例が確認されている。** そのため、価格・騰落率などの数値は
以下のルールに従うこと。

- 可能な限り2件以上の情報源（検索結果のスニペット、または複数ページのWebFetch）を
  突き合わせ、大きく矛盾しないか確認する。
- 表示する数値には必ず「◯月◯日時点」「速報値」など、いつの情報かを併記する。
- 出典が古い、または矛盾する場合は、無理に数値を確定させず「詳細は主要ニュースサイトで
  ご確認ください」という表現に留める。
- 分単位のリアルタイム性は保証しない前提とする（このアプリの目的はリアルタイム取引
  ではなく、投資判断の参考情報整理であるため）。

---

## 3. ダッシュボードHTMLの生成（確定デザイン: タブ切り替え形式）

自己完結的な1ファイルのHTML（インラインCSS/JS、外部ライブラリ不使用）として、
以下の構成でダッシュボードを作る。デザインの方向性はユーザー確認済み（2026-07-02）。

- ヘッダー: タイトル「情勢・市場デイリーダイジェスト」、生成日時（JST）
- **タブ切り替えナビゲーション**を横並びで配置し、以下5つのタブを持つ
  （ウォッチリストが空の日も「ウォッチリスト」タブ自体は残し、空メッセージを表示する）。
  1. 世界情勢
  2. 日本情勢
  3. 株式市場
  4. 為替・金利・債券
  5. ウォッチリスト（注目トピック）
- タブは一度に1パネルだけ表示するJS（`.tab-btn` クリックで対応する `.panel` に
  `active` クラスを付け替える、素の JavaScript。localStorage 等のブラウザ保存APIは
  使用しない）。
- 各パネルの中身:
  - 要点3〜5行の箇条書き＋出典リンク＋「◯月◯日時点」の注記
  - 株式市場／為替・金利・債券タブは、値を「数値タイル」（大きめフォントのカード）で
    ハイライト表示する。数値が確認できない場合は "--" とし、
    「本番実行時に複数ソース照合の上で記載」のプレースホルダー文言を表示してよい
    （無理に不確かな数値を埋めない）。
- ページ上部に「本日のダイジェストは公開情報を要約したものである」旨の簡潔な注記バナー、
  ページ下部に固定の免責事項:
  「本ダイジェストは公開情報の要約であり、投資助言ではありません。投資判断は
  ご自身の責任で行ってください。数値は複数ソースを参考にした概算であり、
  正確な最新値は各金融情報サービスでご確認ください。」
- 配色はダーク背景＋アクセントカラー1色を基調としたダッシュボード風とする。
  参考実装（ドライラン版）が公開リポジトリの `archive/_test_dryrun.html` に保存済みなので、
  HTML構造・CSSクラス名はこれを踏襲してよい。

---

## 4. GitHub APIでの保存

生成したHTMLを変数 `HTML_CONTENT` として、以下の手順でGitHubに保存する。
全APIリクエストに共通ヘッダー: `Authorization: Bearer {GITHUB_PAT}`, `Content-Type: application/json`

### 4-1. latest.html をパブリックリポジトリに保存/更新

```
# ① 既存ファイルのSHAを取得（ファイルが存在しない場合は404になるので sha=null として扱う）
GET https://api.github.com/repos/osa-01/financial-services-pub/contents/latest.html

# ② PUT で保存（HTMLはUTF-8でbase64エンコードする）
PUT https://api.github.com/repos/osa-01/financial-services-pub/contents/latest.html
Body:
{
  "message": "Update digest YYYY-MM-DD",
  "content": "<HTMLをUTF-8でbase64エンコードした文字列>",
  "sha": "<①で取得したsha。新規の場合はフィールドごと省略>"
}
```

### 4-2. archive/YYYY-MM-DD.html をパブリックリポジトリに保存

```
PUT https://api.github.com/repos/osa-01/financial-services-pub/contents/archive/YYYY-MM-DD.html
Body:
{
  "message": "Add archive YYYY-MM-DD",
  "content": "<base64エンコードしたHTML>"
}
```
アーカイブは毎回新規作成なのでSHAは不要。`archive/` ディレクトリはGitHubが自動で作成する。

### 4-3. artifact_state.json をプライベートリポジトリで更新

```
PUT https://api.github.com/repos/osa-01/financial-services/contents/config/artifact_state.json
Body:
{
  "message": "Update state YYYY-MM-DD",
  "content": "<base64エンコードしたJSON>",
  "sha": "<手順0で取得したsha>"
}
```

更新するJSONの内容:
```json
{
  "last_generated_at": "<ISO 8601形式 JST>",
  "last_generated_date_jst": "YYYY-MM-DD"
}
```

**実装上の注意**: HTMLコンテンツをbase64エンコードする際は、Pythonを推奨する：
```python
import base64
content_b64 = base64.b64encode(html_content.encode('utf-8')).decode('ascii')
```
シェルのechoやbase64コマンドは改行・特殊文字の扱いで文字化けが起きやすい。

---

## 5. 完了メッセージ（スケジュールタスクの通知メール本文）

セッションの最後のメッセージとして、以下を含む簡潔な日本語サマリーを出力する：

- 今日の一言まとめ（世界情勢・日本情勢・市場の要点を3〜5行）
- ウォッチリスト銘柄で特筆すべき動きがあれば1〜2行
- ダッシュボードURL: https://osa-01.github.io/financial-services-pub/latest.html

---

## 6. ウォッチリストの更新（通常のチャットセッションで随時）

このスケジュール実行とは別に、ユーザーが通常のチャットで「◯◯をウォッチリストに
追加して」等と伝えてきた場合は、以下の手順で `config/watchlist.json` を更新する：

1. GitHub APIで現在のwatchlist.jsonを取得し、SHAを記録する
2. 内容を編集（銘柄の追加・削除）
3. GitHub APIで更新（SHA指定）

```
GET https://api.github.com/repos/osa-01/financial-services/contents/config/watchlist.json
→ sha を記録

PUT https://api.github.com/repos/osa-01/financial-services/contents/config/watchlist.json
Body: {"message": "Update watchlist", "content": "<base64>", "sha": "<sha>"}
```
