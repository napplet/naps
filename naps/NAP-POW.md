NAP-POW
=======

NIP-13 Proof-of-Work Miner
--------------------------

`draft`

**NAP ID:** NAP-POW
**Domain:** `pow`
**Depends:**
- `identity` — layering · required — the runtime stamps the user pubkey (identity domain) into every mined event
- `relay` — capability · optional — a `mine` result may be published via `relay`
- `outbox` — capability · optional — `mineAndPublish` uses outbox-aware fanout; same publish-consent policy as `outbox`
**Web binding (NIP-5D):** `window.napplet.pow` · `shell.supports("pow")`

## Description

NAP-POW provides napplets with a shell-mediated [NIP-13](https://github.com/nostr-protocol/nips/blob/master/13.md) proof-of-work miner: the napplet hands over an event template and a target difficulty, the runtime mines a `nonce` until the event id carries the required number of leading zero bits, and returns the mined event. A napplet could grind nonces itself, but it runs in a single sandboxed context where a tight hashing loop blocks its own UI; the runtime can spread the search across **1..n web workers** (the web binding) or native threads (other bindings), report aggregate and per-worker progress, and arbitrate CPU across competing jobs. Difficulty mining is a long-running, cancellable, pausable operation, so NAP-POW models each request as a **job** the napplet can observe and control, not a single blocking call.

Mining commits the user's pubkey and `created_at` into the event id (NIP-13), so the runtime — which owns identity ([NAP-IDENTITY](https://github.com/napplet/naps/pull/12)) and signing — stamps those fields before the search begins; the napplet supplies only `kind`, `content`, and `tags`. Two entry points cover the common cases: `mine` returns the mined event to the napplet for it to use or publish later, and `mineAndPublish` mines, signs, and publishes the result through the shell's outbox-aware fanout in one step. As with all NAPs, napplets never receive signing keys; signing and publishing remain shell operations.

## API Surface

```typescript
interface NappletPow {
  mine(template: EventTemplate, target: number, opts?: PowOptions): PowJob;
  mineAndPublish(template: EventTemplate, target: number, opts?: PowOptions): PowJob;
  queue(): Promise<PowJobSummary[]>;                 // via pow.queue
  job(jobId: string): Promise<PowProgress>;          // via pow.job — snapshot of one job
  hashrate(): Promise<PowHashrate>;                  // via pow.hashrate — miner-wide, live
  cancel(jobId: string): Promise<boolean>;           // via pow.cancel
  pause(jobId?: string): Promise<void>;              // via pow.pause  — one job, or all if omitted
  resume(jobId?: string): Promise<void>;             // via pow.resume — one job, or all if omitted
  formatHashRate(hashesPerSecond: number): string;   // sugar: 1500 -> "1.5 kH/s" (local, no message)
}

interface EventTemplate {
  kind: number;
  content: string;
  tags?: string[][];                 // a ["nonce", …] tag, if present, is replaced by the shell
  created_at?: number;               // hint; the shell stamps the committed value at job start
}

interface PowOptions {
  workers?: number;                  // desired worker count (1..n); a hint, the shell caps it
  priority?: number;                 // queue-ordering hint; higher runs sooner. Shell MAY ignore. Default 0
  timeoutMs?: number;                // max time spent MINING before the job ends "error"; excludes queue wait
  commitCreatedAt?: boolean;         // stamp created_at when mining begins and freeze it for the search (default true)
}

interface PowJob {
  jobId: string;                     // shell-unique handle for this job
  target: number;                    // requested difficulty (leading zero bits)
  started: Promise<void>;            // resolves when the job leaves the queue and mining begins
  completed: Promise<PowResult>;     // resolves on success; rejects on cancel/timeout/error
  on(event: "state", cb: (s: PowStateChange) => void): void;     // queued -> mining -> paused/done/cancelled/error
  on(event: "progress", cb: (p: PowProgress) => void): void;
  on(event: "done", cb: (r: PowResult) => void): void;
  on(event: "error", cb: (e: PowError) => void): void;
  cancel(): Promise<boolean>;
  pause(): Promise<void>;
  resume(): Promise<void>;
}

interface PowStateChange {
  jobId: string;
  state: PowState;
  position?: number;                 // 0-based queue position while state === "queued"
}

interface WorkerStat {
  workerId: number;
  bestPow: number;                   // best leading-zero-bits this worker has found
  hashes: number;                    // hashes tried by this worker
  hashRate: number;                  // H/s for this worker
}

interface PowProgress {
  jobId: string;
  target: number;
  state: PowState;
  bestPow: number;                   // best difficulty found across all workers
  bestNonce?: string;
  hashes: number;                    // total hashes tried for this job
  hashRate: number;                  // H/s for this job (sum of its workers)
  workers: WorkerStat[];             // one entry per active worker on this job
  elapsedMs: number;
}

interface PowHashrate {
  hashRate: number;                  // total H/s across the whole miner
  workers: number;                   // active worker count
  perWorker: { workerId: number; hashRate: number }[];
  byJob: { jobId: string; hashRate: number }[];
}

interface PowResult {
  jobId: string;
  ok: boolean;
  event: NostrEvent;                 // mined event; id meets target; carries the nonce tag
  pow: number;                       // achieved difficulty (>= target)
  nonce: string;
  hashes: number;
  elapsedMs: number;
  published?: PowPublishResult;      // present only for mineAndPublish
  error?: string;
}

interface PowPublishResult {
  eventId: string;
  relays: Record<string, boolean>;   // outbox fanout outcome per relay
}

interface PowJobSummary {
  jobId: string;
  target: number;
  state: PowState;
  priority: number;
  position?: number;                 // 0-based queue position while state === "queued"
  bestPow: number;
  hashRate: number;
  kind: number;
  mode: "mine" | "mineAndPublish";
}

interface PowError { jobId: string; error: string; }

type PowState = "queued" | "mining" | "paused" | "done" | "cancelled" | "error";
```

**`mine(template, target, opts?)`** — **Submits** a mining job and returns a `PowJob` handle immediately. The job is *enqueued*: depending on the shell's concurrency policy it begins mining at once or waits behind other jobs, reporting `state: "queued"` (with a queue `position`) until a worker slot frees. `job.started` resolves when mining actually begins; `job.completed` resolves with the mined **unsigned** event once `target` is met. The handle is valid the moment it is returned — `cancel`, `pause`, and queue inspection all work while the job is still queued.

When mining begins (not at submit time), the shell stamps the user pubkey and `created_at`, replaces any `nonce` tag with `["nonce", "<nonce>", "<target>"]`, and searches for a nonce whose event id has at least `target` leading zero bits, fanned across `opts.workers` workers (capped by shell policy). Stamping `created_at` at *mining start* — not enqueue — keeps a job that waited in the queue from committing a stale timestamp; `commitCreatedAt: true` (default) then freezes it for the whole search. `opts.timeoutMs` bounds **mining** time only and does not count queue wait. The mined event is returned unsigned; the napplet may publish it through [NAP-RELAY](https://github.com/napplet/naps/pull/2)/[NAP-OUTBOX](https://github.com/napplet/naps/pull/32), and the shell MUST sign the committed event **verbatim**, since altering `pubkey`, `created_at`, `tags`, `content`, or `kind` would invalidate the proof of work.

**`mineAndPublish(template, target, opts?)`** — Identical lifecycle and queue semantics to `mine` (submit → `queued` → `mining` → `done`), but on success the shell signs the mined event and publishes it using outbox-aware fanout (the user's write relays, plus recipient inbox relays for directed events). There is **no relay argument**: relay selection is shell policy, identical to [NAP-OUTBOX](https://github.com/napplet/naps/pull/32) `publish`. `PowResult.published` carries the fanout outcome; `job.completed` rejects with `"publish failed"`/`"publish denied"` if the proof of work succeeded but publishing did not.

**`queue()`** — Returns a summary of all of the napplet's tracked jobs (queued, mining, paused) including each job's `priority` and, for queued jobs, its `position`. For a miner/status napplet building a queue view. Ordering reflects the shell's current schedule.

**`job(jobId)`** — Returns a one-shot `PowProgress` snapshot for a single job: its `target`, `bestPow`, and per-worker `bestPow` / `hashRate`.

**`hashrate()`** — Returns the live miner-wide hash rate: the total, the per-worker breakdown, and the per-job breakdown. Rates are raw hashes per second; scaling to `kH/s`, `MH/s`, `GH/s` is a display concern — `formatHashRate` is local SDK sugar and sends no message.

**`cancel(jobId)`** — Stops a job and frees its workers. Resolves `true` if a job was running. The job's `completed` promise rejects with `"cancelled"`.

**`pause(jobId?)` / `resume(jobId?)`** — Suspend or continue a job, preserving its best-nonce-so-far and hash count. With `jobId` omitted, the whole miner pauses/resumes (e.g. to yield the CPU). A paused job reports `state: "paused"` and `hashRate: 0`.

## Wire Protocol

`pow.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `pow.mine` | napplet -> shell | `id`, `jobId`, `template`, `target`, `options?` |
| `pow.mine.result` | shell -> napplet | `id`, `jobId`, `accepted`, `state`, `position?`, `error?` |
| `pow.mineAndPublish` | napplet -> shell | `id`, `jobId`, `template`, `target`, `options?` |
| `pow.mineAndPublish.result` | shell -> napplet | `id`, `jobId`, `accepted`, `state`, `position?`, `error?` |
| `pow.state` | shell -> napplet | `jobId`, `state`, `position?` |
| `pow.progress` | shell -> napplet | `jobId`, `progress` |
| `pow.done` | shell -> napplet | `jobId`, `result` |
| `pow.error` | shell -> napplet | `jobId`, `error` |
| `pow.queue` | napplet -> shell | `id` |
| `pow.queue.result` | shell -> napplet | `id`, `jobs` |
| `pow.job` | napplet -> shell | `id`, `jobId` |
| `pow.job.result` | shell -> napplet | `id`, `progress`, `error?` |
| `pow.hashrate` | napplet -> shell | `id` |
| `pow.hashrate.result` | shell -> napplet | `id`, `hashrate` |
| `pow.cancel` | napplet -> shell | `id`, `jobId` |
| `pow.cancel.result` | shell -> napplet | `id`, `jobId`, `cancelled`, `error?` |
| `pow.pause` | napplet -> shell | `id`, `jobId?` |
| `pow.pause.result` | shell -> napplet | `id`, `jobId?`, `error?` |
| `pow.resume` | napplet -> shell | `id`, `jobId?` |
| `pow.resume.result` | shell -> napplet | `id`, `jobId?`, `error?` |

Key design notes:
- The napplet generates `jobId` and uses it to correlate the shell's `pow.progress` / `pow.done` / `pow.error` push messages, exactly as NAP-OUTBOX uses `subId`. The shell rejects a duplicate `jobId` with `accepted: false`.
- `pow.mine` / `pow.mineAndPublish` are acknowledged by their `.result`, which reports the job's **initial** `state` — `"queued"` (with `position`) if it is waiting, or `"mining"` if it started immediately. The mined event arrives later in `pow.done`, not in the ack.
- `pow.state` is a shell push (no `id`) emitted on every lifecycle transition (`queued` → `mining` → `paused`/`done`/`cancelled`/`error`), carrying the new `state` and, while queued, the current `position`. It is how a napplet observes a queued job starting without polling; `job.started` resolves on the first transition to `"mining"`.
- `created_at` is committed when the job **starts mining**, not when it is submitted, so queue wait never produces a stale timestamp. `options.timeoutMs` likewise measures mining time only.
- `options.priority` is a scheduling hint (higher runs sooner); the shell MAY honor or ignore it. It never lets one napplet preempt another's jobs beyond shell policy.
- `pow.progress` is a shell push with no `id`. The shell SHOULD throttle it (e.g. at most a few per second) regardless of hash rate.
- `target` is the required leading-zero-bit count (NIP-13); `bestPow` / `pow` are achieved leading-zero bits of the event id.
- `pause`/`resume` without `jobId` act on the whole miner.

### Examples

**Mine a note to difficulty 21 with 4 workers — enqueued behind another job, then started and read back:**
```
-> {
     "type": "pow.mine",
     "id": "m1",
     "jobId": "job-1",
     "template": { "kind": 1, "content": "gm", "tags": [] },
     "target": 21,
     "options": { "workers": 4, "priority": 0 }
   }
<- { "type": "pow.mine.result", "id": "m1", "jobId": "job-1", "accepted": true, "state": "queued", "position": 1 }
<- { "type": "pow.state", "jobId": "job-1", "state": "mining" }
<- {
     "type": "pow.progress",
     "jobId": "job-1",
     "progress": {
       "jobId": "job-1", "target": 21, "state": "mining",
       "bestPow": 18, "bestNonce": "773201", "hashes": 2310000, "hashRate": 412000,
       "workers": [
         { "workerId": 0, "bestPow": 18, "hashes": 590000, "hashRate": 104000 },
         { "workerId": 1, "bestPow": 16, "hashes": 580000, "hashRate": 103000 },
         { "workerId": 2, "bestPow": 17, "hashes": 575000, "hashRate": 102500 },
         { "workerId": 3, "bestPow": 15, "hashes": 565000, "hashRate": 102500 }
       ],
       "elapsedMs": 5600
     }
   }
<- {
     "type": "pow.done",
     "jobId": "job-1",
     "result": {
       "jobId": "job-1", "ok": true, "pow": 21, "nonce": "1099148",
       "hashes": 4980000, "elapsedMs": 12100,
       "event": {
         "id": "000007abc…", "pubkey": "user…", "kind": 1, "content": "gm",
         "tags": [["nonce", "1099148", "21"]], "created_at": 1234567890
       }
     }
   }
```

**Mine and publish (outbox-aware, no relay argument):**
```
-> {
     "type": "pow.mineAndPublish",
     "id": "p1",
     "jobId": "job-2",
     "template": { "kind": 1, "content": "powered note", "tags": [] },
     "target": 24
   }
<- { "type": "pow.mineAndPublish.result", "id": "p1", "jobId": "job-2", "accepted": true, "state": "mining" }
<- {
     "type": "pow.done",
     "jobId": "job-2",
     "result": {
       "jobId": "job-2", "ok": true, "pow": 24, "nonce": "55230914",
       "hashes": 31200000, "elapsedMs": 74000,
       "event": { "id": "000000def…", "pubkey": "user…", "kind": 1, "content": "powered note", "tags": [["nonce", "55230914", "24"]], "created_at": 1234567999, "sig": "…" },
       "published": { "eventId": "000000def…", "relays": { "wss://relay.example.com": true } }
     }
   }
```

**Inspect the queue, then pause one job:**
```
-> { "type": "pow.queue", "id": "q1" }
<- { "type": "pow.queue.result", "id": "q1",
     "jobs": [
       { "jobId": "job-2", "target": 24, "state": "mining", "priority": 0, "bestPow": 22, "hashRate": 418000, "kind": 1, "mode": "mineAndPublish" },
       { "jobId": "job-3", "target": 28, "state": "queued", "priority": 5, "position": 0, "bestPow": 0, "hashRate": 0, "kind": 1, "mode": "mine" }
     ] }
-> { "type": "pow.pause", "id": "ps1", "jobId": "job-2" }
<- { "type": "pow.pause.result", "id": "ps1", "jobId": "job-2" }
```

**Live miner-wide hash rate:**
```
-> { "type": "pow.hashrate", "id": "h1" }
<- { "type": "pow.hashrate.result", "id": "h1",
     "hashrate": {
       "hashRate": 824000, "workers": 8,
       "perWorker": [ { "workerId": 0, "hashRate": 103000 } ],
       "byJob": [ { "jobId": "job-1", "hashRate": 412000 }, { "jobId": "job-2", "hashRate": 412000 } ]
     } }
```

### Error Handling

Result and `pow.error` messages MAY include `error`. Common errors: `"duplicate jobId"`, `"invalid target"` (negative, or above the shell's cap), `"invalid template"`, `"timeout"` (target not met within `timeoutMs`), `"cancelled"`, `"unknown jobId"`, `"queue full"`, `"publish denied"` and `"publish failed"` (mineAndPublish only), and `"policy denied"`.

A timeout or cancel ends the job with `state: "error"` / `"cancelled"`, emits `pow.error`, and rejects the handle's `completed` promise. Best-effort progress emitted before the failure remains valid.

## Shell Behavior

- The shell MAY queue submitted jobs and schedule them under its own concurrency policy; it MUST accept a job into the queue (or reject it with `accepted: false`) synchronously in the `.result`, reporting the initial `state` and, when `"queued"`, the `position`.
- The shell MUST emit `pow.state` on every lifecycle transition so a queued job's start is observable without polling, and MUST stamp the committed `pubkey` (from user identity) and `created_at` at the moment mining begins — not at submit — then mine over that exact serialization; `commitCreatedAt: true` (default) freezes `created_at` for the whole search.
- The shell SHOULD honor `options.priority` as an ordering hint where its policy allows, but MUST NOT let priority become a way for one napplet to starve or preempt another's jobs beyond policy.
- The shell MUST insert or replace the `["nonce", "<nonce>", "<target>"]` tag per NIP-13 and MUST NOT return a result whose id has fewer than `target` leading zero bits.
- The shell MUST treat `options.workers` as a hint and MAY clamp it to available cores and per-napplet policy.
- The shell MUST respond to every request with a result carrying the same `id`, and MUST key every push (`pow.progress`/`pow.done`/`pow.error`) to the napplet-supplied `jobId`.
- The shell MUST sign mined events only through its normal signing path; for a previously-`mine`d event submitted for publish, the shell MUST sign it verbatim without re-stamping committed fields, or reject it.
- For `mineAndPublish`, the shell MUST publish using outbox-aware fanout (no napplet-supplied relays) and SHOULD apply the same publish consent policy as NAP-OUTBOX.
- The shell SHOULD throttle `pow.progress` and SHOULD preserve best-nonce and hash counters across `pause`/`resume`.
- The shell MAY queue jobs, cap concurrent jobs, total workers, and maximum `target`, and MAY enforce ACL checks on mine, publish, and queue inspection.

## Security Considerations

- Mining is unbounded CPU work, and difficulty cost is exponential in `target`. The shell MUST cap maximum `target`, concurrent jobs, and total workers, and SHOULD enforce `timeoutMs`, so a napplet cannot pin the device or drain a battery by requesting an astronomically hard target or flooding the queue.
- Signing and publishing remain shell-mediated; napplets never receive keys. `mine` returns an **unsigned** event — the proof of work is public and carries no secret — and signing happens only at publish time, gated by shell consent.
- Proof of work commits `pubkey` and `created_at`. The shell MUST preserve every committed field when signing a previously-mined event; a shell that re-stamps `created_at` or reorders tags silently destroys the proof of work the napplet paid for.
- Hash-rate and queue telemetry expose device capability and user activity. Shells MAY coarsen or rate-limit `pow.hashrate` / `pow.queue` for untrusted napplets.
- A napplet could request mining of content it intends to attribute to the user. Because the id commits the user pubkey, `mineAndPublish` MUST run the same content/consent checks the shell applies to any signed publish.

## References

- [NIP-13](https://github.com/nostr-protocol/nips/blob/master/13.md) — Proof of Work
- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)

## Implementations

- (none yet)
</content>
</invoke>
