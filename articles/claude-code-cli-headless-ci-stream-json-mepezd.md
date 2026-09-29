---
title: "Claude Code CLIをheadlessモードでCIに組み込む：-pとstream-jsonでPRを自動レビューする"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "githubactions", "cli", "cicd"]
published: true
---

## 「Claudeさん、このPR見て」を毎回貼るのをやめたい

PRを開くたびにClaude Codeのチャットへdiffを貼り付けてレビューを頼む。悪くはないが、これは人間がボトルネックになっている。Claude Code CLIには`-p`(`--print`)を付けるだけでエージェントループを一回転させて結果を返し、そのまま終了する「headlessモード」がある。これを使えばPRが開かれた瞬間にCIが自動でレビューを走らせられる。

本記事は「なぜheadlessモードが便利か」ではなく、実際に手を動かして GitHub Actions に組み込むところまでを step by step で書く。

## 背景: エージェントを「対話」から「実行系」へ

2026年に入り、OpenAIのGPT-6 Astraが画面を直接見てOS操作を自律実行したり、GoogleがAX(Agent Executor)というオープンソースのエージェントオーケストレーションを出したりと、LLMエージェントを人間の対話相手ではなく「パイプラインの実行単位」として組み込む動きが加速している。Claude Codeも例外ではなく、CLIは元々ターミナルUIを前提にしているが、`-p`フラグ一発で非対話・単発実行のプロセスとして呼び出せる。

CIに組み込む上で重要なのは3つのフラグだけだ。

- `--output-format stream-json` — 出力を1行1JSONのイベントストリームにする
- `--allowedTools` — 実行を許可するツールをホワイトリストで絞る
- `--permission-mode` — 権限確認をどう扱うか決める

この3つを理解すれば、あとは普通のCIジョブと同じ感覚で書ける。

## Step 1: ローカルでheadlessモードの出力を見る

まずローカルで挙動を確認する。

```bash
npm install -g @anthropic-ai/claude-code

claude -p "このディレクトリのpackage.jsonの依存関係を3行で要約して" \
  --allowedTools "Read,Glob" \
  --permission-mode dontAsk \
  --output-format stream-json
```

`stream-json`を指定すると、`system`(init)→`assistant`(思考・ツール呼び出し)→`result`(最終結果)という順でJSONオブジェクトが1行ずつ標準出力に流れてくる。CIログにそのまま吐いても人間が追えるし、後段でパースもしやすい。

ここで`--allowedTools`に`Read,Glob`しか渡していない点に注目してほしい。CI上でコードを読ませてレビューコメントを作らせたいだけなら、`Edit`や`Bash`をエージェントに渡す理由はない。渡さなければ、そもそも書き換えのしようがない。

## Step 2: 権限モードの罠を踏まない

CIでよく見る書き方が`--permission-mode bypassPermissions`だ。「CI環境だから全部許可でいい」という発想は分かるが、これには実装上の落とし穴がある。`bypassPermissions`が有効なとき、`--allowedTools`で絞ったつもりのホワイトリストが効かないケースがAnthropic自身のリポジトリでも既知の問題として報告されている([anthropics/claude-code#12232](https://github.com/anthropics/claude-code/issues/12232))。つまり「読み取り専用のはずが書き込みツールまで動く」という状態になりうる。

CI上で信頼できない差分(外部コントリビューターのPRなど)を読ませる構成では、`bypassPermissions`ではなく`dontAsk`を使い、`--allowedTools`のホワイトリストと組み合わせるほうが安全側に倒せる。

```bash
claude -p "$(cat review-prompt.md)" \
  --allowedTools "Read,Grep,Glob" \
  --permission-mode dontAsk \
  --output-format stream-json
```

## Step 3: GitHub Actions ワークフローを書く

PRのdiffをプロンプトに埋め込み、レビュー結果をJSONLで受け取るジョブを組む。

```yaml
# .github/workflows/claude-review.yml
name: Claude PR Review
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Build diff + prompt
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > /tmp/diff.patch
          cat <<'EOF' > /tmp/prompt.md
          以下はPRの差分です。バグ・セキュリティ・パフォーマンスの観点で
          問題があれば箇条書きで指摘してください。問題がなければ「LGTM」とだけ書いてください。
          コードの書き換えは行わないでください。
          EOF
          cat /tmp/diff.patch >> /tmp/prompt.md

      - name: Run Claude Code review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "$(cat /tmp/prompt.md)" \
            --allowedTools "Read,Grep,Glob" \
            --permission-mode dontAsk \
            --output-format stream-json \
            > /tmp/review.jsonl

      - name: Post review as PR comment
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: node scripts/post-review.mjs
```

## Step 4: stream-jsonをパースする

CIでよくある事故は「exit codeが0だったから成功」という判定だ。headlessモードでも同じ罠があり、`result`イベントが1件も出ないまま正常終了することがある。パース側では「resultイベントの有無」を明示的にチェックする。

```js
// scripts/post-review.mjs
import { readFileSync } from "node:fs";
import { execFileSync } from "node:child_process";

const lines = readFileSync("/tmp/review.jsonl", "utf-8").trim().split("\n");
const events = lines.map((line) => JSON.parse(line));

const resultEvent = events.find((e) => e.type === "result");
if (!resultEvent || !resultEvent.result) {
  console.error("result イベントが見つからない。exit 0 でも中身が空のことがある");
  process.exit(1);
}

const body = `## Claude Code review\n\n${resultEvent.result}`;

execFileSync("gh", [
  "pr", "comment", process.env.PR_NUMBER,
  "--body", body,
], { stdio: "inherit" });
```

`execFileSync`を使い、引数を配列で渡している点も地味に重要だ。シェル経由の文字列結合にすると、レビュー本文にバッククォートやダブルクォートが混じった瞬間にコマンドインジェクションの入口になる。CIとはいえ、LLMが生成したテキストを次のコマンドに渡す以上、ここは信頼できない入力として扱う。

## Step 5: 動作確認

実際に小さな差分でPRを作り、Actionsのログで`system`→`assistant`→`result`の順にイベントが流れているか、`--allowedTools`で絞った以外のツールが呼ばれていないかを確認する。ここで`assistant`イベント中に`Edit`や`Bash`の呼び出しが混ざっていたら、プロンプトかフラグの設定を疑う。

## まとめ

Claude Code CLIのheadlessモードは、`-p` + `--output-format stream-json` + `--allowedTools` + `--permission-mode`の4点さえ押さえれば、対話ツールをCIの1ジョブへそのまま持ち込める。ポイントは「動く」ことより「絞ったつもりの権限が本当に絞れているか」と「exit codeでなく中身で成功を判定しているか」の2つで、どちらもCIに組み込んだ瞬間に効いてくる落とし穴だった。次はレビューだけでなく、`--allowedTools`にテスト実行系を足して失敗時だけ自動でIssueを起こす、という一段先の自動化に進める。