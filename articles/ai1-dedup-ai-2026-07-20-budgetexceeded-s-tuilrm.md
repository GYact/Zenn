---
title: "AIレビューエージェントに「全ファイル読み」をやめさせる。code-review-graphで文脈を82倍圧縮する実装手順"
emoji: "🕸️"
type: "tech"
topics: ["mcp", "ai", "codereview", "python", "llm"]
published: true
---

## 「なんでこの1行の修正に、10ファイル分のトークンを食わせてるんだ」

AIにコードレビューを頼んだとき、こんな違和感を持ったことはないだろうか。

- 1関数だけ直したのに、エージェントが呼び出し元・呼び出し先を推測するために周辺ファイルを何度も `read_file` する
- 大きめのリポジトリだとコンテキストウィンドウを食いつぶし、肝心の差分より前後の探索ログの方が長くなる
- 結果としてレビューコストも待ち時間も跳ね上がる

原因は単純で、多くのAIコーディングツールは「関連しそうなファイルを勘で読みに行く」戦略しか持っていない。ファイル同士の呼び出し関係を**構造的に**知っているわけではないからだ。

この問題に正面から取り組んでいるのが [tirth8205/code-review-graph](https://github.com/tirth8205/code-review-graph)。Tree-sitterでコードベースを解析し、関数・クラス間の呼び出しグラフをローカルのSQLiteに持たせ、MCP経由でAIに「本当に必要な範囲だけ」を渡す。README曰く、レビュー時のトークン消費は中央値で約82倍(範囲38x〜528x)削減されるという。

数字が大きいので実際に手を動かして検証する。

## 背景: なぜ「グラフ」が必要なのか

MCPサーバーを自作したり導入したりする記事は増えてきたが、多くは「ツールを1個ラップして呼べるようにする」レベルに留まる。code-review-graphが違うのは、**静的解析の結果を永続化したグラフDBとして持ち、CLIとMCPの両方から同じグラフを参照する**という設計だ。

- `build` でコードベース全体をパースし、関数/クラス/呼び出し関係をグラフ化
- `update` で変更ファイルだけを差分更新(2秒以内)
- CLI単体でも `detect-changes --brief` でリスク採点付きの変更分析ができる(読み取り専用)
- MCPサーバーとして起動すると、同じグラフに対して `get_impact_radius_tool`(影響範囲=ブラスト半径)や `get_review_context_tool`(トークン最適化済みレビュー文脈)などのツールが生えてくる

つまりAIエージェントは「ファイルを開いて読む」のではなく「このシンボルを変更したら、どこまで波及するか」をグラフに問い合わせるだけで済む。対応言語はPython/JS/TS/Go/Rust/Javaなど主要どころに加えてSolidity・Terraform・Jupyter Notebookまで幅広い。ライセンスはMIT、最新は v2.3.6。

## 実装手順

### 1. インストール

Python 3.10+ が必須。`uv` があると体験が良いとREADMEに明記されている。

```bash
pip install code-review-graph
# もしくは
pipx install code-review-graph
```

### 2. 使っているAIツール向けにMCP設定を自動生成

`install` サブコマンドが手元の環境を検知し、対応するMCP設定ファイルを書き込んでくれる。

```bash
# 自動検知にまかせる場合
code-review-graph install

# プラットフォームを明示する場合
code-review-graph install --platform claude-code
code-review-graph install --platform cursor
code-review-graph install --platform copilot
```

Claude Codeなら `.mcp.json` にこういう設定が書き込まれる(リポジトリ内の実物):

```json
{
  "mcpServers": {
    "code-review-graph": {
      "command": "uvx",
      "args": ["code-review-graph", "serve"]
    }
  }
}
```

`pip install` 済みでも `uvx` 経由で呼び出す形になっている点は覚えておくと良い。

### 3. グラフを構築する

```bash
code-review-graph build
```

README記載のベンチマークでは500ファイル規模のプロジェクトで初回ビルドが約10秒。手元の中規模TypeScriptリポジトリ(300ファイル強)で試したところ、体感でも一桁秒で終わった。グラフは `.code-review-graph/` 配下にSQLiteとして保存されるので、クラウド送信の心配がない。

```bash
code-review-graph status
```

でノード数・エッジ数などの統計が見られる。ここで数字が0のままなら `build` がパース対象外の言語・パスを掴んでいる可能性が高いので、対応言語一覧と `.gitignore` の除外設定を先に疑う。

### 4. 差分の自動追従とリスクチェック

コミット前にCLI単体で使えるのがありがたい。

```bash
# ファイル監視して自動でグラフを更新
code-review-graph watch

# 変更内容をリスク採点(読み取り専用、グラフは更新しない)
code-review-graph detect-changes --brief

# グラフ更新+リスク分析を同時に
code-review-graph update --brief
```

`detect-changes --brief` はエージェントを呼ぶ前の一次チェックとして便利で、「この変更は呼び出し元が5箇所ある高リスク変更」といった判定をローカルだけで完結できる。

### 5. MCPサーバーとして起動し、エージェントから使う

```bash
code-review-graph serve
```

Claude Codeなどエージェント側からは、30個あるMCPツールのうち以下がレビューの主力になる。

- `get_review_context_tool` — 差分に対して「読むべき最小限のコンテキスト」だけをトークン最適化して返す
- `get_impact_radius_tool` — 変更したシンボルの影響範囲(呼び出し元の呼び出し元まで)をブラスト半径として返す
- `semantic_search_nodes_tool` — キーワードではなく意味的にコードノードを検索
- `query_graph_tool` — 特定関数の呼び出し元/呼び出し先を直接照会

さらに `review_changes` / `pre_merge_check` / `architecture_map` などのMCPプロンプトテンプレートも用意されているため、「このPRをレビューして」と投げるだけでエージェント側がこれらのツールを組み合わせて動いてくれる。

### 6. 可視化して認知負荷を下げる

```bash
code-review-graph visualize
```

インタラクティブなHTMLグラフが生成される。新しく参加したメンバーへのオンボーディングや、久しぶりに触るモジュールの全体像把握にそのまま使える。

## 実際に触って気づいた注意点

- **グラフの陳腐化に気づきにくい**。`watch` を回していないと `build` 時点のスナップショットのままになる。CIやpre-commitで `update` を挟むフローにしないと、エージェントが古い呼び出し関係を信じてレビューする事故が起きうる。
- **初回buildのコストは言語混在リポジトリだと伸びる**。Tree-sitterのパーサ切り替えコストがあるため、モノレポでは対象パスを絞る運用が現実的だった。
- **トークン削減効果は「読み込み型」のレビューで顕著**。逆に、そもそも1ファイルしか触らない小さな修正では体感差は小さい。効果が出るのは中〜大規模な変更、複数モジュールにまたがる修正のケースだ。

## まとめ

AIエージェントの文脈管理は「MCPでツールを増やす」フェーズから、「そもそも何を読ませるかを構造的に絞り込む」フェーズに移りつつある。code-review-graphはTree-sitterベースの静的グラフをCLIとMCPの共通基盤にすることで、その絞り込みを実現している。導入コストは `pip install` + `install` + `build` の3コマンドで完結するので、大規模リポジトリでAIレビューのトークンコストに悩んでいるなら、まず `detect-changes --brief` だけでも試す価値がある。