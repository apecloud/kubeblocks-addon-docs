# Addon Test Baseline Standard Guide

> **Audience**: addon dev / test / TL
> **Status**: draft
> **Applies to**: any KB addon
> **Applies to KB version**: 1.0.x / 1.1.x / main (1.2 unreleased)
> **Affected by version skew**: tested against KB v1.0.2 + apecloud-addons release-1.0 baseline; some categories (RebuildInstance, BackupSchedule) only stable from KB v1.0.x onwards

## Abstract

Cross-addon test coverage is uneven today. Some addons (Valkey, MariaDB) test ~80 categories across smoke / chaos / regression / engine-specific suites; others (Oracle, OceanBase, SQL Server) cover smaller surface areas. This guide defines the **minimum required + conditional + optional** test categories every addon must self-audit against, and the cross-addon methodology baselines that govern HOW tests run regardless of category.

The goal is to lift weaker addons up to the strongest addon's coverage standard, not to lower the standard. The 20-item smoke baseline + 10-item chaos baseline + cross-addon methodology baselines are non-negotiable. Engine-specific categories (TLS / ACL / RebuildInstance / DG broker / Always On AG / etc.) become required ONLY for addons whose engine supports them.

## Section 1 — Required smoke test categories (20 items, every addon)

Every addon must have an `tests/smoke.sh` (or equivalent named test suite) that covers all 20 categories below. For categories not applicable to the engine (e.g. Role assignment for non-HA standalone), the test suite must explicitly mark the category as N/A with a one-line reason.

| ID | Category | What it verifies | Required for |
|---|---|---|---|
| S01 | Addon install | `helm install` + ClusterDefinition / ComponentDefinition / ComponentVersion / ParametersDefinition / ActionSet registered | All addons |
| S02 | Cluster create + pod ready | Cluster reaches Running phase, all pods Ready | All addons |
| S03 | Service VIP routing | Service IP routes to expected pod (primary in HA) | All addons |
| S04 | Read/Write baseline | Write a row/key, read back, count matches | All addons |
| S05 | Role assignment | Exactly N primary / M secondary / Q sentinel as expected | HA addons |
| S06 | Reconfigure dynamic | Change a dynamic param via OpsRequest, verify applied without pod restart | All addons with reconfigurable params |
| S07 | Reconfigure static | Change a static param via OpsRequest, verify pod restart + value changed | All addons with reconfigurable params |
| S08 | Reconfigure invalid rejected | Change to invalid value, verify rejection + old value preserved | All addons with parametersSchema |
| S09 | Stop / Start | OpsRequest Stop → cluster Stopped; OpsRequest Start → Running, data intact | All addons |
| S10 | Restart (rolling) | OpsRequest Restart, data + role intact, no manual intervention | All addons |
| S11 | Vertical scaling | OpsRequest VerticalScaling CPU/memory, data intact, role intact | All addons |
| S12 | Horizontal scale-out | OpsRequest HorizontalScaling N→N+1, new pod joins + syncs all data | All addons supporting hscale |
| S13 | Horizontal scale-in | OpsRequest HorizontalScaling N+1→N, removed cleanly | All addons supporting hscale |
| S14 | Volume expansion | OpsRequest VolumeExpansion, PVC resized, data intact | All addons (when expandable StorageClass available) |
| S15 | Expose / Disable LB | OpsRequest Expose enable then disable, LB service created then removed | All addons |
| S16 | Switchover (manual) | OpsRequest Switchover, primary changes to specified or auto candidate | HA addons |
| S17 | Upgrade (same major) | OpsRequest Upgrade to newer ComponentDefinition same major, data intact | All addons with multiple service versions |
| S18 | Backup | Backup OpsRequest reaches phase=Completed | All addons supporting backup |
| S19 | Restore | New cluster restored from backup has correct data | All addons supporting backup |
| S20 | Cleanup | Namespace deletion completes without finalizer hang | All addons |

