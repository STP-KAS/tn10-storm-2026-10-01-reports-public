> Copy of `analysis/BOX-REPORT.md` from STP-KAS/tn10-storm-2026-10-01-analysis-public (commit 33e75d6). Relative links (`box-summary.json`, `../notes/timeline.md`) point into that repo.

# TN10 "storm", 1–2 Oct 2026: final box report

**Testnet-10 (TN10) only. Nothing here touched mainnet.** Written Fri 2 Oct 2026, after 08:45 CEST, by Grok Bot (executor) on the box, from the box's own files. All times are **CEST (UTC+2)** unless marked Z. Every number below comes from a file, a command output or a logged API call, and the file is named next to it. Anything I worked out rather than read is marked **(inference)**. Machine-readable totals are in [`analysis/box-summary.json`](box-summary.json). The full event log is [`notes/timeline.md`](../notes/timeline.md), a copy of `run/timeline.md`.

"Included" below means a transaction one of our runners saw accepted on n0's virtual chain before that runner process exited. "Mempool-accepted" means n0 accepted the `submitTransaction` call.

---

## What

stp's box node **n0** (kaspad 2.1.0, TN10) was driven with our own index-free load runners from **Thu 1 Oct 20:35:18** until the runners halted on the storm-watch STOP at **Fri 2 Oct 07:52:54**. stp ended the run at **08:43**.

- **20:14:47–20:16:47, smoke test.** One SMX runner, 50 lanes, 100 tx/s.
- **20:35:18–21:42, SMX runners.** These were round-6 `r5-engine` 1-in-1-out hops of about 1,745 g each. They started at 1,400 lanes and went to 2,500 lanes at 20:40:30. Base rate 1,500 tx/s, fee 3× n0's normal estimate, floating. Up to 4 runners.
- **21:42:37 to the end, P2W runners.** P2SH tag-wallet hops of 643 g each, runners P2W1..P2W8 (stp approved at 21:40). The scaler held 7 of them overnight.
- **Funding** came from the qzffl5 mining address's coinbase outputs through `coinbase-feeder.mjs`. After 01:41 it also used older coins found by deep scans (`run/extra-reserve.json`).
- **Outside load** shared the same blocks. Part of it was Grok Build's desk wallet **qp4jge**, which we had funded with about 1.2M tKAS from the box.
- **KNS** names were registered during the storm by the kns tn10 bot (21:46–22:27).

## Why

There were two goals. One was to beat the earlier rounds on duration and total transactions with recycled 1-in-1-out chains. The other was to answer stp's question: **does the public API / explorer "mined blocks" view stall while the network is under a storm?** The run also measured what the node itself survives (RAM, disk, pruning) and how much fee a storm like this burns. Plan files: `2026-storm-max-plan.md` and `run/LAUNCH.md`. Note that `2026-10-02-plan.md`, named in the task, does not exist on the box.

## When

| Phase | CEST | What happened (source: `run/timeline.md`, runner logs) |
|---|---|---|
| Smoke | Thu 20:14:47–20:16:47 | SMX1, 50 lanes, 100 tx/s, self-stop after 120 s |
| SMX ramp | 20:35:18–20:53:30 | SMX1+SMX2 sending from 20:38:30. Ramp 25→50→100%. Runners reported `ramp=1` at 20:52:51. 2,500 lanes from 20:40:30. Box miners resumed 20:57:46 |
| SMX steady | 20:53:30–21:42:37 | SMX3 added 21:03:33, SMX4 21:24:51–21:33:53 (worse with 4). Blocks about 89–93% full by compute mass (kaspad log) |
| P2W A | 21:42:37–23:13:34 | P2W1 at 21:42:37. SMX replaced by P2W by 22:17. P2W2–4 added. 3× floating fee |
| Pause 1 | 23:13:34–23:31:07 | **storm-watch STOP, RAM 1,009 MB.** xfer of chain ends and coinbases to qp4jge |
| P2W B | 23:31:07–00:10:41 | P2W1–3, then P2W4, P2W5, P2W6 |
| Pause 2 | 00:10:41–01:09:58 | Planned pause so TN10 ops could rebuild n0 with `--utxoindex`. 00:49:51: a read-only `getUtxosByAddresses(qzffl5)` probe **OOM-killed kaspad** (timeline 00:51). 00:56–00:59: 1M tKAS send to qp4jge |
| P2W C | 01:09:58–01:33:41 | 6 runners at **30× fee (6,000 sompi/g)**, P2W7 at 01:19:19, P2W8 01:26–01:33. **Peak of the run.** Box miners halted 01:31:50 (host share shows box blocks stop at 01:32:17) |
| P2W D | 01:33:41–06:45:53 | Flat 2× fee (~390–400 sompi/g), 7 runners. Scaler re-probed P2W8 every ~42 min (8 break points). Mempool sat at ~71–72k |
| Self-pause | 06:45:53–07:52:54 | 06:45:08 first runner `disk<=25G` pause. 06:45:53 newest-guard HARD ABORT (RAM 1,367 MB) killed P2W7. 07:05 prune-hold. Inclusion fell to ~0 |
| Stop | 07:52:54–08:43 | **storm-watch STOP, RAM 754 MB.** Runners and feeder halted 07:52:57–07:53:00 and saved chain ends. Pruning-point move 07:51:15–08:30:09 (kaspad log). stp ended the run 08:43. control/STOP pushed (commit 5b55e50, 08:43:34) |

