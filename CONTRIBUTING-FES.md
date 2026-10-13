# FES router fork — development conventions & defect context

Working notes for the `core`/`fix-*` branch tree in the **FES fork**
(`Fulton-Engineering-Services/router`), used to develop and prove fixes for the
PD two-stage concurrency defect before any upstream PR.

## Branch model

- `main` — upstream tracking branch (`upstream/main` = `vllm-project/router` main).
- **`core`** — our integration branch, based on **tag `v0.1.16`** (`db02f35`,
  annotated tag object `31f6b462`). This is the exact binary the workspace runs
  and logged the defect against, so `core` preserves repro fidelity. A later,
  separate step merges `core` → a `main`-tracking branch before any PR.
- **`fix-<slug>`** — one branch per root cause, each in its own git worktree
  under `<repo>/.git-worktrees/<branch>/`, branched from `core`. Each is
  developed + self-reviewed, then merged back into `core`.

DCO: every commit MUST be `git commit -s` (Signed-off-by), identity
`J. Patrick Fulton <patrick@fultonengineeringservices.com>`. No commit-msg hook
is installed; enforce manually.

## The defect (evidence report)

Full analysis: `fulton-infra-ecosystem/.omp/reviews/router-pd-concurrency-defect.md`
(produced from codekeeper repo `vllm-router`). Summary:

**Symptom.** With ≥2 concurrent chat requests via the router, one returns 200
and the other hangs indefinitely; after a hang the router wedges for all traffic
until the container is restarted. Both vLLM engines are provably idle (py-spy)
during the hang; decode's own access log shows its POSTs returned 200; prefill's
access log shows only ONE POST. The deployed binary is tag `v0.1.16`; the report
maps every step to `v0.1.16` source file:line (Section 8 addendum).

**Root causes assigned to fix branches:**

| branch | root cause | key source (v0.1.16 `src/routers/http/vllm_pd_router.rs` unless noted) |
|---|---|---|
| `fix-timeout-deadline` | **R1** — the two-stage flow has no deadline of its own; the only bound is the shared client `.timeout(request_timeout_secs)` (default 1800 s, `src/config/types.rs:897` → `src/server.rs:961`). A stalled leg blocks ~30 min silently. | `process_vllm_two_stage_request` (`:1280`); prefill await (`≈:1400`); decode await (`≈:1560+`); decode-status log (`:1583`) |
| `fix-mooncake-handshake` | **R2** — router hand-builds prefill `kv_transfer_params` (`:205-209`, `do_remote_decode`+`transfer_id`) and decode params (`:341-354`, `do_remote_prefill`+`remote_bootstrap_addr`+`remote_engine_id`) with no contract validation (missing params only **warn**, `:1586-1590`, then proceed). In Mooncake PD the prefill request is held open until its KV transfer completes. | `build_prefill_kv_transfer_params` (~`:205`), `build_decode_kv_transfer_params` (~`:341`), `query_mooncake_bootstrap` (`:97-160`) |
| `fix-concurrent-dispatch` | **R3** — direct-URL mode takes the SEQUENTIAL prefill→decode path: `try_build_concurrent_requests` (`:1680`) returns `None` unless `kv_connector == Nixl` (`:1686`). Our deployment is Mooncake, so the router has no concurrency here. HEAD added a `tokio::join!` path (`process_vllm_two_stage_request_discovered`, HEAD `:879`) not wired for direct `--prefill/--decode`. | `try_build_concurrent_requests` (`:1680-1697`), `process_concurrent_two_stage` (`:1700+`) |
| `fix-limiter-leak` | **post-hang wedge** — a never-completing handler leaks its concurrency-limiter token (returned only after `next.run(request).await` resolves) and leaves worker load counters incremented, so one hung request can wedge capacity. | `src/middleware.rs:466-540` (token acquired `:507-509`, returned `:511-514`); worker load `src/core/worker.rs:459-470` |

**Ruled out** (do not spend effort): per-worker in-flight slot/semaphore,
connection-pool-of-1, lock held across the flow, dropped-sender channel.
Token-bucket limiter default capacity is 32768, so one leaked token alone does
not wedge all traffic — the persistent wedge is R2 compounded by R1.

## Build & test

- Rust toolchain present locally (`cargo 1.98.1`). Isolated build per worktree:
  `cd .git-worktrees/<branch> && cargo build --release && cargo test`.
- Do NOT touch another branch's worktree. Shared object DB is fine; each
  worktree has its own index/untracked files.
- Ship-to-cluster image build: `infra/cluster-config/scripts/build-router-image.sh`
  (builds `Dockerfile.router` from the fork checkout, publishes
  `10.100.170.3:5000/vllm-router-fork:<tag>-<sha>`), then set
  `VLLM_ROUTER_IMAGE_NAME=vllm-router-fork` + tag in `cluster.env` and relaunch.

## Reviewer subagent contract

Each fix subagent spawns ONE reviewer (`reviewer` agent) before declaring done.
Reviewer checks: correctness vs the assigned root cause; no regression to the
other three; DCO sign-off present; `cargo build`/`cargo test` green; evidence
cited file:line against `core` (v0.1.16). Reviewer writes findings next to the
work (worktree-local), fix subagent addresses valid findings, then the branch is
handed to the parent for merge into `core`.
