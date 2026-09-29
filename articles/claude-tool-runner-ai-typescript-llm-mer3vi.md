---
title: "Claude Tool Runnerで「独自ツールを呼ぶAIエージェント」をTypeScriptで実装する"
emoji: "🛠️"
type: "tech"
topics: ["claude", "typescript", "llm", "ai", "anthropic"]
published: true
---

## 「Claude Agent SDK」で検索して迷子になった話

「Claude エージェント 実装」で調べると、Claude Codeをライブラリ化した `@anthropic-ai/claude-agent-sdk` の紹介記事がよく出てくる。ファイル読み書きやBashが最初から使えて便利そうに見えるが、実際に欲しかったのは「自社の在庫APIや監視システムを叩く、こちらが定義した2〜3個のツールだけを持つ小さなエージェント」だった。Claude Codeの機能一式は要らない。

調べていくと、これは別物だとわかった。`@anthropic-ai/sdk`(通常のAPI SDK)には `client.beta.messages.toolRunner` という、**自分で定義したツールのループ実行だけを引き受けてくれるヘルパー**がある。Claude Code相当のハーネスではなく、「ツール呼び出し→実行→結果を返す→次のツール呼び出し…」というループをSDK側が回してくれるだけの薄い仕組みだ。今回はこれを使って、実際に動くツール呼び出しエージェントをゼロから組んでみる。

## Tool Runner が向いている場面

Claude APIでツールを使う場合、選択肢は大きく2つある。

- **手動ループ**: `client.messages.create` を自分で `while` 文で回し、`tool_use` ブロックを拾って結果を返す。制御はすべて自分の手の中にあるが、コード量が増える。
- **Tool Runner**: `betaZodTool` でツールを定義すると、ループ・パラレル実行・停止判定をSDKが代行する。Zodスキーマからツール定義のJSON Schemaも自動生成される。

「独自ツールを何個か持たせたいだけで、承認フローや特殊なトランスポートなど凝った制御は不要」という今回のようなケースは、素直にTool Runnerに乗るのが早い。現時点(SDK 0.128系)ではベータ扱いなので、`client.beta.messages` 名前空間から呼ぶ点だけ注意する。

## 実装: サービス監視エージェント

「サービスの状態を確認して、必要なら再起動する」という小さな運用エージェントを作る。ツールは2つ。

- `get_service_status`: 指定サービスの稼働状態を取得
- `restart_service`: 状態が悪い場合のみ再起動を許可(正常稼働中のサービスは拒否する)

### セットアップ

```bash
npm install @anthropic-ai/sdk zod
```

### ツールを定義する

`betaZodTool` に `inputSchema`(Zod)と `run`(実行関数)を渡すだけでツールになる。スキーマの説明文(`.describe()`)はモデルへのヒントとしてそのまま使われるので、曖昧な引数名ほど丁寧に書く。

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const client = new Anthropic();

// モックのサービス状態ストア(本番ではDBや監視APIに置き換える)
const SERVICES = new Map([
  ["api-gateway", { status: "degraded", latencyMs: 820 }],
  ["billing-worker", { status: "healthy", latencyMs: 45 }],
]);

const getServiceStatus = betaZodTool({
  name: "get_service_status",
  description: "指定したサービスの稼働状態とレイテンシを取得する",
  inputSchema: z.object({
    serviceName: z.string().describe("api-gateway や billing-worker など"),
  }),
  run: async (input) => {
    const service = SERVICES.get(input.serviceName);
    if (!service) {
      return JSON.stringify({ error: `unknown service: ${input.serviceName}` });
    }
    return JSON.stringify(service);
  },
});

