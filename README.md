# ラフテルズ HR ニュースレター — Claude Code 引き継ぎメモ

## プロジェクト概要

毎週月曜9時に「飲食・人事 ウィークリーニュースレター」を自動生成し、
Netlify 固定 URL へ配信するシステム。生成・配信は Cowork（Claude.ai）側の
スケジュールタスクが担当し、このリポジトリはニュースレター本体の HTML（アーティファクト）を
バージョン管理・引き継ぎするためのソース置き場。

---

## 重要ファイル

| ファイル | パス |
|----------|------|
| メイン HTML | `newsletter_v7.html`（リポジトリ直下） |
| スケジュールタスク SKILL.md | `C:\Users\laftels\Claude\Scheduled\daily-hr-newsletter\SKILL.md`（Cowork 側、このリポジトリ外） |

> **注意：** SKILL.md は Cowork の接続フォルダ内にあり、このリポジトリの Edit ツールでは編集できない。
> 変更は Cowork 側で `mcp__scheduled-tasks__update_scheduled_task` を使う（taskId: `daily-hr-newsletter`）。

---

## Netlify 設定

```
URL      : https://laftels-newsletter.netlify.app
Site ID  : 1730b445-a9c9-48f0-9cc0-1743d2cb82cb
Token    : （Personal Access Token。セキュリティ上このリポジトリには記載しない。
             Cowork 側の会話メモ・パスワードマネージャー等、安全な場所に保管し、
             このドキュメントには絶対にコミットしないこと）
```

> **セキュリティメモ：** 元の引き継ぎメモには Netlify の Personal Access Token が平文で
> 含まれていた。トークンは Netlify アカウント全体を操作できる強い権限を持つため、
> git 履歴に残さないこと。もし過去に共有された値に心当たりがある場合は、
> [Netlify のユーザー設定](https://app.netlify.com/user/applications)から失効・再発行することを推奨する。

### デプロイ方法（SHA1 ダイジェスト方式）

bash で curl は proxy ブロックされるため、Chrome MCP の `javascript_tool` で XHR を使う（Cowork 環境固有の制約）。

```javascript
// app.netlify.com タブ上で実行
const TOKEN = "<Netlifyトークン。安全な場所から取得してここに貼り付ける>";
const SITE_ID = "1730b445-a9c9-48f0-9cc0-1743d2cb82cb";
const SHA1 = "<SHA1ハッシュ>";  // python3 -c "import hashlib; ..." で計算
const r = await new Promise((resolve) => {
  const xhr = new XMLHttpRequest();
  xhr.open("POST", `https://api.netlify.com/api/v1/sites/${SITE_ID}/deploys`, true);
  xhr.setRequestHeader("Authorization", `Bearer ${TOKEN}`);
  xhr.setRequestHeader("Content-Type", "application/json");
  xhr.onload = () => resolve(JSON.parse(xhr.responseText));
  xhr.onerror = () => resolve({ error: "network error" });
  xhr.send(JSON.stringify({ files: { "/index.html": SHA1 }, async: false }));
});
// r.required === [] なら即 ready。値あればファイルをPUTアップロード。
```

> `window.fetch` は Netlify アプリが上書きしているため必ず XHR を使うこと。
> `application/zip` は CORS プリフライトで失敗するため JSON ダイジェスト方式のみ使う。

---

## スケジュールタスク（Cowork 側）

- **タスク ID**: `daily-hr-newsletter`
- **スケジュール**: 毎週月曜 9:00（cron: `0 9 * * 1`）
- **実行内容**:
  - STEP 1: 飲食業界ニュース10件（大手5 + 中小5）Exa検索
  - STEP 2: 人事労務トピック選定
  - STEP 3: アイスブレイク3件 + 労務提案ストーリー
  - STEP 4: ラフテルズ顧問先ニュース（直近3ヶ月のみ）
  - STEP 5: `newsletter_v7.html` の `WEEKLY_ISSUE_DATA` を差し替え → アーティファクト更新
  - STEP 6: Netlify へ自動デプロイ（XHR + SHA1）

## Exa MCP ツール名（Cowork 側）

```
mcp__cfba6524-a79c-4d25-9a05-0b69f8f9ac1e__web_search_exa
mcp__cfba6524-a79c-4d25-9a05-0b69f8f9ac1e__web_fetch_exa
```

---

## newsletter_v7.html の構造

- 844行 / 約51,498バイト
- 単一ファイル HTML（フレームワーク不使用、バニラJS）
- `const WEEKLY_ISSUE_DATA = null; // AUTO-UPDATED BY SCHEDULED TASK`
  → Cowork のスケジュールタスクがここを毎週書き換える
- Cowork のアーティファクト実行環境を前提としており、`window.cowork.askClaude` /
  `window.cowork.callMcpTool` に依存する（「今日の号を生成」「深掘り質問」機能）。
  そのため単体の静的ページとして開いた場合、これらの機能は動作しない。
- スタンドアロン時（Cowork 外）は「ニュース生成に失敗しました」表示 → 正常動作
- `clientNews` が空配列 `[]` の場合は HTML が自動で「📭 新規の情報なし」を表示
- 号データは `localStorage`（キー: `hr_nl_v7`）に保存され、サイドバーから過去号を検索・閲覧できる

### WEEKLY_ISSUE_DATA の型

```json
{
  "date": "YYYY-MM-DD",
  "newsItems": [
    {"tag":"M&A","headline":"...","body":"...","scale":"大手|中小","source":"...","url":"...","date":"YYYY-MM-DD"}
  ],
  "topic": {
    "emoji":"📋","title":"...","simpleExplain":"...","keyNumbers":[...],"visual":{...},
    "example":"...","points":["...","...","..."],"sourceUrl":"...","sourceName":"..."
  },
  "icebreakers": [
    {"newsRef":"...","opening":"...","point":"..."}
  ],
  "laborProposal": {
    "hook":"...","story":"...","closingQuestion":"..."
  },
  "clientNews": [
    {"company":"...","headline":"...","body":"...","source":"...","url":"...","date":"YYYY-MM-DD"}
  ],
  "errors": []
}
```

---

## 既知の注意点

| 問題 | 原因 | 対処 |
|------|------|------|
| bash curl が 403 | `localhost:3128` proxy が api.netlify.com をブロック | Chrome MCP の javascript_tool で XHR |
| `window.fetch` が TypeError | Netlify アプリが fetch をモンキーパッチ | XMLHttpRequest を直接使う |
| zip POST が CORS エラー | Content-Type: application/zip でプリフライト失敗 | JSON ダイジェスト方式に切り替え済み |
| SKILL.md が Edit 不可 | Cowork の接続フォルダ外のパス | update_scheduled_task MCP で更新 |
| JSON に backtick / シングルクォート | HTMLテンプレートリテラル内で構文エラー | ダブルクォートのみ使用 |

---

## Cowork アーティファクト ID

```
daily-hr-newsletter
```

更新は Cowork 側の `mcp__cowork__update_artifact` で行う（`html_path` 指定）。

---

## このリポジトリでの位置づけ

- このリポジトリは Cowork のチャットで作られたニュースレター生成アプリのソースを
  バックアップ・引き継ぎするためのもの。実際の週次生成・Netlifyデプロイの実行主体は
  引き続き Cowork 側のスケジュールタスク。
- `newsletter_v7.html` を更新した場合は、Cowork 側にも同内容を反映（アーティファクト更新）しないと
  実際の配信物には反映されない点に注意。
