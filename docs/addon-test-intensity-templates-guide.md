# Cross-Addon Test Intensity Templates

> **Audience**: addon test engineers (per-engine test engineers)
> **Status**: draft
> **Applies to**: any KB addon
> **Applies to KB version**: any
> **Affected by version skew**: none directly; per-engine intensity may need engine-specific adjustment

## Abstract

Current cross-addon test sets pass single-cycle but rarely run at high intensity. This document gives engine-neutral templates for raising test intensity along 5 axes:

1. Probabilistic verification N (Rule 2 candidate)
2. Long-running soak with random chaos injection (24h+)
3. Data scale baseline (10GB / 100M rows)
4. Concurrency baseline (N×100 connections, N×1000 writers)
5. Multi-chaos overlay (kill + partition + reconfigure simultaneous)

Templates are designed to be forked into per-addon runners with engine-specific adapters. The 5 categories below correspond directly to the missing strength dimensions in the cross-addon baseline section 4.3 (Strength criterion).

## Template 1 — N-multiplier wrapper

### Purpose

Run any single test case N times and report PASS rate + first-failure cycle. Replaces "single-cycle PASS = accepted" with "N-of-M PASS = accepted with strength evidence".

### Engine-neutral wrapper script

```bash
#!/bin/bash
# n-multiplier.sh - wrap any single test case N times, collect evidence per cycle, report aggregate
# Usage: N=20 ./n-multiplier.sh -t dist -test invalid-reconfigure
#        N=50 ./n-multiplier.sh -t repl -test switchover

set -euo pipefail

N="${N:-10}"
TEST_SUITE="${TEST_SUITE:-smoke}"
TEST_CASE="${TEST_CASE:-T01}"
RUN_ROOT="${RUN_ROOT:-evidence/n-mult-$(date +%Y%m%d-%H%M%S)-${TEST_SUITE}-${TEST_CASE}-N${N}}"

mkdir -p "$RUN_ROOT"

PASS=0
FAIL=0
FIRST_FAILURE=""

for i in $(seq 1 "$N"); do
    cycle_dir="$RUN_ROOT/cycle-$(printf "%03d" "$i")"
    mkdir -p "$cycle_dir"

    if ./run-tests.sh -t "$TEST_SUITE" --test-id "$TEST_CASE" \
        --evidence-dir "$cycle_dir" 2>&1 | tee "$cycle_dir/cycle.log"; then
        PASS=$((PASS + 1))
        echo "PASS" > "$cycle_dir/result"
    else
        FAIL=$((FAIL + 1))
        echo "FAIL" > "$cycle_dir/result"
        if [ -z "$FIRST_FAILURE" ]; then
            FIRST_FAILURE="$i"
        fi
    fi

    # Rule 1 evidence preserved per cycle, even on PASS
    # cleanup happens here only AFTER evidence saved
done

cat > "$RUN_ROOT/SUMMARY.md" <<EOF
# N-Multiplier Run Summary

- Test: $TEST_SUITE / $TEST_CASE
- N: $N
- Pass: $PASS / $N
- Fail: $FAIL / $N
- Pass rate: $(echo "scale=2; $PASS * 100 / $N" | bc)%
- First failure cycle: ${FIRST_FAILURE:-none}
- Run root: $RUN_ROOT
EOF

# Pack evidence + sha256
cd "$(dirname "$RUN_ROOT")"
tar czf "$(basename "$RUN_ROOT").tar.gz" "$(basename "$RUN_ROOT")"
sha256sum "$(basename "$RUN_ROOT").tar.gz" > "$(basename "$RUN_ROOT").tar.gz.sha256"

echo "===> N-multiplier run complete: $PASS/$N PASS"
exit $([ "$FAIL" -eq 0 ] && echo 0 || echo 1)
```

### Acceptance criteria

- For known high-frequency bugs: N=10 all PASS = accepted
- For known intermittent bugs: N=20-50 all PASS = accepted (the lower the prior failure rate, the higher N must be)
- For unknown new categories: N=10 first-pass; if any failure within first 10, escalate N to 20+ for follow-up

## Template 2 — 24h soak with random chaos injection

### Purpose

Replace fixed-sequence longrun (current 1108 12h) with 24h+ soak that intersperses random chaos events. Catches "rare combination" failures the fixed sequence misses.

### Engine-neutral soak orchestrator

