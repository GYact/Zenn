---
title: "Claude Sonnet 5.5へ載せ替えると400になる4つの書き方：smokeテストで先に潰す"
emoji: "🧪"
type: "tech"
topics: ["claude", "anthropic", "typescript", "llm", "api"]
published: true
---

モデルIDを `claude-sonnet-5-5` に書き換えただけで、本番のジョブが全部 400 で落ちる。載せ替えで起きがちな事故です。

Sonnet 5.5 は、以前のモデルで普通に通っていたリクエストの形をいくつか拒否します。この記事では、拒否される形を最小リクエストで再現するsmokeテストを書き、新しい書き方に直すところまでを手順にします。

前提は `@anthropic-ai/sdk` と TypeScript です。仕様はClaude API公式スキル同梱の資料(2026-09-25時点のキャッシュ)で確認しました。細部は変わりうるので、最終的な正本は公式ドキュメントです。

## 400になる書き方

Sonnet 5.5 で400になるのは次の4つです。

| 旧来の書き方 | 5.5での扱い | 直し方 |
|---|---|---|
| `thinking: {type: "disabled"}` | 400 | `{type: "between_tools"}` か、thinkingを残して `effort` を下げる |
| `thinking: {type: "enabled", budget_tokens: N}` | 400 | `{type: "adaptive"}` か省略 |
| `tool_choice: {type: "any"}` / `{type: "tool", name}` | 400 | `auto` + プロンプトでの指示 + `strict: true` |
| 既定値以外の `temperature` / `top_p` / `top_k` | 400 | 送らない |

assistantメッセージの末尾に書くprefillも、4.6以降のモデルでは400です。JSONを強制していた古いコードはここに当たります。

## Step 1: 旧い形が本当に落ちることを確認する

最初に「落ちる形」をテストにします。落ちる前提のテストを持っておくと、後でSDKやモデルの仕様が変わったときに気づけます。

```typescript
// smoke.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const MODEL = "claude-sonnet-5-5";

const base = {
  model: MODEL,
  max_tokens: 256,
  messages: [{ role: "user", content: "1+1は?" }],
} as const;

type Params = Anthropic.MessageCreateParamsNonStreaming;
// between_tools など SDK の型に未登録の値を送るため、一度 unknown 経由で通す
const asParams = (p: unknown) => p as Params;

async function expect400(name: string, params: Params) {
  try {
    await client.messages.create(params);
    console.log(`NG   ${name}: 通ってしまった`);
    process.exitCode = 1;
  } catch (e) {
    if (e instanceof Anthropic.BadRequestError) {
      console.log(`OK   ${name}: 400 (${e.message.slice(0, 80)})`);
    } else {
      throw e; // 429や5xxを「期待どおり」に数えない
    }
  }
}

await expect400("thinking disabled", asParams({ ...base, thinking: { type: "disabled" } }));
await expect400(
  "budget_tokens",
  asParams({ ...base, thinking: { type: "enabled", budget_tokens: 2048 } }),
);
await expect400("temperature", asParams({ ...base, temperature: 0.2 }));
```

`BadRequestError` だけを期待どおりとして数えているのは、レート制限や5xxを「400が出たので正常」と取り違えないためです。

## Step 2: thinkingを切っていた経路を直す

分類やルーティングなど、thinkingが要らない経路で `disabled` を使っていたはずです。5.5では次の順で検討します。

1. thinkingを残し、`output_config: { effort: "low" }` で軽くする。まずこれを試す
2. それでも切りたい経路だけ `{ type: "between_tools" }` を使う

`between_tools` には制約があります。`effort` が `high` 以下でないと400になり、`display` や `budget_tokens` は併記できません。会話の途中でeffortを変えることもできません。

```typescript
const light = asParams({
  ...base,
  thinking: { type: "adaptive" },
  output_config: { effort: "low" },
});

const noThink = asParams({
  ...base,
  thinking: { type: "between_tools" }, // 他のフィールドを足さない
  output_config: { effort: "high" },
});
```

