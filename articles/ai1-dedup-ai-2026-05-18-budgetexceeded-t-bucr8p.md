---
title: "Anthropic Batches APIで大量LLMリクエストのコストを50%削減する実装手順"
emoji: "💸"
type: "tech"
topics: ["anthropic", "claude", "typescript", "llm", "ai"]
published: true
---

## その `for await` ループ、本当に同期で回す必要があるか？

5万件のユーザーレビューをカテゴリ分類するバッチジョブを組んだとき、Claude Sonnet に素直に `for` ループを回したら、完了前から「月換算したらいくらになるんだ」と計算して手が止まった。リアルタイム応答が不要なのに同期 API を使い続けるのは、純粋にコストの無駄だ。

Anthropic の **Message Batches API** を使えば、非同期処理に切り替えるだけでトークン単価が **50% オフ** になる。実装の変更量は思っているより少ない。この記事では TypeScript でゼロから動かすまでの手順を順番に書く。

## Batches API の概要

| 項目 | 内容 |
|------|------|
| 割引率 | 通常価格の 50% オフ |
| 最大リクエスト数 | 1バッチあたり 10,000 件 |
| 結果取得まで | 通常 1 時間以内（最大 24 時間） |
| 対応モデル | Claude 全モデル |

Prompt Caching（最大 90% オフ）と組み合わせれば、理論上は **95% 以上のコスト削減** も狙える。

## Step 1: SDK のインストール

```bash
pnpm add @anthropic-ai/sdk
```

`.env` に API キーを設定する。

```env
ANTHROPIC_API_KEY=sk-ant-...
```

## Step 2: バッチリクエストの作成

`requests` 配列の各要素に `custom_id` と、通常の Messages API と同じ `params` を渡すだけだ。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// 分類したいテキストの一覧（実際はDBやCSVから取得）
const reviews = [
  { id: "review-001", text: "配送が遅かったが、商品自体は満足" },
  { id: "review-002", text: "サポートの対応が非常に丁寧だった" },
  { id: "review-003", text: "品質が低く、すぐに壊れた" },
  // ...最大 10,000 件まで
];

const SYSTEM_PROMPT = `ユーザーレビューを以下のカテゴリに分類してください。
カテゴリ: delivery / product_quality / support / price / other
JSONのみ返却: {"category": "...", "sentiment": "positive|neutral|negative"}`;

const batch = await client.messages.batches.create({
  requests: reviews.map((review) => ({
    custom_id: review.id,
    params: {
      model: "claude-sonnet-4-6",
      max_tokens: 100,
      system: SYSTEM_PROMPT,
      messages: [{ role: "user", content: review.text }],
    },
  })),
});

console.log(`Batch created: ${batch.id}`);
console.log(`Status: ${batch.processing_status}`);
// → Batch created: msgbatch_01XFDUDYJgAACTU8zNo9sU7C
// → Status: in_progress
```

## Step 3: 完了まで待つポーリング

`processing_status` が `"ended"` になるまで定期的に確認する。

```typescript
async function waitForBatch(batchId: string): Promise<Anthropic.MessageBatch> {
  const POLL_INTERVAL_MS = 30_000; // 30秒間隔

  while (true) {
    const batch = await client.messages.batches.retrieve(batchId);

    console.log(
      `[${new Date().toISOString()}] status: ${batch.processing_status} ` +
        `(succeeded: ${batch.request_counts.succeeded}, ` +
        `errored: ${batch.request_counts.errored}, ` +
        `processing: ${batch.request_counts.processing})`
    );

    if (batch.processing_status === "ended") {
      return batch;
    }

    await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS));
  }
}

const completedBatch = await waitForBatch(batch.id);
```

実際の運用では、ポーリングループを Vercel Cron や Cloud Scheduler で回す設計のほうがコンピュートの無駄がない。

## Step 4: 結果の取得と処理

`.results()` は AsyncIterable を返す。JSONL 形式を逐次処理するので、10,000 件でもメモリ効率がいい。

```typescript
type ClassificationResult = {
  category: string;
  sentiment: "positive" | "neutral" | "negative";
};

const results: Array<{
  id: string;
  data: ClassificationResult | null;
  error?: string;
}> = [];