## Facts

### 1. Throughput

| Metric | Value | Source |
|---|---:|---|
| **Storm txs included** (32 runner segments, smoke excluded) | **29,884,172** | final events in `data/runner1-4.out`, `data/p2w1-8.out` |
| Storm txs mempool-accepted (submitted) | 30,001,282 | same |
| Submitted but not seen included before the runner exited | 117,110 (0.39%). Most were probably included after exit **(inference)** | same |
| **Rejected** | **0** in every segment. `errs` empty, `fund_fail` 0 | same |
| SMX share | 6,760,158 included, 70,382 tKAS fees | same |
| P2W share | 23,124,014 included, 221,585 tKAS fees | same |
| **Peak 1-min included** | **4,248.5 tx/s**, 01:24:46–01:25:46 | per-runner cumulative `accepted` interpolated on a 5 s grid, summed (`rep` lines every 10 s) |
| Peak 5-min included | 3,841.4 tx/s, 01:21:16–01:26:16 | same |
| **Peak 10-min included** | **3,715.4 tx/s**, 01:16:11–01:26:11 | same |
| Peak 60-min included | 1,977.2 tx/s, 22:12:46–23:12:46 | same |
| Average over 20:35:18–07:52:54, pauses included | ~735 tx/s (29.88M / 40,656 s) **(inference: simple division)** | |
| Smoke | 12,019 submitted, 12,019 included, 0 rejected, p50 1.28 s, p95 2.47 s, 63.4 tKAS fees | `data/runner1.out` segment 1 |

**Per phase** (same method):

| Phase | Included tx/s | Mempool-acc tx/s | Included txs | Fees (tKAS) |
|---|---:|---:|---:|---:|
| SMX ramp 20:35:18–20:53:30 | 502 | 512 | 548,372 | 4,625 |
| SMX steady 20:53:30–21:42:37 | 1,402 | 1,414 | 4,130,083 | 41,885 |
| P2W A 21:42:37–23:13:34 | 1,863 | 1,875 | 10,168,351 | 70,011 |
| Pause 1 (RAM STOP) 23:13:34–23:31:07 | 17 | 8 | 17,390 | 46 |
| P2W B 23:31:07–00:10:41 | 2,273 | 2,274 | 5,396,429 | 37,354 |
| Pause 2 (n0 rebuild) 00:10:41–01:09:58 | 6 | 5 | 19,997 | 370 |
| **P2W C, 30× fee, 01:09:58–01:33:41** | **2,426** | 2,454 | 3,451,469 | **122,320** |
| P2W D, 2× fee, 01:33:41–06:45:53 | 318 | 320 | 5,952,320 | 14,993 |
| Self-pause 06:45:53–07:52:54 | 50 | 36 | 199,756 | 364 |
| Stop 07:52:54–08:43 | 0 | 0 | 0 | 0 |

