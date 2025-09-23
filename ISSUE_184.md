# Issue 184 — Properly Benchmark `on_initialize` Hook Weight

## 1. Executive Summary
- GitHub issue [#184](https://github.com/paritytech/polkadot-sdk/issues/184) highlights that several FRAME pallets return `BlockWeights::get().max_block` from `on_initialize`. This hard-coded full-block weight blocks inclusion of inherents and signed extrinsics whenever those hooks fire.
- The behaviour is confirmed in the current repository snapshot for `pallet-session` (`[substrate/frame/session/src/lib.rs:608](substrate/frame/session/src/lib.rs#L608)`) and `pallet-democracy` (`[substrate/frame/democracy/src/lib.rs:568](substrate/frame/democracy/src/lib.rs#L568)`). Both pallets skip proper benchmarking and fall back to the worst-case constant.
- Downstream incidents, notably asset-hub Rococo stalling after the 1.12.0 upgrade ([issue #4559](https://github.com/paritytech/polkadot-sdk/issues/4559)), and follow-up tracking ([issue #4573](https://github.com/paritytech/polkadot-sdk/issues/4573)), demonstrate the operational impact.
- A low-level fix inside this repository requires full benchmarking coverage and runtime integration changes for the affected pallets, plus a follow-up audit for other pallets that still rely on the same hack.

## 2. Detailed Problem Statement
- `on_initialize` is executed before any inherents or extrinsics. Its return value reserves part of the per-block weight budget; returning `max_block` leaves no weight for inherents like `set_validation_data`, which are mandatory for parachain blocks.
- `pallet-session` currently does the following:
  - Checks `T::ShouldEndSession::should_end_session(n)` and, if true, rotates the session.
  - Regardless of the actual operations executed, returns `T::BlockWeights::get().max_block` (`[substrate/frame/session/src/lib.rs:612](substrate/frame/session/src/lib.rs#L612)` → `[substrate/frame/session/src/lib.rs:616](substrate/frame/session/src/lib.rs#L616)`).
  - This behaviour was introduced as a temporary measure in [substrate#5738](https://github.com/paritytech/substrate/pull/5738) and never replaced.
- `pallet-democracy` delegates `on_initialize` to `begin_block` (`[substrate/frame/democracy/src/lib.rs:568](substrate/frame/democracy/src/lib.rs#L568)`). Whenever it launches a new referendum or finalises expiring ones, it assigns `weight = max_block_weight` (`[substrate/frame/democracy/src/lib.rs:1644](substrate/frame/democracy/src/lib.rs#L1644)` → `[substrate/frame/democracy/src/lib.rs:1660](substrate/frame/democracy/src/lib.rs#L1660)`).
- The situation was tolerable while `CheckWeight` ignored proof-of-validity size and while inherents were permissive, but runtime 1.12 tightened enforcement (see PR [polkadot-sdk#4326](https://github.com/paritytech/polkadot-sdk/pull/4326)): inherents must now fit into the remaining weight budget, and the combined weight/length check fails whenever the hook reports full weight usage.

## 3. Current State Analysis
### 3.1 `pallet-session`
- `rotate_session` (`[substrate/frame/session/src/lib.rs:716](substrate/frame/session/src/lib.rs#L716)`) performs several operations that scale with the validator set:
  - Calls `SessionManager::end_session` and `SessionManager::start_session`.
  - Rewrites `Validators`, `QueuedKeys`, and `QueuedChanged` storage items.
  - Iterates over validators to compare old vs. new keys and to load queued keys, potentially triggering storage reads per validator.
  - Invokes `SessionHandler::on_new_session`, which may perform additional work in runtime components (e.g., BABE, GRANDPA, parachain staking).
- Existing benchmarks (`[substrate/frame/session/benchmarking/src/inner.rs](substrate/frame/session/benchmarking/src/inner.rs)`) stop short of benchmarking `rotate_session`. Relevant cases are tagged `#[benchmark(extra)]`, and a comment notes broken assumptions about key encodings (`[substrate/frame/session/benchmarking/src/inner.rs:171](substrate/frame/session/benchmarking/src/inner.rs#L171)` → `[substrate/frame/session/benchmarking/src/inner.rs:177](substrate/frame/session/benchmarking/src/inner.rs#L177)`). No generated weight covers the hook path.
- The hacky `max_block` return is therefore the only guardrail. Networks with frequent session changes (e.g., every hour on Kusama) experience periodic blocks that cannot include any other transactions.

### 3.2 `pallet-democracy`
- `begin_block` iterates through referenda queues and scheduling logic. It sets `weight = max_block_weight` when:
  - `launch_next` succeeds, i.e., a new referendum is launched (`[substrate/frame/democracy/src/lib.rs:1644](substrate/frame/democracy/src/lib.rs#L1644)` → `[substrate/frame/democracy/src/lib.rs:1649](substrate/frame/democracy/src/lib.rs#L1649)`).
  - Any referendum reaches its end and is baked, triggering the `for` loop and setting the weight to the maximum (`[substrate/frame/democracy/src/lib.rs:1656](substrate/frame/democracy/src/lib.rs#L1656)` → `[substrate/frame/democracy/src/lib.rs:1660](substrate/frame/democracy/src/lib.rs#L1660)`).
- Benchmarks exist for `on_initialize_base` and `on_initialize_base_with_launch_period` (`[substrate/frame/democracy/src/weights.rs:310](substrate/frame/democracy/src/weights.rs#L310)` and `[substrate/frame/democracy/src/weights.rs:336](substrate/frame/democracy/src/weights.rs#L336)`). They model storage access scaling with `r` (number of referenda). However, the runtime code does not use these weights to bound the hook.
- Democracy workloads vary significantly between chains. Without parameterised weights tied to configuration (e.g., maximum referenda tracked at once), the current implementation swings between underestimating or over-reserving the block budget.

### 3.3 Frame `CheckWeight`
- `CheckWeight` enforces both weight and proof-size limits (`[substrate/frame/system/src/extensions/check_weight.rs:199](substrate/frame/system/src/extensions/check_weight.rs#L199)`). It now considers the aggregate `max_block` minus what `on_initialize` consumes, meaning inherents fail inclusion when no weight remains.
- In the Rococo incident (#4559), the runtime logged `Extrinsic exceeds total pov size` for inherents, and block production halted because `set_validation_data` could not be included after `on_initialize` consumed the entire budget.

## 4. Related Incidents and References
- [Issue #184](https://github.com/paritytech/polkadot-sdk/issues/184): Original request to benchmark and return actual weight for the hook.
- [Issue #4573](https://github.com/paritytech/polkadot-sdk/issues/4573): Follow-up bug noting that pallets should not return `max_block` in `on_initialize`; explicitly lists `pallet-session`.
- [Issue #4559](https://github.com/paritytech/polkadot-sdk/issues/4559): Production outage on asset-hub Rococo tied to the same root cause.
- Historical references: `pallet-session` hack introduced in substrate commit [`c0793b58c509f352b6006e402350ac9ad8017823`](https://github.com/paritytech/substrate/blob/c0793b58c509f352b6006e402350ac9ad8017823/frame/session/src/lib.rs#L571) (link cited in the issue body) and the corresponding democracy code at [`frame/democracy/src/lib.rs#L1625-L1626`](https://github.com/paritytech/substrate/blob/c0793b58c509f352b6006e402350ac9ad8017823/frame/democracy/src/lib.rs#L1625-L1626).

## 5. Solution Space
### 5.1 `pallet-session`
- **Benchmark coverage**
  - Extend the benchmarking harness to simulate session rotation with realistic validator counts, queued validator changes, disabled validator lists, and session key updates.
  - Capture both the read/write cost and proof-size contributions from `Validators`, `QueuedKeys`, `QueuedChanged`, `DisabledValidators`, and any `SessionHandler` interactions.
- **Weight function design**
  - Introduce a new `WeightInfo::rotate_session(validators: u32, queued: u32, keys_changed: u32)` (or similar signature) that reflects benchmark results.
  - Account for constant cost of calling `ShouldEndSession::should_end_session`, which is currently charged as part of the “empty block” base weight.
- **Runtime integration**
  - Replace the unconditional `max_block` return with the benchmarked weight calculation plus defensive checks (e.g., saturate at `max_block` and emit a log if limits are exceeded).
  - Update runtime configuration to expose limits used during benchmarking (for example, `MaxValidators` in staking, maximum queued sessions).

### 5.2 `pallet-democracy`
- **Benchmark coverage**
  - Build harness helpers to seed `PublicProps`, `ReferendumInfoOf`, and `MaturingReferenda` with controllable sizes. Cover scenarios: launching a referendum, tallying maturing referenda, and idle blocks.
  - Measure both weight and proof-size impact when moving the worst-case number of referenda from `Ongoing` to `Finished`.
- **Weight function design**
  - Extend `WeightInfo` with functions representing each branch, e.g., `on_initialize_launch(r)` and `on_initialize_bake(r)`.
  - Sum the relevant weight contributions in `begin_block` instead of forcing `max_block`.
  - Continue to include the existing base weights for queue scanning operations.
- **Runtime integration**
  - Introduce configuration constants (if missing) that cap referenda counts or queue lengths to match benchmark assumptions.
  - Add warnings or `debug_assert!` that trigger if runtime state exceeds benchmarked bounds.

### 5.3 Cross-cutting safeguards
- Audit other pallets for similar patterns (`rg "max_block" substrate/frame -g'*.rs'` already reveals candidates such as `pallet-elections-phragmen` tests and migrations). Prioritise any active hook returning `max_block` in production runtimes.
- Consider adding a lightweight `try-runtime` check that panics when `on_initialize` returns weight close to or exceeding `max_block`, ensuring CI catches regressions.
- Explore reserving a small inherent-only budget in block weight configuration, though this should complement, not replace, correct benchmarking.

## 6. Implementation Challenges
- **Complex benchmarks:** Rotating a session requires emulating staking state, session managers, and handler hooks. Reworking the current extra benchmarks (which already note broken assumptions) will need careful fixture design.
- **Generic dependencies:** The session pallet is generic over runtime-specific `SessionManager` logic. Benchmarks must either use the default mock or provide a representative implementation that exercises storage writes comparable to production.
- **Branch explosion in democracy:** `begin_block` mixes multiple responsibilities. Splitting the logic to charge weights accurately without altering behaviour presents a refactoring challenge.
- **Proof-size accounting:** Updated `CheckWeight` considers both execution time and PoV size. Benchmarks must capture proof-size deltas, not only the `ref_time` component, to avoid under-estimating real costs.
- **Runtime configurability drift:** Values like `MaxReferenda`, `ValidatorCount`, and session lengths differ across runtimes (Polkadot, Kusama, Collectives, parachains). Benchmarks must either cover the worst common case or expose configuration-dependent parameters.
- **Ensuring backward compatibility:** Any change to reported weights affects transaction fee calculation and may require runtime migrations or updates to benchmarking artefacts consumed by downstream chains.

## 7. Proposed Work Breakdown (Low-Level)
1. **Requirements gathering (1–2 days)**
   - Inventory all runtimes using `pallet-session` and `pallet-democracy`; record validator caps, session lengths, referendum queue limits.
   - Confirm runtime maintainers’ expectations for maximum on-chain limits.
2. **`pallet-session` benchmarking overhaul (multi-day)**
   - Fix the existing `#[benchmark(extra)]` that manipulates session keys (`[substrate/frame/session/benchmarking/src/inner.rs](substrate/frame/session/benchmarking/src/inner.rs)`).
   - Add a benchmark that drives `rotate_session` through the worst case: large validator set, high churn, disabled validators reset, `QueuedKeys` full.
   - Generate new weight files with `frame-omni-bencher` and update the pallet to consume the weights.
   - Adapt runtimes (Polkadot, Kusama, Collectives, parachains) to use the new weight function in `on_initialize`.
3. **`pallet-democracy` benchmarking improvements (multi-day)**
   - Create builders that enqueue referenda and mimic the launch schedule.
   - Benchmark the distinct branches (launch vs. bake) and update `WeightInfo` accordingly.
   - Refactor `begin_block` so `on_initialize` sums benchmarked weights instead of assigning `max_block`.
   - Update runtimes and regenerate weight files.
4. **Regression protection**
   - Add `debug_assert!` or telemetry counters in both pallets to report when weights approach configured limits.
   - Implement integration tests (or try-runtime scripts) that simulate a block containing both hooks plus inherents, ensuring block production succeeds.
5. **Documentation & communication**
   - Update FRAME docs (e.g., `[substrate/frame/session/README.md](substrate/frame/session/README.md)`) with the new benchmarking requirements.
   - Leave GitHub issue/PR comments summarising the fix and linking to the updated weight files.
   - Notify parachain teams that run custom runtimes to adopt the new weights.

## 8. Validation Strategy
- **Unit/bench tests:** Ensure new benchmarks compile and run across Rust channels used in CI.
- **Runtime tests:** Use `try-runtime on-runtime-upgrade` and `try-runtime on-runtime-upgrade --execute-blocks` to verify blocks containing session rotation and heavy democracy activity execute within weight limits.
- **Integration:** Spin a local relay-chain/parachain environment with patched runtime, enabling logging around `on_initialize` weight usage to confirm inherents still fit.
- **Monitoring:** After deployment, monitor block weight metrics (telemetry) to ensure observed weights align with benchmark expectations.

## 9. Open Questions
- What validator-count upper bound should benchmarks target (e.g., 400 for Polkadot, 1000 for Kusama)?
- How many referenda can realistically mature in a single block, and should the runtime enforce that limit explicitly?
- Does any downstream tooling depend on the old behaviour (e.g., assuming session rotation reserves the full block)?
- Should `CheckWeight` reserve a fixed inherent budget irrespective of reported `on_initialize` weight as an additional safeguard?
- Are there other pallets (e.g., staking, elections, scheduler) that must be patched concurrently to avoid similar block stalls?

## 10. Deliverables Checklist
- [ ] Updated benchmarking code for `pallet-session` and generated `weights.rs` files.
- [ ] Updated benchmarking code for `pallet-democracy` and regenerated weights.
- [ ] Refactored `on_initialize` implementations using benchmarked weights.
- [ ] Tests or try-runtime scripts validating inherents inclusion under worst-case hook execution.
- [ ] Documentation and issue updates capturing assumptions, limits, and follow-up audit results.

---
Prepared from scratch against the repository contents under `/Users/j.leung/workspace/code/blockchain/polkadot/polkadot-sdk` and the referenced GitHub issues as of the current date.
