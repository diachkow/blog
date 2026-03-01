---
title: "Building Reliable AI Pipelines in Production"
date: 2026-02-10
tags: ["ai", "production"]
description: "What I learned running LLM-powered pipelines for 18 months."
readingtime: 8
pinned: true
---

Running AI pipelines in production is nothing like running them in a notebook. The gap between "it works on my machine" and "it works at 3 AM on a Saturday" is enormous. After 18 months of operating LLM-powered systems in production, here is what actually matters.

## The core problem

LLMs are nondeterministic by nature. Same input, different output. Sometimes wildly different. This breaks every assumption traditional software engineering has about testing, validation, and reliability.

You can't write a unit test that says "assert output == expected" because the output will be different every time. You need a fundamentally different approach.

> The hardest part of AI in production isn't the AI. It's everything around it.

## What a production pipeline actually looks like

A typical pipeline in our system follows this rough shape:

```python
class Pipeline:
    def __init__(self, model: str, fallback: str):
        self.primary = LLMClient(model)
        self.fallback = LLMClient(fallback)
        self.validator = OutputValidator()
        self.cache = SemanticCache(ttl=3600)

    async def run(self, input: PipelineInput) -> PipelineOutput:
        cached = await self.cache.get(input)
        if cached and self.validator.check(cached):
            return cached

        for attempt in range(3):
            try:
                result = await self.primary.generate(input)
                if self.validator.check(result):
                    await self.cache.set(input, result)
                    return result
            except RateLimitError:
                await asyncio.sleep(2 ** attempt)
            except ModelError:
                break

        return await self.fallback.generate(input)
```

Nothing fancy. Retry logic, fallback models, validation, caching. The boring stuff that keeps you from getting paged.

## Lessons learned

### Always have a fallback model

Your primary model will go down. It will hit rate limits. It will start returning garbage for no apparent reason. Have a cheaper, faster fallback that produces acceptable (not great) results.

We use a tiered approach:

- **Primary**: GPT-4 class model for quality
- **Fallback**: GPT-3.5 class for availability
- **Emergency**: Rule-based system with no LLM at all

### Validate everything

Every LLM output goes through validation before it touches anything downstream. This means:

- Schema validation (is it valid JSON? does it match the Pydantic model?)
- Content validation (are required fields populated? are values in expected ranges?)
- Safety checks (no PII leakage, no harmful content)

```python
class OutputValidator:
    def check(self, output: PipelineOutput) -> bool:
        try:
            parsed = OutputSchema.model_validate_json(output.raw)
            return (
                parsed.confidence > 0.7
                and len(parsed.result) > 0
                and not self.contains_pii(parsed.result)
            )
        except ValidationError:
            return False
```

### Cache aggressively

LLM calls are slow and expensive. Semantic caching — where you match on meaning rather than exact string equality — cut our costs by roughly 40% and reduced p95 latency from 3.2s to 800ms.

### Structured output is not optional

Asking an LLM to "return JSON" is not a strategy. Use function calling, tool use, or constrained decoding. Parse with Pydantic. Retry on parse failure. This alone eliminated about 60% of our production incidents.

### Monitor the right things

Standard metrics (latency, error rate, throughput) are necessary but not sufficient. You also need:

- **Output quality scores** — run a sample through an eval pipeline
- **Drift detection** — are outputs changing character over time?
- **Cost tracking** — per-pipeline, per-customer token usage
- **Fallback rate** — how often are you hitting the backup model?

## The part nobody talks about

The hardest operational problem isn't technical. It's setting expectations. Stakeholders hear "AI" and expect perfection. What they get is a system that's right 94% of the time, and the remaining 6% can be confidently wrong.

Build human-in-the-loop flows for high-stakes decisions. Make confidence scores visible. Design your UX around uncertainty, not around the happy path.

---

Production AI is mostly just production engineering with an extra layer of nondeterminism. The same principles apply: defense in depth, graceful degradation, observability everywhere. The models are the easy part. Keeping them running is the job.