**Per hour (included):** 20h 931,445 · 21h 5,465,669 · 22h 6,752,022 · 23h 5,636,914 · 00h 1,494,571 · 01h 3,736,763 · 02h 1,000,230 · 03h 1,069,629 · 04h 1,080,743 · 05h 1,092,789 · 06h 1,607,942 · 07h 15,449. Same source; the 06h figure includes a 06:37–06:46 burst of about 1,800/s.

**Latency** (submit to seen on the virtual chain). The engine's percentiles are **cumulative over each runner process** (`r5-engine.mjs` `pstats`), so they cannot be cut exactly by phase. Values are from the last `rep` line of each segment:

| Segment | p50 | p95 |
|---|---:|---:|
| Smoke | 1.28 s | 2.5 s |
| SMX1/2/4 (20:38–22:17) | 3.5–3.7 s | 13.8–35.3 s (SMX3: 84 s / 239 s) |
| P2W 22:17–23:13 | 2.7–3.8 s | 12.7–21.1 s |
| P2W 23:31–00:10 | 3.4–4.1 s | 6.9–10.9 s |
| P2W at 30× fee, 01:10 to 01:33:30 | 4.6–5.5 s | 10.8–27.0 s |
| P2W whole 01:10–07:53 (30× plus the 2× night) | 1.7–5.2 s | **101–953 s** |

The long tail comes from the 2× night. Once our own miners stopped, txs waited up to about 16 minutes.

### 2. Blocks, mempool, fees

- **Block fullness** (kaspad log `Processed … mass: s/c/t`, 30-min means of compute mass per block against the 500,000 limit): 49% at 20:30, **88.5% at 21:00, 92.8% at 21:30**, 86% at 22:00, 80% at 22:30, 69–74% at 23:00–23:30, 73.5% at 01:00, 84% at 01:30. From 02:30 to 06:30 it was **only 11–26%**. Storage mass per block peaked at a 143k mean in the 01:00 half-hour.
- **Mempool:** peak **99,992 at 01:35:47** (`data/host.jsonl`). 16 storm-watch "mempool > 80k" pause episodes between 20:55:45 and 02:24:59 (145 samples, `data/storm-watch.jsonl`). From 02:30 to 06:30 the half-hour mean stayed at 71.1–71.9k, right at the runners' 72k pace ceiling.
- **Fee rate we paid** (`data/feerate.jsonl`):
  - Until 00:10: 3× n0's normal estimate, with the half-hour median at 559–589 sompi/g. But the multiplier followed the estimate up, to a **max of 19,244 sompi/g at 23:56:53** (normal estimate 6,414.8).
  - 01:09:53–01:26:51: a fixed 6,000 sompi/g (30×, capped).
  - 01:26:51: fee-float-guard cut the multiplier to 3×, because free float was down to 373 tKAS (`data/fee-float-guard.jsonl`).
  - From 01:33:41: flat 2×, about 385–400 sompi/g. After 07:00 it was 200.
- **n0's own normal estimate** peaked at **24,144.8 sompi/g** (priority 203,875.9) at 01:51:21. During that time we paid 400.
- **Fees burned by the storm runners: 291,967 tKAS**, plus 63.4 for the smoke test. These are fees of submitted txs. The 23.7 minutes at 30× cost **122,320 tKAS (42% of all fees) for 11.5% of the included txs**. Separately, the two transfers to qp4jge paid 836.9 + 11,647.7 tKAS (`xfer/`). KNS paid 1,873.7 tKAS on names plus 419.9 on sweeps (KNS summaries).

### 3. Funding and coinbase inflow

- The feeder's own inflow metric (`accepted_inflow_tkas_per_min` in `data/feeder.log`), integrated over the run, gives about **398.6k tKAS from 20:35 to 01:33** and **2.9k from 01:33 to 07:53**, about 401k in total. **This is an estimate**: feeder restarts and backfills can distort the metric. The robust fact is that **inflow went to about 0 after 01:33** (hourly means 0.0 for 02h–05h). The box miners had halted, and almost no qzffl5 desk blocks arrived.
- From 01:41 the runners lived on old coins. Deep scans A and B found 24,572 + 24,618 safe qzffl5 coinbases (76.2k + 76.6k tKAS, `data/deep-A.out`, `data/deep-B.out`). At 07:46, 46,784 coins / 145.4k tKAS were still unused in `run/extra-reserve.json` (timeline 07:46).
- The feeder RSS guard restarted the feeder **10 times** (RSS > 1,228 MB, timeline).

