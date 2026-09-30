---
title: "Claude Message Batches APIで1000件の分類ジョブを半額で回す：TypeScript実装と結果突き合わせの落とし穴"
emoji: "📦"
type: "tech"
topics: ["claude", "anthropic", "typescript", "batchapi", "llm"]
published: true
---

## 夜中に回すだけのジョブを、リアルタイムAPIの料金で払っていないか

問い合わせチケットの分類、過去ログのタグ付け、ドキュメント要約。こうした「今すぐ結果が要らない」処理を `messages.create` のループで回していると、待ち時間もレート制限の管理も料金も全部自分で持つことになります。

AnthropicのMessage Batches APIは、リクエストをまとめて投げて非同期で結果を受け取る仕組みです。公式ドキュメントでは通常料金の50%引きで、多くのバッチは1時間以内に終わるが最大24時間かかりうる、と説明されています。上限件数やサイズは変わることがあるので、実装前に公式の Batch processing のページで確認してください。

この記事では、1000件のチケットを分類するバッチを TypeScript で作り、投入・ポーリング・結果回収まで動かします。最後に、実際に踏みやすい落とし穴を3つ挙げます。

## 前提

- Node.js 20 以上、pnpm
- `ANTHROPIC_API_KEY` を環境変数に設定済み（値をコードやログに書かない）

```bash
pnpm add @anthropic-ai/sdk
pnpm add -D tsx typescript @types/node
```

## Step 1: 入力を「1リクエスト＝1件」に変換する

バッチの各リクエストには `custom_id` が必須です。結果はこのIDで突き合わせます。チケットのDB主キーをそのまま使うのが安全です。

```ts
// submit.ts
import Anthropic from "@anthropic-ai/sdk";
import { writeFileSync } from "node:fs";

type Ticket = { id: string; body: string };

const client = new Anthropic();

const SYSTEM = `問い合わせを次のカテゴリのいずれか1つに分類し、カテゴリ名だけを返してください。
billing / bug / feature_request / other`;

export async function submit(tickets: Ticket[]) {
  const batch = await client.messages.batches.create({
    requests: tickets.map((t) => ({
      custom_id: t.id,
      params: {
        model: "claude-haiku-4-5-20251001",
        max_tokens: 16,
        system: SYSTEM,
        messages: [{ role: "user", content: t.body }],
      },
    })),
  });
  // プロセスが落ちても回収できるよう、batch.id は必ず永続化する
  writeFileSync(".batch-id", batch.id);
  return batch.id;
}
```

分類だけなので軽量な Haiku 4.5 を選び、`max_tokens` も小さく絞っています。バッチ割引と合わせると、この種のジョブでは費用がかなり下がります。

## Step 2: ポーリングで完了を待つ

`processing_status` が `ended` になれば終了です。`ended` は全件成功の意味ではない点に注意してください（後述）。

```ts
// wait.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

export async function waitForEnd(batchId: string, intervalMs = 60_000) {
  for (;;) {
    const b = await client.messages.batches.retrieve(batchId);
    console.log(b.processing_status, b.request_counts);
    if (b.processing_status === "ended") return b;
    await new Promise((r) => setTimeout(r, intervalMs));
  }
}
```

`request_counts` には `processing` / `succeeded` / `errored` / `canceled` / `expired` の件数が入ります。ログに出しておくと進み具合が分かります。1分間隔なら、長く待つジョブでもAPI呼び出しは軽く済みます。

## Step 3: 結果を回収して突き合わせる

結果はストリームで返ります。

```ts
// collect.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const CATEGORIES = new Set(["billing", "bug", "feature_request", "other"]);

export async function collect(batchId: string) {
  const ok = new Map<string, string>();
  const retry: string[] = [];

  for await (const item of await client.messages.batches.results(batchId)) {
    const r = item.result;
    if (r.type === "succeeded") {
      const block = r.message.content[0];
      const label = block?.type === "text" ? block.text.trim() : "";
      if (CATEGORIES.has(label)) ok.set(item.custom_id, label);
      else retry.push(item.custom_id); // 想定外の出力は再投入対象
    } else {
      retry.push(item.custom_id); // errored / canceled / expired
    }
  }
  return { ok, retry };
}
```

実行は `tsx` で3ファイルをつなぐだけです。

```ts
// run.ts
import { submit } from "./submit";
import { waitForEnd } from "./wait";
import { collect } from "./collect";

const tickets = [/* DBから取得した1000件 */];
const id = await submit(tickets);
await waitForEnd(id);
const { ok, retry } = await collect(id);
console.log(`成功 ${ok.size} / 再投入 ${retry.length}`);
```

## 落とし穴3つ

**1. 結果の順序は入力順と一致しない。** インデックスで突き合わせると、別のチケットに別のラベルが付きます。必ず `custom_id` で引いてください。DBへの書き込みも `custom_id` をキーにした upsert にしておくと、回収を2回走らせても壊れません。

**2. `ended` でも全件成功とは限らない。** 個別リクエストが `errored`、または24時間以内に処理されず `expired` になることがあります。上のコードのように、成功以外を `retry` に集めて次のバッチに入れ直す設計にします。再投入は `custom_id` の集合から作れば、ジョブ全体を最初からやり直さずに済みます。

**3. `batch.id` を保存しないと迷子になる。** 投入後にスクリプトが落ちると、結果は取れる状態なのにIDが分からず、料金だけ発生します。`submit` の直後に永続化し、`wait` / `collect` は ID だけから再開できる形にしておくのが確実です。

## 使い分けの目安

バッチが向くのは、待てる・件数が多い・失敗を再投入できる処理です。ユーザーを待たせるチャットや、1件ごとに次の入力が変わるエージェントループには向きません。私は分類やタグ付けのような「夜間に流して朝に見る」ジョブだけをバッチに寄せ、残りは通常のAPIに残しています。

## まとめ

- Message Batches API は `custom_id` 付きのリクエスト配列を投げ、`processing_status` が `ended` になったら結果を回収する
- 突き合わせは順序ではなく `custom_id`、`ended` は全件成功の意味ではない
- `batch.id` の永続化と、失敗分だけの再投入を最初から組み込んでおく

まずは100件程度で試し、`request_counts` の内訳と、想定外の出力がどれだけ出るかを確認してから本番件数に広げてください。