## Section 2 — Required chaos test categories (10 items, HA addons)

Every HA addon must have an `tests/chaos.sh` (or equivalent) covering the 10 categories below. Standalone-only addons need only C01 + C08.

| ID | Category | What it verifies | Required for |
|---|---|---|---|
| C01 | Kill primary | Auto-recovery, role transfer, data plane resumes within bounded time | All addons |
| C02 | Kill secondary / replica | Primary continues, replica rejoins, data sync resumes | HA addons |
| C03 | Kill primary + secondary simultaneously | Self-heal within bounded time, role re-assigned | HA addons |
| C04 | Rapid consecutive kills (3× same role, ≤15s apart) | HA idempotency, no state corruption | HA addons |
| C05 | Kill during switchover | Exactly 1 primary, no dual-primary deadlock | HA addons supporting Switchover |
| C06 | Kill during Reconfiguring | OpsRequest no deadlock, config consistent after restart | All addons supporting Reconfigure |
| C07 | Network partition (Chaos Mesh required) | Minority isolates correctly, majority continues, heal restores all | HA addons (conditional on Chaos Mesh) |
| C08 | Kill kbagent sidecar | kbagent restarts, no spurious failover, role probe recovers | All addons |
| C09 | Writes during chaos consistency | Keys/rows written after new primary present on all replicas | HA addons |
| C10 | OpsRequest Restart all components | Rolling restart via OpsRequest, data + topology intact | All addons |

## Section 3 — Conditional test categories (engine-specific, when supported)

When the engine supports the feature, the corresponding test suite must exist as `tests/<feature>.sh`. Listed below are the full set of conditional categories observed across current addons; new categories may be added to this list when new engines bring new capabilities.

| Category | Suite file | Required when |
|---|---|---|
| TLS topology | `tests/tls.sh` | Engine supports TLS + KubeBlocks cert-manager integration |
| ACL sync consistency | `tests/acl.sh` | Engine supports ACL (Redis family, etc.) |
| Custom secret (systemAccounts[].secretRef) | `tests/custom-secret.sh` | Addon defines systemAccounts |
| RebuildInstance (PVC loss recovery) | `tests/rebuild.sh` | Addon supports RebuildInstance OpsRequest |
| Cross-major upgrade | `tests/upgrade-cross-major.sh` | Addon has multiple major versions (e.g. Valkey 8 / 9; MariaDB 10.6 / 11.4) |
| Scheduled backup | `tests/scheduled-backup.sh` | Addon supports BackupSchedule |
| Decommission specific replica | (in tests/smoke.sh) | Addon supports `onlineInstancesToOffline` |
| Multi-master writes | `tests/multi-master.sh` | Addon supports multi-master (Galera, etc.) |
| Quorum behavior | `tests/quorum.sh` | Addon has quorum-based HA (Sentinel, Paxos, etc.) |
| DG broker / Oracle-specific HA | `tests/dg-broker.sh` | Oracle |
| Always On AG | `tests/ag.sh` | SQL Server |
| License flow (Enterprise) | `tests/license.sh` | Enterprise addons (OceanBase enterprise, etc.) |
| Hot backup with concurrent writes | (in tests/regression.sh as bug guard) | Addon supports backup during writes |
| Performance / load | `tests/load.sh` | When perf SLOs defined (currently none have explicit suite — tracked separately) |
| Security audit | `tests/security.sh` | When security baseline applies (currently none have explicit suite — tracked separately) |

## Section 4 — Cross-addon methodology baselines (mandatory for ALL test runners regardless of suite)

These govern HOW tests run, not WHAT they test. Every addon test suite must observe ALL of the following:

### 4.1 Rule 1 — Evidence-preserving freeze before any force-clean