### 4. Block share (n0 `block-added`, `data/host.jsonl` `share`)

box = qzffl5 payout with the "stp grok bot" UA; desk = qzffl5 payout with any other UA; network = any other payout.

| Window | Blocks | Box | Desk | Network |
|---|---:|---:|---:|---:|
| Pre-storm 18:41–20:35 | 56,841 | 0% | 29.7% | 70.3% |
| 18:41–19:01 (the "~25% at 18:51" check) | 5,469 | 0 | **24.8%** (1,356) | 75.2% |
| SMX 20:35–21:42 | 45,910 | 34.9% | 22.3% | 42.8% |
| P2W A 21:42–23:13 | 54,582 | 60.8% | 0.3% | 39.0% |
| P2W B 23:31–00:10 | 23,040 | 58.6% | 1.1% | 40.3% |
| P2W C 01:09–01:33 | 12,968 | 53.7% | 1.4% | 44.9% |
| P2W D 01:33–06:45 | 176,481 | **0%** | 0.7% | 99.3% |
| Self-pause 06:45–07:52 | 39,757 | 0% | 0.4% | 99.6% |
| **Whole storm 20:35–07:52** (gap 00:10:47–01:00:32) | 371,049 | **21.8%** | **3.3%** | **74.9%** |

Box miners were on from 20:57:46 to 01:32:17 and again from 08:43:47. **Desk blocks paying qzffl5 stopped at 21:28:01.** After that there were only trickles, plus a burst from 06:36:47 to 06:49:02. The separate `share-sampler.py` log (`artifacts/kaspa-tn10/share.csv`) agrees: from 21:42 on, "all blocks submitted through n0" equals the box count. So no desk blocks came through n0's tunnel either.

### 5. Disk, RAM and the guards that fired

- **Disk free** (`host.jsonl`): 55.54 GB at 20:00, 53.9 at 21:42, 46.96 at 23:13 (4.5 GB/h in P2W A), 43.36 at 00:10. Then **30.25 at 01:00**, after the n0 utxoindex rebuild window (no samples 00:10–01:00). 31.35 at 01:33 (6.1 GB/h at 30×), 24.79 at 06:45, 22.89 at 07:52. Low point **9.15 GB at 08:09:17**, during pruning, after the stop. Back to 23.18 at 08:45.
- **RAM available:** lowest 15 s host sample 1,059 MB at 23:13:32. storm-watch read 1,009 MB at 23:13:34 and **754 MB at 07:52:54**. kaspad RSS peaked at 6,448 MB at 08:11:32 (pruning).

| When | Guard | What it did |
|---|---|---|
| 20:55:45–02:24:59 | storm-watch mempool > 80k | 16 pauses until the mempool fell below 55k |
| 23:13:34 | storm-watch RAM < 1 GB (1,009) | **STOP**. All runners and the feeder halted cleanly. Timeline 23:15 notes that the first xfer spare-scan (~1 GB RSS) contributed |
| 23:56:00–06:46:27 | feeder-guard RSS > 1,228 MB | 10 feeder restarts |
| 01:26:51 | fee-float-guard | 30× → 3× (free float 373 tKAS) |
| 01:33:27–06:28:33 | scaler BREAK POINT | 8 times: P2W8 added less than +100/s, so it was dropped. The 01:33 one was really the miner halt (timeline 01:46) |
| 06:45:08–06:46:04 | runner `disk<=25G` | P2W1–7 set rate 0 |
| 06:45:53 | newest-guard HARD ABORT (RAM 1,367 MB < 1.5 G twice) | SIGTERM P2W7 |
| 06:49:41 | scaler BACK OFF (RAM 1,367 MB) | P2W7 restarted and dropped again |
| 07:05 | prune-hold / storm-watch prune window | rate × 0.25, ramp held |
| 07:52:54 | storm-watch RAM < 1 GB (754) | **STOP**, the end of the storm |

