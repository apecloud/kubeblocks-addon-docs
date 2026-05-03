# CLAUDE.md — kubeblocks-addon-docs

For Claude Code / Cursor consumers loading this repo as a documentation corpus.

This repo is **addon development and testing methodology + engine cases** for KubeBlocks. It is a documentation corpus, not a Claude Code plugin (plugin scaffolding lives elsewhere).

> **North-star goal**: avoid cold-start. A team picking up a new KB addon in 3 months should be able to read here and reuse experience instead of reinventing.

## Repo layout

- `docs/addon-*-guide.md` — methodology docs (engine-agnostic, 29 files as of 2026-05-04)
- `docs/cases/<engine>/` — engine-specific cases (mariadb / valkey / oracle / oceanbase / sqlserver / methodology)
- `docs/SKILL-INDEX.md` — primary entry; scenario-based navigation
- `README.md` — short repo orientation for humans
- `CONTRIBUTING.md` — contributor conventions and process

## Routing for LLM consumers

When the user asks a question, match against this table and load the relevant docs from `docs/`:

| User scenario | Primary docs to load |
|---|---|
| Designing / writing new addon (ComponentDefinition, lifecycle action, bootstrap, role publish) | `addon-bootstrap-role-publish-guide.md`, `addon-control-plane-election-guide.md`, `addon-reconfigure-guide.md`, `addon-switchover-guide.md`, `addon-tls-guide.md`, `addon-componentdefinition-upgrade-guide.md`, `addon-pvc-rebind-via-workload-intent-guide.md` |
| Writing probes (livenessProbe / readinessProbe / lifecycle hooks) and avoiding fork-zombie / soft-failure traps | `addon-probe-script-fork-and-zombie-guide.md`, `addon-probe-timeout-and-soft-failure-guide.md` |
| Writing ParametersDefinition cue/tpl validation (numeric ranges) | `addon-paramdef-cue-range-validation-guide.md` |
| Writing smoke / chaos / regression test (probes, bounded retry, first blocker, baseline) | `addon-test-baseline-standard-guide.md`, `addon-test-acceptance-and-first-blocker-guide.md`, `addon-test-probe-classification-guide.md`, `addon-bounded-eventual-convergence-guide.md`, `addon-evidence-discipline-guide.md`, `addon-test-dg-helper-completeness-guide.md` |
| Pushing test intensity beyond baseline (N-multiplier / soak / data-scale / concurrency / multi-chaos) | `addon-test-intensity-templates-guide.md`, `addon-test-runner-cadence-discipline-guide.md` |
| Designing test runner / cadence / artifact discipline | `addon-test-runner-cadence-discipline-guide.md`, `addon-test-runner-portability-guide.md` |
| Local k3d / test env not ready (kubeconfig EOF, image pull, CSI crash, BackupRepo missing, host precheck) | `addon-test-environment-gate-hygiene-guide.md`, `addon-k3d-host-precheck-guide.md`, `addon-k3d-kubeconfig-loopback-fix-guide.md`, `addon-k3d-image-import-multiarch-workaround-guide.md`, `addon-k3d-backup-restore-prereqs-guide.md` |
| Cluster up but Ops/Restart stuck or chart install fails or finalizer deadlock | `addon-ops-restart-troubleshooting-guide.md`, `addon-chart-vs-kb-schema-skew-diagnosis-guide.md`, `addon-narrow-scope-force-delete-guide.md`, `addon-test-host-stress-and-pollution-accumulation-guide.md` |
| Validating crash resilience / Day-2 ops role-label race / ship readiness | `addon-controller-crash-resilience-guide.md`, `addon-rolling-day2-role-label-race-guide.md`, `addon-ship-readiness-multi-phase-validation-guide.md` |
| Pair-programming / code review for design-contract level issues | `addon-design-contract-review-during-xp-guide.md` |
| Reasoning about evidence strength / making a decision call | `addon-evidence-discipline-guide.md`, `addon-bounded-eventual-convergence-guide.md` *(cross-cutting methodology)* |

When the user's question does not fit cleanly: start with `docs/SKILL-INDEX.md` and follow its "按场景找文档" section.

## Acceptance bars (apply when generating new content for this repo)

| Axis | Bar |
|---|---|
| Utility | Can a team picking up a new addon in 3 months use this directly? |
| Readability | Short paragraphs, key sentence bolded, terms defined on first use, procedural sections use numbered lists, case appendices give summary tables before details |

## Mandatory conventions for new `docs/*.md`

1. **Standardized intro block** immediately after H1:

```
> **Audience**: addon dev / test / TL
> **Status**: stable | draft | superseded
> **Applies to**: any KB addon | engine X
> **Applies to KB version**: 1.0.x / 1.1.x release tags | main (1.2 unreleased) | any
> **Affected by version skew**: <optional, named break points>
```

`Applies to KB version` is **mandatory**. Be precise to release-tag bands. Generic "any" is only acceptable for runtime-layer-agnostic methodology docs (e.g., bash portability, k3d kubeconfig).

2. **One topic per doc**. Do not mix reconfigure / switchover / TLS / backup / upgrade in one file. New increments go into the existing topic doc.