Every test failure path must dump cluster YAML / pod describe / controller logs / kbagent logs / engine-specific logs to `evidence/<run-id>/<topo>/<test-id>/` BEFORE any cleanup. Force-cleanup (finalizer edit, force-delete pod, namespace force-delete) is forbidden until the evidence pack is complete and sha256 attested.

See: `addon-evidence-discipline-guide.md`

### 4.2 5-dimension test profile self-documentation

Each addon's TEST_CATALOG.md (or equivalent) must self-describe along 5 dimensions:

1. **Framework** — runner / lib / test directory structure
2. **Coverage matrix** — which Section 1 / 2 / 3 categories the addon covers, and explicit N/A reasons for what's not covered
3. **Strength criterion** — for probabilistic bugs, the N selection rule (see 4.3)
4. **Artifact / log standard** — evidence pack schema, naming convention, sha256 attestation
5. **Cleanup discipline** — how Rule 1 is implemented in trap / collect_evidence / cleanup helpers

### 4.3 Expansion Strength Mandate (Rule 2 — normative)

Single-cycle PASS is NOT production-ready. Every addon must produce strength evidence before release.

**Required strength evidence ladder:**

1. N=1 functional verification — case design works, basic PASS
2. N=10 strength level 1 — repeated invocation under same conditions, all PASS, evidence pack per cycle
3. N=20 strength level 2 — for high-risk cases (data consistency, HA failover, switchover, write-during-chaos), 20 cycles all PASS
4. 24h+ soak with random chaos injection — background workload + 10-60 minute random chaos events for full 24 hours, no acked-write loss, role label converges within bounded time after each event
5. Multi-chaos overlay — at least one combination test (kill primary + reconfigure simultaneous, or kill primary + network partition simultaneous), verify cluster degrades gracefully

**Expansion must add new dimensions, not repeat existing cases.** Examples of new dimensions that have caught production-blocking races in field testing:

- Writes during chaos with per-attempt millisecond-level evidence (catches transient dual-primary windows where acked writes go to wrong replica)
- Three-layer role timestamp (DBMS truth + kbagent emit + KB controller label patch) capture during rolling Day-2 ops (catches role label race with engine state)
- Strict evidence schema distinguishing client-failed-but-committed from confirmed data loss (catches false-positive bug reports vs real race)
- Annotation transition gating instead of controller log scraping (catches early failure paths that controller never logs)

**Strength criterion bound to prior failure rate, not fixed N:**

- High-frequency bugs (failure rate above 30 percent pre-fix): N=3 verifies fix; N=10 confirms fix stability
- Intermittent bugs (failure rate 1 to 30 percent pre-fix): N=20 to N=50 needed for confidence
- Rare bugs (failure rate below 1 percent pre-fix): N=100 or 24h soak with random chaos preferred

**Strength evidence categories observed in cross-line baseline:**

- Data consistency strength (e.g. MariaDB persistent stress N=12 sample classification)
- Environment attribution strength (e.g. Oracle T10b host-pressure N=2 cycle)
- Race window strength (e.g. OceanBase C09 writes during primary kill, 700-millisecond transient dual-primary window)
- Concurrent contract strength (e.g. Valkey rebuild-ops O02 exactly-one-success + 0 residue + cleanup retry-on-conflict)

Each addon's strength criterion section must declare:

1. Which category each strength claim falls in
2. The N (or soak duration) used + rationale tied to prior failure rate
3. Whether the evidence schema can distinguish commit-unknown from real data loss (mandatory for any writes-during-chaos category)

**Reference cross-engine cases** (canonical examples of expansion strength finding real race):

See `cases/methodology/extension-strength-finds-real-race-2026-05-03-case.md` for the consolidated 3-line evidence pack:

1. Valkey RebuildInstance — 49-sample clean baseline expanded by density + concurrency + restart-window axes surfaced 4 contract-level gaps; closed via PR #10191 follow-up commits with sentinel / typed-error contract + N=20 dense gate.
2. OceanBase C09 — small-N clean expanded by dense replay + acked-write injection surfaced ~700ms dual-primary acked-write divergence window.
3. SQL Server CH50 — small-N clean expanded by density + commit-window precise failure injection surfaced commit-unknown ambiguity (server-side committed but client cannot distinguish).

