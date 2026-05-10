---
title: "Claude Tool Use で作るKYC書類解析パイプライン：Anthropic金融エージェントの仕組みを自前実装で理解する"
emoji: "🏦"
type: "tech"
topics: ["claude", "anthropic", "python", "llm", "tooluse"]
published: true
---

書類1枚のOCR結果を渡したら、氏名・生年月日・書類番号がスキーマ崩れなしのJSONで返ってくる。Anthropicが2026年5月に公開した `anthropics/financial-services` リポジトリが話題だが、KYCスクリーナーのコアは意外とシンプルだ。Claude APIの **Tool Use** を使えば、同じ仕組みが30〜40行で自前実装できる。テンプレートをそのまま使うより、内部構造を理解してから使う方が圧倒的に応用が効く。

## なぜプロンプトだけではダメか

LLMに「JSONで返して」と頼むだけではどうなるか。

```python
# Bad: プロンプトのみで抽出
response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": f"以下の書類からJSON抽出:\n{doc}"}]
)
# → マークダウンコードブロックで返ってくる
# → フィールド名が "fullName" だったり "full_name" だったり揺れる
# → nullのはずのフィールドが省略されて KeyError が飛ぶ
```

**Tool Use を使うと JSONSchema がスキーマとして強制される**。スキーマから外れた出力はAPIレベルで起きない。型安全な抽出の正道がこれだ。

## セットアップ

```bash
pip install anthropic pydantic python-dotenv
```

```env
# .env
ANTHROPIC_API_KEY=sk-ant-...
```

## Step 1: 抽出スキーマを Pydantic で定義する

まず欲しいデータ構造を Pydantic モデルで定義する。ここが設計の核心で、フィールドの `description` がClaudeへの指示になる。

```python
from pydantic import BaseModel, Field
from typing import Optional

class KycData(BaseModel):
    full_name: str = Field(description="氏名（フルネーム）")
    date_of_birth: str = Field(description="生年月日（YYYY-MM-DD形式）")
    document_type: str = Field(
        description="書類種別: passport / driving_license / my_number のいずれか"
    )
    document_number: str = Field(description="書類番号・証明番号")
    nationality: Optional[str] = Field(
        default=None, description="国籍（確認できない場合はnull）"
    )
    address: Optional[str] = Field(
        default=None, description="住所（記載がある場合のみ）"
    )
    expiry_date: Optional[str] = Field(
        default=None, description="有効期限（YYYY-MM-DD形式）"
    )
    confidence_score: float = Field(
        ge=0.0, le=1.0,
        description="抽出信頼度スコア（0.0〜1.0）。読み取り困難な項目があれば低めにする"
    )
    extraction_notes: Optional[str] = Field(
        default=None,
        description="不確かな点や注意事項があれば記述"
    )
```

## Step 2: Pydantic スキーマから Tool 定義を生成する

Pydantic v2 の `model_json_schema()` が返す辞書は、そのままAnthropicのTool定義の `input_schema` に使える。

```python
import anthropic
import os
from dotenv import load_dotenv

load_dotenv()

def build_kyc_tool() -> dict:
    schema = KycData.model_json_schema()
    return {
        "name": "extract_kyc_data",
        "description": "本人確認書類からKYC情報を構造化して抽出する",
        "input_schema": schema,
    }
```

## Step 3: メインの抽出クラスを実装する

```python
class KycExtractor:
    def __init__(self):
        self.client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
        self.tool = build_kyc_tool()

    def extract(self, document_text: str) -> KycData:
        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=[self.tool],
            # tool_choice で強制呼び出し — これがないと Claude が
            # 「ツールなしで答えよう」と判断することがある
            tool_choice={"type": "tool", "name": "extract_kyc_data"},
            messages=[
                {
                    "role": "user",
                    "content": (
                        "以下の本人確認書類のテキストからKYC情報を抽出してください。"
                        "読み取れない項目はnullにし、confidence_scoreで信頼度を示してください。\n\n"
                        f"---\n{document_text}\n---"
                    ),
                }
            ],
        )

        tool_use_block = next(
            block for block in response.content if block.type == "tool_use"
        )
        # Pydantic でバリデーションしながら変換
        return KycData.model_validate(tool_use_block.input)
```

`tool_choice={"type": "tool", "name": "extract_kyc_data"}` がミソだ。これを外すと、Claudeが「テキストで答えればいいか」と判断してしまうケースがある。

## Step 4: 画像スキャンにも対応する（実書類向け）

現実のKYCはOCR済みテキストではなく、スキャン画像が届く。ClaudeはBase64画像を直接読み取れる。

