---
title: "Claude APIのCitationsで回答に「検証できる出典」を付ける：文字位置の突き合わせまで実装する"
emoji: "📝"
type: "tech"
topics: ["claude", "anthropic", "typescript", "rag", "llm"]
published: true
---

「根拠も付けて答えて」とプロンプトに書くと、LLMは根拠らしい文を返します。ところがその引用が原文に存在するかは、別途確かめない限りわかりません。社内ドキュメントQAでこれをやると、存在しない一文が「出典」として表示されます。

Claude APIには **Citations** があります。`document` ブロックに `citations: { enabled: true }` を付けると、回答の各テキストブロックに「原文のどこを根拠にしたか」が構造化データで付いてきます。この記事では、その使い方に加えて、返ってきた引用が本当に原文と一致するかを自前で検証するところまでを TypeScript で実装します。

## 完成形

- 社内規程のテキストを `document` として渡す
- 回答を脚注付き Markdown（`[1]` `[2]`）に整形する
- 各引用の `cited_text` が原文の指定範囲と一致するかを機械的に検証する

## Step 1: セットアップ

```bash
mkdir citations-demo && cd citations-demo
pnpm init
pnpm add @anthropic-ai/sdk
pnpm add -D tsx typescript @types/node
```

キーは環境変数 `ANTHROPIC_API_KEY` から SDK が読みます。コードには書かず、`.env` は Git 管理から除外してください。

## Step 2: 引用つきでリクエストする

```typescript
// ask.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

export const policy = `第1条 経費精算は発生月の翌月5営業日までに申請する。
第2条 1件3万円以上の支出は事前に上長の承認を得る。
第3条 領収書は電子データでの保管を原則とし、原本は不要とする。`;

export async function ask(question: string) {
  return client.messages.create({
    model: "claude-sonnet-5-5",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: { type: "text", media_type: "text/plain", data: policy },
            title: "経費規程",
            citations: { enabled: true },
          },
          { type: "text", text: question },
        ],
      },
    ],
  });
}
```

ポイントは2つです。

- 引用させたい文書は `document` ブロックで渡す。プロンプト本文に貼り付けても引用は付かない
- `source.type: "text"` の場合、引用は文字位置（`char_location`）で返る。PDF なら `page_location` になる

## Step 3: レスポンス構造を読む

回答は1つの文字列ではなく、`content` の配列に分割されて返ります。引用が付くテキストブロックには `citations` 配列が付きます。

```json
{
  "type": "text",
  "text": "3万円以上の支出は事前に上長の承認が必要です。",
  "citations": [
    {
      "type": "char_location",
      "cited_text": "1件3万円以上の支出は事前に上長の承認を得る。",
      "document_index": 0,
      "document_title": "経費規程",
      "start_char_index": 28,
      "end_char_index": 51
    }
  ]
}
```

`end_char_index` は排他的（その位置の文字は含まない）です。上の数値は説明用なので、実際の値は実行結果で確認してください。

## Step 4: 脚注付きMarkdownに整形する

```typescript
// render.ts
import type Anthropic from "@anthropic-ai/sdk";

type Msg = Anthropic.Message;

export function renderWithFootnotes(msg: Msg) {
  const notes: string[] = [];
  let body = "";

  for (const block of msg.content) {
    if (block.type !== "text") continue;
    body += block.text;
    for (const c of block.citations ?? []) {
      if (c.type !== "char_location") continue;
      notes.push(`${c.document_title}: 「${c.cited_text}」`);
      body += `[${notes.length}]`;
    }
  }
  const footer = notes.map((n, i) => `[${i + 1}] ${n}`).join("\n");
  return footer ? `${body}\n\n${footer}` : body;
}
```

## Step 5: 引用が本物かを検証する

ここがこの記事の中心です。位置情報は API が返すものですが、受け取った側で「その範囲が本当に `cited_text` と一致するか」を確かめておくと、UI に出す前に壊れた引用を落とせます。

```typescript
// verify.ts
import type Anthropic from "@anthropic-ai/sdk";

export function verifyCitations(msg: Anthropic.Message, source: string) {
  const bad: string[] = [];
  let total = 0;

  for (const block of msg.content) {
    if (block.type !== "text") continue;
    for (const c of block.citations ?? []) {
      if (c.type !== "char_location") continue;
      total++;
      const actual = source.slice(c.start_char_index, c.end_char_index);
      if (actual !== c.cited_text) bad.push(c.cited_text);
    }
  }
  return { total, bad };
}
```

## Step 6: 動かす

```typescript
// main.ts
import { ask, policy } from "./ask";
import { renderWithFootnotes } from "./render";
import { verifyCitations } from "./verify";

const msg = await ask("領収書の原本は取っておく必要がありますか？");
const { total, bad } = verifyCitations(msg, policy);

console.log(renderWithFootnotes(msg));
console.error(`citations=${total} mismatched=${bad.length}`);
if (total === 0) console.error("引用が0件: 出典なしの回答として扱う");
```

```bash
pnpm tsx main.ts
```

## ハマりどころ

**引用が0件の回答は普通にある。** 文書に答えがない質問や、挨拶のような返答には引用が付きません。「出典付きのはず」と決め打ちせず、`total === 0` のときは「根拠なし」と UI に明示する分岐を用意してください。

**位置は元の `data` 文字列基準。** 検証時に改行コードの正規化などをかけた文字列と比べると、必ずずれます。API に渡した文字列そのものを保持して照合します。

**`cited_text` はトークン課金の対象外。** 引用として返る `cited_text` の分は出力トークンに数えられない、というのが Citations の仕様です（料金の最新は公式ドキュメントで確認してください）。自前で「引用箇所を原文から抜いて返して」と頼む方式より安くなる可能性があります。

**他の機能との併用は事前に試す。** Citations は出力の形式に制約を持ちます。構造化出力など他の機能と組み合わせる場合は、本番に入れる前に小さなリクエストで通ることを確認してください。

## まとめ

- Citations は `document` ブロック + `citations.enabled` で有効になり、引用は `content[].citations` に構造化されて返る
- 引用 0 件の回答は正常系として扱い、画面上でも区別する
- `source.slice(start, end) === cited_text` の照合を挟めば、UI に出す引用を機械的に検証できる

手元の規程やマニュアル1本を `policy` に差し替えるだけで、社内QAの出典表示まで試せます。まずは5行の短い文書で、引用の位置がどう返るかを `console.log` で見てみてください。