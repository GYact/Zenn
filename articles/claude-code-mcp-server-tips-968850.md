---
title: "Claude Code の MCP サーバー活用術：外部ツール連携で開発効率を最大化する"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "mcp", "ai", "開発効率化", "llm"]
published: true
---

## はじめに

Claude Code を使い始めて最初に気づくのは、「コードを書いてくれる AI」としての能力だ。しかし本当のポテンシャルは、**MCP（Model Context Protocol）サーバーを組み合わせた外部ツール連携**にある。

MCP とは Anthropic が策定したオープンな通信プロトコルで、AI モデルが外部ツール・データソース・サービスを呼び出すための標準インターフェースを定義する。Claude Code はこのプロトコルをネイティブサポートしており、設定ファイルに数行追加するだけで GitHub・Supabase・Notion・ブラウザ自動化など多様なサービスと直接連携できる。

本記事では、実務で Claude Code + MCP を使い込んだ知見をもとに、**設定方法・実践的なユースケース・運用上のハマりどころ**を具体的にまとめる。

---

## MCP の基本構造を理解する

MCP は **クライアント（Claude Code）↔ サーバー（外部ツール）** の 2 層構造で動く。

```
Claude Code
  └─ MCP Client
       ├─ mcp__github__*        → GitHub API
       ├─ mcp__supabase__*      → Supabase (DB / Edge Functions)
       ├─ mcp__playwright__*    → ブラウザ操作
       └─ mcp__notion__*        → Notion API
```

Claude Code は、会話の中でツールを呼び出す必要があると判断すると、MCP サーバーに JSON-RPC でリクエストを投げ、結果を受け取って次の推論に使う。重要なのは「AI が自律的にツールを選択・実行する」点で、人間がいちいちコマンドを打つ必要がない。

---

## セットアップ：`claude_desktop_config.json` または `settings.json`

MCP サーバーの登録は `~/.claude/settings.json` で行う（プロジェクト単位なら `.claude/settings.json`）。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    },
    "supabase": {
      "command": "npx",
      "args": [
        "-y",
        "@supabase/mcp-server-supabase@latest",
        "--supabase-url", "${SUPABASE_URL}",
        "--supabase-service-role-key", "${SUPABASE_SERVICE_ROLE_KEY}"
      ]
    }
  }
}
```

**ポイント：**
- 環境変数は `${VAR_NAME}` 形式で参照できる（シェル環境から引き継ぎ）
- サーバーは Claude Code 起動時に子プロセスとして立ち上がる
- `npx -y` で都度最新版を取得するか、バージョン固定かはチームの方針で決める

---

## 実践ユースケース

### 1. GitHub + コードベース横断レビュー

```
「PR #234 のレビューをして。関連するファイルと過去の Issue も確認したうえで」
```

Claude Code は以下を自律実行する：

1. `mcp__github__get_pull_request` で差分取得
2. `mcp__github__list_issues` で関連 Issue 検索
3. ローカルファイルを `Read` で読み込み
4. 総合的なレビューコメントを生成

手動で PR → Issue → コード を往復する作業が、一発のプロンプトで完結する。

### 2. Playwright によるブラウザ自動化デバッグ

フロントエンドのバグ修正で特に有効だ。

```
「http://localhost:3000/dashboard を開いて、ログイン後にエラーが出る手順を再現して」
```

```
# Claude Code が実行する流れ
1. mcp__playwright__browser_navigate("http://localhost:3000")
2. mcp__playwright__browser_snapshot()  ← DOM 構造を取得
3. mcp__playwright__browser_fill_form({ ... })  ← フォーム入力
4. mcp__playwright__browser_click({ ... })
5. mcp__playwright__browser_console_messages()  ← エラーログ確認
```

スクリーンショットと DOM スナップショットを見ながらデバッグするため、「再現できない」バグの調査時間が大幅に短縮できる。

### 3. Supabase MCP で DB スキーマを活かしたコード生成

```
「users テーブルと orders テーブルの構造を確認して、注文履歴を取得する
 Server Action を書いて」
```

`mcp__supabase__list_tables` → `mcp__supabase__execute_sql` でスキーマを自律取得するため、Claude Code は**実際の型定義に基づいたコードを生成**する。ハルシネーションで存在しないカラムを参照するミスが激減した。

### 4. Notion をドキュメントストアとして活用

```
「Notion の "API仕様" ページを参照して、この endpoint のレスポンス型を定義して」
```

Claude Code が Notion から最新仕様を取得し、その内容に沿った TypeScript 型を生成する。仕様書とコードの乖離問題を AI が自動的に埋めてくれる。

---

## 運用で学んだハマりどころ

### CORS・認証ヘッダーの見落とし

Edge Functions や社内 API に MCP 経由でアクセスする場合、**カスタムヘッダー（`apikey`, `x-client-info` 等）が CORS の `Access-Control-Allow-Headers` に含まれていない**と OPTIONS が通っても POST が 403 になる。

```typescript
// Edge Function の CORS 設定例（最低限必要なヘッダー）
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type",
};
```

### MCP サーバーの起動失敗はサイレントに失敗する

MCP サーバーが起動できなくても Claude Code は落ちない。代わりに**そのツールが使えない状態で動き続ける**。`claude --debug` で起動ログを確認する習慣をつけよう。

```bash
# デバッグモードで起動して MCP サーバーの状態確認
claude --debug 2>&1 | grep -i mcp
```

### ツール名の衝突と優先順位

複数の MCP サーバーで同名のツールが存在すると、後から登録したものが勝つ。`settings.json` での定義順と、会話中の明示的なツール指定（「github の方の search_code を使って」）を組み合わせて制御する。

---

## Context 節約の設計パターン

MCP ツールの呼び出し結果はそのままコンテキストウィンドウを消費する。大量データを返す API は特に注意が必要だ。

```
# NG: 一度に全件取得を指示
「全ての Issue を取得して分析して」→ コンテキスト爆発

# OK: 絞り込みを明示
「直近30日・open 状態の Issue のみ取得して、バグ報告に絞って分析して」
```

また、繰り返しアクセスするデータ（スキーマ定義、設計ドキュメント等）は `/init` コマンドや `CLAUDE.md` にあらかじめ記載しておくことで、MCP 呼び出し回数自体を減らせる。

---

## まとめ

MCP サーバーを組み合わせることで、Claude Code は「コード補完ツール」から**「コードベースとサービスを横断して自律的に動くエージェント」**に変わる。

| 用途 | MCP サーバー | 効果 |
|------|------------|------|
| コードレビュー | `@modelcontextprotocol/server-github` | PR・Issue・コードを横断参照 |
| フロントデバッグ | `@playwright/mcp` | ブラウザ操作・ログ取得の自動化 |
| DB 連携コード生成 | `@supabase/mcp-server-supabase` | スキーマベースの正確な生成 |
| 仕様駆動開発 | Notion MCP | ドキュメントとコードの同期 |

導入ハードルは低い。まず 1 つ MCP サーバーを追加して、「AI が自律的にツールを叩く」体験を積み上げていくことをお勧めする。そこから適用範囲を広げていくと、開発フローの想像以上の部分が自動化できることに気づくはずだ。