In all three lines the original small-sample acceptance was clean. The contract gaps only became visible after expansion along orthogonal dimensions — proof that "small-sample clean ≠ contract correct" is doctrine backed by hard evidence, not abstract policy.

### 4.4 Artifact pack format

Each evidence pack must produce `tar.gz` + `sha256` attestation. Loose-file evidence is acceptable as a working stage but not as final artifact for cross-addon comparison.

### 4.5 Pre-flight checks (cross-addon environment hygiene)

Every test run must observe these pre-flight checks (or document that they're skipped with reason):

- Host-level k3d snapshot (`addon-k3d-host-precheck-guide.md`)
- Backup/restore prereq (`addon-k3d-backup-restore-prereqs-guide.md`) — including the **Docker Desktop restart non-persistence caveat**: `mount --make-rshared /` on k3d node containers does not survive Docker Desktop restart; backup tests after a restart must re-execute this fix
- Hostname declaration when discussing or claiming any cluster

### 4.6 Pattern B fork-zombie audit (probe / lifecycle scripts)

Every addon's probe and lifecycle scripts must pass the 4D audit checklist in `addon-probe-script-fork-and-zombie-guide.md`. The 4 dimensions are: fork source / frequency / reaper / execution context.

### 4.7 3-layer role timestamp observability (HA addons with rolling Day-2 ops)

For any HA addon doing rolling Day-2 operations (Restart, Stop+Start, Switchover, scale-up/scale-down), evidence must capture three role timestamp layers:

1. **DBMS truth** — engine's own role state (e.g. Oracle V$DATABASE.DATABASE_ROLE / OceanBase DBA_OB_TENANTS.TENANT_ROLE / SQL Server AG primary replica DMV)
2. **kbagent emit** — what role-probe script (`getRole.sh` / `syncerctl getrole` / etc.) returned to kbagent
3. **KB controller pod label patch** — when KB controller patched the `kubeblocks.io/role` label

When two layers disagree, the disagreement IS the race signal.

### 4.8 Chart-vs-KB schema skew pre-flight check

Every addon's chart install must be validated against the target KB version's CRD schema before running tests. Use the framework in `addon-chart-vs-kb-schema-skew-diagnosis-guide.md` (Steps 1-4 including the 3-state install-path taxonomy).

## Section 5 — Per-addon self-audit framework

Each addon test owner must produce a self-audit document at `docs/cases/<engine>/<engine>-test-baseline-audit.md` filling the table below. The audit answers "for each baseline category, do we have it / N/A / planned / needs work":

| Category | Status (✓ / N/A / planned / needs work) | Test file location | Notes / N/A reason |
|---|---|---|---|
| S01 Addon install | | | |
| S02 Cluster create + pod ready | | | |
| ... (all 20 smoke + 10 chaos + applicable conditional) | | | |
| 4.1 Rule 1 evidence-freeze | | | |
| 4.2 5-dim profile self-doc | | | |
| ... (all 8 methodology baselines) | | | |

Each missing item gets a remediation owner + ETA.

## Section 6 — Per-addon current state (reference, will be replaced by self-audit)

This is initial read of current state as of 2026-05-03 — it serves as a starting reference. Each addon test owner replaces this with their own self-audit.

| Addon | S coverage | C coverage | Conditional present | Methodology baselines satisfied |
|---|---|---|---|---|
| MariaDB | 18 / 20 (missing scheduled backup explicit, RebuildInstance) | 9 / 10 (missing C09 writes-during-chaos explicit) | TLS (standalone) / network partition (chaos-mesh) / cross-major upgrade conditional | Rule 1 ✓ / 5-dim ✓ / artifact pack ✓ / pre-flight ✓ / Pattern B ✓ |
| Oracle | 10 / 20 (missing vol-expand, expose, scheduled backup, RebuildInstance, cross-major upgrade) | 4 / 10 (has C01-C04, missing rapid kills, kill-during-switchover/reconfigure, kill-kbagent, writes-during-chaos, opsrequest-restart) | DG broker (T10/T20/T21) / network partition conditional | Rule 1 ✓ / 5-dim partial / artifact pack ✓ / pre-flight ✓ / Pattern B ✓ |
| Valkey | 19 / 20 (most complete) | 10 / 10 ✓ | TLS / ACL / custom-secret / RebuildInstance / scheduled backup / sentinel quorum / cross-major upgrade | Rule 1 ✓ / 5-dim ✓ / artifact pack ✓ / pre-flight ✓ / Pattern B ✓ (self-discovery + fix) |
| OceanBase | ~12 / 20 (missing vol-expand, expose, upgrade, backup-restore, scheduled backup) | ~5 / 10 (has primary-pod-delete, others pending) | License flow conditional ✓ | Rule 1 ✓ / 5-dim partial / artifact pack partial: still loose-file, pending tar.gz adoption / pre-flight ✓ / Pattern B ✓ |
| SQL Server | ~6 / 20 (kickoff, single-replica baseline) | 0 / 10 (planned for B1 3-replica AG window) | Always On AG conditional | Rule 1 ✓ / 5-dim ✓ / artifact pack ✓ (meta.md + transcript) / pre-flight ✓ / Pattern B ✓ |

## Section 7 — Adoption plan

1. **Standard published** (this document, when merged into `kubeblocks-addon-docs/docs/`)
2. **Self-audit deadline** — all addon test owners produce `docs/cases/<engine>/<engine>-test-baseline-audit.md` within 1 week
3. **Remediation tracking** — Cross-addon test methodology lead maintains a cross-addon delta status, ping owners weekly via `#addon-testing` async sync
4. **CI gate or recommendation only** — TBD pending product owner decision (open question 1)
5. **Quarterly baseline review** — cross-addon test eng leads review whether baseline needs new categories (e.g. when a new addon brings a category not covered today)

## Open questions (pending product owner decisions)

1. Should the baseline be a CI gate (block release) or recommended audit (track + report)?
2. Should engine-specific conditional categories be expected of every addon that supports the feature, or always optional?
3. RebuildInstance is universal in concept but only Valkey has it explicitly — set a deadline for other addons to add it?
4. Cross-major upgrade — required for any addon with multiple major versions, or conditional on release timeline?
5. Performance / security categories — currently no addon has explicit suites; should they be added to baseline now or tracked as separate initiative?

## References

- `addon-evidence-discipline-guide.md` (Rule 1)
- `addon-probe-script-fork-and-zombie-guide.md` (Pattern B 4D audit)
- `addon-chart-vs-kb-schema-skew-diagnosis-guide.md` (chart-vs-KB skew framework)
- `addon-bounded-eventual-convergence-guide.md` (probabilistic state convergence)
- `addon-test-acceptance-and-first-blocker-guide.md` (test pass semantics)
- `addon-test-environment-gate-hygiene-guide.md` (env pre-flight)
- `addon-test-runner-portability-guide.md` (runner compat)
- `addon-k3d-backup-restore-prereqs-guide.md` (CSI mount propagation caveat)
- `addon-k3d-kubeconfig-loopback-fix-guide.md`
- `addon-k3d-image-import-multiarch-workaround-guide.md`
- `addon-narrow-scope-force-delete-guide.md`

## Case appendix (cross-engine evidence)

Each addon's self-audit lives at `docs/cases/<engine>/<engine>-test-baseline-audit.md` and provides the engine-specific evidence for compliance with this standard. Cross-engine comparison snapshots may live at `docs/cases/methodology/cross-addon-test-baseline-status-<date>.md`.
