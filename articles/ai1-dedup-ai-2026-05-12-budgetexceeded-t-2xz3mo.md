---
title: "プロンプト劣化をCIで検出する：Vitest + LLM-as-Judge で評価パイプラインを自前実装する"
emoji: "⚖️"
type: "tech"
topics: ["llm", "vitest", "anthropic", "testing", "ci"]
published: true
---

「プロンプトを少し調整したら、翌日からユーザーの満足度が静かに落ちていった。単体テストは全部 green のまま。気づいたのは1週間後だった。」

LLM アプリを本番で動かし始めると、必ずこの問題に直面する。**プロンプトは差分 3 行なのに、出力の品質は大幅に変わる**。しかも従来の assertEquals では絶対に検知できない。

この記事では、**Vitest + Anthropic Claude の LLM-as-Judge パターン**を使って、プロンプト品質を CI で定量的に評価するパイプラインをゼロから実装する。外部サービスへの依存は最小限、動くコードを中心に説明する。

---

## なぜ普通のユニットテストが使えないのか

```typescript
// これは使えない
expect(await summarize(article)).toBe("TypeScript は静的型付き言語です。");
```

LLM の出力は**非決定論的**だ。同じ入力でも毎回わずかに変わる。「完全一致」でテストしても温度パラメータで壊れる。かといって「文字列が含まれる」程度のテストでは品質劣化を捕捉できない。

必要なのは「どれだけ的確か」を **0〜100 のスコアに変換して閾値比較する**仕組みだ。

---

## LLM-as-Judge とは

評価者に別の LLM を使い、「この出力は基準を満たしているか？1〜5 点で採点してください」と問いかける手法。2024〜2025 年に急速に普及し、現在は人間の評価との一致率が **約 80%** まで確認されている（LangSmith・EvidentlyAI の調査より）。

ポイントは**ジャッジを上位モデル、被評価を下位モデル**に分けること。コストを抑えつつ評価精度を確保できる。

---

## 実装：ステップバイステップ

### Step 1: セットアップ

```bash
pnpm add -D vitest
pnpm add @anthropic-ai/sdk
```

`.env` に API キーを置く（`.gitignore` に追加することを忘れずに）：

```
ANTHROPIC_API_KEY=sk-ant-...
```

### Step 2: ジャッジ関数を実装する

`src/eval/judge.ts` として保存する。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

export interface JudgeResult {
  score: number;   // 1-5
  reasoning: string;
}

export async function judgeResponse(
  input: string,
  output: string,
  criteria: string
): Promise<JudgeResult> {
  const message = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 256,
    messages: [
      {
        role: "user",
        content: `あなたは LLM 出力の評価者です。以下の基準で出力を採点してください。

【評価基準】
${criteria}

【入力】
${input}

【出力】
${output}

JSON 形式のみで応答してください: {"score": <1-5の整数>, "reasoning": "<一文で理由>"}
5点=完璧、1点=完全に外れている。`,
      },
    ],
  });

  const text =
    message.content[0].type === "text" ? message.content[0].text : "";

  const match = text.match(/\{[\s\S]*?\}/);
  if (!match) throw new Error(`Judge が JSON を返しませんでした: ${text}`);

  return JSON.parse(match[0]) as JudgeResult;
}
```

**注意点が2つある。**

1. `model` は**ジャッジには上位モデルを使う**こと。Haiku に採点させると精度が落ちる
2. Claude は稀に JSON を ` ```json ``` ` で囲んで返すため、正規表現でブロックを抽出している

### Step 3: 評価テストを書く

`src/eval/summarize.eval.test.ts`：

```typescript
import { describe, it, expect } from "vitest";
import Anthropic from "@anthropic-ai/sdk";
import { judgeResponse } from "./judge";

const client = new Anthropic();

async function summarize(text: string): Promise<string> {
  const msg = await client.messages.create({
    model: "claude-haiku-4-5-20251001",
    max_tokens: 256,
    messages: [
      {
        role: "user",
        content: `以下を3文以内で要約してください:\n\n${text}`,
      },
    ],
  });
  return msg.content[0].type === "text" ? msg.content[0].text : "";
}