const restartService = betaZodTool({
  name: "restart_service",
  description: "サービスを再起動する。degraded/unhealthy のときのみ許可される",
  inputSchema: z.object({
    serviceName: z.string(),
    reason: z.string().describe("再起動が必要と判断した理由"),
  }),
  run: async (input) => {
    const service = SERVICES.get(input.serviceName);
    if (!service) {
      return JSON.stringify({ ok: false, error: `unknown service: ${input.serviceName}` });
    }
    if (service.status === "healthy") {
      // 正常なサービスを勝手に落とさせない
      return JSON.stringify({ ok: false, error: "service is healthy; refusing restart" });
    }
    SERVICES.set(input.serviceName, { status: "healthy", latencyMs: 40 });
    return JSON.stringify({ ok: true, restarted: input.serviceName, reason: input.reason });
  },
});
```

ここでのポイントは、`restart_service` の安全弁を **ツールの実行関数の中**に置いていることだ。「正常稼働中のサービスは再起動しない」という業務ルールをプロンプトの指示文だけに頼ると、モデルが指示を読み飛ばした時に事故る。ツール側でガードしておけば、モデルの判断がどうであれ実際の副作用は止まる。

### ループを回す

Tool Runnerに渡せば、あとは `await` するだけで最終応答まで面倒を見てくれる。

```typescript
async function main() {
  const finalMessage = await client.beta.messages.toolRunner({
    model: "claude-opus-5",
    max_tokens: 4096,
    tools: [getServiceStatus, restartService],
    messages: [
      {
        role: "user",
        content:
          "api-gateway と billing-worker の状態を確認して、必要なら再起動して。最後に結果を一言で報告して。",
      },
    ],
  });

  for (const block of finalMessage.content) {
    if (block.type === "text") {
      console.log(block.text);
    }
  }
}

main();
```

このスクリプトは `@anthropic-ai/sdk` 0.128.0 + `zod` 4.6.5 の組み合わせで `tsc --noEmit --strict` を実行し、型エラーが出ないことを確認済み(実行にはAPIキーが必要なため、型検証まで)。実際に`ANTHROPIC_API_KEY` を設定して走らせると、以下のような流れになる。

1. `get_service_status` を `api-gateway` と `billing-worker` の両方に対して**並列**で呼び出す(Tool Runnerはデフォルトで並列呼び出しに対応している)
2. `api-gateway` が `degraded` だとわかると `restart_service` を呼ぶ
3. `billing-worker` は `healthy` なので再起動をスキップし、そのままテキストで報告する

途中の `tool_use` → `tool_result` のやり取りをログに出したい場合は、`toolRunner` は `for await` でも消費できる(非ストリーミング時は各反復で完全なメッセージが1つ流れてくる)。

```typescript
const runner = client.beta.messages.toolRunner({
  model: "claude-opus-5",
  max_tokens: 4096,
  tools: [getServiceStatus, restartService],
  messages: [{ role: "user", content: "api-gatewayの状態を見て" }],
});

for await (const message of runner) {
  console.log(message.stop_reason, message.content);
}
```

## つまずきやすい点

- **`toolRunner` は `pause_turn` を自動で再開しない**。Web検索など長時間のサーバー側ツールを混ぜる場合、`stop_reason === "pause_turn"` を見て `runner.pushMessages(...)` で自分で再開させる必要がある。今回のような自作ツールだけの構成なら基本的に踏まない罠だが、後からサーバーツールを足すときにハマりやすい。
- **`run` の戻り値は「モデルに見せるテキスト」**であって、成功/失敗のフラグではない。今回のように `JSON.stringify({ ok: false, ... })` を返すと、モデルはその文字列を読んで次の行動を判断する。本当に実行を止めたいエラー(不正なJSONなど)は `is_error: true` を使う手動ループ側の話になるので、Tool Runnerでは「失敗も含めて文章として返す」設計に寄せるのが素直。
- **モデルIDと料金**: 2026年6月時点でAnthropicが公開している価格は、Claude Opus 5が入力$5/出力$25(100万トークンあたり)、Claude Sonnet 5が$2/$10、Claude Haiku 4.5が$1/$5。監視エージェントのように呼び出し頻度が高くなりがちな用途では、まずSonnet 5やHaiku 4.5で様子を見て、判断の質が必要な場面だけOpus系に上げる構成が現実的なコストになりやすい。

## まとめ

Claude Agent SDK(Claude Codeのライブラリ版)とTool Runner(API SDKのツール実行ヘルパー)は名前が紛らわしいが、前者はハーネスと組み込みツール一式、後者は「自分のツールのループ実行だけを代行する薄いレイヤー」という別物だった。独自APIを叩くだけの小さな運用エージェントなら、Tool Runnerで十分に動く。次はここに `web_search` などのサーバーツールを混ぜて、`pause_turn` の再開処理を試す予定。