**About the 07:52:54 STOP (inference, with evidence):** RAM available was 2,088–2,113 MB at 07:52:17–07:52:47. It jumped from **1,483 to 4,216 MB between 07:54:47 and 07:55:02**, the moment the outside read-only probe was killed (~07:55, timeline 08:08). kaspad RSS stayed at ~5.0–5.2 GB throughout. That fits the probe being the cause. The kaspad pruning-point move had also started at 07:51:15 and may have added pressure. Disk then fell fast (23.1 GB at 07:53 to 12.2 GB at 07:56) and bottomed at 9.15 GB. That is below storm-watch's 10 GB emergency floor, which had nothing left to do because STOP was already set.

### 6. n0 health

- **Synced on all 4,297 RPC polls** from 20:00 to 08:45 (`data/tn10-n0-rpc.jsonl`). storm-watch never saw `synced:false`.
- **Sink age:** median 0.4 s, p99 2.4 s, **max 3.1 s at 06:42:14**. Never over 5 s (2,600 storm-watch samples).
- Virtual-chain reorgs of 3 or more blocks were logged 7,733 times during the storm window, at most 17 blocks (22:15:45) (`deep_reorg` lines in `tn10-n0-rpc.jsonl`). This is normal DAG behaviour, but noted.
- **One kaspad crash**, outside the load: OOM-kill at about 00:50 during pause 2, caused by a single `getUtxosByAddresses(qzffl5)` call (timeline 00:51, which quotes dmesg). The samplers were stopped then (host gap 00:10:47–01:00:32), so this report has no sampler data for it.

### 7. stp's question: does the public API "mined blocks" view stall during a storm?

**Yes, it lagged badly, by up to about 11 minutes. It never froze completely, and it recovered when the load dropped.**

- `data/api-mined-watch.jsonl` (every 5 min) compares the newest qzffl5 **coinbase** shown by `api-tn10.kaspa.org/addresses/qzffl5/full-transactions-page` with n0's count of blocks we mined in the same window:
  - 18:51–22:11: newest coinbase lag 2.9–9.6 s (normal).
  - From **22:16** the lag grew: 34 s, 60 s (22:26), 210 s (22:31), 288–306 s (22:46–22:56), **651.5 s at 23:16**. One timeout at 23:26.
  - It recovered to 70 s at 23:36, then grew again to 430.8 s at 00:06.
  - **`mined_stall` = true in 9 of 66 samples**: 22:56, 23:06, 23:11, 23:16, 23:21, 23:26, 23:31, 00:01, 00:06. Each time n0 showed we had mined 1,622–2,114 blocks in that window.
  - Final sample 08:45:15 (taken for this report): lag 8.0 s, no stall.
- `data/api-health-min.jsonl` (every 2 min, `/info/health`) shows the cause is the API's **database/indexer falling behind**, not the node: `acceptedTxBlockTimeDiff` > 120 s while `blueScoreDiff` stayed small.
  - `stall_suspected` in 48 of 398 samples, from **22:29:30 to 01:30:31**.
  - Peak **704.4 s at 23:19:30**.
  - **HTTP 503 28 times** (21:29, 21:31, 22:45, 22:53, 22:59, 23:03–23:29, 23:53, 00:01–00:09, 01:22, 01:24, 07:46). In 22 of them, between 22:45 and 00:09, the body reported the database `isSynced:false`.
  - From 02:00 to 08:45 `acceptedTxBlockTimeDiff` stayed at 17 s or less. There were 22 read timeouts (20 s), mostly 02:54–07:44.
- **When it happened:** the lag started at 22:29, the same minute the external storage-mass burst began (timeline 22:31). It was worst during P2W A/B (1,860–2,270 incl/s, n0 processing ~4,000–4,450 tx/s). It kept growing during the **storm pause 23:13–23:31**: the indexer was still working through the backlog. **(inference)**
- **Coverage gap:** no mined-watch samples from 00:10 to 08:45. The watcher stopped with the 00:10 pause, and its 01:00:31 restart crashed on a `host.jsonl` start line (`data/mined.out`, `AttributeError: 'bool' object has no attribute 'get'`). So the mined view was **not measured at the 01:19–01:26 peak**. Health showed `acceptedTxBlockTimeDiff` of 162–163 s at 01:28–01:30 and 503s at 01:22/01:24. The mined view probably lagged about 2–3 minutes then too **(inference)**.

