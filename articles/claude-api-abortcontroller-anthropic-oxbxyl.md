---
title: "Claude APIのストリーミングを途中で止める：AbortControllerで「閉じたタブ」に課金し続けない実装"
emoji: "🛑"
type: "tech"
topics: ["claude", "anthropic", "typescript", "nodejs", "streaming"]
published: true
---

チャットUIで長文の回答を生成させ、途中でタブを閉じたとします。このとき、サーバー側のClaude APIリクエストは止まっているでしょうか。

自分のコードを確認したら止まっていませんでした。クライアントが切断してもサーバーは最後まで生成を受け取り続けます。誰にも届かない出力トークンのために課金され、`max_tokens` を大きく取った呼び出しほど無駄が増えます。

この記事では、Node.js + TypeScript で次の3点を実装します。

- クライアント切断を検知して、Claudeへのストリームを中断する
- 中断までに生成済みのテキストを失わずに保存する
- 本当に止まったかを、ログで確認する

## 前提

- Node.js 20 以上
- `@anthropic-ai/sdk`(`pnpm add @anthropic-ai/sdk`)
- `ANTHROPIC_API_KEY` は環境変数で渡します(コードに書かない)

モデルIDは `claude-sonnet-5-5` を使います。

## Step 1: まず「止まらない」実装を作る

ベースラインです。SSEでテキストを流すだけのサーバーです。

```ts
// server.ts
import http from "node:http";
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

http
  .createServer(async (req, res) => {
    res.writeHead(200, {
      "content-type": "text/event-stream",
      "cache-control": "no-cache",
    });

    const stream = client.messages.stream({
      model: "claude-sonnet-5-5",
      max_tokens: 4096,
      messages: [{ role: "user", content: "分散システムの合意アルゴリズムを詳しく解説して" }],
    });

    stream.on("text", (t) => res.write(`data: ${JSON.stringify(t)}\n\n`));
    await stream.finalMessage();
    res.end();
  })
  .listen(3000);
```

`curl -N localhost:3000` で受信中に Ctrl+C を押してみてください。curl は終わりますが、サーバー側の `stream` は生成を受け取り続けます。

## Step 2: 切断を検知して abort する

`res` の `close` イベントを使います。ここで `req` の `close` を使うと、ボディ読み取り完了時点で発火する環境があるため、`res` 側で判定して、正常終了と区別します。

```ts
const controller = new AbortController();

res.on("close", () => {
  if (!res.writableEnded) controller.abort();
});

const stream = client.messages.stream(
  {
    model: "claude-sonnet-5-5",
    max_tokens: 4096,
    messages: [{ role: "user", content: prompt }],
  },
  { signal: controller.signal },
);
```

SDKの第2引数 `signal` に `AbortSignal` を渡すと、abort時にHTTP接続が閉じられます。SDKが投げる `APIUserAbortError` は握りつぶさず、中断由来かどうかを判定して扱います。

```ts
import Anthropic, { APIUserAbortError } from "@anthropic-ai/sdk";

try {
  await stream.finalMessage();
  res.end();
} catch (e) {
  if (e instanceof APIUserAbortError) {
    // 意図した中断。エラー扱いしない
  } else {
    throw e;
  }
}
```

これを入れないと、切断のたびに未処理の例外がログに出てアラートが鳴ります。

## Step 3: 中断時点までのテキストを残す

途中まで生成された回答は捨てたくない場合があります。たとえば、ユーザーが再接続したときに続きから見せたい場合です。`text` イベントで自分でバッファに貯めておけば、中断後も手元に残ります。

```ts
let partial = "";

stream.on("text", (t) => {
  partial += t;
  if (!res.writableEnded) res.write(`data: ${JSON.stringify(t)}\n\n`);
});

try {
  const msg = await stream.finalMessage();
  await save({ text: partial, status: "complete", usage: msg.usage });
  res.end();
} catch (e) {
  if (!(e instanceof APIUserAbortError)) throw e;
  await save({ text: partial, status: "aborted" });
}
```

`save` は自分のDB書き込み関数です。`status` を分けておくと、後から「途中で切れた回答」だけを集計できます。中断した回答には `usage` が付かないので、トークン数は推定になります。

## Step 4: 止まったことを確かめる

「abortしたつもり」を避けるため、Step 1 と Step 2 で同じ操作をして、ログを比べます。ストリーム終了時に経過時間を出す1行を足します。

```ts
const t0 = Date.now();
stream.on("end", () => console.log(`end after ${Date.now() - t0}ms chars=${partial.length}`));
stream.on("abort", () => console.log(`abort after ${Date.now() - t0}ms chars=${partial.length}`));
```

`curl -N` を3秒後に Ctrl+C して比べます。

- Step 1: 生成が終わるまで `end` が出ない。`chars` は最後まで増える
- Step 2: 切断直後に `abort` が出る。`chars` はその時点で止まる

本当に課金が止まったかは、Anthropic Console の利用量で、同じプロンプトの中断あり・なしを数回流して見比べるのが確実です。中断までに生成された分は課金対象になると考えておくのが安全です。止められるのは、その後の生成分です。

## 落とし穴

- **リバースプロキシのバッファリング**: nginx などがSSEをバッファすると、切断がサーバーに伝わるのが遅れます。`X-Accel-Buffering: no` を返すか、プロキシ側のバッファを切ります
- **1リクエスト内の複数呼び出し**: ツール呼び出しの途中ならループ全体で同じ `signal` を使い回します。呼び出しごとに新しい `AbortController` を作ると、2回目以降が止まりません
- **リトライとの相性**: SDKの自動リトライは abort 後には再試行しません。自前のリトライを書く場合は、`signal.aborted` を先に見てから再試行します

```ts
if (controller.signal.aborted) return; // 切断済みなら再試行しない
```

## まとめ

- 既定のストリーミングは、クライアントが切れてもサーバー側の生成を止めません
- `res.on("close")` で `AbortController.abort()` を呼び、SDKの `signal` に渡せば止まります
- `APIUserAbortError` は正常系として扱い、途中経過は `text` イベントで自前に貯めます
- 止まったかどうかは、`end` と `abort` のログで確かめられます

ストリーミングを出した時点で、中断の設計は実装の一部になります。手元の実装で、タブを閉じたときに何が起きているかを一度確認してみてください。