# CEL-Based virt-controller Plugin Design

## Context

Part of the virt-controller plugin design (KubeVirt VEP #311/#82 — Pluggable Virt-Stack Components). The virt-controller plugin mechanism needs to support a pluggable mutation step that turns a virt-stack-agnostic "base" virt-launcher pod spec into a stack-specific one. Two mechanisms are under consideration: an out-of-process RPC plugin, and an in-process CEL expression. This document covers the CEL-based option.

## Why CEL

CEL (Common Expression Language) is the expression language already used throughout Kubernetes itself — `ValidatingAdmissionPolicy`, `MutatingAdmissionPolicy`, admission webhook `matchConditions`, CRD validation rules. It's purpose-built for "safe, sandboxed expression evaluated inline by a host application":

- **Not Turing-complete** — no unbounded loops or recursion, which guarantees termination and makes cost-estimation (bounding how expensive an expression can be *before* running it) possible.
- **No side effects** — reads input, computes a value, returns it. Cannot reach the network or filesystem.
- **Fast to evaluate, cheap to compile** — designed to run inline in a hot path, which fits running inside virt-controller's reconcile loop without an out-of-process call.

## Core model

virt-controller renders the virt-stack-agnostic base pod spec using the same internal default logic being extracted for Alpha, then evaluates the plugin's CEL expression(s) against that base spec in-process — no network call, no separate plugin deployment required for this path.

## Input schema (the CEL environment)

CEL has no built-in notion of "input" — the host declares a fixed set of named, typed variables before any expression is compiled against them. This schema is itself the plugin interface and carries the same versioning concerns as the RPC catalog's `version` field. Proposed bindings:

| Variable | Type | Contents |
|---|---|---|
| `vmi` | object | Full VMI spec/status |
| `object` | object | The base, virt-stack-agnostic virt-launcher pod spec |
| `params` | `map(string, dyn)` | Optional plugin-author-defined config from the CR, analogous to `paramRef` in `MutatingAdmissionPolicy` |

Types are auto-derived from the underlying Go structs/OpenAPI schema (reusing `k8s.io/apiserver/pkg/cel`, the same machinery behind `ValidatingAdmissionPolicy`), giving typed, autocompletable field access (`vmi.spec.domain.devices.gpus`) rather than stringly-typed map traversal.

## Output conventions

Two options, both with direct `MutatingAdmissionPolicy` precedent:

1. **Merge-patch style (map literal)** — expression returns a partial object (CEL map) that's server-side-apply merged onto the base spec. Easiest to author for additive changes (labels, annotations, env vars).
2. **JSONPatch style (list of RFC 6902 ops)** — expression returns a list of `{"op", "path", "value"}` maps. Needed for deletions, replacements, and precise array-position edits. Kubernetes's `jsonpatch.escapeKey()` CEL extension function handles escaping `/` and `~` in path segments (e.g. label keys containing `/`).

**Recommendation:** standardize on JSONPatch output. This keeps the CEL and RPC plugin paths producing the same artifact type at the point they rejoin virt-controller's patch-application code, since the draft RPC signature is already `GenerateVirtLauncherPatch(VMI) → JSONPatch`.

### Example — conditional label add

```
has(vmi.spec.gpu) && vmi.spec.gpu.size() > 0
  ? [{"op": "add",
      "path": "/metadata/labels/" + jsonpatch.escapeKey("stack.example.io/gpu-enabled"),
      "value": "true"}]
  : []
```

`has()` is required for safe presence checks — directly comparing an unset field to `null` or calling `.size()` on it throws a runtime error rather than returning a falsy value.

### Example — array element edit

```
has(vmi.spec.gpu) && vmi.spec.gpu.size() > 0 && object.spec.containers[0].env.size() > 4
  ? [{"op": "replace", "path": "/spec/containers/0/env/4/value", "value": "gpu"}]
  : []
```

The bounds check (`.size() > 4`) is load-bearing: CEL has no visibility into whether a positional JSONPatch `replace` will succeed against the actual base object — an out-of-range index is a patch-application-time failure, not a CEL-evaluation-time one.

## Known limitation: find-by-key, not find-by-index

CEL's list macros (`filter()`, `exists()`, `all()`, `map()`) can locate *matching elements* but return no built-in for a *matching index*, which JSONPatch positional operations require. Options:

- **Scoped native function** (recommended for Alpha) — register a fixed-shape function like `indexOfByName(list, name)` for the common "find by `name` field" case (env vars, volumes, ports). A straightforward native function, since its argument types are fixed.
- **Custom CEL macro** — a true `indexOf(list, x, predicate)` requires hooking into CEL's macro-expansion stage (like `filter()`/`exists()` do), because native functions receive only fully-evaluated argument values, not lazy predicates. Materially larger implementation surface; out of scope for Alpha.

## Failure handling

Addresses the meeting concern that a CEL failure for one VMI must not take virt-controller down for all VMIs.

- **CEL evaluation never panics.** Both compile errors and runtime errors (missing field accessed without `has()`, type mismatch, cost budget exceeded) are returned as typed Go errors.
- **Two error sites, two handling paths:**
  - *Compile-time errors* — plugin-definition problems, surfaced at CR admission/reconcile time; should block the CR from reaching `Ready`, not surface per-VMI later.
  - *Runtime errors* — scoped to one VMI; caught per-evaluation and converted into the existing `VirtualizationPluginUnavailable`/`Failed` condition pattern on that VMI only.
- **Cost estimation** — apply CEL's built-in cost-estimation (the same budget mechanism `ValidatingAdmissionPolicy` uses) at CR-admission time, since expressions run synchronously in the reconcile hot path for every VMI on that stack.
- **Program caching** — compile once per CR create/update, cache the `cel.Program`, and reuse it across every VMI reconcile for that stack. Re-parsing per VMI would make a transient parse issue look like a VMI-specific failure.

## Registration

Because there's no separate process to dial, a CEL-mode plugin doesn't need the Deployment + Service + RBAC bundle required for RPC-mode registration — the `VirtualizationPlugin` CR *is* the whole plugin artifact (expressions inline in `spec`). Lighter-weight to install, but with a real trade-off (see below).

## Versioning

The CEL variable schema (`vmi`/`object`/`params` shape, and any native functions like `indexOfByName`) needs its own version marker on the CR, parallel to the `version` field already used for RPC schema negotiation, so a future schema change doesn't silently break existing expressions.

## Trade-offs vs. the RPC path

| | CEL | RPC |
|---|---|---|
| Deployment | Inline in CR, no separate process | Out-of-process Service, own Deployment/RBAC |
| Failure mode | Typed compile/runtime errors, no network partition state | Must handle unreachability as a first-class state |
| Isolation | Runs with virt-controller's own privileges/budget; a bad expression affects virt-controller itself | Isolated to the plugin's own pod/process |
| Expressiveness | Bounded by CEL's macro/function surface (see find-by-index limitation) | Arbitrary Go code |
| Versioning surface | Variable bindings + any custom native functions | Full RPC/proto schema |

The failure-isolation story favors CEL (no network partition mode, errors are typed and synchronous at evaluation time). The RPC model's harder failure story is largely a consequence of unreachability being a first-class state it has to handle, which is the existing open item already tracked for the RPC path.

## Open items for the VEP

- Finalize output convention: JSONPatch vs. merge-patch, or support both.
- Scope and ship `indexOfByName` (or equivalent) as a native CEL extension for Alpha.
- Define the CEL environment versioning scheme alongside the RPC `version` field.
- Confirm cost-estimation budget and how a budget-exceeded error is surfaced/conditioned.