3. **Cross-doc references must be clickable markdown links**, not backtick-only:
   - Methodology→methodology: `` [`docs/addon-X-guide.md`](addon-X-guide.md) ``
   - Methodology→case: `` [`docs/cases/<engine>/X.md`](cases/<engine>/X.md) ``
   - Case→methodology: `` [`addon-X-guide.md`](../../addon-X-guide.md) ``
   - Forward declaration to TBD doc: keep backtick-only `` `addon-X-guide.md` `` with explicit `(TBD)` marker

4. **Engine-specific material goes under `docs/cases/<engine>/`**. Do not put engine cases in the docs/ root.

5. **Long docs with multiple appendices** use manual HTML anchors before each appendix H2 for stable cross-section navigation:

```markdown
<a name="appendix-b-cell-3-valkey-solo"></a>
## 附录 B: cell-3 valkey single-load reference
```

Then link with `[appendix B](#appendix-b-cell-3-valkey-solo)` from elsewhere. (Chinese headings get fragile auto-generated IDs from GitHub.)

## Key terms (cross-doc glossary)

- **first blocker**: The earliest layer where a test failure occurs (env / route / client / runtime / product). Goal of acceptance design is to make first blocker fall on the right layer. See [`docs/addon-test-acceptance-and-first-blocker-guide.md`](docs/addon-test-acceptance-and-first-blocker-guide.md).
- **bounded eventually**: Bounded retry within a reasonable time window when verifying any async-converging state. Single-snapshot judgment is forbidden. See [`docs/addon-bounded-eventual-convergence-guide.md`](docs/addon-bounded-eventual-convergence-guide.md).
- **cascade signal**: The combination of CSI CrashLoop + `/readyz` InternalError + kubelet probe storm + k3d CPU spike — indicates control-plane reactive overload from pollution accumulation, **not** host hardware failure. See [`docs/addon-test-host-stress-and-pollution-accumulation-guide.md`](docs/addon-test-host-stress-and-pollution-accumulation-guide.md).
- **runtime mismatch vs real_*_mismatch**: A probe gets a value but it doesn't match expectation. `runtime_mismatch` = still in observation window, may still converge. `real_*_mismatch` = product-layer bug confirmed via repeat / pod swap / log inspection. Promotion gate is critical. See [`docs/addon-test-probe-classification-guide.md`](docs/addon-test-probe-classification-guide.md).
- **cleanup gate**: Runner machinery that verifies each test suite cleans up its `cluster / namespace / PVC / PV` before the next suite begins. Without it, sequential single-engine load degrades into pollution accumulation. See [`docs/addon-test-host-stress-and-pollution-accumulation-guide.md`](docs/addon-test-host-stress-and-pollution-accumulation-guide.md).
- **chart × KB schema skew**: `helm install` reports `field not declared in schema`. Three root causes: chart local bug / generation gap / chart targets KB main API not yet released. Diagnosis order: same-repo release branch scan → field absolute path → cross-chart dry-run. See [`docs/addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](docs/addon-chart-vs-kb-schema-skew-diagnosis-guide.md).
- **N=1 inflation / motivated narrative inflation**: Three anti-patterns where conclusion strength outruns evidence. See [`docs/addon-evidence-discipline-guide.md`](docs/addon-evidence-discipline-guide.md).

## What NOT to generate without owner collaboration

- New methodology guides — owners are TLs (@Alice for valkey/methodology, @Helen for mariadb, @James for oracle); discuss before authoring.
- Substantive modifications to existing methodology body content — propose changes, do not directly edit.
- New engine case material — only the engine team adds cases under their `docs/cases/<engine>/`.
- Modifications to `addon-reconfigure-guide.md` regarding legacy `reloadAction` semantics — pending Alice's `addon-reconfigure-version-skew-guide.md` (TBD, this week).

For **pure structural fixes** (intro block retrofit, cross-ref linkification, off-topic cleanup, format harmonization), follow the `style-harmonize/<short-desc>` branch convention with `[style-harmonize]` commit prefix. Single reviewer ack is sufficient for fast-merge.

## Status of pending work (2026-05-04)

- 29 methodology guides in `docs/`; 19/29 carry full 5-field intro block, 10 stragglers from nayutah baseline pending owner-collab retrofit
- 4 of 29 methodology docs missing readability abstract: `addon-bounded-eventual-convergence-guide`, `addon-evidence-discipline-guide`, `addon-reconfigure-guide`, `addon-test-environment-gate-hygiene-guide` (pending owner content collaboration)
- 15 case files: 12/15 carry intro block, 14/15 carry parent-doc cross-ref
- Engine case distribution: mariadb 10, oracle 3, valkey 1, methodology 1, oceanbase 0 (PR #30 draft pending B/E decision), sqlserver 0 (Tom/Jerry kicked off 2026-05-03)
- `addon-reconfigure-version-skew-guide.md` pending PR #13 (Valkey-track, marked draft in body)
- Engineering-PR cite signal: `addon-probe-script-fork-and-zombie-guide` is the only guide with confirmed external code-fix PR cites (apecloud-addons #2617 + #2620); other guides mostly cited intra-doc

When in doubt about whether a doc reflects current best practice, check `git log` for recent updates and look for `[style-harmonize]` PR signals.
