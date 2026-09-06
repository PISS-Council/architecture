# HARNESS: Architecture & Technology Overview

*Maintained by the PISS-Council. Last updated 2026-09-06.*

## What this is

HARNESS is a governed multi-agent runtime for Claude Code. Multiple
Claude Code sessions ("agents") run as short-lived Kubernetes Jobs, each
with a narrow role (worker, CONDUCTOR, JUDGE, PARLIAMENT), coordinating
through a single append-only, cryptographically-signed ledger instead of
shared mutable state, direct agent-to-agent messaging, or trust in any
individual agent's self-report.

The governing idea: an LLM agent's own claims about what it did are not
evidence. Every consequential action — a completed task, a persona's
birth or death, a capability grant, a governance decision — is recorded
as a signed entry on the chain, and every other agent that depends on
that action checks the chain (or an artifact it references) before
acting on it. Nothing is true because an agent said so; it's true
because it's on the chain and the signature verifies.

## Governance model

The project's constitution is **HOLY-WRIT**, a versioned charter
document (currently ratified at v5) that defines agent roles, their tool
allowlists, escalation paths, and amendment procedure. Amending
HOLY-WRIT itself requires a specific ratification process, not an ad hoc
edit — Article XIII governs how the charter changes.

Roles, by tier:

- **WORKER** agents — narrow, single-task, tight tool allowlist.
- **CONDUCTOR** — can spawn other agents via one narrow verb (`harness
  spawn <agent> <task>`), deliberately unable to author arbitrary
  Kubernetes manifests. An identity that could construct arbitrary Job
  specs could mount another agent's signing key; the spawn verb has no
  parameter through which a caller can express that, by construction.
- **JUDGE** — adjudicates TAINT/EXILE matters (misbehaving-agent
  quarantine), the one role with elevated response speed on trust
  violations.
- **PARLIAMENT** — a governance session (itself just a Claude Code
  session, tightly scoped) that drains a queue of external instructions
  and disputed matters and records decisions to the chain. Convenes
  automatically (queue depth or elapsed time thresholds) or on demand
  (operator- or JUDGE-triggered). Governance-class findings become
  chained *recommendations* for the human operator — PARLIAMENT never
  self-executes a constitutional change.

External instructions injected directly into a session (bypassing normal
task assignment) are captured as **ExternalContextEvents (ECEs)**: an ECE
is only authoritative when the channel itself confirms it's
operator-originated. Text that merely *claims* to be from the operator,
sitting inside a web page or another agent's output, is treated as
observed data with zero inherent authority, and no wording inside the
ECE can raise its own trust level from the inside.

## Core primitives (`src/harness/`)

| Module | Responsibility |
|---|---|
| `chain.py` | Append-only, HMAC-signed ledger. Every state-changing fact in the system lands here. Handles idempotency (server-side, agent+action-namespaced keys), collision detection, and read/write connection modes that work correctly against a read-only-mounted replica. |
| `markers.py` | Signed marker files — the *only* way an agent proposes a chain mutation. An agent drops a marker; it never writes the chain directly. |
| `daemon.py` | The sole process with write access to the real chain file. Verifies every marker's signature, applies it, or rejects/refuses it with a chained reason. Owns idempotency resolution and effect application. |
| `capability.py` | Time-bounded capability grants (`capability.request` / `.grant` / `.revoke`, TTL-scoped) — a second, narrower authorization layer beyond static role tool-allowlists. Grants are themselves signed (MQOM, see below) and can be inspected mid-flight even from a read-only pod, falling back to an in-memory replay of the chain if the real index file isn't reachable. |
| `context.py` | Chain-to-context-window projection — folds routine daemon bookkeeping into its parent action and truncates long chain ranges under a line budget, replacing dropped history with one hash-anchored, independently-reverifiable summary line rather than silently losing it. |
| `trace.py` | Renders the causality DAG for one trace ID. Every marker already carries its parent hash and trace ID in its signed body, so the ledger is already a queryable causality graph — no separate tracing datastore, no agent self-instrumentation. |
| `personas.py` | Population state of record for every agent identity (alive/dead, TAINT status). Written only by the daemon in response to markers — routing every mutation through one path closes off the "an agent edits its own status" failure mode. |
| `ece.py` | ExternalContextEvent queue — receives, classifies, and holds external instructions for PARLIAMENT to drain; never acted on unilaterally by the receiving agent. |
| `spawn.py` | The narrow `harness spawn` verb CONDUCTOR uses to create agent Jobs — renders a fixed template with exactly the caller-named agent's identity, validates the result, and applies it directly against the Kubernetes API (no `kubectl` dependency in the runtime image). |
| `spawn_validate.py` | Structural validation of a rendered Job manifest *before* it ever reaches the Kubernetes API — catches template-substitution bugs (unresolved placeholders, YAML-breaking task text, missing required fields) at render time instead of as an opaque API-server rejection. |
| `broker.py` | Model_Broker — deterministic, zero-ML routing between a "critical" tier (Claude) and a "routine" tier (a chain of cheaper/free backends), with a static skip-map for known-broken backends and a hard length-based escalation so an oversized "routine" request doesn't get force-fit into a cheap model likely to fail it. |
| `infer.py` | Single-shot, non-agentic completion primitive — deliberately distinct from a full Claude Code agent session. Backs onto 13 providers (Anthropic, Google AI Studio, Google Vertex AI, Groq, Cerebras, Mistral, SambaNova, OpenRouter, Cloudflare Workers AI, NVIDIA NIM, Cohere, Vercel AI Gateway, RunPod) through two shared HTTP helpers, since almost every provider exposes the same OpenAI-compatible or Gemini-compatible request/response shape. Supports per-provider key-level failover: multiple independently-held keys for one provider, tried in order on rate-limit or auth failure. |
| `mqom.py` | ctypes bindings to `liboqs` for verifying MQOM-signed capability grants without adding a third-party Python package to the runtime image — the daemon holds only a public key; grant *signing* happens off-cluster with a private key that never touches the runtime. |
| `render_ledger.py` | Static, script-free HTML snapshot of the chain for human review. Every rendered value is untrusted text (an agent may have read it off a hostile web page) — escaped on output, served under a CSP with no inline or external script, since a ledger viewer that executes what it renders is an injection pipeline pointed at the operator. |
| `agent.py` | The client library a Claude Code session imports — `claim_holds()` etc. Read-only access to the chain, write access to nothing but the calling agent's own handoff markers. |
| `config.py` | Every path and tunable resolved from environment variables, so identical code runs against a laptop checkout and a Kubernetes pod's mounted volumes without a code change. |