effortの既定は `high` のままですが、段階の較正は Sonnet 5 から変わっています。エージェント的なコーディングや多段のツール利用は `medium`、チャットは `low` から測り直すのがよさそうです。

ストリーミングで思考を見せているUIは、`thinking.display` の既定が `omitted` なので、何も指定しないと思考テキストが空になります。見せたいなら `{ type: "adaptive", display: "summarized" }` を明示します。

## Step 3: forced tool_choiceをstrictとプロンプトに置き換える

「JSONが欲しいから `tool_choice` でツールを強制する」実装は5.5では400です。置き換えは次の2通りです。

```typescript
const tools: Anthropic.Tool[] = [
  {
    name: "classify_ticket",
    description: "問い合わせを分類して結果を返す。必ずこのツールで答える。",
    strict: true, // 引数がスキーマに厳密に一致する
    input_schema: {
      type: "object",
      properties: {
        category: { type: "string", enum: ["billing", "bug", "other"] },
        urgent: { type: "boolean" },
      },
      required: ["category", "urgent"],
      additionalProperties: false,
    },
  },
];

const res = await client.messages.create({
  model: MODEL,
  max_tokens: 1024,
  tools,
  tool_choice: { type: "auto" },
  system: "回答は必ず classify_ticket ツールの呼び出しで返すこと。",
  messages: [{ role: "user", content: "請求が二重になっています。至急対応を" }],
});

const call = res.content.find((b) => b.type === "tool_use");
if (!call) throw new Error(`ツールが呼ばれなかった: stop_reason=${res.stop_reason}`);
console.log(call.input);
```

`strict: true` は `tool_choice` ではなくツール定義の直下に置きます。スキーマには `additionalProperties: false` と `required` が必要です。

強制の代わりは「プロンプトで指示するだけ」なので、呼ばれない可能性が残ります。上のコードの `!call` の分岐が必要な理由です。

ツールが要らず、JSONが欲しいだけなら、ツールを経由せず `output_config: { format: ... }` のStructured Outputsに寄せるほうが素直です。

## Step 4: 新しい形が通ることも確認する

Step 1のテストに、直した形が通る確認を足します。「落ちる形が落ちる」だけでは、移行後のコードが動くことの証明になりません。

```typescript
async function expectOk(name: string, params: Params) {
  const r = await client.messages.create(params);
  console.log(`OK   ${name}: stop_reason=${r.stop_reason}, out=${r.usage.output_tokens}tok`);
}

await expectOk("adaptive + low effort", light);
await expectOk("between_tools", noThink);
```

CIで回すなら、実際にAPIを叩くので数十トークン程度の課金が毎回発生します。`max_tokens: 256` に絞っているのはそのためです。頻度はモデル更新やSDK更新のときだけに絞れば十分です。

## 続けて確認したいこと

- Sonnet 5.5 は、安全分類器が拒否したときも HTTP 200 で `stop_reason: "refusal"` を返します。`content` を読む前に必ず `stop_reason` を見ます
- 思考ブロックは生成したモデルとその会話に結び付きます。履歴を書き換える実装(古いターンの削除や要約)があると、思考ブロックが無効になりえます
- 単価はスキル同梱の表で入力 $2 / 出力 $10 per 1M tokens でした。Sonnet 4.6 の $3 / $15 から載せ替える場合は、トークナイザの差も含めて `count_tokens` で測り直します

## まとめ

載せ替えの400は、本番に出る前にsmokeテストで検出できます。

1. 落ちる4つの形(disabled、budget_tokens、forced tool_choice、temperature)を、400が返ることで確認する
2. thinkingは `effort: "low"` を先に試し、切る必要があれば `between_tools` を使う
3. forced tool_choiceは `auto` + `strict: true` + プロンプトに置き換え、ツールを呼ばないケースを分岐で拾う
4. 直した形が通ることも同じスクリプトで確認する

私の環境ではまだこのスクリプトを本番系統に当てておらず、効果は未計測です。まず手元の `thinking` と `tool_choice` を grep して、旧い形がどこに残っているか数えるところから始めるのがよいと思います。