```bash
#!/bin/bash
# soak-runner.sh - 24h random-chaos soak against an existing cluster
# Usage: SOAK_DURATION=24h CHAOS_INTERVAL_MIN=10 CHAOS_INTERVAL_MAX=60 ./soak-runner.sh

set -euo pipefail

SOAK_DURATION="${SOAK_DURATION:-24h}"
CHAOS_INTERVAL_MIN="${CHAOS_INTERVAL_MIN:-10}"  # min minutes between chaos
CHAOS_INTERVAL_MAX="${CHAOS_INTERVAL_MAX:-60}"  # max minutes between chaos
RUN_ROOT="${RUN_ROOT:-evidence/soak-$(date +%Y%m%d-%H%M%S)}"

mkdir -p "$RUN_ROOT"

# Chaos action library (engine-adapted)
CHAOS_ACTIONS=(
    "kill-primary"
    "kill-secondary"
    "kill-kbagent"
    "rapid-3-kill"
    "kill-during-reconfigure"
    "writes-during-chaos"
    # add engine-specific: e.g. oracle adds dgmgrl-disable, mssql adds AG-failover-trigger
)

start_time=$(date +%s)
end_time=$((start_time + $(date -d "$SOAK_DURATION" +%s) - $(date +%s)))
chaos_count=0

# Background continuous workload
./workload-driver.sh --duration "$SOAK_DURATION" --evidence-dir "$RUN_ROOT/workload" &
WORKLOAD_PID=$!

while [ "$(date +%s)" -lt "$end_time" ]; do
    # Sleep random interval between chaos events
    interval=$(( CHAOS_INTERVAL_MIN + RANDOM % (CHAOS_INTERVAL_MAX - CHAOS_INTERVAL_MIN + 1) ))
    sleep "${interval}m"

    # Pick random chaos
    action="${CHAOS_ACTIONS[$RANDOM % ${#CHAOS_ACTIONS[@]}]}"
    chaos_dir="$RUN_ROOT/chaos-$(printf "%03d" "$chaos_count")-${action}"
    mkdir -p "$chaos_dir"

    echo "[$(date)] Triggering chaos: $action" | tee -a "$RUN_ROOT/orchestrator.log"
    ./chaos-actions/"$action".sh --evidence-dir "$chaos_dir" || \
        echo "[$(date)] Chaos $action FAILED" >> "$RUN_ROOT/orchestrator.log"

    chaos_count=$((chaos_count + 1))
done

kill "$WORKLOAD_PID" 2>/dev/null || true
wait

# Aggregate
echo "Total chaos events: $chaos_count" >> "$RUN_ROOT/SUMMARY.md"
# ... evidence pack + sha256 same as Template 1
```

### Acceptance criteria

- 24h soak with random chaos at 10-60min intervals: PASS = workload writes never lost, role label always converges within bounded time after each chaos, no resource leak (check zombie count, fd count, memory baseline drift)

## Template 3 — Data scale baseline

### Purpose

Verify backup/restore/scale operations still complete + behave correctly under realistic data volumes (10GB, 100M rows). Single-cycle smoke at 30 keys / 30 rows misses scale-related issues.

### Engine-neutral data loader

```bash
#!/bin/bash
# data-scale-loader.sh - load N rows or N GB before invoking standard test
# Usage: DATA_SIZE=10GB ./data-scale-loader.sh && ./run-tests.sh -t dist --test-id T18-backup

set -euo pipefail

DATA_SIZE="${DATA_SIZE:-10GB}"  # supports MB/GB/rows
TARGET_TABLE="${TARGET_TABLE:-stress_test_table}"

# Engine-specific row size estimation
ROW_SIZE_BYTES="${ROW_SIZE_BYTES:-1024}"  # average row size; engine adapter overrides

# Compute target row count
if [[ "$DATA_SIZE" =~ ^([0-9]+)GB$ ]]; then
    TARGET_BYTES=$((${BASH_REMATCH[1]} * 1024 * 1024 * 1024))
    TARGET_ROWS=$((TARGET_BYTES / ROW_SIZE_BYTES))
elif [[ "$DATA_SIZE" =~ ^([0-9]+)M$ ]]; then
    TARGET_ROWS=$((${BASH_REMATCH[1]} * 1000000))
else
    TARGET_ROWS="$DATA_SIZE"
fi

echo "Loading $TARGET_ROWS rows into $TARGET_TABLE..."

# Engine-specific load command (mariadb / oceanbase / mssql / valkey adapters)
./engine-adapter-load.sh "$TARGET_TABLE" "$TARGET_ROWS"

# Capture pre-test baseline metrics
./engine-adapter-metrics.sh > pre-test-metrics.txt
```

### Acceptance criteria per engine

- Backup of 10GB: completes within reasonable time (engine SLO)
- Restore of 10GB to new cluster: data plane resumes + row count verified
- Scale-out from 3→4 with 10GB existing data: new replica syncs all data within bounded time
- Reconfigure during 10GB workload: no data loss, no transaction lost

## Template 4 — Concurrency baseline

### Purpose

Existing tests use single-thread DML. Real production has hundreds-thousands of concurrent connections + writers. Add concurrency baseline to expose connection-pool exhaustion, lock contention, replication lag amplification.

### Engine-neutral concurrent driver (using ksdotgo / sysbench / engine-native pattern)

