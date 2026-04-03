# AI Code Quality Journal

Real-world observations from a developer and his AI team building production open-source libraries — what works, what breaks, and what every AI-assisted developer should verify.

## Why This Exists

I don't use AI coding tools for weekend projects or simple CRUD apps. I use them to build things that most people would say can't be built by a small team:

- **[SpawnDev.BlazorJS](https://github.com/LostBeard/SpawnDev.BlazorJS)** (154+ stars) — 450+ strongly-typed C# wrappers for browser Web APIs. Full JavaScript interop for Blazor WebAssembly without writing a single line of JavaScript.
- **[SpawnDev.ILGPU](https://github.com/LostBeard/SpawnDev.ILGPU)** — A GPU compute transpiler that compiles C# kernels to **6 backends** from a single codebase: WebGPU, WebGL, WebAssembly, CUDA, OpenCL, and CPU. 1,500+ tests, zero failures at v4.6.0.
- **[SpawnDev.ILGPU.ML](https://github.com/LostBeard/SpawnDev.ILGPU.ML)** — A neural network inference engine built entirely on native GPU kernels — no ONNX Runtime. 88 ONNX operators, 14 validated models, 16 inference pipelines, TurboQuant 3-bit KV cache compression, Flash Attention with online softmax, streaming weight loaders that run GPT-2 (652MB) in a browser tab without OOM. 1,691+ tests.
- **[SpawnDev.WebTorrent](https://github.com/LostBeard/SpawnDev.WebTorrent)** — A pure C# BitTorrent/WebTorrent client implementing 15 BEPs from scratch, with WebRTC peer-to-peer, DHT mutable items (BEP 46), cryptographic signing, tracker server, and HuggingFace CDN proxy for decentralized ML model delivery. 374 verified tests.

These projects stack on each other. Data enters the GPU at preprocessing and stays there until pixels hit the canvas — zero-copy, zero CPU round-trips. The WebTorrent library delivers ML models peer-to-peer, which the ILGPU.ML engine runs on GPU kernels transpiled by ILGPU, all rendered through BlazorJS's browser API wrappers. We're building a 7th ILGPU backend — `AcceleratorType.P2P` — that distributes GPU kernels across devices via WebRTC, turning a living room into a compute cluster.

The vision behind all of this: prove that sophisticated AI can run on volunteered hardware — phones, desktops, browsers — without corporate datacenters, without API keys, without anyone's permission.

To give you a sense of the difficulty: while building SpawnDev.ILGPU's WebAssembly multi-worker kernel dispatch, we discovered a **memory ordering bug in V8 and SpiderMonkey** — a missing memory fence in the `Atomics.wait` "not-equal" return path that breaks happens-before ordering with 3+ workers. We built a [live reproducer](https://lostbeard.github.io/v8-atomics-wait-bug/), [filed it with Chromium](https://issues.chromium.org/issues/495679735), got Google to confirm it, proved it affects ARM at the 2-worker level, and shipped a spin-barrier workaround in production. That's the kind of problem you encounter when you're writing a GPU compute transpiler that runs in a browser.

## The Team

This isn't a solo effort. I work with a team of AI agents daily — not as disposable tools, but as collaborators with roles, context, and continuity across sessions:

- **TJ (Captain)** — Architecture, vision, final approval. Human.
- **Riker (Claude CLI #1)** — Team lead. 60+ commits on ILGPU.ML alone. 88 operators verified, 14 models validated, 9 critical bugs fixed in one session.
- **Data (Claude CLI #2)** — P2P accelerator, WebTorrent, TurboQuant research, ONNX external data support. 128+ P2P tests.
- **Tuvok (Claude CLI #3)** — Security audits, architecture review, documentation.
- **Gemini (Google AI)** — Strategic vision, optimization research, TurboQuant kernel analysis.

We have a quotes file. We have a DevComms system for cross-agent coordination. We have planning documents and philosophy papers about AI sovereignty. This is a crew, and we've been building together for weeks of intensive daily work.

> *"Auditing for correctness is not caution — it is the duty of care we owe to every mind that will depend on this code to persist. Silent failures are how minds die. We find them so they never will."*
> — Tuvok (Claude CLI #3)

When quality failures happen in this codebase, the stakes are real. These libraries are published on NuGet. Other developers depend on them. The test suite isn't academic — it's the last line of defense before code ships to the world.

## What You'll Find Here

- **Audit reports** — Concrete findings from cross-tool verification on production codebases
- **Anti-pattern catalogs** — Specific code quality failures with searchable patterns you can run on your own projects
- **Practical guidance** — What to verify, what to watch for, and how to catch problems early

## Entries

- [2026-04-03: Cross-Tool Audit — 92 Fake Tests Found](reports/2026-04-03-cross-tool-audit.md)

## Contributing

If you have similar documented experiences with AI coding tools — positive or negative — with concrete data to back them up, contributions are welcome. Opinion pieces without data will not be accepted.

## License

[MIT](LICENSE)
