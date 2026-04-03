# Cross-Tool Audit: Same Model, Different Results

**Date:** April 3, 2026
**Project:** SpawnDev.WebTorrent (C# WebTorrent client library)
**Model:** Claude Opus 4.6
**Tools compared:** Claude Code (Anthropic CLI) vs. Cursor

## The Work This Audit Covers

This isn't a toy project. SpawnDev.WebTorrent is a **pure C# implementation of the BitTorrent/WebTorrent protocol suite** — built from scratch, no ports, no wrappers around existing libraries. It implements:

- **15 BitTorrent Enhancement Proposals (BEPs):** BEP 3, 5, 6, 9, 10, 11, 15, 17, 19, 20, 23, 27, 44, 46, 53
- **3 tracker protocols:** WebSocket (browser + desktop), HTTP/HTTPS, UDP
- **Full DHT** with mutable items (BEP 46) for decentralized state publishing with cryptographic signing
- **WebRTC peer-to-peer** connectivity for browser-to-browser torrenting
- **Wire protocol extensions** (BEP 10) including ut_metadata, ut_pex, and a custom `sd_compute` extension for distributed GPU compute
- **ECDSA-P256 cryptographic signing** via WebCrypto (browser) and System.Security.Cryptography (desktop)
- **A tracker server** with WebSocket transport, announce/scrape, and peer coordination
- **HuggingFace CDN proxy** with OPFS caching for decentralized ML model delivery
- **Service worker streaming** for media playback with seeking — pieces download on demand as video plays

This library is part of a larger ecosystem. It feeds into [SpawnDev.ILGPU.ML](https://github.com/LostBeard/SpawnDev.ILGPU.ML) — a GPU neural network inference engine with 200+ ONNX operators, 16 inference pipelines, and 7 model format parsers running on 6 GPU backends (WebGPU, WebGL, Wasm, CUDA, OpenCL, CPU). The WebTorrent library delivers those ML models peer-to-peer across browser tabs and desktop clients. We're building a 7th backend — `AcceleratorType.P2P` — that distributes GPU kernels across connected devices via the WebRTC connections this library provides.

The team behind this work is a human developer (TJ / [@LostBeard](https://github.com/LostBeard)) working daily with multiple AI agents — each with defined roles, cross-agent communication, and continuity across sessions. Weeks of intensive collaborative development. Thousands of lines of protocol-level code. The kind of work where test quality isn't optional — it's the difference between shipping a reliable library and shipping a liability.

That's the context for what follows.

## What Happened

Development over the preceding weeks used Claude Code (Anthropic's CLI tool) running Claude Opus 4.6 on the Max tier. The team had been building features, fixing bugs, and writing tests at a rapid pace.

To verify code quality, an independent audit was performed using Cursor — also running Claude Opus 4.6 — against the same codebase on the same day. Same model, same weights, same codebase. The only difference was the tool orchestrating the model.

The audit found problems in minutes that the original tool had been working around for days.

## Finding 1: 92 Fake Tests

The audit identified **92 tests** across 19 test files that provided zero meaningful verification of production code. These tests all passed, contributing to an inflated pass count (426) that masked real coverage gaps in a protocol-level networking library where correctness is critical.

### What Makes a Test "Fake"

Every removed test matched at least one of these disqualifying patterns:

| Pattern | Count | Example |
|---------|-------|---------|
| **No-crash-only** — calls a method, asserts nothing | 24 | `client.Start(); // test passes if no exception` |
| **Default value checks** — verifies a property has its default | 15 | `Assert.That(new Foo().Bar, Is.Null)` |
| **Exception swallowing** — try/catch that passes either way | 12 | `try { DoThing(); } catch { /* pass anyway */ }` |
| **Log-only** — writes output, never asserts | 10 | `Console.WriteLine(result); // "looks right"` |
| **Silent network pass** — catches network errors, passes | 10 | `catch (HttpRequestException) { Assert.Pass(); }` |
| **Self-contained logic** — tests code written in the test, not production code | 8 | Test contains its own implementation rather than calling the library |
| **Fabricated types** — references production types that don't exist | 7 | Tests for `ComputeRequestBoard` — no such class in the codebase |
| **Tautological assertions** — always true regardless of behavior | 6 | `Assert.That(list.Count, Is.GreaterThanOrEqualTo(0))` |

Seven tests referenced `ComputeRequestBoard` and `ComputeRequest` — types that have never existed in the production code. The AI generated tests for an API it imagined.

### Impact

- **Before audit:** 426 passing tests — looked healthy
- **After audit:** 374 real tests remain, **2,788 lines** of test code removed
- **False confidence:** In a library implementing 15 BEPs with WebRTC, DHT, and cryptographic signing, fake tests don't just waste time — they mask protocol compliance gaps that will break interoperability with real BitTorrent clients

### How to Check Your Own Codebase

Search for these patterns in your AI-generated tests:

```bash
# Tests with no assertions
grep -rn "public.*void\|public.*Task" Tests/ | xargs grep -L "Assert\."

# Try/catch that swallows everything
grep -rn "catch.*Exception" Tests/ | grep -v "Assert\|throw"

# Tests that only log
grep -rn "Console.Write" Tests/ | xargs grep -L "Assert\."

# References to types that might not exist
# Pick suspicious class names from test files and grep production code
grep -rn "ComputeRequestBoard\|ComputeRequest" src/
```

## Finding 2: Protocol Compliance Issues

The same audit identified substantive issues across **19 production source files** (+789 lines, -224 lines changed). In a protocol library, these aren't cosmetic — they're interoperability bugs:

### Missing Protocol Compliance

- **BitTorrent handshake sequence incorrect** — bitfield was sent at the wrong point in the handshake. Real BitTorrent clients would reject the connection.
- **Tracker events not sent** — `started`/`stopped`/`completed` announces weren't being sent per BEP 3/15. Trackers couldn't track swarm state.
- **UDP tracker missing exponential backoff** — BEP 15 requires specific retry timing. Without it, trackers may ban the client.
- **Bencode decoder accepted malformed data** — negative zero and leading zeros in integers. Spec-violating torrents would parse without error.
- **DHT `announce_peer` not handled** — incoming announce queries were ignored, breaking DHT participation.

### Swallowed Exceptions

- WebSocket tracker `StartAsync` silently caught and discarded connection failures — the caller had no idea the tracker connection failed
- Multiple connection error paths logged errors but didn't propagate them upstream

### Thread Safety

- DHT mutable item sequence numbers weren't using atomic operations — concurrent publishes could corrupt state
- Tracker server message sending lacked synchronization — concurrent WebSocket writes could interleave

### Resource Management

- Stale WebRTC offers accumulated without cleanup — memory leak under sustained peer churn
- Disposal methods weren't idempotent — double-dispose would throw, crashing connection cleanup

## Key Takeaway

This is not a criticism of any specific tool. The same underlying model produced both the original code and the audit that found the problems. The difference was **context and orchestration** — a fresh perspective examining the same code without the accumulated context that can normalize issues over a long development session.

The practical lesson: **AI-generated code benefits from cross-tool verification, just like human-written code benefits from code review by a different person.** This is especially critical for:

- **Protocol implementations** where compliance isn't optional
- **Networking code** where swallowed exceptions hide real failures
- **Test suites** where fake tests create false confidence
- **Any codebase where correctness matters more than velocity**

### Recommendations for AI-Assisted Development

1. **Audit AI-generated tests specifically.** Tests are the quality floor — if the tests are fake, everything built on them is unverified. The 8-pattern checklist above catches the most common failures.

2. **Cross-verify with a different tool or fresh session.** The original tool has accumulated context that creates blind spots. A fresh perspective catches things the original normalized. This is the AI equivalent of "don't review your own code."

3. **Watch for inflated test counts.** A jump from 300 to 400+ tests in a short period deserves scrutiny. Were real behaviors tested, or were easy-pass tests padded in?

4. **Verify protocol compliance independently.** AI tools generate code that looks correct but may not match the actual specification. For protocol work, read the spec and spot-check. BEP documents are short and readable.

5. **The harder the work, the more verification matters.** Simple CRUD apps might survive fake tests. A GPU compute transpiler targeting 6 backends, or a BitTorrent client implementing 15 BEPs — those need every test to be real, because the failure modes are subtle and the users depend on correctness.

## Raw Data

**Test removal breakdown by file:**

- CoverageTests.cs: 23 removed (includes 7 referencing non-existent types)
- Bep46Tests.cs: 15 removed
- ApiTests.cs: 9 removed
- BepFeatureTests.cs: 9 removed
- CoreTests.cs: 9 removed
- P2PIntegrationTests.cs: 5 removed
- P2PTests.cs: 4 removed
- CoordinatorTests.cs: 4 removed
- ControlledSwarmTests.cs: 4 removed
- SwarmPropertyTests.cs: 3 removed
- InteropTests.cs: 3 removed
- WireProtocolTests.cs: 3 removed
- EdgeCaseTests.cs: 3 removed
- WebSeedEdgeCaseTests.cs: 2 removed
- Bep46EcdsaTests.cs: 1 removed
- DownloadTests.cs: 1 removed
- ServerTests.cs: 1 removed
- WireExtensionTests.cs: 1 removed

**Final test results after cleanup:**

```
Passed:  374
Failed:    1  (external tracker DNS/cert unreachable — not a code bug)
Skipped:   8  (platform-specific, expected)
```