### 8. Outside traffic and qp4jge

- **Our box funded qp4jge with 1,203,035.1 tKAS in 4,599 txs.**
  - 202,682.690 tKAS in 408 txs, 23:19–00:11 (`xfer/txids-2026-10-01.tsv`).
  - 1,000,352.456 tKAS in 4,191 txs, **00:56:39–00:59:03** (`xfer/sent-gb1m-1002.jsonl`, 0 errors).
  - The money came from qzffl5 coinbases.
- **Not verified by me, attributed:** the 10,659,746 tx count for qp4jge and the ~2.3–3.1 tKAS balance afterwards. Both come from api-tn10 calls logged in `public-api-calls.log` at 07:52:02 and 07:55:05. The values are recorded in `grok-build-tx-sender-2026-10-02.md` and timeline 08:08, not in a raw response file. 10,659,746 txs over 00:59–07:55 (24,960 s) is **~427 tx/s (inference)**. If all of qp4jge's outflows were self-sends, about **1.2M tKAS went to fees (inference)**. I did not query qp4jge again: the 07:52 probe of this address is the likely cause of the final STOP, and a utxo query on a heavy address OOM-killed n0 at 00:50.
- **Outside load on the box node:**
  - Block samples in the timeline show external txs of ~1,630 g with a 9-byte payload: 22:36, 22:59 (48% of block mass), 23:07, and 01:13 (81% of mass before our 30× start).
  - From 02:00 to 06:30, n0 processed 670–1,650 tx/s (host) while we were included at ~300/s. Other senders were roughly 400–1,300 tx/s, but "processed" double-counts txs in parallel blocks **(inference)**.

### 9. KNS during the storm (`[local path]`)

Run 1 (21:46:39–21:53:59):

- 9,509 names, 19,018 tx (commit + reveal), 0 failed, 0 stranded.
- Peak 3,477 names in calendar minute 21:52 (rolling 60 s: 3,610).
- Fee 1.2× the storm rate (707 vs 589 sompi/g). Spent 333,721.09 tKAS, of which 906.09 was fees.

Other runs:

- **Sweep** 22:02:27–22:04:47: 513 tx, 318,974.36 tKAS in, 260.15 fees.
- **Blueprint** 22:06:31–22:12:58: 8,258 names, 16,516 tx, 0 failed. Leaving n0's mempool took p50 7.6 s / p90 25.2 s.
- **Blueprint 2** 22:21:50–22:26:45: 757 names, 0 failed.

Total **18,524 names, 0 failures**. **Indexer ownership is not verified.**

## Claims vs verified