```bash
#!/bin/bash
# concurrent-workload.sh - drive N×100 concurrent connections with N×1000 writers
# Usage: CONNECTIONS=100 WRITERS=1000 DURATION=10m ./concurrent-workload.sh

set -euo pipefail

CONNECTIONS="${CONNECTIONS:-100}"
WRITERS="${WRITERS:-1000}"
DURATION="${DURATION:-10m}"
RUN_ROOT="${RUN_ROOT:-evidence/concurrent-$(date +%Y%m%d-%H%M%S)-C${CONNECTIONS}-W${WRITERS}}"

mkdir -p "$RUN_ROOT"

# Engine adapter chooses sysbench / pgbench / redis-benchmark / ksdotgo
./engine-adapter-bench.sh \
    --connections "$CONNECTIONS" \
    --writers "$WRITERS" \
    --duration "$DURATION" \
    --evidence-dir "$RUN_ROOT" \
    --target-cluster "${TARGET_CLUSTER}"

# Capture results
./engine-adapter-metrics.sh > "$RUN_ROOT/post-bench-metrics.txt"

# Verify no lost transactions, no replica lag explosion
./engine-adapter-replication-lag.sh > "$RUN_ROOT/replication-lag.tsv"
```

### Acceptance criteria

- Connection saturation: N×100 concurrent connections do NOT crash cluster, return-error-rate within engine SLO
- Write throughput: N×1000 writers maintain throughput without OOM / disk-full / replication backlog
- Replica lag during heavy writes: stays within bounded window after writes pause

## Template 5 — Multi-chaos overlay

### Purpose

Single chaos = "kill primary". Real failure modes often combine: primary dies + network partition + reconfigure ops happens during recovery. Test these combinations to expose state machine bugs.

### Multi-chaos orchestrator

```bash
#!/bin/bash
# multi-chaos.sh - apply 2-3 chaos actions concurrently, verify cluster recovery
# Usage: ./multi-chaos.sh kill-primary network-partition reconfigure

set -euo pipefail

CHAOS_ACTIONS=("$@")
RUN_ROOT="${RUN_ROOT:-evidence/multi-chaos-$(date +%Y%m%d-%H%M%S)-$(IFS=_; echo "${CHAOS_ACTIONS[*]}")}"

mkdir -p "$RUN_ROOT"

# Apply each chaos in parallel
PIDS=()
for action in "${CHAOS_ACTIONS[@]}"; do
    chaos_dir="$RUN_ROOT/chaos-${action}"
    mkdir -p "$chaos_dir"
    ./chaos-actions/"$action".sh --evidence-dir "$chaos_dir" &
    PIDS+=($!)
done

# Wait for all chaos to complete
for pid in "${PIDS[@]}"; do
    wait "$pid"
done

# Verify cluster eventually recovers
./engine-adapter-wait-cluster-running.sh --timeout 10m
./engine-adapter-verify-data-integrity.sh > "$RUN_ROOT/data-integrity.txt"
./engine-adapter-verify-role-correctness.sh > "$RUN_ROOT/role-correctness.txt"
```

### Recommended multi-chaos combinations

| Combination | What it stresses |
|---|---|
| kill-primary + kill-secondary | Self-heal under simultaneous failure |
| kill-primary + reconfigure | Reconfigure rollback during failover |
| kill-primary + network-partition | Split-brain prevention |
| kill-kbagent + kill-primary | Orchestration layer + data layer simultaneous failure |
| 3× rapid-kill + writes-during-chaos | HA idempotency under sustained pressure |
| network-partition + scale-out | Topology change during quorum stress |

## Engine-specific adapter checklist

Each addon test owner provides 3 adapter scripts:

- `engine-adapter-load.sh <table> <rows>` — bulk insert N rows
- `engine-adapter-metrics.sh` — capture connection count / replica lag / CPU / memory snapshot
- `engine-adapter-bench.sh` — invoke engine-native bench tool (sysbench / pgbench / etc.)

OB / SQL Server / MariaDB / Valkey / Oracle each fork these stubs.

## Adoption recommendation

Apply templates incrementally — don't try all 5 at once:

1. **Week 1**: N-multiplier wrapper on existing cases (Template 1) — easy win, exposes intermittent bugs immediately
2. **Week 2**: Multi-chaos overlay (Template 5) on top 3 combinations — exposes state machine bugs
3. **Week 3**: 24h soak with random chaos (Template 2) — replaces fixed longrun
4. **Week 4**: Data scale + Concurrency baselines (Templates 3 + 4) — exposes scale-related issues

Each week produces evidence-pack tar.gz + sha256 attestation per Rule 1.

## Cross-addon coordination

Cross-addon test methodology lead maintains the templates above as engine-neutral skeleton. Per-addon test engineers fork into their runners. New intensity patterns surfaced by any line should be sedimented back to this guide for cross-line reuse.

Reach out to the cross-addon test methodology lead if you want code review on a specific intensity case before commit.
