---
title: "同じPDFを毎回base64で送るのをやめる：Claude Files APIで「1回アップロード、N回質問」を実装する"
emoji: "📎"
type: "tech"
topics: ["claude", "anthropic", "typescript", "llm", "api"]
published: true
---

## 100ページのPDFに5問きくと、5回アップロードしている

契約書や仕様書のPDFに対して「要約して」「解約条項は？」「支払サイトは？」と質問を重ねる機能を作ったとします。base64で素直に実装すると、質問のたびに数MBのリクエストボディを送り直します。

Files APIを使うと、ファイルを1回アップロードして `file_id` を受け取り、以降のリクエストはIDで参照できます。この記事では、次の3点を動くコードで実装します。

- アップロードと `file_id` 参照
- 1ファイルに複数の質問を並列で投げる
- 使い終わったファイルを確実に消す後始末

## 前提と注意点

- Files APIはベータを卒業しています。参考にした公式スキル資料には「`client.beta.files` から `client.files` へ移り、ベータヘッダ不要」とあります。この記事のコードは `client.files` と通常の `client.messages` を使います。
- 同資料のサンプルは旧ベータ形式のままの部分があります。SDKのバージョンで `client.files` が生えているかを先に確認してください。型エラーが出たら SDK を更新します。
- ファイルの保存・一覧・削除は無料で、メッセージ内で読ませた分が入力トークンとして課金されます。つまり**節約されるのは転送量とアップロード処理で、トークン課金は変わりません**。トークン側を下げたいなら別途プロンプトキャッシュが必要です。
- 1ファイルの上限は500MB、組織全体で100GBです。ファイルは削除するまで残ります。
- Amazon BedrockとGoogle Vertex AIでは使えません。

モデルIDは `claude-opus-5-5` を使います。

## Step 1: セットアップ

```bash
pnpm add @anthropic-ai/sdk
```

認証は環境変数 `ANTHROPIC_API_KEY` から読まれます。コードにキーは書きません。

## Step 2: アップロードして file_id を得る

```typescript
import Anthropic, { toFile } from "@anthropic-ai/sdk";
import fs from "node:fs";

const client = new Anthropic();

export async function uploadPdf(path: string): Promise<string> {
  const uploaded = await client.files.upload({
    file: await toFile(fs.createReadStream(path), undefined, {
      type: "application/pdf",
    }),
  });
  console.log(`uploaded: ${uploaded.id} (${uploaded.size_bytes} bytes)`);
  return uploaded.id;
}
```

`toFile` の第3引数でMIMEタイプを明示しています。ストリームから作るとファイル名やタイプが推定できないことがあるため、指定しておくほうが確実です。

## Step 3: file_id を document ブロックで参照する

```typescript
export async function ask(fileId: string, question: string): Promise<string> {
  const res = await client.messages.create({
    model: "claude-opus-5-5",
    max_tokens: 16000,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: { type: "file", file_id: fileId },
            title: "契約書",
          },
          { type: "text", text: question },
        ],
      },
    ],
  });

  return res.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => b.text)
    .join("");
}
```

注意点が2つあります。

1. ブロックの種類はファイルのMIMEタイプと一致させます。PDFやテキストは `document`、画像は `image` です。
2. `document` ブロックは質問文のtextより前に置きます。公式のPDF入力の説明もこの順です。

## Step 4: 1ファイルに複数質問を並列で投げる

```typescript
const questions = [
  "この契約の解約条項を引用つきで要約して",
  "支払いサイトと遅延損害金の定めは？",
  "自動更新の条件は？",
];

const fileId = await uploadPdf("./contract.pdf");

try {
  const answers = await Promise.all(questions.map((q) => ask(fileId, q)));
  answers.forEach((a, i) => console.log(`Q${i + 1}: ${questions[i]}\n${a}\n`));
} finally {
  await client.files.delete(fileId);
}
```

`Promise.all` で同時に投げると、レート制限(429)に当たる可能性があります。SDKは既定で2回までリトライしますが、質問が数十件に増えるなら同時実行数を絞ってください。

## Step 5: 後始末を `finally` に入れる

Files APIのファイルは**自分で消すまで残ります**。リクエスト処理中に例外が出ると削除が飛びます。そのため上のコードは `finally` で削除しています。

放置ファイルを棚卸しする場合は、一覧を取って古いものを消します。

```typescript
for await (const f of client.files.list()) {
  console.log(`${f.id}\t${f.filename}\t${f.size_bytes}`);
}
```

削除の前に、`filename` と `created_at` を見て対象を確かめてください。組織全体のストレージを共有しているので、他のジョブのファイルを巻き込む可能性があります。この一覧は確認用にとどめ、削除は ID を明示して行うのが安全です。

## 引用を付けたい場合

`document` ブロックに `citations: { enabled: true }` を足せば、回答に引用位置が付きます。PDFでは `page_location`(1始まりのページ番号)で返ります。ただし `output_config.format`(構造化出力)とは併用できず、400になります。「JSONで返して、かつ引用も」は1リクエストでは組めないので、2段階に分けて設計します。

## まとめ

- ファイルは1回アップロードして `file_id` で参照します。同じ資料に何度も質問する用途で効きます。
- 節約できるのは送信量です。入力トークンの課金は変わらないため、コストはキャッシュで別途下げます。
- 削除は自動で行われません。`finally` で消す処理を入れておくと、ストレージに残りません。
- BedrockとVertex AIでは使えません。そのため、マルチクラウド構成ではbase64送信との分岐が要ります。

この記事のコードは、実際に実行して確かめたものではありません。SDKの型定義や公式資料の記述をもとに書いています。手元で動かす際は、SDKのバージョンと `client.files` の有無を最初に確認してください。