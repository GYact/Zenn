---
title: "マルチモデルルーティング入門：GPT・Claude・Geminiを使い分ける実装パターン"
emoji: "🔀"
type: "tech"
topics: ["llm", "openai", "anthropic", "python", "ai"]
published: true
---

## 「どのモデルを使えばいいか」という問いは、もう古い

2026年現在、LLMの選択肢は爆発的に増えた。GPT-5.5、Claude Opus 4.7、Gemini 2.5 Pro……それぞれに得意・不得意があり、コストも全然違う。

「とりあえずGPT-4に投げとけ」という時代は終わった。現場で求められるのは、タスクに応じて最適なモデルを自動で選ぶ**マルチモデルルーティング**だ。

この記事では、実際に手を動かして動くルーティング層を作りながら、その設計パターンを解説する。

---

## マルチモデルルーティングとは何か

一言でいうと「タスクの特性に応じてLLMを動的に切り替える仕組み」だ。

例えば：
- **単純なFAQ応答** → 安いモデル（gpt-4o-mini など）で十分
- **複雑なコード生成** → Claude Opus 4.7 や GPT-5.5 の出番
- **長文の要約・翻訳** → Gemini 2.5 Pro（コンテキストウィンドウが広い）

これを人間が毎回判断するのではなく、自動化することがゴールだ。

---

## 実装：シンプルなルーターを作る

まず依存関係を整理する。

```bash
pip install openai anthropic google-generativeai tiktoken
```

### 1. 統一インターフェースの定義

各プロバイダのSDKはAPIが微妙に違う。まずラッパーで吸収する。

```python
# llm_client.py
from dataclasses import dataclass
from typing import Literal
import openai
import anthropic
import google.generativeai as genai

ModelProvider = Literal["openai", "anthropic", "google"]

@dataclass
class LLMResponse:
    content: str
    model: str
    provider: ModelProvider
    input_tokens: int
    output_tokens: int

class UnifiedLLMClient:
    def __init__(self):
        self.openai = openai.OpenAI()
        self.anthropic = anthropic.Anthropic()
        genai.configure()  # GOOGLE_API_KEY を環境変数から読む

    def call(
        self,
        provider: ModelProvider,
        model: str,
        prompt: str,
        system: str = "",
        max_tokens: int = 1024,
    ) -> LLMResponse:
        if provider == "openai":
            return self._call_openai(model, prompt, system, max_tokens)
        elif provider == "anthropic":
            return self._call_anthropic(model, prompt, system, max_tokens)
        elif provider == "google":
            return self._call_google(model, prompt, system, max_tokens)
        raise ValueError(f"Unknown provider: {provider}")

    def _call_openai(self, model, prompt, system, max_tokens):
        messages = []
        if system:
            messages.append({"role": "system", "content": system})
        messages.append({"role": "user", "content": prompt})

        resp = self.openai.chat.completions.create(
            model=model, messages=messages, max_tokens=max_tokens
        )
        return LLMResponse(
            content=resp.choices[0].message.content,
            model=model,
            provider="openai",
            input_tokens=resp.usage.prompt_tokens,
            output_tokens=resp.usage.completion_tokens,
        )

    def _call_anthropic(self, model, prompt, system, max_tokens):
        resp = self.anthropic.messages.create(
            model=model,
            max_tokens=max_tokens,
            system=system or "You are a helpful assistant.",
            messages=[{"role": "user", "content": prompt}],
        )
        return LLMResponse(
            content=resp.content[0].text,
            model=model,
            provider="anthropic",
            input_tokens=resp.usage.input_tokens,
            output_tokens=resp.usage.output_tokens,
        )

    def _call_google(self, model, prompt, system, max_tokens):
        gmodel = genai.GenerativeModel(
            model_name=model,
            system_instruction=system or None,
        )
        resp = gmodel.generate_content(
            prompt,
            generation_config=genai.types.GenerationConfig(max_output_tokens=max_tokens),
        )
        # Google APIはトークン数の取得方法がやや異なる
        usage = resp.usage_metadata
        return LLMResponse(
            content=resp.text,
            model=model,
            provider="google",
            input_tokens=usage.prompt_token_count,
            output_tokens=usage.candidates_token_count,
        )
```

### 2. ルーティングロジックの実装

タスクの複雑さをヒューリスティックで判定するルーターを作る。プロダクションでは機械学習ベースにするのがベストだが、まずは動くものを優先する。

