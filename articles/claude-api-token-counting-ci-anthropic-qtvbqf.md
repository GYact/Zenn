---
title: "Claude APIのToken Counting APIで「送る前に」コストを止める：プロンプト肥大をCIで検知する実装手順"
emoji: "🧮"
type: "tech"
topics: ["claude", "anthropic", "typescript", "llm", "api"]
published: true
---

システムプロンプトに「ちょっとだけルールを足した」だけで、請求が先月の1.4倍になっていた。そんな経験はないでしょうか。原因はたいてい、リクエストを送ってから `usage` を見て初めて気づく点にあります。

この記事では、Claude API の Token Counting API を使って、**送信前にトークン数を測り、上限を超えたら止める**仕組みを TypeScript で作ります。最後は CI でプロンプトの肥大を検知するところまで進めます。

## 前提と環境

- Node.js 20 以上、パッケージは pnpm
- `@anthropic-ai/sdk`
- API キーは環境変数 `ANTHROPIC_API_KEY`(コードやチャットに貼らない)

```bash
pnpm add @anthropic-ai/sdk
pnpm add -D tsx typescript
```

モデルIDは記事執筆時点(2026-10-02)で `claude-sonnet-5-5` を使います。料金やモデルは変わるので、運用前に公式ドキュメントで確認してください。

## Step 1: まず1回だけ数えてみる

Token Counting API は、`messages.create` と同じ形の引数を渡すと、実際には生成せず入力トークン数だけを返します。

```ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const { input_tokens } = await client.messages.countTokens({
  model: "claude-sonnet-5-5",
  system: "あなたは社内ヘルプデスクの回答者です。",
  messages: [{ role: "user", content: "VPNにつながりません" }],
});

console.log(input_tokens);
```

公式ドキュメントでは、このエンドポイントは利用無料で、レート制限は `messages.create` とは別枠と案内されています。数えるだけなら課金を気にせず呼べますが、仕様は変わりうるので最新の記載を確認してください。返る値は見積もりで、実際の課金トークン数と少しずれることがあります。そのため、上限判定には余裕を持たせます。

## Step 2: 内訳を差分で出す

合計だけ分かっても、どこが太ったのか分かりません。`system` / `tools` / `messages` を1つずつ外して数え、差分から内訳を出します。

```ts
import Anthropic from "@anthropic-ai/sdk";
import type { MessageParam, Tool } from "@anthropic-ai/sdk/resources/messages";

const client = new Anthropic();
const MODEL = "claude-sonnet-5-5";

type Req = { system?: string; tools?: Tool[]; messages: MessageParam[] };

const count = async (r: Req) =>
  (await client.messages.countTokens({ model: MODEL, ...r })).input_tokens;

export async function breakdown(req: Req) {
  const [total, noSystem, noTools] = await Promise.all([
    count(req),
    count({ ...req, system: undefined }),
    count({ ...req, tools: undefined }),
  ]);
  return {
    total,
    system: total - noSystem,
    tools: total - noTools,
    messages: total - (total - noSystem) - (total - noTools),
  };
}
```

`messages` は残りとして求めています。`tools` を足すとツール利用向けの内部プロンプトが加わる実装のため、`tools` 側の数字には定義そのもの以外の固定分も含まれる点に注意してください。絶対値より、**変更前後の差**を見る使い方が向いています。

## Step 3: 送信前ガードを作る

数えた結果が予算を超えたら、API を呼ばずに例外にします。

```ts
export class TokenBudgetError extends Error {
  constructor(
    public readonly tokens: number,
    public readonly limit: number,
  ) {
    super(`input ${tokens} tokens exceeds budget ${limit}`);
  }
}

export async function createWithBudget(
  req: Req & { max_tokens: number },
  inputBudget: number,
) {
  const { max_tokens, ...countable } = req;
  const tokens = await count(countable);
  if (tokens > inputBudget) throw new TokenBudgetError(tokens, inputBudget);

  return client.messages.create({ model: MODEL, max_tokens, ...req });
}
```

会話履歴が伸びるチャットや、ファイルを丸ごと貼るエージェントで効きます。例外を捕まえたら、古い履歴の要約や切り詰めに回せます。

```ts
try {
  await createWithBudget(req, 30_000);
} catch (e) {
  if (e instanceof TokenBudgetError) {
    req.messages = req.messages.slice(-6); // 直近だけ残して再試行
    await createWithBudget(req, 30_000);
  } else {
    throw e;
  }
}
```

事前カウントは往復が1回増えます。短い入力が大半のワークロードなら、`messages` の文字数が閾値を超えたときだけ数える運用で十分です。

## Step 4: CIでプロンプト肥大を検知する

冒頭の「ルールを足したら請求が増えた」を、プルリクエストの時点で止めます。システムプロンプトとツール定義だけを数えて、基準値と比較するスクリプトです。

```ts
// scripts/check-prompt-size.ts
import Anthropic from "@anthropic-ai/sdk";
import { readFileSync } from "node:fs";
import { tools } from "../src/tools";

const client = new Anthropic();
const system = readFileSync("prompts/system.md", "utf8");
const BASELINE = Number(process.env.PROMPT_BASELINE); // 例: 2400
const TOLERANCE = 1.1; // 10%まで許容

const { input_tokens } = await client.messages.countTokens({
  model: "claude-sonnet-5-5",
  system,
  tools,
  messages: [{ role: "user", content: "ping" }],
});

console.log(`固定部分: ${input_tokens} tokens (baseline ${BASELINE})`);
if (input_tokens > BASELINE * TOLERANCE) {
  console.error("固定プロンプトが基準を10%以上超えています。意図した変更なら BASELINE を更新してください");
  process.exit(1);
}
```

GitHub Actions では、API キーを Secrets に置いて呼び出します。

```yaml
- run: pnpm tsx scripts/check-prompt-size.ts
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    PROMPT_BASELINE: "2400"
```

基準値は、まず現状を1回測ってから決めます。ここを推測で置くと、初回から赤になります。

## 落とし穴

- **キャッシュは考慮されない**: カウントは入力全体の量です。プロンプトキャッシュが効く部分の課金割引までは反映されないので、「課金額」ではなく「入力の大きさ」として扱います。
- **画像・PDF も数えられる**: ただし base64 を毎回送るので、事前カウントの通信量自体が大きくなります。
- **`ping` ダミーの長さ**: CI では固定部分の比較が目的なので、ユーザーメッセージは短い固定文字列にします。
- **モデルを変えたら基準値も測り直す**: トークナイザがモデルごとに違う可能性があるため、モデルID を変えた PR では BASELINE も更新します。

## まとめ

1. `countTokens` に `create` と同じ引数を渡せば、生成せず入力トークン数が分かる
2. `system` / `tools` を外して差分を取ると、肥大の原因が見える
3. 予算超過は送信前に例外にして、履歴の切り詰めへ回せる
4. 固定プロンプトは CI で基準値と比べ、静かな肥大を PR で止める

請求書で気づく前に、手元で測れるようになります。まずは Step 2 の `breakdown` を自分のプロンプトに1回かけて、最も太い部分を確認してみてください。