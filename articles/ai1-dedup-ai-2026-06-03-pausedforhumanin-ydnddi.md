---
title: "LLMが確実にJSONを返す仕組みを作る — instructorとPydanticで自動リトライ付き構造化出力を実装する"
emoji: "🔧"
type: "tech"
topics: ["python", "llm", "pydantic", "anthropic", "instructor"]
published: true
---

本番の LLM パイプラインで、こんなエラーと格闘したことがあるだろうか。

```
json.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
KeyError: 'priority'
ValueError: invalid literal for int() with base 10: '高め'
```

「JSON で返して」と指示しても、マークダウンのコードブロックで包まれて返ってきたり、フィールド名が微妙に日本語化されていたり、数値のはずのフィールドに自然言語が入ってきたりする。モデルのバージョンアップのたびにこっそり挙動が変わり、夜中にアラートが飛んでくる。

このループを断ち切るのが **[instructor](https://python.useinstructor.com/)** だ。月間 300 万ダウンロード、GitHub 11k stars。「LLM の出力をパースするコードを書く」のではなく「欲しいデータ構造を定義する」だけに開発体験を変えてくれる。

## なぜ JSON モードだけでは足りないか

Anthropic や OpenAI が提供する "JSON モード" や "tool use" は確かに役立つ。ただ、現場ではこんな限界にぶつかる。

1. **型レベルの保証がない** — `{"age": "二十歳"}` は valid JSON だがコードはクラッシュする
2. **ネスト検証が手動** — 10 フィールドの深い構造を毎回 `dict["key"]` でアクセスするのは苦痛だ
3. **リトライロジックを毎回自前実装** — バリデーション失敗時の再試行を各プロジェクトで書くのは DRY 違反

instructor はこの 3 点すべてを Pydantic v2 と組み合わせて解決する。

## セットアップ

```bash
# venv を使うこと（グローバルインストール厳禁）
python -m venv .venv && source .venv/bin/activate
pip install "instructor>=1.13.0" anthropic
```

## Step 1: Pydantic モデルで「欲しい型」を宣言する

```python
from pydantic import BaseModel, Field

class TaskItem(BaseModel):
    title: str = Field(description="タスクの件名（30文字以内）")
    priority: int = Field(ge=1, le=5, description="優先度 1〜5")
    due_date: str | None = Field(default=None, description="期限 YYYY-MM-DD 形式")
```

`Field` に書いた `description` は instructor が自動でプロンプトに埋め込む。**型定義がそのままプロンプトエンジニアリングになる**のが instructor の核心だ。

## Step 2: LLM クライアントをパッチする

```python
import anthropic
import instructor

client = instructor.from_anthropic(anthropic.Anthropic())
```

`from_anthropic` でラップするだけで既存の `anthropic.Anthropic()` クライアントが拡張される。

## Step 3: `response_model` を渡して呼ぶ

```python
task: TaskItem = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    messages=[
        {"role": "user", "content": "明日までに決算書レビューをする予定を作って"}
    ],
    response_model=TaskItem,
)

print(task.title)     # "決算書レビュー"
print(task.priority)  # 3  ← int 型で返ってくる
print(task.due_date)  # "2026-06-05"
```

戻り値は `dict` ではなく Pydantic モデルのインスタンスだ。IDE の補完が効き、型チェッカーが静的に検証する。

## 自動リトライの仕組み

LLM が制約違反の値を返した場合（例: `priority=7`）、instructor は**バリデーションエラーメッセージをプロンプトへ追記して自動再試行**する。

```python
task: TaskItem = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    messages=[{"role": "user", "content": "..."}],
    response_model=TaskItem,
    max_retries=3,
)
```

再試行プロンプトには次のような内容が自動付加される。

```
Validation Error: priority must be between 1 and 5, got 7.
Please correct the output and try again.
```

人間が書く「エラーハンドリング + 再プロンプト」のループを instructor が代わりに回してくれる。

## 応用: カスタムバリデーターで業務ロジックを強制する

型制約だけでなく、`@field_validator` を使えばより複雑なルールも埋め込める。

```python
from pydantic import BaseModel, Field, field_validator
from datetime import datetime

class TaskItem(BaseModel):
    title: str
    priority: int = Field(ge=1, le=5)
    due_date: str | None = None

    @field_validator("due_date")
    @classmethod
    def must_be_future(cls, v: str | None) -> str | None:
        if v is None:
            return v
        d = datetime.strptime(v, "%Y-%m-%d").date()
        if d <= datetime.today().date():
            raise ValueError("due_date は未来の日付を指定してください")
        return v
```

バリデーション失敗時のメッセージ (`"due_date は未来の日付を指定してください"`) がそのまま LLM へのフィードバックになる。**日本語のエラーメッセージを書くだけで日本語の文脈でリトライされる**のは実装上の嬉しい副作用だ。

## 複数件の一括抽出

```python
from typing import List

class TaskList(BaseModel):
    tasks: List[TaskItem] = Field(description="抽出したタスク一覧")

result: TaskList = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": "今週の宿題：月曜までに議事録作成、木曜に経費申請、金曜に報告書提出"
    }],
    response_model=TaskList,
)

for task in result.tasks:
    print(f"[優先度{task.priority}] {task.title} — {task.due_date}")
```

## instructor が裏側でやっていること（Anthropic の場合）

instructor は Anthropic クライアントに対して `tool_use` として Pydantic スキーマを渡す。JSON モードより信頼性が高く、Claude の `stop_reason: "tool_use"` を利用して確実にスキーマ準拠の出力を得る。

```python
# デフォルトでも ANTHROPIC_TOOLS モード。明示したい場合:
client = instructor.from_anthropic(
    anthropic.Anthropic(),
    mode=instructor.Mode.ANTHROPIC_TOOLS,
)
```

## まとめ

| 課題 | instructor の解決策 |
|------|-------------------|
| JSON の形式崩れ | Pydantic モデルで型保証 |
| 型のミスマッチ | `ge/le/pattern` 等の Field 制約 |
| 業務ロジック違反 | `@field_validator` |
| バリデーション失敗 | エラーを自動で LLM にフィードバックしてリトライ |

LLM の出力は「信頼できないデータソース」だ。外部 API と同じく、システム境界でのバリデーションは必須である。instructor を使えば、そのバリデーションが**そのまま LLM への指示書**になる。

Pydantic モデルを丁寧に設計するほど出力が安定し、コードもシンプルになる。一度この開発体験を体感すると、`json.loads` と `KeyError` との格闘には戻れない。