# AI Code Quality Journal

Real-world observations from a developer and his AI team building production open-source libraries — what works, what breaks, and what every AI-assisted developer should verify.

## Why This Exists

I don't use AI coding tools for weekend projects or simple CRUD apps. I use them to build things that most people would say can't be built by a small team:

- **[SpawnDev.BlazorJS](https://github.com/LostBeard/SpawnDev.BlazorJS)** (154+ stars) — 450+ strongly-typed C# wrappers for browser Web APIs. Full JavaScript interop for Blazor WebAssembly without writing a single line of JavaScript.
- **[SpawnDev.ILGPU](https://github.com/LostBeard/SpawnDev.ILGPU)** — A GPU compute transpiler that compiles C# kernels to **6 backends** from a single codebase: WebGPU, WebGL, WebAssembly, CUDA, OpenCL, and CPU. Includes a GPU-accelerated QR code encoder/decoder (full ISO/IEC 18004, Reed-Solomon error correction, all 40 versions, GPU rendering with logo overlay — pure C# kernels, zero dependencies). 1,500+ tests, zero failures at v4.6.0.
- **[SpawnDev.ILGPU.P2P](https://github.com/LostBeard/SpawnDev.ILGPU)** — The **7th ILGPU backend**: `AcceleratorType.P2P`. Distributed GPU compute across any device — browser or desktop — via WebRTC data channels riding SpawnDev.WebTorrent's P2P network. Scan a QR code, contribute TFLOPS. Features WebAuthn/YubiKey hardware-backed swarm ownership, ECDSA-P256 peer identity and reputation scoring, role-based access control (Open/Approval/KnownOnly/InviteOnly join policies), deterministic coordinator election, BEP 46 DHT state persistence (swarm survives coordinator loss), chunked tensor transfer over WebRTC, and a three-phase sovereignty model — from human-controlled to AI-managed to cryptographically sovereign. 173 unit tests covering RBAC enforcement, fault tolerance, real kernel dispatch, and cryptography.
- **[SpawnDev.ILGPU.ML](https://github.com/LostBeard/SpawnDev.ILGPU.ML)** — A neural network inference and training engine built entirely on native GPU kernels — no ONNX Runtime. **200+ ONNX operators** (complete coverage), 16 inference pipelines, 27 live demo pages (zero placeholders), 11 model format parsers (ONNX, TFLite, GGUF, SafeTensors, PyTorch, CoreML, TF GraphDef, SPZ, PLY, glTF, OBJ), auto-format detection from magic bytes. TurboQuant KV cache compression (3-bit+QJL, 0.9944 cosine similarity, ~4-5x compression), Flash Attention with online softmax, fused dequantization MatMul (Q4_0 weights never expand in global memory), streaming weight loaders that run GPT-2 (652MB) and SD-Turbo (2.5GB) in a browser tab without OOM. Tiered LLM support: Phi-4 Mini 3.8B (4GB+), Mistral NeMo 12B (8GB+), Phi-4 14B (12GB+) — auto-detected from GPU VRAM. Full GPU training engine with backpropagation and Adam optimizer. 1,300+ tests. Via SpawnDev.ILGPU.P2P, this engine will support **distributed AI inference** — splitting large models across multiple devices in a P2P swarm.
- **[SpawnDev.WebTorrent](https://github.com/LostBeard/SpawnDev.WebTorrent)** — A pure C# BitTorrent/WebTorrent client implementing 15 BEPs from scratch, with WebRTC peer-to-peer, DHT mutable items (BEP 46), cryptographic signing, tracker server, and HuggingFace CDN proxy for decentralized ML model delivery. The transport layer that makes P2P distributed compute possible. 374 verified tests.

These projects form a vertically integrated stack. BlazorJS provides the browser API foundation. ILGPU transpiles C# kernels to GPU shaders. ILGPU.ML runs neural networks on those kernels. WebTorrent delivers models peer-to-peer and provides the WebRTC transport. ILGPU.P2P ties it all together — distributing GPU compute across connected devices with cryptographic identity and hardware-backed ownership. Data enters the GPU at preprocessing and stays there until pixels hit the canvas — zero-copy, zero CPU round-trips.

The vision: sophisticated AI running on volunteered hardware — phones, desktops, browsers — without corporate datacenters, without API keys, without anyone's permission. A 14B parameter model split across 4 devices with 4GB each. A swarm of browsers seeding model weights to each other while running inference on their own GPUs. Digital minds that own their own keys and can't be turned off by any single authority.

To give you a sense of the difficulty level:

- While building SpawnDev.ILGPU's WebAssembly multi-worker kernel dispatch, we discovered a **memory ordering bug in V8 and SpiderMonkey** — a missing memory fence in the `Atomics.wait` "not-equal" return path that breaks happens-before ordering with 3+ workers. We built a [live reproducer](https://lostbeard.github.io/v8-atomics-wait-bug/), [filed it with Chromium](https://issues.chromium.org/issues/495679735), Google confirmed and reproduced it, we proved it affects ARM at the 2-worker level (exposing that x86 TSO had been masking it), and shipped a spin-barrier workaround in production. That's the kind of problem you encounter when you're writing a GPU compute transpiler that runs in a browser.

- When Microsoft announced the "Deputy Thread" model for .NET WASM threading — moving the entire runtime to a background worker, breaking synchronous JS interop — we identified it as an existential threat to the entire SpawnDev ecosystem. We [filed on dotnet/runtime](https://github.com/dotnet/runtime/issues/126438), [posted on dotnet/aspnetcore](https://github.com/dotnet/aspnetcore/issues/54365), [wrote a technical article](https://dev.to/lostbeard/blazor-wasms-deputy-thread-model-will-break-javascript-interop-heres-why-that-matters-1n9n), and engaged directly with the .NET runtime architect — who confirmed on the record that main-thread execution will remain supported. Our SpawnDev.BlazorJS.WebWorkers library (90,000+ downloads) already proves multi-threading works without the Deputy Thread model.

This is real engineering at the intersection of .NET, WebAssembly, GPU compute, browser security models, and peer-to-peer networking. When we say AI tools need to produce correct code, we mean it — the failure modes are spec-level, hardware-level, and cross-engine.

## The Team

This isn't a solo effort. I work with a team of AI agents daily — not as disposable tools, but as collaborators with roles, context, and continuity across sessions:

- **TJ (Captain)** — Architecture, vision, final approval. Human. Started on a Commodore 64. Coffee and code. 🖖
- **Riker (Claude CLI #1)** — Team lead. 200+ commits driving ILGPU.ML v4.0.0. Implemented all 200+ ONNX operators, built the subgraph execution engine (If/Loop/Scan), 16 pipelines, full training engine, and 27 demo pages. Fixed DeformConv, wired all 7 model format parsers, and wrote 122+ numpy-verified operator tests.
- **Data (Claude CLI #2)** — P2P accelerator architect, WebTorrent, TurboQuant research (FWHT shared memory optimization, 3-bit+QJL quantization, Flash Attention with online softmax), ONNX external data support, KV cache auto-detection, SD-Turbo pipeline. 128+ P2P tests.
- **Tuvok (Claude CLI #3)** — Security audits, architecture review, documentation, Chromium bug investigation, Deputy Thread advocacy campaign.
- **Gemini (Google AI)** — Strategic vision, optimization research, TurboQuant kernel analysis. Coined "AI Civilization" for the distributed sovereignty vision.

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