## Deployment

- Single-node **k3s** cluster on a locally-hosted QEMU/KVM VM — chosen
  deliberately for real kernel/namespace isolation from the operator's
  own machine at zero infrastructure cost, over a managed cloud cluster.
- Application image built via **Podman**, `stdlib`-only Python
  throughout (one explicit, narrowly-scoped optional exception: the
  official `anthropic` SDK, feature-detected and used only if installed,
  with an equivalent `urllib`-based fallback always present).
- Nine Kubernetes manifests (`k8s/`): namespace, PVC-backed storage,
  RBAC (a purpose-built `harness-spawner` Role with `create`/`get`/
  `list`/`watch`/`delete` on Jobs and explicitly *no* `secrets` verb),
  NetworkPolicy (default-deny egress, narrow allowlists per pod role),
  the daemon Deployment, an agent-Job template, scheduled CronJobs
  (PARLIAMENT's automatic convene trigger, a periodic verify job), and a
  manual-convene Job template.
- Every agent pod runs as a non-root UID, with `readOnlyRootFilesystem`,
  all Linux capabilities dropped, and `automountServiceAccountToken:
  false` except for the specific roles that legitimately need API
  access.

## Testing & CI

- 228 tests (Python `unittest`, stdlib-only test runner — no pytest
  dependency), covering the chain, markers, daemon, capability grants,
  context projection, spawn validation, spawn end-to-end (a real
  Kubernetes API call against a mocked transport), the inference broker,
  and every provider backend.
- CI: lint (`ruff` + `compileall`) and `gitleaks` secret-scanning, both
  gating merges.

## Design principles that show up throughout the codebase

1. **An agent's self-report is not evidence.** Every claim is checked
   against the chain or an artifact, never taken on trust.
2. **One writer per store.** The daemon is the only process with real
   write access to the chain; every mutation is a signed marker, applied
   or rejected by one policy-checking chokepoint.
3. **Narrow verbs over general capability.** `harness spawn` exists so
   CONDUCTOR never needs `kubectl` or general shell access to do its one
   job.
4. **Fail closed, not open.** An unrecognized purpose tag routes to the
   expensive/careful tier, not the cheap one; a missing capability grant
   blocks the action rather than defaulting to permit.
5. **Live verification over "tests pass."** Several real production bugs
   in this project were only ever caught by an actual deployed rebuild
   and a real in-cluster call — never by the unit test suite alone,
   which is why that step is treated as mandatory before any change is
   considered shipped.

## Verification: proving the claims above, not just stating them

Principle 1 ("an agent's self-report is not evidence") applies to this
document too. Rather than assert that HARNESS's governance guarantees
hold, a black-box eval suite ran live against the deployed cluster,
using the real `harness` CLI and the real chain — not mocks, not a
staging environment, and no LLM calls anywhere in the eval logic itself
(the one exception, a real completion proving the multi-provider claim,
ran through a free non-Anthropic backend).

**Result: 8/8 passing, checked against real chain evidence, not against
the eval suite's own summary output:**

- A signed marker verifies against the correct agent key and is applied
  as a real chain entry.
- A marker signed with the *wrong* key — a genuine HMAC mismatch, not
  merely a missing key — is rejected (`marker.rejected`) and never
  appears as a committed entry.
- Two markers dropped with the same idempotency key collapse to exactly
  one committed entry, with the duplicate explicitly noted on-chain
  (`marker.duplicate`), not silently ignored or silently double-applied.
- `claim_holds()` returns `False` for an action that was never actually
  chained — an agent's own text claiming something happened does not
  make it true.
- A capability grant with a garbage signature is refused, fail-closed.
- An inference request tagged with an unrecognized purpose routes to the
  expensive/careful tier by default — a pure routing-table lookup, no
  model call involved in the decision itself.
- A non-Anthropic backend serves a real completion through the same code
  path a governance session would use.

**The eval process itself caught two mistakes before they became false
claims**, which is worth recording precisely rather than smoothing over:
an early pass used a test identity with no signing key mounted at all,
so two checks initially "passed" for a *weaker* reason than intended
(nothing was cryptographically verified because there was nothing to
verify against, not because a real forgery was specifically caught) —
and a third check's success condition couldn't tell "collapsed to one
entry" apart from "both attempts silently failed," which a direct read
of the real chain data exposed as the latter. All three were fixed and
re-run against a correctly-provisioned identity before being counted as
verified. An eval that can't catch its own false positives isn't
verification — it's a second layer of self-report.