```python
import base64
from pathlib import Path

def load_image_as_base64(image_path: str) -> tuple[str, str]:
    path = Path(image_path)
    media_type_map = {
        ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
        ".png": "image/png", ".webp": "image/webp",
    }
    media_type = media_type_map.get(path.suffix.lower(), "image/jpeg")
    with open(path, "rb") as f:
        data = base64.standard_b64encode(f.read()).decode("utf-8")
    return data, media_type

class KycImageExtractor(KycExtractor):
    def extract_from_image(self, image_path: str) -> KycData:
        image_data, media_type = load_image_as_base64(image_path)

        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=[self.tool],
            tool_choice={"type": "tool", "name": "extract_kyc_data"},
            messages=[
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "image",
                            "source": {
                                "type": "base64",
                                "media_type": media_type,
                                "data": image_data,
                            },
                        },
                        {
                            "type": "text",
                            "text": "この本人確認書類からKYC情報を抽出してください。",
                        },
                    ],
                }
            ],
        )

        tool_use_block = next(
            block for block in response.content if block.type == "tool_use"
        )
        return KycData.model_validate(tool_use_block.input)
```

## Step 5: バッチ処理と信頼度ルーティング

実務では複数書類を一括処理し、信頼度が低いものだけ人間レビューに回す。

```python
from dataclasses import dataclass

CONFIDENCE_THRESHOLD = 0.85

@dataclass
class ExtractionResult:
    source_id: str
    data: KycData | None
    error: str | None = None

def batch_extract(
    extractor: KycExtractor,
    documents: list[tuple[str, str]],  # [(source_id, text), ...]
) -> tuple[list[ExtractionResult], list[ExtractionResult]]:
    """Returns: (auto_approved, needs_human_review)"""
    auto_approved: list[ExtractionResult] = []
    needs_review: list[ExtractionResult] = []

    for source_id, text in documents:
        try:
            kyc_data = extractor.extract(text)
            result = ExtractionResult(source_id=source_id, data=kyc_data)

            if kyc_data.confidence_score >= CONFIDENCE_THRESHOLD:
                auto_approved.append(result)
            else:
                needs_review.append(result)
        except Exception as e:
            needs_review.append(
                ExtractionResult(source_id=source_id, data=None, error=str(e))
            )

    return auto_approved, needs_review
```

## 動かして確認

```python
if __name__ == "__main__":
    extractor = KycExtractor()

    sample = """
    パスポート
    氏名: 山田 太郎 / YAMADA TARO
    生年月日: 1990年3月15日
    パスポート番号: TK1234567
    有効期限: 2030年3月14日
    国籍: 日本
    """

    result = extractor.extract(sample)
    print(result.model_dump_json(indent=2))
```

```json
{
  "full_name": "山田 太郎",
  "date_of_birth": "1990-03-15",
  "document_type": "passport",
  "document_number": "TK1234567",
  "nationality": "日本",
  "address": null,
  "expiry_date": "2030-03-14",
  "confidence_score": 0.97,
  "extraction_notes": null
}
```

## 実務で詰まりやすい3点

**モデル選択**：画像処理はトークン消費が多い。まず `claude-haiku-4-5-20251001` で試し、精度が足りなければ `claude-sonnet-4-6` に切り替えるのが費用対効果の鉄則だ。テキスト抽出ならHaikuで十分なことも多い。

**個人情報の扱い**：KYC書類には最高レベルの個人情報が含まれる。Anthropicのデータ処理規約とプロンプトキャッシュの有効/無効設定を確認すること。本番では Privacy-first モードや専用エンドポイントを検討する。

**エラーハンドリング**：`next(block for block in ...)` は `tool_use` ブロックが存在しない場合に `StopIteration` を投げる。本番では `next(..., None)` でガードし、`None` の場合は再試行かフォールバックを実装すること。

## まとめ

- `tool_choice={"type": "tool", "name": "..."}` で強制呼び出しするとスキーマ準拠が保証される
- Pydantic の `model_json_schema()` を Tool 定義に直接転用することで型安全な抽出が実現できる
- `confidence_score` を Claude に自己評価させ、自動承認と人間レビューへのルーティングに使う
- 画像はBase64で渡せばOCRなしで書類スキャンに対応できる

KYCは「難しそう」に見えて、コアは **書類→構造化データの変換** にすぎない。このパターンは請求書処理・契約書レビュー・医療書類の解析など、あらゆる「書類を読んで項目を埋める」業務に横展開できる。`anthropics/financial-services` のテンプレートを使う前に、まず自分の手で動かしてみてほしい。
```

記事を出力しました。ポイントは：

- **フック**：「30〜40行で同じ仕組みが実装できる」で引き込む
- **構成**：問題提起（プロンプトだけではダメ）→ Step別実装 → 実務の注意点
- **差別化**：Pydanticスキーマを直接Tool定義に流用する手法、`tool_choice` 強制呼び出しの理由、信頼度によるルーティングの実装まで踏み込んだ

他のKY=1/2候補が「なぜ金融AIが流行るか」「フレームワーク比較」を書いているとすれば、この記事は「手を動かしたい人向け実装ガイド」として住み分けられます。