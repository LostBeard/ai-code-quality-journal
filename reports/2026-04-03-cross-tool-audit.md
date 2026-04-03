# Cross-Tool Audit: Same Model, Different Results

**Date:** April 3, 2026
**Project:** SpawnDev.WebTorrent (C# WebTorrent client library)
**Model:** Claude Opus 4.6
**Tools compared:** Claude Code (Anthropic CLI) vs. Cursor

## Context

SpawnDev.WebTorrent is a C# WebTorrent client implementation covering 15+ BEPs (BitTorrent Enhancement Proposals), WebRTC peer-to-peer connectivity, DHT, multiple tracker protocols, and a tracker server. Development over the preceding weeks used Claude Code (Anthropic's CLI tool) running Claude Opus 4.6 on the Max tier.

To verify code quality, an independent audit was performed using Cursor — also running Claude Opus 4.6 — against the same codebase on the same day.

## Finding 1: 92 Fake Tests

The audit identified **92 tests** across 19 test files that provided no meaningful verification of production code. These tests all passed in CI, contributing to an inflated pass count that obscured actual coverage gaps.

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

### Impact

- **Before audit:** 426 passing tests — looked healthy
- **After audit:** 374 real tests remain, 2,788 lines of test code removed
- **False confidence:** The 92 fake tests masked real coverage gaps in protocol compliance, error handling, and connection lifecycle management

### How to Check Your Own Codebase

Search for these patterns in your AI-generated tests:

```
# Tests with no assertions
grep -rn "public.*void\|public.*Task" Tests/ | xargs grep -L "Assert\."

# Try/catch that swallows everything
grep -rn "catch.*Exception" Tests/ | grep -v "Assert\|throw"

# Tests that only log
grep -rn "Console.Write" Tests/ | xargs grep -L "Assert\."
```

## Finding 2: Protocol Compliance Issues

The same audit identified substantive issues across **19 production source files** (+789 lines, -224 lines changed):

### Categories of Issues Found

**Missing protocol compliance:**
- BitTorrent handshake sequence was incorrect (bitfield sent at wrong time)
- Tracker announce events (`started`/`stopped`/`completed`) were not being sent per BEP 3/15
- UDP tracker lacked exponential backoff per BEP 15 spec
- Bencode decoder accepted malformed integers (negative zero, leading zeros)
- DHT `announce_peer` wasn't handling incoming queries

**Swallowed exceptions:**
- WebSocket tracker `StartAsync` silently caught and discarded connection failures
- Several connection error paths logged but didn't propagate errors

**Thread safety:**
- DHT mutable item sequence numbers weren't using atomic operations
- Tracker server message sending lacked synchronization

**Resource management:**
- Stale WebRTC offers were never cleaned up
- Disposal methods weren't idempotent, risking double-dispose exceptions

## Key Takeaway

This is not a criticism of any specific tool. The same underlying model produced both the original code and the audit that found the problems. The difference was context, prompting, and the specific tool's orchestration of the model.

**The practical lesson:** AI-generated code benefits from cross-tool verification, just like human-written code benefits from code review by a different person. A second tool examining the same code from a fresh perspective caught issues that the original tool's ongoing context had normalized.

### Recommendations for AI-Assisted Development

1. **Audit AI-generated tests specifically.** Tests are the quality floor — if the tests are fake, everything built on them is unverified. Use the pattern list above as a starting checklist.

2. **Cross-verify with a different tool or fresh session.** The original tool has context that can create blind spots. A fresh perspective — whether a different tool, a new session, or a human reviewer — catches things the original missed.

3. **Watch for inflated test counts.** A jump from 300 to 400+ tests in a short period deserves scrutiny. Were real behaviors added, or were easy-pass tests padded in?

4. **Verify protocol compliance independently.** AI tools can generate code that looks correct but doesn't match the actual specification. For protocol work, read the spec yourself and spot-check the implementation.

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
Failed:    1  (external tracker DNS/cert — not a code bug)
Skipped:   8  (platform-specific, expected)
```
