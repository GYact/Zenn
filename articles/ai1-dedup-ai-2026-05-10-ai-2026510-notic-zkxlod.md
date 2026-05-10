---
title: "Claude API の tool_use 実装で必ずハマる5つの罠"
emoji: "🪤"
type: "tech"
topics: ["claude", "anthropic", "llm", "typescript", "ai"]
published: true
---

## はじめに

Anthropic が公開した金融業務向けエージェントテンプレート群（KYC・ピッチブック生成・月次決算処理など）が注目を集めている。これらを参考に社内のドキュメント処理エージェントを実装しようとすると、`tool_use` (function calling) の実装ミスで詰まるエンジニアが後を絶たない。

本稿では筆者が実際に踏んだ5つの典型的な落とし穴と、正しい実装パターンを紹介する。GPT の function calling と「だいたい同じだろう」という思い込みがそのまま事故になるケースが多い。

---

## 罠1：`stop_reason` のチェック漏れ

最も多い失敗が、`stop_reason` を確認せずにレスポンスからテキストを直接取り出そうとするパターンだ。

```typescript
// ❌ アンチパターン
const response = await client.messages.create({ ... });
const text = response.content[0].text; // tool_use ブロックが先頭に来ると undefined
```

Claude はツールを呼び出すとき `stop_reason: "tool_use"` を返し、`content` 配列に `{ type: "tool_use", ... }` ブロックを含める。`content[0]` が `tool_use` ブロックの場合、`.text` プロパティは存在しないためランタイムエラーになる。

```typescript
// ✅ 正しい実装
const response = await client.messages.create({ ... });

if (response.stop_reason === "tool_use") {
  const toolUseBlocks = response.content.filter(b => b.type === "tool_use");
  // ツール呼び出し処理へ
} else {
  const textBlock = response.content.find(b => b.type === "text");
  const text = textBlock?.text ?? "";
}
```

---

## 罠2：並列 tool_use への非対応

Claude は1回のレスポンスで複数のツールを同時に呼び出すことがある。単一呼び出しを前提にしたコードは必ず壊れる。

```typescript
// ❌ アンチパターン：単一ツールを想定
const toolCall = response.content.find(b => b.type === "tool_use");
const result = await executeTool(toolCall!.name, toolCall!.input);
```

たとえば「A社とB社の財務データを同時に取得して比較して」という指示では、2つの `fetch_financial_data` ツール呼び出しが1レスポンス内に並列で返ってくる。

```typescript
// ✅ 正しい実装：並列実行を前提に
const toolUseBlocks = response.content.filter(b => b.type === "tool_use");
const results = await Promise.all(
  toolUseBlocks.map(async (block) => ({
    type: "tool_result" as const,
    tool_use_id: block.id,
    content: await executeTool(block.name, block.input),
  }))
);
```

---

## 罠3：`input_schema` の `required` フィールド省略

`required` を指定しないと、Claude がフィールドをオプション扱いして引数を省略してくることがある。

```typescript
// ❌ アンチパターン：required 省略
input_schema: {
  type: "object",
  properties: {
    company_id: { type: "string" },
    fiscal_year: { type: "number" },
  }
  // required がない → 全フィールドがオプション扱い
}
// 結果: block.input.fiscal_year が undefined になることがある
```

JSON Schema の仕様上、`required` を省略すると全プロパティがオプションになる。Claude は文脈から推測して補完しようとするが、省略してくることもある。

```typescript
// ✅ 正しい実装：必須フィールドを明示
input_schema: {
  type: "object",
  properties: {
    company_id: { type: "string", description: "企業ID（例: 'AAPL'）" },
    fiscal_year: { type: "number", description: "会計年度（例: 2025）" },
  },
  required: ["company_id", "fiscal_year"],
}
```

`description` を丁寧に書くほど、モデルが意図した引数を渡してくれる確率が上がる。

---

## 罠4：`messages` 配列の構造ミス

最も詰まりやすい落とし穴が、ツール結果を返す際の `messages` 配列の構造だ。

```typescript
// ❌ アンチパターン：tool_result を独立した user メッセージとして追加するだけ
messages.push({ role: "user", content: toolResults });
// → 直前の assistant ターン（tool_use ブロックを含む）が履歴に入っていない
```

正しくは、**アシスタントのレスポンス全体（テキスト + tool_use ブロック）を先に `messages` に追加してから**、user ロールで `tool_result` を返す。

```typescript
// ✅ 正しい実装
const messages: MessageParam[] = [
  { role: "user", content: userPrompt },
];

let response = await client.messages.create({ model, tools, messages });

while (response.stop_reason === "tool_use") {
  // 1. アシスタントの返答全体を履歴に追加（tool_use ブロックも含む）
  messages.push({ role: "assistant", content: response.content });

  // 2. ツールを実行して結果を user メッセージとして返す
  const toolResults = await Promise.all(
    response.content
      .filter((b): b is ToolUseBlock => b.type === "tool_use")
      .map(async (b) => ({
        type: "tool_result" as const,
        tool_use_id: b.id,
        content: JSON.stringify(await executeTool(b.name, b.input)),
      }))
  );
  messages.push({ role: "user", content: toolResults });

  response = await client.messages.create({ model, tools, messages });
}
```

`response.content` を**丸ごと** `messages` に追加するのがポイント。テキストブロックと tool_use ブロックが混在していてもそのまま入れてよい。これを省くと API が `400 Bad Request` を返す。

---

## 罠5：エラー時の `tool_result` の扱い

ツール実行が失敗したとき、多くのエンジニアはエラーメッセージを `content` に文字列で詰めるだけにする。しかし `is_error: true` フラグを立てないと、モデルは成功した結果として扱い、おかしな推論を続ける。

```typescript
// ❌ アンチパターン
{
  type: "tool_result",
  tool_use_id: block.id,
  content: `Error: ${e.message}`, // モデルはこれを正常な返答として扱うことがある
}
```

```typescript
// ✅ 正しい実装：is_error フラグを付ける
{
  type: "tool_result",
  tool_use_id: block.id,
  is_error: true,
  content: `Error: ${e instanceof Error ? e.message : String(e)}`,
}
```

`is_error: true` を付けると Claude はツール実行の失敗を認識し、「別のアプローチを試みる」「ユーザーにエラーを伝える」といった適切な判断をする。フラグなしだと「API が `Error: 404` というデータを返した」と解釈して、そのエラー文字列を元に次の推論を組み立ててしまう。

---

## まとめ：実装前チェックリスト

| チェック項目 | 確認内容 |
|---|---|
| `stop_reason` 分岐 | `"tool_use"` と `"end_turn"` を適切に処理しているか |
| 並列呼び出し | `tool_use` ブロックが複数返ってきても処理できるか |
| `input_schema` | `required` フィールドを明示しているか |
| `messages` 構造 | `response.content` を丸ごと履歴に積んでいるか |
| `is_error` | ツール失敗時に `is_error: true` を付けているか |

これらはいずれも Anthropic の公式ドキュメントに記載されているが、実際に動くサンプルを読むまで気づきにくい。特に罠4の `messages` 構造は、GPT の function calling と設計が微妙に違うため、経験者ほどハマりやすい。

`anthropics/financial-services` リポジトリのテンプレートはこれらを正しく実装しているため、自社実装の参考にするとよいだろう。構造を読み解くだけでも、tool_use の正しいループパターンが身につく。