const PASS_THRESHOLD = 3; // 5点中3点以上でパス

const TEST_CASES = [
  {
    name: "技術記事の要約",
    input:
      "TypeScript は Microsoft が開発した静的型付き言語で、JavaScript のスーパーセットです。コンパイル時に型エラーを検出し、大規模アプリケーションの保守性を高めます。2012年に公開されて以来、React・Angular・Node.js エコシステムで広く採用されています。",
    criteria: "元の内容を正確に捉えており、3文以内にまとまっているか",
  },
  {
    name: "エラーメッセージの要約",
    input:
      "ECONNREFUSED: Connection refused at 127.0.0.1:5432. The PostgreSQL database is not running or the port is incorrect.",
    criteria: "問題の原因と対処法が簡潔に含まれているか",
  },
];

describe("summarize() — LLM-as-Judge eval", () => {
  for (const tc of TEST_CASES) {
    it(tc.name, async () => {
      const output = await summarize(tc.input);
      const result = await judgeResponse(tc.input, output, tc.criteria);

      console.log(
        `[${tc.name}] Score: ${result.score}/5 — ${result.reasoning}`
      );
      console.log(`Output: ${output}`);

      expect(
        result.score,
        `品質スコアが閾値 ${PASS_THRESHOLD} を下回りました: ${result.reasoning}`
      ).toBeGreaterThanOrEqual(PASS_THRESHOLD);
    }, 30_000); // LLM 呼び出しは 30 秒タイムアウト
  }
});
```

`console.log` でスコアと理由を出力しておくと、CI ログで劣化原因をすぐ読める。

### Step 4: vitest.config.ts でタイムアウトを設定

LLM 呼び出しはデフォルトの 5 秒タイムアウトで必ず失敗する。

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    testTimeout: 60_000,
    hookTimeout: 30_000,
    include: ["**/*.eval.test.ts"],
    reporters: ["verbose"],
  },
});
```

`*.eval.test.ts` をパターンで限定しておくことで、通常のユニットテスト（`*.test.ts`）と分離して実行できる。

### Step 5: GitHub Actions に組み込む

`.github/workflows/eval.yml`：

```yaml
name: LLM Eval

on:
  pull_request:
    paths:
      - "src/**/*.ts"
      - "prompts/**"

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"

      - run: pnpm install --frozen-lockfile

      - name: Run LLM evals
        run: pnpm vitest run --config vitest.config.ts
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

`paths` フィルターでプロンプトファイルや TypeScript の変更があった PR だけ eval を走らせる。全 PR で毎回動かすとコストが積み上がる。

---

## 実運用で気をつけること

**スコア閾値は 3/5 から始める。** いきなり 4/5 にすると non-determinism でフレーキーになる。数週間のデータを集めてから段階的に上げる。

**テストケースは CSV で管理する。** コードに埋め込むと増えるほど PR が汚れる。`eval/cases.csv` を読み込む方式にすると、エンジニア以外がケースを追加しやすい。

**コストの目安。** Sonnet（ジャッジ）+ Haiku（被評価）の組み合わせで、1 テストケース約 $0.003 程度。20 ケースでも 1 回の eval 実行コストは $0.06 以下に収まる。

**スコアを時系列で記録する。** CI ログだけだと傾向が見えない。Supabase や JSON ファイルにスコアを蓄積し、プロンプト変更とスコア推移を突き合わせると「どの変更で落ちたか」が一目でわかる。

---

## まとめ

| ステップ | やること |
|---------|---------|
| judge.ts | Anthropic SDK で採点関数を実装 |
| *.eval.test.ts | Vitest の it() でジャッジを呼び閾値チェック |
| vitest.config.ts | タイムアウト 60s、eval ファイルを限定 |
| GitHub Actions | `paths` フィルターでコスト最適化 |

LLM アプリは「動くかどうか」ではなく「どれだけ良いか」が問われる。`assertEquals` が効かない領域に、評価者としての LLM を投入することで、**プロンプト変更の影響を CI が定量的に拒否できる**ようになる。

まず judge.ts 1 ファイルを書くだけで、明日からプロンプト変更に怯えなくて済む。