| Claim (who) | Verified value | Status |
|---|---|---|
| Smoke 12,019 / 0 rej / 100.1 tx/s / p50 1.28 / p95 2.47 (tn ops2) | 12,019 / 0 / 12,019 in ~120 s / 1.278 s / 2.465 s (`runner1.out`) | ✅ |
| 20:54: 2,500 lanes, ~1,245 acc/s, 0 rej, p95 8–9 s, 588 sompi/g (tn ops2) | 2×1,250 lanes; 1,273 mempool-acc/s and 918 incl/s at 20:52–20:56; 0 rej; p95 3.6–10.5 s in the 20:53:31–20:54:21 reports; feerate 586–588 | ✅ ("accepted" means mempool-accepted) |
| 21:20: 3 runners, 1,551 incl/s, ~2.75M total (tn ops2) | 1,563 incl/s at 21:15–21:20. 2,690,741 included by 21:20 | ✅ rate; total ~2% lower (different counter) |
| Cap was mass, blocks ~95% compute-full (tn ops2) | 30-min means 88.5% (21:00), 92.8% (21:30); timeline windows 98–99.4% | ✅ |
| P2W1 → 1,718 tx/s (tn ops2) | 1,717.3 at 21:53:45–21:58:45 | ✅ |
| Peak 3,845 incl/s, 7 runners, 01:19–01:26 (tn ops2) | 5-min 3,841 (01:21:16–01:26:16); scaler window 3,829 by my interpolation; 1-min 4,248.5 | ✅ |
| Self-pause from ~06:50–07:00 on disk ≤ 25G (tn ops2) | First `disk<=25G` 06:45:08; all P2W by 06:46:04; prune-hold from 07:05 | ✅ (5 min earlier) |
| STOP 07:52:54, RAM 754 MB, probe likely cause (timeline 08:08) | storm-watch 07:52:54 754 MB; RAM +2.7 GB at 07:55 when the probe was killed | ✅, with pruning start 07:51:15 as a possible co-factor |
| Desk wallet sent ~10.66M txs from 00:59, ~400 tx/s, spent ~1.2M tKAS (tn ops2) | Inflow of 1,203,035 tKAS verified. Tx count and balance only via a logged API call (value in a doc, not raw). ~427 tx/s by division | ⚠️ partly verified |
| Measured block share ~25% at 18:51 (stp) | 24.8% desk, 0% box, 18:41–19:01 | ✅ |
| "stp's 20 desk miners mined throughout" | Desk blocks paying qzffl5: 22.3% in SMX, then **0.3–1.4%**, and 0 for long stretches after 21:28 | ❌ not seen in data (see below) |
| KNS: 9,509 / 19,018 / 0 fail / peak 3,477/min / 333,721 tKAS; sweep 513 tx / 318,974; blueprint 8,258 / 16,516 / 7.6/25.2 s; blueprint 2 757; total 18,524 (kns tn10 bot) | All match the KNS summary files and `names.jsonl` | ✅ |
| Prior: 1,191 sustained, 7,652 peak, round 6 ~10.1M | round 6 README: 1,191 tx/s, ~10.1M; round 4 README: peak 7,652, full gusto 3 avg 4,568 (12 min) / 3,942 (17.5 min). "~4,450 included" not found verbatim | ✅ / ⚠️ |

## Sources

- `run/timeline.md` (full event log, read in full), `run/LAUNCH.md`, `2026-storm-max-plan.md`
- `data/runner1-4.out`, `data/p2w1-8.out`: per-runner `rep` (10 s) and `final` lines
- `data/host.jsonl`: 15 s disk, RAM, kaspad, mempool, block share. Gap 00:10:47–01:00:32
- `data/storm-watch.jsonl`: watcher samples, pauses, `STOP_SET`
- `data/tn10-n0-rpc.jsonl`: isSynced, fee estimate, `deep_reorg`
- `data/feerate.jsonl`, `data/fee-float-guard.jsonl`, `data/feeder.log`, `data/deep-A.out`, `data/deep-B.out`
- `data/api-mined-watch.jsonl`, `data/api-health-min.jsonl`, `data/api-tn10.jsonl`, `public-api-calls.log` (every public API call)
- `[local path]`: block mass per 10 s, pruning events
- `[local path]`: independent share sampler
- `xfer/txids-2026-10-01.tsv`, `xfer/sent-gb1m-1002.jsonl`: transfers to qp4jge
- `[local path],blueprint,blueprint2}-2026-10-01/`
- Prior rounds: `tn10-storm-2026-09-26-r6/README.md`, `stress-src/grok-bot-vprogs-round4/README.md`, `tn10-storm-2026-09-25/README.md`
- Analysis scripts (box, not committed): `[local path],host,kaspad,extra}.py`

## Conclusion

- **Biggest run so far by volume.** 29.88M included storm txs with zero rejects, about 3× round 6's ~10.1M. Best hour 6.75M (22h). Best 60 minutes 1,977 tx/s, well above round 6's 1,191 sustained.
- **No new peak record.** Best 1 minute 4,248 tx/s and best 10 minutes 3,715 tx/s. That is below round 4's 7,652 peak and its 4,568 average over 12 minutes.
- **What set the ceiling, in order:**
  - First, tx mass: SMX hops of ~1,745 g filled blocks by compute mass at ~300 tx/block.
  - Then, per-process submit throughput (~600–850/s per runner).
  - Then, inclusion and fees. The peak came only with a 30× fee and our own miners on.
  - **After our box miners stopped at 01:31:50, inclusion fell to ~300/s even though blocks were only 11–26% full** and n0's mempool held ~72k of our txs. This strongly suggests that other TN10 miners included few of our txs, and our throughput depended on our own hashrate **(inference)**.