```python
# router.py
import tiktoken
from llm_client import UnifiedLLMClient, LLMResponse

# コスト定義（$/1M tokens、2026年5月時点の概算）
MODEL_REGISTRY = {
    "fast": {
        "provider": "openai",
        "model": "gpt-4o-mini",
        "input_cost": 0.15,
        "output_cost": 0.60,
        "max_context": 128_000,
    },
    "smart": {
        "provider": "anthropic",
        "model": "claude-opus-4-7",
        "input_cost": 15.0,
        "output_cost": 75.0,
        "max_context": 200_000,
    },
    "long_context": {
        "provider": "google",
        "model": "gemini-2.5-pro",
        "input_cost": 1.25,
        "output_cost": 5.00,
        "max_context": 1_000_000,
    },
}

def count_tokens(text: str) -> int:
    enc = tiktoken.get_encoding("cl100k_base")
    return len(enc.encode(text))

def select_model(prompt: str, task_hint: str = "") -> dict:
    token_count = count_tokens(prompt)
    prompt_lower = prompt.lower()
    hint_lower = task_hint.lower()

    # 長文 → Gemini
    if token_count > 50_000:
        return MODEL_REGISTRY["long_context"]

    # コード生成・推論系 → Claude
    code_signals = ["code", "implement", "debug", "algorithm", "関数", "実装", "コード"]
    if any(s in prompt_lower or s in hint_lower for s in code_signals):
        return MODEL_REGISTRY["smart"]

    # 単純なQ&A → 安いモデル
    return MODEL_REGISTRY["fast"]


class MultiModelRouter:
    def __init__(self):
        self.client = UnifiedLLMClient()
        self.total_cost = 0.0

    def route(self, prompt: str, task_hint: str = "", system: str = "") -> LLMResponse:
        config = select_model(prompt, task_hint)
        print(f"[Router] Using: {config['provider']}/{config['model']}")

        response = self.client.call(
            provider=config["provider"],
            model=config["model"],
            prompt=prompt,
            system=system,
        )

        # コスト計算
        cost = (
            response.input_tokens / 1_000_000 * config["input_cost"]
            + response.output_tokens / 1_000_000 * config["output_cost"]
        )
        self.total_cost += cost
        print(f"[Router] Cost: ${cost:.6f} | Total: ${self.total_cost:.4f}")

        return response
```

### 3. 動作確認

```python
# main.py
from router import MultiModelRouter

router = MultiModelRouter()

# シンプルな質問 → gpt-4o-mini にルーティングされる
resp1 = router.route("Pythonのリスト内包表記とは何ですか？")
print(resp1.content[:200])

# コード生成 → claude-opus-4-7 にルーティングされる
resp2 = router.route(
    "二分探索木を実装してください",
    task_hint="code"
)
print(resp2.content[:200])

# 長文要約 → gemini-2.5-pro にルーティングされる（ダミーで大きなテキストを渡す）
long_text = "概要: " + "これは非常に長いドキュメントです。 " * 3000
resp3 = router.route(long_text)
print(resp3.content[:200])
```

実行すると以下のようなログが出る：

```
[Router] Using: openai/gpt-4o-mini
[Router] Cost: $0.000023 | Total: $0.0000
[Router] Using: anthropic/claude-opus-4-7
[Router] Cost: $0.001240 | Total: $0.0013
[Router] Using: google/gemini-2.5-pro
[Router] Cost: $0.008750 | Total: $0.0100
```

---

## 設計上のポイント

### フォールバックを忘れずに

本番では必ず `try/except` でフォールバックを実装すること。あるプロバイダが落ちていても、別モデルで継続できるようにする。

```python
def route_with_fallback(self, prompt: str, **kwargs) -> LLMResponse:
    primary = select_model(prompt)
    fallback_chain = ["fast", "smart", "long_context"]

    for tier in fallback_chain:
        config = MODEL_REGISTRY[tier]
        try:
            return self.client.call(provider=config["provider"], model=config["model"], prompt=prompt)
        except Exception as e:
            print(f"[Router] Failed {config['model']}: {e}, trying next...")
    raise RuntimeError("All models failed")
```

### キャッシュで重複コストを削減

同じプロンプトへの応答はキャッシュすることでコストを大幅削減できる。Redisを使った実装が現実的だ。

```python
import hashlib, json, redis

cache = redis.Redis()
TTL = 3600  # 1時間キャッシュ

def get_cache_key(prompt: str, model: str) -> str:
    return hashlib.sha256(f"{model}:{prompt}".encode()).hexdigest()

def cached_call(self, config, prompt, system=""):
    key = get_cache_key(prompt, config["model"])
    cached = cache.get(key)
    if cached:
        print("[Router] Cache hit!")
        return LLMResponse(**json.loads(cached))

    response = self.client.call(
        provider=config["provider"],
        model=config["model"],
        prompt=prompt,
        system=system,
    )
    cache.setex(key, TTL, json.dumps(response.__dict__))
    return response
```

---

## まとめ

マルチモデルルーティングの核心は3つだ：

1. **統一インターフェース**：プロバイダ差異をラップして、上位層から透過的に扱えるようにする
2. **ルーティングロジック**：トークン数・タスクの性質・コストを考慮した選択基準を持つ
3. **フォールバックとキャッシュ**：可用性とコスト効率の両立

今回のヒューリスティックベースのルーターは出発点に過ぎない。次のステップとして、**応答品質の評価結果をフィードバックしてルーター自体を学習させる**アプローチが現場では有効だ。「モデルを1つ選ぶ」のではなく「モデルを使いこなすインフラを作る」という思考転換が、2026年のLLMエンジニアに求められている。