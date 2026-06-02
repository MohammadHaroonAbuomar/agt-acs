# agt-policies (5.0.0a1)

AGT 5.0 policy layer. This package is the Python host surface that
connects Agent OS adapters to the AGT-vendored Agent Control
Specification engine in `policy-engine/`. It owns AGT-specific
manifest resolution, snapshot construction, v4 compatibility mapping,
and verdict normalization while the native ACS runtime owns the
stateless policy decision.

Use this package when an AGT host needs to evaluate a policy at an
intervention point and enforce the returned verdict. Framework
adapters call into it through the v5 runtime bridge; direct AGT hosts
can use the same `agt.policies` primitives to build snapshots and call
the runtime explicitly.

## What is here

- `agt.manifest_resolution` — folder discovery + scope filtering +
  rule merge layer that runs in the host before the engine sees a
  manifest. Implements `spec/agt/AGT-RESOLUTION-1.0.md`.
  (`discover`, `scope`, `merge`, `build`.)
- `agt.policies.snapshot` — snapshot builder per
  `spec/agt/AGT-SNAPSHOT-1.0.md`.
- `agt.policies.bridge` — renders a v4 `GovernancePolicy` into an ACS
  manifest + OPA rego module.
- `agt.policies.result` — `EvaluationResult` (replaces v4
  `PolicyCheckResult`).
- `agt.policies.runtime` — Python wrapper over the ACS Python SDK that
  loads a resolved manifest, runs intervention points, applies the
  transform verdict, enforces approval, and emits AGT telemetry events.

## Runtime flow

1. The host identifies the intervention point, such as `input` or
   `pre_tool_call`.
2. `SnapshotBuilder` creates the complete AGT snapshot for that call,
   including the agent/session envelope and current budget counters.
3. `AgtRuntime` resolves the manifest when needed, sanitizes AGT-only
   fields for the native engine, and calls the ACS Python SDK.
4. The returned ACS verdict is mapped to `EvaluationResult`, including
   `verdict`, `reason`, optional `transform`, optional `evidence`, and
   the `input_identity` / `enforced_identity` audit fields.
5. The host enforces the result. `allow`, `warn`, and `transform`
   proceed; `deny` blocks; `escalate` routes through the configured
   approval resolver or fails closed.

## Compatibility bridge

Existing Agent OS adapters still accept the v4 `GovernancePolicy`
dataclass. `agt.policies.bridge` renders that policy into an ACS
manifest plus a generated Rego bundle. The bridge preserves v4
semantics where they differ from the native ACS defaults, including an
empty `allowed_tools` list meaning no allowlist and `max_tool_calls=0`
meaning deny every tool call.

The generated compatibility policy is identified as `agt_legacy_rules`
inside the resolved ACS manifest. If merged governance rules are
present but no intervention point binds to `agt_legacy_rules`,
resolution fails closed rather than producing rules that never run.

## Security invariants

The host layer is fail-closed by design. Notably: governance files
that resolve outside the workspace root are rejected; directory-style
scopes (`dir/`) cover their subtree; a parent `deny` cannot be
neutralised by a child `allow` whose condition overlaps it; malformed
budget counters and approval-resolver timeouts deny rather than
silently allow.

Resolved Rego bundles are materialized outside the governed workspace
for runtime use and cleaned up when the runtime closes. This prevents a
workspace-writable policy bundle from being overwritten between
resolution and evaluation.

## Install (development)

```sh
cd agent-governance-python/agt-policies
pip install -e ".[dev]"
pytest
```

Tests that exercise `agt.policies.runtime` require the native ACS Python
SDK from `policy-engine/sdk/python`. In a repository checkout, build it
first:

```sh
cd ../../policy-engine
pip install ./sdk/python
```

OPA-backed Rego evaluations also require `opa` on `PATH` or
`ACS_OPA_PATH` pointing at an OPA executable.
