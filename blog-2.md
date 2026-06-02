---
title: "Building Caresse #2: Orchestrating a Multi-Phase LLM Pipeline"
subtitle: "One user prompt, 8 sequential stages, a different model per stage, and a self-critique step that improves the output before it ever reaches TTS — why one big prompt fails where several smaller, chained ones succeed."
tags: [llm, ai, dotnet, prompt-engineering, building-caresse]
series: "Building Caresse"
series_index: 2
reading_time: "9 min"
language: en
medium:
  member_only: true  # PREMIUM — second article, audience built by #1
  publication_target: "Better Programming"
canonical: https://caresse.app
---

# Building Caresse #2: Orchestrating a Multi-Phase LLM Pipeline

> *This is the second article in **Building Caresse**, a series taking you inside the architecture of a shipped, bootstrapped AI product — the .NET 10 backend, the Kotlin Multiplatform app, the trade-offs, the dead ends. Every article draws from the production codebase behind [caresse.app](https://caresse.app).*
>
> *If you missed #1 on hexagonal architecture and the provider registry pattern, start there — this article builds directly on it.*

Most LLM production articles show you a single prompt and call it done. You get a `chat/completions` call, a well-engineered system prompt, maybe a retry loop — and that's the "pipeline."

That's not what we have. [Caresse](https://caresse.app) is an immersive-audio app that takes a short user configuration and returns a fully generated multi-voice audio scene. The gap between those two things is **8 sequential LLM stages**, each with its own model selection, its own prompt builder, and its own failure mode. A single big prompt can't do this. We know — we tried. This article explains exactly why, and how we replaced it with an architecture that actually works in production.

<!--more-->

---

## Why one big prompt fails

Before getting to the solution, the failure mode is worth understanding — because it's subtle. When we first attempted to generate a full scene with one large prompt, the output wasn't obviously broken. It was *good enough to ship*, and that was the problem.

The failure looked like quality regression rather than an error. The model would follow the first half of the instructions and quietly deprioritize constraints buried further down. Structural decisions made early would drift by the end of a long generation. Output length bled in both directions — sometimes too short, sometimes exceeding TTS limits — always unpredictably.

None of this threw an exception. All of it degraded the product.

The second failure was **cost**. A monolithic prompt that requires a premium model runs that model for the entire duration. Splitting the work lets you run cheap, fast models for the mechanical stages and reserve the expensive ones for the steps where quality actually matters.

The third failure was **observability**. A single prompt producing a single blob of text gives you almost nothing to measure. Which part got worse when you updated the system prompt? You can't tell — you have one output, no seams, no intermediate states to inspect.

Eight smaller stages solve all three.

---

## The pipeline shape

Here's the sequence as it runs in production:

```
Bootstrap → Framing → NarrativeA → NarrativeB
          → NarrativeC → Refiner → Metadata → TextToSpeech
```

The names describe roles, not implementations. What each stage *does* — the prompts, the constraints, the narrative logic — is where the product lives, and that's not what this article is about. What this article covers is the **infrastructure that makes the pipeline maintainable**: how stages are wired together, how context flows between them, how providers are selected, and where things break.

Each stage implements a common interface:

```csharp
public interface IGenerationStage
{
    GenerationStage Stage { get; }
    AiProvider Provider { get; }
    Task<StageResult> ExecuteAsync(GenerationContext context, CancellationToken ct);
}
```

`GenerationContext` is the pipeline's shared state — it carries the user configuration, the outputs of prior stages, and metadata accumulated along the way. No stage reaches back into the database. No stage knows about HTTP. The context moves forward; each stage appends its result; the next stage reads what it needs.

Two structural observations worth calling out:

**The Bootstrap stage generates structure, not content.** It produces the scaffolding — names, voice assignments, duration targets — that every downstream stage must stay consistent with. Front-loading structural decisions is the single most important thing you can do in a chained pipeline. Without it, each stage invents its own assumptions and inconsistency is guaranteed.

**One stage runs after the narrative is complete.** It takes the assembled output and improves it — not the whole thing, just the weakest part. Targeted self-critique is cheap, fast, and surprisingly effective. Full-text rewrites at this stage are expensive, slow, and often make things *worse* — the model loses context and introduces new inconsistencies. One targeted pass is the right scope.

---

## Provider selection per stage

Every stage carries an `AiProvider` property. The value comes from configuration — not hardcoded.

```json
{
  "Pipeline": {
    "Stages": {
      "Bootstrap":  { "Provider": "OpenAi",   "Model": "gpt-4o-mini" },
      "Framing":    { "Provider": "Grok",     "Model": "grok-2" },
      "NarrativeC": { "Provider": "OpenAi",   "Model": "gpt-4o" },
      "Refiner":    { "Provider": "DeepSeek", "Model": "deepseek-chat" },
      "Metadata":   { "Provider": "OpenAi",   "Model": "gpt-4o-mini" }
    }
  }
}
```

Changing which model runs a stage is a config change, not a code change. This matters more than it sounds: we've updated this map 15–20 times in the first six months. A new model launches, we test it in UAT on isolated stages, and if it's better or cheaper we promote it to production without touching application code.

The wiring uses the provider registry from #1 directly:

```csharp
var executor = _llmProviders[stage.Provider];
var result = await executor.ExecuteAsync(stage.ToPrompt(context), ct);
context = context.WithStageResult(stage.Stage, result);
```

No conditionals, no `if grok then else if openai`. The registry handles dispatch; the stage handles its own prompt construction; the orchestrator chains them and moves on.

---

## Prompt builders: the Strategy pattern

Each stage has a dedicated prompt builder. The builders are the only layer that sees user-facing configuration — scenario type, language, intensity. The orchestrator never touches these values; it calls `stage.ToPrompt(context)` and the builder handles the rest.

```csharp
public interface IPromptBuilder<TStage>
{
    Prompt Build(GenerationContext context);
}
```

This containment is the reason prompt iteration doesn't break the pipeline. Every change to tone, language variants, or A/B variants lives inside one builder, isolated from orchestration logic and from every other stage.

We have unit test coverage on every prompt builder. The tests are simple: given this `GenerationContext`, assert the `Prompt` output contains the required directives and doesn't contain the forbidden ones. Boring tests. They've caught three regressions where a refactor silently dropped a constraint — regressions that would have been invisible without per-stage observability.

---

## Context chaining and the compression problem

The hardest engineering problem in a multi-phase pipeline is context management. Every stage needs *some* history; no stage needs *all* of it.

Our `GenerationContext` carries two kinds of history:

1. **Structural outputs** — the Bootstrap result travels in full to every downstream stage. Names, assignments, structural decisions: these never get compressed.
2. **Stage summaries** — each Narrative stage produces both its full output and a short summary. Downstream stages receive the summaries, not the full text.

This design came from a frustrating discovery: passing full prior outputs made the model *more* repetitive, not more consistent. Summaries force the model to reason about narrative state rather than echo prior text. The full outputs live in `GenerationContext` in memory — assembled later for TTS — but they don't go back into the LLM's context window.

---

## Error handling between stages

A pipeline that fails silently is worse than one that fails loudly.

- Each stage has a **retry budget**: 2 attempts with exponential backoff before escalating.
- A stage failure aborts the pipeline and returns a typed error to the use case.
- We never partially commit — either the full pipeline succeeds or nothing is stored.

The failure mode we spent the most time on wasn't HTTP errors or rate limits. It was **constraint violations**: output that's syntactically valid but semantically wrong — voice assignments swapped, sentence budgets blown, language drift mid-scene. These don't throw exceptions. Per-stage validators catch them after every stage completes:

```csharp
var validationResult = stageValidator.Validate(result, context);
if (!validationResult.IsValid)
    throw new StageConstraintViolationException(stage.Stage, validationResult.Errors);
```

About 3% of production generations hit a constraint violation and retry. Without the validators, those 3% would have shipped bad audio.

---

## LLM as judge: a dedicated model for quality evaluation

Rule-based validators catch structural violations — but they can't tell you whether the output is actually *good*. For that, we use a separate LLM call whose only job is to evaluate the output of another LLM call.

The pattern is sometimes called "LLM as judge" or "AI as judge." The idea: instead of trying to define "quality" in code, you describe it in natural language to a model and ask it to score the output. The evaluator runs on a different model than the generator — deliberately. Using the same model to evaluate its own output is like asking someone to proofread their own writing: the same blind spots apply.

In practice, the judge sits at two points in our pipeline:

1. **After the last narrative stage** — before the Improver runs. The judge scores the assembled narrative on a small set of dimensions (not published here — that's the product). If the score is below threshold, we skip the Improver and retry the generation from scratch. There's no point improving a fundamentally weak output; it's cheaper to regenerate.

2. **After the Improver** — a lightweight second pass. Did the improvement actually improve things, or did it introduce a regression? This second judge is cheaper, faster, and runs on a smaller model. Its only job is to compare before and after.

The interface looks like any other stage:

```csharp
public interface IJudgePort
{
    Task<JudgeResult> EvaluateAsync(JudgeRequest request, CancellationToken ct);
}

public record JudgeResult(float Score, bool PassesThreshold, string Reasoning);
```

The `Reasoning` field is the part we didn't expect to use — and ended up relying on heavily. When a generation fails the judge, the reasoning goes into the retry context. The regenerating stage receives a short note: *"the previous attempt had this specific problem."* The retry is guided, not blind. That small feedback loop cut our retry rate in half.

A few things we learned the hard way:

**Don't ask the judge to do too many things at once.** A single evaluation prompt that scores tone, consistency, pacing, and language quality simultaneously produces mediocre scores on all of them. Separate concerns — even if it means two judge calls.

**The judge needs its own retry budget.** LLMs occasionally return malformed evaluation responses (wrong JSON shape, score out of range). Treating a malformed judge response as a failed generation sends users into retry loops that have nothing to do with content quality.

**Cost matters.** The judge adds latency and cost to every generation. We run it on the cheapest model that produces reliable structured output — the goal is a consistent signal, not a perfect literary critic.

The result: a pipeline where "quality" is not a vibe you check manually after deploy. It's a number, per generation, logged, queryable, and improvable over time.

---

## The real value: 8 measurable checkpoints

If you're building an LLM feature and wondering whether to split it into stages, here's the forcing question we use:

> **What would you measure to know if your prompt got worse?**

If the answer is "the full output quality," you have no seams. Debugging means reading the whole output, forming a subjective opinion, and adjusting blindly. That works for prototypes.

Our 8 stages give us 8 checkpoints. When something degrades, we isolate it to a stage in minutes. When we update a prompt builder, we know exactly what changed and where. When we switch a provider, we measure the delta on one stage in isolation.

That's the actual return on investment — not the architecture diagram, not the clean interfaces. The ability to iterate on a live, revenue-generating product without guessing.

---

## Coming next in *Building Caresse*

**Building Caresse #3** covers the testing strategy that keeps this pipeline honest: Testcontainers for a real PostgreSQL instance, Azurite for Azure Blob Storage, a custom `TestAuthHandler` that replaces Firebase JWT in tests, and WireMock.Net for the RevenueCat API — all running in a single xUnit suite, no mocks, no flakiness, no dependency on production infrastructure.

If this was useful, follow me on Medium to catch #3 — the testing article tends to be the one engineers share with their teams.

---

## See it running

This isn't a thought experiment — it's the backend of a shipped product:

- 🌐 **[caresse.app](https://caresse.app)** — overview, screenshots, supported languages (FR / EN / ES / PT)
- 📱 **[Caresse on Google Play](https://play.google.com/store/apps/details?id=com.flareai.caresse)** — live, iOS coming

*Questions, pushback, or want me to dig into a specific layer? Reply on Medium or reach out via [caresse.app](https://caresse.app) — the next article topic often comes from reader questions.*

---

**Technical level:** 🟩🟩🟦 Confirmed — assumes comfort with C# and async patterns. LLM experience helpful but not required.

**Series:** Building Caresse — inside the stack of a bootstrapped AI product.
**Previous — #1:** hexagonal architecture and the multi-provider registry.
**Next up — #3:** integration testing .NET without touching production.
