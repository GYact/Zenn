---
title: "Claude APIのプロンプトキャッシュを「効いているか」まで測って導入する手順"
emoji: "🧊"
type: "tech"
topics: ["claude", "anthropic", "typescript", "llm", "promptcaching"]
published: true
---

`cache_control` を1行足したのに、請求が思ったほど下がらない。その原因の多くは、キャッシュが効いていないのに気づけていないことです。

この記事では、Claude API のプロンプトキャッシュを TypeScript で導入します。手順は、最小構成で動かす、`usage` でヒット率を測る、外れる原因を潰す、の順です。

なお、コードは公式 SDK の仕様に沿って書いていますが、この記事のために実行した出力は載せていません。数値は手元で測ってください。倍率や最小トークン数、TTL は改定されうるので、実装前に公式ドキュメントの Prompt caching のページを確認してください。

## 前提: キャッシュされるのは「先頭からの一致部分」

キャッシュは、リクエストの先頭から `cache_control` を置いたブロックまでの prefix が完全一致したときに読まれます。順序は `tools` → `system` → `messages` です。したがって、次の2点が設計の軸になります。

- 変わらないもの(ツール定義、長い指示、参照ドキュメント)を前に置く
- 変わるもの(日時、ユーザー入力)を後ろに置く

## Step 1: 最小構成で書く

```bash
pnpm add @anthropic-ai/sdk
```

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { readFileSync } from "node:fs";

const client = new Anthropic(); // ANTHROPIC_API_KEY は環境変数から読まれる

// 最小トークン数を超える必要があるので、長い固定文書を読み込む
const guideline = readFileSync("./guideline.md", "utf8");

export async function ask(question: string) {
  const res = await client.messages.create({
    model: "claude-sonnet-5-5",
    max_tokens: 512,
    system: [
      { type: "text", text: "あなたはレビュー担当のエンジニアです。" },
      {
        type: "text",
        text: guideline,
        cache_control: { type: "ephemeral" }, // ここまでが prefix としてキャッシュ対象
      },
    ],
    messages: [{ role: "user", content: question }],
  });
  return res;
}
```

`cache_control` は `system` の最後の固定ブロックに置きます。質問文はその後ろの `messages` にあるので、質問が変わっても prefix は変わりません。

## Step 2: usage で「効いているか」を測る

レスポンスの `usage` に、キャッシュの状況が出ます。

- `cache_creation_input_tokens`: 今回キャッシュへ書き込んだトークン数
- `cache_read_input_tokens`: キャッシュから読んだトークン数
- `input_tokens`: キャッシュに乗らなかった通常の入力トークン数

```typescript
import { ask } from "./ask";

function summarize(u: {
  input_tokens: number;
  cache_creation_input_tokens?: number | null;
  cache_read_input_tokens?: number | null;
}) {
  const write = u.cache_creation_input_tokens ?? 0;
  const read = u.cache_read_input_tokens ?? 0;
  const total = u.input_tokens + write + read;
  return { write, read, uncached: u.input_tokens, hitRate: total ? read / total : 0 };
}

for (const q of ["命名規則は?", "エラー処理の方針は?", "テストの粒度は?"]) {
  const res = await ask(q);
  console.log(q, summarize(res.usage));
}
```

期待する挙動は次のとおりです。

- 1回目は `write` が大きく、`read` は 0
- 2回目以降は `read` が大きく、`write` は 0

3回とも `write` だけが出て `read` が 0 なら、キャッシュは効いていません。次の節へ進みます。

## Step 3: 外れる原因を潰す

**原因1: 固定部分が最小トークン数に届いていない。** 届かない場合、エラーにはならず、黙ってキャッシュされません。`write` も `read` も 0 になるので、Step 2 のログで気づけます。最小値はモデルごとに違います。

**原因2: prefix に毎回変わる値が混ざっている。** 典型例は `system` に現在時刻やリクエストIDを入れる書き方です。

```typescript
// 悪い例: 1文字でも変わると、その後ろのキャッシュは全部外れる
system: [{ type: "text", text: `現在時刻: ${new Date().toISOString()}\n${guideline}`,
           cache_control: { type: "ephemeral" } }]

// 良い例: 変わる値は cache_control より後ろ(messages 側)へ出す
messages: [{
  role: "user",
  content: [{ type: "text", text: `現在時刻: ${now}\n質問: ${question}` }],
}]
```

`tools` の定義順が実行ごとに変わる、JSON のキー順が揺れる、といったことでも外れます。ツール定義は配列をソートして固定してください。

**原因3: 並列で初回を投げている。** キャッシュは最初のレスポンスが始まってから読めるようになります。同じ prefix で10並列に投げると、全部が書き込みになりえます。1本目を先に待ってから残りを流します。

```typescript
const [first, ...rest] = questions;
await ask(first); // ウォームアップ
const results = await Promise.all(rest.map(ask));
```

**原因4: TTL が切れている。** デフォルトのキャッシュ寿命は5分程度で、読まれるたびに延びます。呼び出し間隔が空くバッチでは、ヒットしないまま書き込み料金だけ払うことになります。書き込みは通常入力より割高なので、再利用されない prefix にはかえって不利です。

## Step 4: マルチターン会話に広げる

会話が伸びる用途では、直近の user メッセージの末尾にも `cache_control` を置きます。次のターンで、そこまでの履歴が prefix として読まれます。

```typescript
function withCacheOnLast(messages: Anthropic.MessageParam[]): Anthropic.MessageParam[] {
  return messages.map((m, i) => {
    if (i !== messages.length - 1 || m.role !== "user") return m;
    const blocks =
      typeof m.content === "string" ? [{ type: "text" as const, text: m.content }] : [...m.content];
    const last = blocks[blocks.length - 1];
    blocks[blocks.length - 1] = { ...last, cache_control: { type: "ephemeral" } };
    return { ...m, content: blocks };
  });
}
```

`cache_control` を置ける箇所(ブレークポイント)には上限があります。毎ターン増やし続けると上限に当たるので、「system 末尾に1つ、最新 user に1つ」のように固定するのが安全です。

## Step 5: ヒット率を常時監視する

導入して終わりにすると、プロンプトを編集した拍子に静かに外れます。`usage` を集計して、ヒット率の急落を検知してください。

```typescript
let totalRead = 0;
let totalAll = 0;

export function record(u: Anthropic.Usage) {
  const read = u.cache_read_input_tokens ?? 0;
  totalRead += read;
  totalAll += u.input_tokens + (u.cache_creation_input_tokens ?? 0) + read;
  const rate = totalAll ? totalRead / totalAll : 0;
  if (totalAll > 50_000 && rate < 0.5) {
    console.warn(`cache hit rate low: ${(rate * 100).toFixed(1)}%`);
  }
}
```

しきい値の 0.5 は仮の値です。自分のワークロードでヒット率の通常値を測ってから決めてください。

## まとめ

- `cache_control` は、固定部分の最後のブロックに置く
- 効果は `cache_read_input_tokens` で確認する。請求額だけで判断しない
- 外れる原因は、最小トークン未満、prefix に混ざった可変値、初回の並列送信、TTL 切れの4つ
- ヒット率のログは、導入と同時に入れる

まず、自分のアプリの `system` に「毎回変わる値」が混ざっていないか、`grep` で確認するところから始めてみてください。