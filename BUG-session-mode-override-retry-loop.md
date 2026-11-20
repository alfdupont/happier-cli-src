# Session mode override retried forever, saturating the network link

**Status:** upstream defect, reproducible; observed in `happier-cli` 0.2.20 and confirmed present in 0.2.14
**Severity:** high (silent, unthrottled network amplification; multi-GB data waste, link saturation)
**Component:** CLI / `createSessionModeOverrideSynchronizer`
**Not caused by:** the local `copilot-per-model-reasoning-effort.patch` in this repository (see [Fork provenance](#fork-provenance-this-patch-is-not-involved))

## Summary

When a Copilot session is started with an `--agent-mode` value that the provider's catalog does not allow, `setSessionMode()` fails permanently. The session-mode override synchronizer treats that permanent failure as transient: it keeps the override `pending` and retries it on every metadata sync, with **no backoff, no attempt cap, and no error detail logged**.

A single idle session produced **123,032 retry attempts over 7 hours**, sustaining roughly **1.7 MB/s down and 556 KB/s up** with no user-visible activity, until the process was killed.

## Impact (measured)

| Metric | Value |
| --- | --- |
| Retry attempts | 123,032 (`attempt` 1 → 123,032) |
| Duration | 10:17:26 → 17:17:37 (~7h) |
| Retry log lines | 123,032 of 123,082 total (99.96%) |
| Gap between attempts | avg 205 ms; min 1 ms; 105,956 gaps under 100 ms |
| Sustained rate, session idle | 1.7 MB/s down, 556 KB/s up |
| Implied payload per attempt | ~138 KB down, ~45 KB up |
| Estimated waste, final idle stretch only | ~15 GB down, ~5 GB up |
| Diagnostics captured in 7h | none (error object discarded) |

The session was genuinely idle for the final ~2.5 hours: its last real turn was at 14:36:14. During that stretch the loop ran **unbroken**, with no gap over 60 seconds, so the traffic looked like an unexplained background download.

## Environment

- Package: `happier-cli` 0.2.20-1, built from this repository against the pinned immutable tag `cli-v0.2.20`.
- Affected process: `node /opt/happier-cli/package-dist/index.mjs copilot --happy-starting-mode remote --started-by daemon --permission-mode yolo --agent-mode https://agentclientprotocol.com/protocol/session-modes#autopilot --agent-mode-updated-at 1794215839884 --model claude-opus-5.5`
- Session id: `cmw91sbvt04apqj5dkrd2lr2q`, provider `copilot`, providerMode `acp`
- Remote peer: `[2001:41d0:304:200::58c]:443`

## Root cause

Source of truth: `/opt/happier-cli/package-dist/createAcpRuntime-DKq1SnV2.mjs`

`createSessionModeOverrideSynchronizer` (:66) keeps `pending`, `lastAppliedUpdatedAt`, `lastAttemptedUpdatedAt` and `lastAttemptNumber`. In `applyPendingIfPossible` (:73):

1. `attempt` is computed and incremented for the *same* `updatedAt` forever — it is only logged, never used to back off or to stop.
2. The apply path is `runtime.setSessionMode(modeId)`, and its failure handler is `.catch(() => { ... })` (:95). **The caught error is discarded**, so the reason is unrecoverable from logs. `lastAppliedUpdatedAt` never advances, so `pending` is never cleared.
3. `syncFromMetadata` (:109) recomputes the same pending override from the metadata snapshot and calls `void applyPendingIfPossible()` (:122). Because the pending entry survives the failure, every metadata snapshot re-arms the attempt — a self-sustaining loop with no delay.

Net effect: a permanently invalid instruction is retried at transport speed, forever.

## Trigger: an `--agent-mode` value the provider forbids

The Copilot descriptor in the provider catalog is:

```
copilot: {
  ...
  defaultMode: "default",
  allowedModes: ["default"]
}
```

`https://agentclientprotocol.com/protocol/session-modes#autopilot` is therefore not an admissible mode for Copilot, whose mode switch is declared as `runtimeSwitch: "acp-setSessionMode"`. The CLI accepted the flag at launch instead of rejecting it, and the runtime then tried to apply it 123,032 times.

Corroborating signals from the same log:

- `updatedAt: 1794215839884` decodes to 2026-11-09 10:17:19 CET, two seconds before the process started — it originates from the launch flags, never changes, and so can never converge.
- `deferredReason: "pending_version_mismatch"` logged at 10:17:27, one second after the loop began.
- `[PUSH] Push service did not accept every recipient {"acceptedCount":0,"failedCount":1}` six times.
- Only at teardown did the server report `phase: "terminal_failure"` for both `update-metadata` and `update-state` — a condition that should have stopped the sync instead of continuing it.

## Timeline

| Time | Event |
| --- | --- |
| 10:17:19 | `--agent-mode-updated-at` timestamp minted |
| 10:17:21 | process start |
| 10:17:22.593 | session socket `online` (only once; never reconnected) |
| 10:17:26.906 | first `Failed to apply session mode override`, `attempt:1` |
| 10:17:27.198 | `pending_version_mismatch` |
| 12:01 – 14:36 | normal user turns interleaved with retries |
| 14:36:14 | last real turn; session idle from here |
| 14:47:54 → 17:17:37 | loop runs unbroken, no gap over 60 s, ~12 attempts/s |
| 17:17:34.940 | `shutting_down` (intentional) |
| 17:17:37 | `terminal_failure` on `update-metadata` and `update-state` |

Attempts per 30 minutes show it is event-driven rather than a fixed timer, and that it ran hardest while idle: `13:00 → 17,793`, `15:30 → 19,493`, `16:30 → 20,847`, `17:00 → 12,507`.

## Fork provenance: this patch is not involved

This repository builds upstream releases plus one local patch. That patch is provably not on the failure path:

- `PKGBUILD:23` declares exactly one local patch, `copilot-per-model-reasoning-effort.patch`, applied in `prepare()` (`PKGBUILD:42`, `PKGBUILD:52`) with `--forward --fuzz=0` onto the sha256-pinned immutable release tree.
- The patch edits only two files, 4 hunks total: `package-dist/api-C6NpPUiK.mjs` and `package-dist/withMcpTimeout-B6xGJAfL.mjs`.
- The retry logic lives in `createAcpRuntime-DKq1SnV2.mjs`, which the patch never touches.
- The same defect exists in the previous packaged release: `createAcpRuntime-Dn84uydZ.mjs:65` and `:98` inside `happier-cli-0.2.14-1-x86_64.pkg.tar.zst` contain the identical synchronizer (0.2.14 logged it via `logger.debug`, 0.2.20 via `logger.infoFile`).
- The patch's only change to the Copilot descriptor is `acpModelOptionsScope: "per_model"` (`withMcpTimeout-B6xGJAfL.mjs:2236`). That is a *model-options* field consumed by `createModelOverrideSynchronizer` (`createAcpRuntime-DKq1SnV2.mjs:187`, `:220`, `:233`), a separate code path from the session-mode synchronizer. `allowedModes: ["default"]` appears in the patch as unchanged context and is also present in 0.2.14.
- The one conceivable interaction — the per-model probe churning metadata and raising retry *frequency* — is not supported by the evidence: no `[agentModelsProbe]` warning appears anywhere in the log, the probe runs at startup, and it cannot sustain a 2.5-hour loop.

## Reproduction

1. Install `happier-cli` 0.2.20 (this repository's `PKGBUILD`).
2. Start a Copilot remote session requesting a mode outside `allowedModes`:
   `happier copilot --happy-starting-mode remote --agent-mode https://agentclientprotocol.com/protocol/session-modes#autopilot --permission-mode yolo`
3. Leave the session idle.
4. Observe `~/.happier/logs/*-pid-<pid>.log` filling at ~5 lines/s with `attempt` climbing, and `ss -tunapHei` showing several MB/s on that pid.

Control: relaunch without `--agent-mode` (or with `default`) and the loop does not occur. Building 0.2.20 with the `prepare()` patch removed and repeating the command should reproduce the loop as well, confirming it is upstream.

## Suggested upstream fixes

1. **Backoff and cap.** Use the already-tracked `attempt` counter: exponential backoff with jitter, and stop retrying after a bounded number of attempts on the same `updatedAt`.
2. **Do not swallow the error.** `.catch(() => ...)` should capture `error` and log its name, message and any status code, then escalate to `logger.warnFile` after the first few failures.
3. **Treat terminal failures as fatal.** A `terminal_failure` for session metadata or agent state must clear `pending` and stop the synchronizer rather than re-arm it.
4. **Fail fast on invalid flags.** Validate `--agent-mode` against the provider's `allowedModes` at launch and exit with a clear message, instead of accepting a value the backend can never apply.
5. **Guard against self-rearm.** `syncFromMetadata` should not re-trigger an apply for an `updatedAt` that has already failed, unless a genuinely newer override arrives.

## Workarounds

- Do not pass `--agent-mode` values outside the provider's `allowedModes`; for Copilot use `default`.
- If a session starts looping, kill it — the loop dies with the process and traffic stops immediately.
- Housekeeping: stale runner locks such as `~/.happier/tmp/session-runner-locks/cmw91sbvt04apqj5dkrd2lr2q.json` can linger after a killed session.

## Detection

The loop is easy to catch early because it dominates the log:

```sh
# retry attempts for the newest session log
L=$(ls -t ~/.happier/logs/*.log | head -1)
printf 'loop lines: %s of %s\n' "$(grep -c 'Failed to apply session mode override' "$L")" "$(wc -l < "$L")"
grep -o '"attempt":[0-9]*' "$L" | tail -1
```

Any process whose rate stays flat for minutes while its session is idle, or any `attempt` counter above ~50, indicates this defect.
