---
title: "Building Caresse #1: Hexagonal Architecture for a Multi-Provider AI Pipeline"
subtitle: "Ports, adapters, and a real-world provider registry in .NET 10 that lets us swap LLM and TTS vendors without ever touching the domain — the patterns, the dead ends, the trade-offs."
tags: [dotnet, hexagonal-architecture, clean-architecture, ddd, llm, ai, building-caresse]
series: "Building Caresse"
series_index: 1
reading_time: "9 min"
language: en
medium:
  member_only: true
  publication_target: "Better Programming"
canonical: https://caresse.app
---

# Building Caresse #1: Hexagonal Architecture for a Multi-Provider AI Pipeline

> *This is the first article in **Building Caresse**, a new series taking you inside the architecture of a shipped, bootstrapped AI product — the .NET 10 backend, the Kotlin Multiplatform app, the trade-offs, the dead ends. Every article draws from the production codebase behind [caresse.app](https://caresse.app).*

Most articles about hexagonal architecture stop at the diagram. You get the hexagon, the two kinds of arrows, a vague reference to Alistair Cockburn — and then a toy example with one repository and one controller. That's not architecture, that's a folder rename.

This article is different. It walks through how we structured the real .NET 10 API behind [**Caresse**](https://caresse.app) — an immersive-audio app [live on the Play Store](https://play.google.com/store/apps/details?id=com.flareai.caresse) — that orchestrates **five LLM providers** (OpenAI, Grok, DeepSeek, OpenRouter, Mistral) and **six TTS providers** behind a single internal contract, runs a multi-stage generation pipeline, and ships audio to mobile.

The headline result: **the domain has not been edited a single time when we added or replaced a provider in the last six months.** Not when we swapped one LLM for another on a specific stage. Not when a cheaper provider joined to cut costs on the easy parts. Not when we changed TTS engines for one specific language because another vendor turned out to be better at it.

If you've ever tried to bolt a second LLM into a codebase that "just used OpenAI," you already know what this article is about.

<!--more-->

---

## Why we needed real boundaries

Our API does one thing on paper: turn a structured prompt into an immersive audio file. In practice, it has to:

1. Decide which LLM to call **for each generation stage** (the pipeline has multiple — more on that in a future article)
2. Stream tokens or wait for completion depending on the provider's API style
3. Route each generated line to a TTS provider chosen per voice and language
4. Mix the resulting audio tracks with ambient music
5. Persist the result, charge the user via RevenueCat, handle webhooks

Every one of those steps depends on something external — and every one of those externals **will** change. Providers raise prices, deprecate models, get rate-limited, get acquired. OpenAI's API changes shape twice a year. Voice models that used to be the gold standard get replaced by upstart competitors in months. The cost of a "small" change to an external service ripples through any codebase that doesn't have real boundaries.

If the use case at the center of our domain knows the string "OpenAI" anywhere — in a class name, in a method call, in a config key — we've already lost. That's the test for hexagonal:

> **Can you grep your domain for vendor names and get zero hits?**

For us, the answer is zero. Here's how we got there.

---

## The folder structure that actually matters

Here's the layout we converged on after two refactors:

```
src/
├── Caresse.Domain/
│   ├── Models/                  # Pure C# records, no attributes
│   ├── Ports/
│   │   ├── Driving/             # What the outside world calls INTO us
│   │   └── Driven/              # What WE call OUT to
│   └── UseCases/                # The actual business logic
│
├── Caresse.DrivingAdapters/     # HTTP, gRPC, CLI, webhooks
│   └── Rest/
│
└── Caresse.DrivenAdapters/      # Implementations of the driven ports
    ├── Llm/                     # One adapter per LLM provider
    ├── Tts/                     # One adapter per TTS provider
    ├── Persistence/
    └── Audio/                   # Audio post-processing
```

The naming conventions matter more than they look:

- **`I*Port`** for any interface defined inside the domain. Never `I*Service`, never `I*Repository`. The name reminds you that the domain *owns* the contract — adapters implement it, not the other way around.
- **`*UseCase`** for the orchestrators. One use case = one user-visible operation. If your "use case" is doing two things, split it.
- **`*RestAdapter`** (not `*Controller`) for the HTTP layer. ASP.NET happens to provide controllers, but our code shouldn't pretend HTTP is the only entry point — tomorrow it could be a gRPC adapter, a queue handler, or a CLI.
- **`*Adapter`** suffix for *every* driven implementation. When you see something named `*PromptAdapter`, you know immediately it's an outbound implementation talking to the outside world, not domain logic.

These aren't cosmetic rules. They're how a new contributor finds the right file in under a minute — and how a senior reviewer catches "this shouldn't live here" in a glance at the PR diff.

---

## What we tried first (and abandoned)

The first version of this API was the controller-knows-everything style. One controller, ~600 lines, that did the LLM call, the TTS call, the EF Core save, and the HTTP response in sequence. It worked. Until it didn't.

The first time we needed to swap LLM vendors for one specific stage of the pipeline, the diff touched the controller, the config, three test files, and we ended up with a feature flag branch that everyone hated within a week.

We then tried the classic "service layer" — extract `OpenAiService`, `ElevenLabsService`, inject them into the controller. Better, but still wrong: the controller now knew there were *two* providers, and adding a third meant editing the controller again. The dependency was still in the wrong direction.

The third attempt was hexagonal done badly: we made interfaces, but they lived in the same project as their implementations, and the use cases imported the implementation namespace "just for the DTOs." Every shortcut compounded. Within two months we had circular references and `[InternalsVisibleTo]` attributes patching over leaky abstractions.

The current architecture works because we drew **three hard lines**:

1. The domain project has **zero NuGet dependencies** except `System.*`. No EF Core, no HttpClient, no Serilog.
2. Domain ports define their **own DTOs**. Adapters translate between port DTOs and vendor DTOs.
3. **Compile-time enforcement**: a CI step runs `dotnet list package --include-transitive` on the Domain project and fails if anything outside an allowlist appears.

If those three lines aren't enforced, "hexagonal architecture" is just folder theater.

---

## The provider registry pattern

This is the piece that nobody publishes, so here's the shape — without giving away the whole thing.

The domain knows about an enum:

```csharp
public enum AiProvider { OpenAi, Grok, DeepSeek, OpenRouter, Mistral }
```

And a port (one method, no vendor leakage):

```csharp
public interface IPromptExecutorPort
{
    AiProvider Provider { get; }
    Task<PromptResult> ExecuteAsync(Prompt prompt, CancellationToken ct);
}
```

Each adapter implements the port and declares which provider it handles. There is **no** base class, no shared HTTP helper, no "smart" abstraction layer — every vendor has its own quirks and we let each adapter own them. Trying to factor out commonality between LLM HTTP clients was one of our worst time sinks; we tore it out.

The wiring is where the magic — really, the lack of magic — happens. In `Program.cs`:

```csharp
builder.Services.AddSingleton<IPromptExecutorPort, OpenAiPromptAdapter>();
builder.Services.AddSingleton<IPromptExecutorPort, GrokPromptAdapter>();
// ... one line per provider

builder.Services.AddSingleton<IReadOnlyDictionary<AiProvider, IPromptExecutorPort>>(sp =>
    sp.GetServices<IPromptExecutorPort>().ToDictionary(a => a.Provider));
```

That dictionary is what the use case receives. Selecting a provider becomes a single lookup:

```csharp
var executor = llms[stage.Provider];
var result = await executor.ExecuteAsync(stage.ToPrompt(context), ct);
```

The same pattern applies to TTS — different enum, different port, same registry shape. Same goes for audio post-processing: one port, one adapter, the domain never sees the implementation.

Adding a new provider is exactly three things:

1. Add a value to the enum
2. Create a new `*PromptAdapter` class implementing the port
3. Register it in `Program.cs`

The use case, the domain models, and the controllers are all unchanged. The compiler tells you if you missed a case somewhere — and there shouldn't be any, because the dictionary is exhaustive by construction.

When we added a new vendor mid-pipeline to cut costs on the cheaper stages of [Caresse](https://caresse.app)'s generator, the diff was under 50 lines. Zero in the domain.

---

## Testing the adapters in isolation

This architecture only pays off if you can also test each piece in isolation. Two layers, two strategies:

- **Adapter tests** verify the *protocol* with one vendor — using a fake `HttpMessageHandler` to simulate vendor responses, no network involved. One adapter at a time, one vendor at a time, deterministic.
- **Use case tests** substitute the **entire dictionary** of ports. The orchestration logic is exercised with no vendor at all, just `Substitute.For<IPromptExecutorPort>()` instances asserting "this stage called this executor."

The combination matters: adapter tests cover the integration surface, use case tests cover the business rules. Together they replace a brittle end-to-end suite that used to take 12 minutes and pass 70% of the time.

```csharp
// shape only — use case test
await useCase.ExecuteAsync(request, ct);

await openAi.Received(1).ExecuteAsync(Arg.Any<Prompt>(), Arg.Any<CancellationToken>());
await grok.Received(1).ExecuteAsync(Arg.Any<Prompt>(), Arg.Any<CancellationToken>());
```

Zero HTTP, zero database, runs in milliseconds. Every PR runs the full suite.

---

## What it costs you

I'd be lying if I said the architecture is free.

- **More files.** You will have three or four files where the controller-only version had one. New devs will ask "why" for the first week.
- **More indirection in stack traces.** A call goes RestAdapter → UseCase → Port → Adapter. That's four frames where there used to be one.
- **The registry pattern is opaque to newcomers.** "Where does this port come from?" is a real question, and the answer (`Program.cs` registers all of them, the use case receives a dictionary) is not obvious from any single file.
- **You will be tempted to leak.** "Just this once" — a `JsonPropertyName` attribute in a domain model because it's easier than mapping. Don't. The day you say yes, the architecture starts dying.

We accept these costs because [Caresse](https://caresse.app) has crossed the threshold where they pay back: we genuinely swap providers per stage, we genuinely test each adapter in isolation, and we genuinely run the same use cases from HTTP, webhooks, and background jobs. If you're building a CRUD app with one database and one external API, hexagonal is probably overkill — and that's a fine conclusion to come away with.

---

## The grep test

Try it on your own code:

```bash
grep -rE "OpenAI|ElevenLabs|Cartesia|Grok|DeepSeek" src/YourApp.Domain/
```

If you get zero results, your domain is genuinely decoupled. If you get a hit, that's your next refactor.

For us, the answer is zero — and that's the only metric that matters when the next provider war breaks out and we need to switch in an afternoon.

---

## Coming next in *Building Caresse*

The registry pattern explains *how* we route to providers. **Building Caresse #2** explains *why we route at all*: the multi-stage generation pipeline at the heart of [Caresse](https://caresse.app), where one user prompt becomes a full multi-voice scene, with a different model per stage and a self-critique step that improves the output before it ever reaches TTS.

It also covers the hardest lesson we learned: **one big prompt fails where several smaller, chained ones succeed**, and the failure mode looks like quality regression rather than an error in the logs. We had to build per-stage evaluation to catch it.

If this article was useful, follow me on Medium so you don't miss #2 — it's where this architecture starts paying for itself.

---

## See it running

This isn't a thought experiment — it's the backend of a shipped product:

- 🌐 **[caresse.app](https://caresse.app)** — overview, screenshots, supported languages (FR / EN / ES / PT)
- 📱 **[Caresse on Google Play](https://play.google.com/store/apps/details?id=com.flareai.caresse)** — live, iOS coming

*Questions, pushback, or want me to dig into a specific layer? Reply on Medium or reach out via [caresse.app](https://caresse.app) — I read every message and the next article topic often comes from reader questions.*

---

**Technical level:** 🟩🟩🟦 Confirmed — assumes you're comfortable with C#, .NET DI, and have at least heard of ports & adapters. No prior DDD theory required.

**Series:** Building Caresse — inside the stack of a bootstrapped AI product.
**Next up — #2:** the multi-stage LLM pipeline that turns one prompt into an immersive scene.
