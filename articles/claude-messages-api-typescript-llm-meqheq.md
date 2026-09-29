---
title: "Claude Messages APIで自作エージェントに人間承認ゲートを実装する"
emoji: "📝"
type: "tech"
topics: ["claude", "typescript", "llm", "nodejs", "agent"]
published: true
---

先週、社内用に書いた小さなエージェントスクリプトに「不要なファイルを消して」と頼んだら、確認なしで `rm` が実行されて青ざめたことがある。Claude Code の CLI ならフックで危険なコマンドを止められるが、これは Messages API を直接叩いて組んだ自前のエージェントだった。ツール呼び出し(Tool Use)そのものには承認機構が無い。止めたければ自分でループの中に承認ゲートを作るしかない。

この記事は、Anthropic の Messages API でツール呼び出しループを組み、危険なツールだけ人間の承認を待ってから実行を再開する最小構成を、動くコードで最初から最後まで作る。Claude Code の PreToolUse フックとは別物で、Slack bot やバッチ処理、社内ツールなど「CLI を経由しない自作エージェント」を書く人向けの手順になる。

## 準備