for await (const entry of await client.messages.batches.results(completedBatch.id)) {
  if (entry.result.type === "succeeded") {
    const raw = entry.result.message.content[0];

    if (raw.type !== "text") {
      results.push({ id: entry.custom_id, data: null, error: "unexpected content type" });
      continue;
    }

    try {
      const parsed: ClassificationResult = JSON.parse(raw.text);
      results.push({ id: entry.custom_id, data: parsed });
    } catch {
      results.push({ id: entry.custom_id, data: null, error: `parse failed: ${raw.text}` });
    }
  } else if (entry.result.type === "errored") {
    results.push({
      id: entry.custom_id,
      data: null,
      error: entry.result.error.type,
    });
  } else {
    // "expired": バッチが 24 時間以内に処理されなかった
    results.push({ id: entry.custom_id, data: null, error: "expired" });
  }
}

const succeeded = results.filter((r) => r.data !== null);
const failed = results.filter((r) => r.data === null);
console.log(`Done: ${succeeded.length} succeeded, ${failed.length} failed`);
```

## Step 5: Prompt Caching と組み合わせてさらに削減

System Prompt が長い場合、`cache_control` を付けると Batches 50% オフに加えて、キャッシュヒット時は **Input トークンが 90% オフ** になる。

```typescript
const batch = await client.messages.batches.create({
  requests: reviews.map((review) => ({
    custom_id: review.id,
    params: {
      model: "claude-sonnet-4-6",
      max_tokens: 100,
      system: [
        {
          type: "text",
          text: SYSTEM_PROMPT,
          cache_control: { type: "ephemeral" }, // ← キャッシュを明示
        },
      ],
      messages: [{ role: "user", content: review.text }],
    },
  })),
});
```

System Prompt が 1,024 トークン以上あれば、バッチ全体で 1 回のキャッシュライトが発生し、以降は全リクエストがキャッシュヒットする。

## 全体をまとめたスクリプト

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function runClassificationBatch(texts: { id: string; text: string }[]) {
  // 1. バッチ作成
  const batch = await client.messages.batches.create({
    requests: texts.map(({ id, text }) => ({
      custom_id: id,
      params: {
        model: "claude-haiku-4-5-20251001", // コスト最重視なら Haiku
        max_tokens: 100,
        messages: [{ role: "user", content: `Classify: ${text}\nJSON only: {"category":"..."}` }],
      },
    })),
  });

  console.log(`Batch ${batch.id} submitted (${texts.length} requests)`);

  // 2. 完了待ち
  let current = batch;
  while (current.processing_status !== "ended") {
    await new Promise((r) => setTimeout(r, 30_000));
    current = await client.messages.batches.retrieve(batch.id);
  }

  // 3. 結果取得
  const output: Record<string, string> = {};
  for await (const entry of await client.messages.batches.results(batch.id)) {
    if (entry.result.type === "succeeded") {
      const text = entry.result.message.content[0];
      if (text.type === "text") output[entry.custom_id] = text.text;
    }
  }
  return output;
}
```

## どんな処理に向いているか

- ドキュメントの一括分類・要約・翻訳
- データエンリッチメント（住所の正規化、タグ付けなど）
- 夜間バッチの評価・レポート生成
- LLM-as-Judge によるオフライン評価パイプライン（CIに組み込む場合も）

逆に、チャットやリアルタイム API レスポンスが必要なユースケースには使えない。「即時性が不要か？」を判断基準にするとシンプルだ。

## まとめ

Message Batches API の採用に必要なコード変更は「`create` に配列を渡す」「ポーリングで完了待ち」「`results` を AsyncIterable で処理する」の 3 ステップだけだ。Prompt Caching と組み合わせれば、大量処理ワークロードのコストを大幅に圧縮できる。

非同期でよいバッチ処理を同期 API で回し続けているなら、今すぐ切り替える価値がある。
```

---

以上が完成した Zenn 記事です。

**記事の概要:**
- **テーマ**: Anthropic Message Batches API（デデュプリストに未登場）
- **切り口**: 実装手順 step-by-step（コード中心、手を動かしたい人向け）
- **構成**: コスト爆発の hook → 仕組み解説 → Step 1〜5 の実装 → ユースケース整理
- **コード**: バッチ作成・ポーリング・結果取得・Prompt Caching 組み合わせを TypeScript で実装