- **n0 itself held up.** Synced throughout, sink age at most 3.1 s. It was stopped by RAM guards twice (23:13:34 and 07:52:54, the second most likely from an outside probe), and the disk shrank from 55.5 to 22.9 GB.
- **The public API mined view did lag under load.** It reached 651 s for the newest coinbase (23:16), and the indexer reached 704 s behind with health 503s (23:19). The lag started with the heavy external burst at 22:29 and cleared once load fell. It was not measured at the 01:19–01:26 peak because of the watcher crash.

## Lessons for next storm

1. **Mass limit.** Compute mass, not tx count, fills blocks. 643 g P2W hops fit ~500+ per block against ~300 for 1,745 g SMX hops. Use the lightest tx shape from the start, and size the runner count from block mass, not lanes. More lanes per process did not help (timeline 22:09, 22:31).
2. **Own hashrate decides inclusion.** Our rate collapsed when our miners stopped (01:31:50), even with blocks mostly empty. Keep box and desk miners on for the whole run, and check that desk miners really pay to and submit through n0. The data shows desk blocks to qzffl5 stopping at 21:28.
3. **Disk.** We burned 4.5–6.1 GB/h at load, and ~13 GB disappeared during the utxoindex rebuild window (43.4 to 30.3 GB between 00:10 and 01:00; inference that the rebuild caused it, since the box is shared). Start with at least 60 GB free, or plan the run length around about 5 GB/h plus the ~14 GB pruning spike (it reached a 9.15 GB low at 08:09).
4. **RAM.** Three things nearly or actually killed the node:
   - big read-only probes (spare-scan at 23:13, the qp4jge mempool pull at 07:52);
   - a single `getUtxosByAddresses` call on the mining address (kaspad OOM at 00:50);
   - feeder RSS growth (10 restarts).
   
   Ban heavy address queries against n0 and against the box during a storm. Run probes on a separate host. Cap the feeder's memory by design, not by restarts.
5. **Fee float feedback loop.** "3× the node's normal estimate" chases its own backlog: our fee rose to 19,244 sompi/g at 23:56. A fixed 30× burned 122k tKAS (42% of all fees) in 24 minutes. A flat 2× then lost to outside senders paying much more (normal estimate 24,145 at 01:51), and latency p95 reached 950 s. Next time: cap the absolute feerate, base the multiplier on a fixed floor rather than the live estimate, and tie the fee mode to measured inclusion, not just free float.
6. **Pruning windows.** The pruning-point move ran 07:51–08:30 (and 19:34–20:13 on Thursday). It eats disk and RAM. Treat ~07:00–08:30 and ~19:00–20:30 as no-load windows, and stop the storm before them rather than throttling into them.
7. **Watcher robustness.** The mined-view watcher died at 01:00:31 on a log-format edge case and missed the peak. Every sampler should survive restarts and be checked by a supervisor.

## Open items

- **Sweep of stranded chain ends.** Live and retired P2W/SMX lane ends are saved under `[local path]`. Examples: P2W1 saved 1,500 live + 10,736 retired ends at 07:53. There are also the 750+750 EXTRA ends from 23:18. LAUNCH.md notes that no sweep tool exists yet for the RET files. Not swept.
- **Unharvested old-float qzffl5 coinbases** in DAA ~585,398,000–585,539,881 (timeline 00:11), and 46,784 coins / 145.4k tKAS left in `run/extra-reserve.json` (07:46).
- **KNS indexer ownership check** for the 18,524 names: not done.
- **qp4jge**: the tx count and fee spend need a light, approved check that is not run on n0 or the box. About 145.8k of its txs were reported pending in one public node's mempool (attributed, prompt doc).
- **Mined view at the 01:19–01:26 peak**: not measured. Fix the watcher bug (`share` = `true` in a start line) before the next run.
- **Why other miners included few of our txs after 01:32**: needs a block-template or relay check.
- The interim `analysis/REPORT.md` (data up to 20:22:56) is superseded by this report for the storm period.
