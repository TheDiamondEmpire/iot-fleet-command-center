<p align="center"><img src="assets/banner.svg" alt="IoT Fleet Command Center" width="100%"></p>

# IoT Fleet Command Center: Research Report

> **Where the code is.** The experiment scripts that produced these results are part of the source package, which is distributed separately (see [SOURCE-ACCESS.md](../SOURCE-ACCESS.md)). The raw results they wrote are included in this repository under [`results/`](../results/).

> **Status: protocol v0.2, results recorded.** The hypotheses (section 2) are unchanged from v0.1. Section 4 now describes the system and conditions as actually built and run, and section 7 holds the measured results. Every number below was produced by a script in `experiments/` and is in `results/`. **All devices, faults and traffic are simulated**; nothing here says anything about real hardware, real field failures or production scale. Where a hypothesis was not supported, the text says so.

## Abstract

This study asks: how effectively do staged, health-gated rollouts and fleet anomaly detection limit the impact of faulty firmware and catch failing devices early, and how does ingestion scale with fleet size? It was run on a working prototype with simulated devices on one shared development machine. Headline results, all on simulated devices: a health-gated staged rollout limited a faulty image to the 5-device canary ring in every run for faults affecting at least 30% of devices (100% for ungated baselines) but let it reach 31% of the fleet on average when only 10% of devices were affected (H1 supported only in part); a per-model anomaly rule flagged 97.6% of failing devices at least 24 h ahead with no false positives, versus 21.5% to 84.0% (the best of which flagged 17.9% of healthy devices) for static thresholds (H2 supported in the simulation); the ingest path sustained at least 8,000 simulated devices and 4,000 messages per second with p99 commit latency under about 0.3 s on a heavily shared machine, without reaching a measured ceiling for devices and with the 100,000-device clause untested (H3 not supported as worded); and mutual TLS added 55.6% to connection setup compared with a shared secret over TLS (H4 not supported).

## 1. Research question

How effectively do staged, health-gated rollouts and fleet anomaly detection limit the impact of faulty firmware and catch failing devices early, and how does ingestion scale with fleet size?

## 2. Hypotheses

Wording is unchanged from protocol v0.1. Section 4.6 lists the places where v0.2 had to adapt how a hypothesis is tested, and why.

- H1: Health-gated staged rollouts cap the share of devices receiving a faulty image at the first ring size, versus 100% for an ungated rollout.
- H2: Per-model anomaly detection flags failing devices at least 24 hours before hard failure for at least 70% of injected degradation patterns, at under 2% false-positive rate.
- H3: Ingest throughput scales near-linearly with worker count up to the broker’s connection limit, and p99 ingest latency stays under one second at 100,000 devices.
- H4: Certificate-based identity adds under 10% to connection-setup time compared with shared-secret authentication.

## 3. Background

This section is a positioning note, not a literature review. Citations are from memory of well-known sources and should be checked against the originals before this report is used anywhere formal.

- **Transport and identity.** MQTT is the de-facto device messaging protocol (OASIS 2019) and is normally secured with TLS; per-device X.509 certificates (RFC 5280) give each device its own revocable credential, in contrast to fleet-wide shared secrets. H4 asks what that costs at connection time.
- **Secure firmware update.** The IETF SUIT work describes an update architecture and manifest information model for constrained devices (RFC 9019, RFC 9124): signed manifests, payload digests and anti-rollback. Uptane (Kuppusamy et al. 2016) addresses compromise-resilient updates for vehicles. The prototype implements a much smaller subset (one signing key, a signed manifest with digest and counter, a per-device signed offer) and does not claim conformance to either.
- **Staged rollouts.** Canarying and progressive delivery are standard release-engineering practice (Beyer et al. 2016, on release engineering and canarying). The contribution tested here (H1) is not the idea but how well a simple device-health gate behaves with different fault types and prevalences.
- **Anomaly detection.** Peer-group comparison with robust statistics (median and median absolute deviation) is a classical, assumption-light approach to outlier detection (Leys et al. 2013; Chandola et al. 2009 for the wider survey). H2 tests a deliberately simple per-model detector against static thresholds, not against state-of-the-art learned detectors.
- **Statistics used.** Percentile bootstrap intervals (Efron and Tibshirani 1993), Wilson score intervals for proportions (Wilson 1927), Mann-Whitney and Wilcoxon signed-rank tests, Cliff's delta, Holm-Bonferroni correction.

## 4. Method

### 4.1 System under test

The prototype in this repository: an in-process Aedes MQTT broker (MQTT 3.1.1) that accepts only mutual-TLS clients whose certificate is in the registry; an ingest pipeline (strict schema validation, per-device token-bucket rate limit, back-pressure) writing batches to SQLite with 1 min and 1 h rollups; an alert engine with a per-model anomaly rule and static threshold rules; and a campaign controller that stages signed firmware through rings with health gates and automatic rollback. Details and stand-ins for production infrastructure are in the README.

### 4.2 Environment

- **Hardware and load.** One shared development machine (AMD EPYC 7543P, 8 logical cores visible, Windows 11, Node 25.9). Three unrelated projects were being built and tested on it at the same time, and the machine was **already close to fully busy before every run** (100% for E1, E2, E4 and the main E3 run; 64.6% for the E3c supplement) (the whole-machine CPU utilisation sampled for one second before each experiment and each E3 step is recorded in the result files; see the Results section). Timing results are therefore noisy and conservative in an unknown direction.
- **CPU limits.** Every experiment process is pinned (Windows processor affinity) to logical cores 4 and 5 for the system under test, never more than two cores. E4 and E3 run their load generator in a separate process pinned to one further core (core 6). The generator is not part of the system under test but it did use a third core.
- **Transport.** mTLS is carried over a Windows named pipe, not TCP, because security software on this machine re-signs loopback TLS (see README). TLS handshakes, certificates and ciphers are the real ones; socket-layer costs differ from TCP.
- **Simulation.** Simulated devices connect with their own certificates and run the same firmware verification code as the server. Faults (bad images, battery decay, sensor drift) and physics (discharge, diurnal ambient cycle, sensor noise) are synthetic and were written by the author. E2 uses a time-compressed synthetic history rather than real time.
- **Database.** SQLite in memory for E3 (no disk I/O in the measured path); SQLite in memory for E1, E2 and E4 as well. The effect of a disk-backed store is not measured.
- **Software versions.** Dependency versions are pinned by `package-lock.json` (hash prefix recorded in each summary file). No container image was used.

### 4.3 Baselines

- Ungated all-at-once rollout, and a staged rollout advanced by a timer with no health check (isolates the effect of the gate from the effect of rings)
- Static threshold alerting (battery below 20% and below 30%; sensor outside a fixed range)
- Shared-secret device authentication over TLS (the fair comparison for H4) and over no TLS (reference)

### 4.4 Independent variables

- Rollout policy; fault mode (`degraded`, `bootloop`, `brick`) and fault prevalence among devices (100%, 30%, 10%); ring sizes and gate thresholds were fixed (not varied)
- Anomaly detector vs static rules; fault kind
- Fleet size (E3a) and per-device message rate (E3b); SUT cores (E3c)
- Authentication mode (E4)

### 4.5 Metrics

- Share of fleet that ran the faulty image (ground truth from the simulator), share harmed, time from campaign start to automatic pause
- Detection lead time before hard failure, recall at a 24 h lead, device-level false-positive rate, precision
- Ingest throughput (rows committed per second), p99 latency from device timestamp to committed row, message loss
- Connection setup time and broker CPU per connection

### 4.6 Deviations from protocol v0.1 (adaptations, listed up front)

1. **H3 cannot be tested as worded.** The ingest path is a single Node.js process (Aedes and SQLite are single-threaded here), so there is no "worker count" to vary and no reason to expect linear scaling. E3c therefore compares one pinned core with two. The "100,000 devices" clause was not reached; E3a reports the real ceiling of this machine and nothing is extrapolated.
2. **E3 fleet sizes are a ladder to a measured ceiling,** not the 10k/50k/100k points listed in the v0.1 matrix.
3. **E4 CPU cost** is broker process CPU time per connection, not whole-machine CPU.
4. **The E2 detector was revised once during development.** The first version measured the sensor drift as a 24 h slope of the device residual; on the unit-test fleet it missed every drifting sensor because the per-device daily cycle aliased into the slope. It was replaced by a day-over-day residual change before any E2 experiment ran. All parameters were then frozen; E2 results use seeds 1 to 20, none of which were used during development. A 3-seed pilot (seeds 9001 to 9003) was run once to check the pipeline before the full run and is not reported.
5. **E1 prevalence conditions (p = 0.3 and 0.1) were added** to probe the limits of H1; v0.1 only specified injecting faulty firmware.
6. **E3 was run more than once, and why.** The first complete run (kept as `results/e3-prelim-*`) used an earlier harness revision. Before the final run the harness was changed for these reasons only: (a) throughput was divided by a wall time that included the drain period, understating it, and CPU utilisation was diluted by the same period, so both are now taken at the end of the measured window; (b) the load generator did not restart publishing between repetitions; (c) the per-step affinity result was not recorded; (d) a shared-machine memory guard was added after free memory fell to under 1 GB and a larger step risked starving other projects; (e) ladders can now continue past a failing step (`--no-early-stop`) because a single 1 to 2 s latency excursion ended the preliminary E3c ladder at its first step. Two further runs were killed before finishing (the tool shell that hosted them restarted) and contribute nothing. The final run uses the same pass criteria and the same default stopping rule as the preliminary run; no threshold was changed. Both complete runs are reported and they disagree on neighbouring steps (section 7, E3).

## 5. Experiment matrix

| ID  | Experiment             | Manipulation (as run)                                                                                                                                                | Measured                                                      |
| --- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| E1  | Rollout safety         | 100 simulated devices; faulty image (degraded at p = 1.0, 0.3, 0.1; bootloop p = 1.0; brick p = 1.0) under gated, timer-staged and all-at-once policies; 10 runs per cell; healthy control | Share of fleet exposed, share harmed, time to pause          |
| E2  | Early failure detection | 20 simulated fleets of 200 devices over 17 days, 10% battery-decay and 10% sensor-drift faults; anomaly rule vs static rules                                         | Lead time, recall, false positives, precision                 |
| E3  | Ingest scaling         | Device ladder at 0.1 msg/s/device; rate ladder at 500 devices; the rate ladder again on one core; up to the first failing step                                       | Throughput, p99 commit latency, loss, SUT CPU and memory      |
| E4  | Identity overhead      | mTLS vs TLS + shared secret vs plain + shared secret; 10 blocks of 200 sequential connections per mode, interleaved                                                  | Connect time, broker CPU per connection                       |

## 6. Statistical plan

- **Replication:** run counts are fixed before data collection and reported in each table: E1 10 per cell, E2 20 replicate fleets, E3 3 repetitions per step, E4 10 blocks per mode.
- **Summaries:** means (or medians where stated) with seeded percentile-bootstrap 95% confidence intervals (2,000 resamples), plus spread in the raw files. Proportions use Wilson intervals.
- **Comparisons:** non-parametric tests: Mann-Whitney U (unpaired, E1) and Wilcoxon signed-rank (paired replicates, E2). With few replicates and discrete outcomes these tests have limited resolution, and some contrasts are so large that the p-value is bounded by the sample size.
- **Effect sizes:** Cliff's delta alongside p-values.
- **Multiple comparisons:** Holm-Bonferroni within each experiment's family of comparisons (adjusted p-values reported).
- **No silent exclusions:** no run was excluded. Runs that hit the 90 s campaign timeout, if any, are counted in the final-state column.
- **Negative results are results:** see Discussion.

## 7. Results

All tables are generated from `results/*.json` and `results/*.csv` by `experiments/summarize.ts` (full text in `results/summary.md`). Console logs of the runs are in `results/logs/`. **Every number is from simulated devices on the shared machine described in section 4.2, which was at 100% CPU busy before E1, E2, E3 (main) and E4 and 64.6% before the E3c supplement.**

### E1. Rollout safety

**Manipulation:** 100 simulated devices receive a correctly signed but faulty image. Policies: `gated` (rings of 5, 20, 35 and 40 devices, health gate, automatic rollback), `timer` (same rings, advanced by a 2 s timer with no health check), `all-at-once` (one ring of 100). Faults: `degraded` (installs fine, then error and reboot counters climb) at prevalence 1.0, 0.3 and 0.1; `bootloop` (new image fails its self-test and the simulated A/B bootloader reverts) at 1.0; `brick` (device hangs after install) at 1.0. A healthy image is the false-trip control.
**Measured:** share of the fleet that ran the faulty image (ground truth from the simulator), share harmed, time from campaign start to the automatic pause. n = 10 runs per cell, seeds recorded in `results/e1-runs.csv`.

| Condition (fault / policy) | n | Result: share of fleet that ran the faulty image (mean) | 95% CI (bootstrap) | Notes |
| --- | --- | --- | --- | --- |
| degraded, p = 1.0 / gated | 10 | 5.0% (5 devices) | [5.0%, 5.0%] | Paused in 10/10 runs, median 2,178 ms after start (about 11 telemetry intervals); rolled back in 10/10; 0% still faulty at the end. |
| degraded, p = 1.0 / timer | 10 | 100.0% | [100.0%, 100.0%] | Never paused; 100% harmed and still faulty. |
| degraded, p = 1.0 / all-at-once | 10 | 100.0% | [100.0%, 100.0%] | Never paused; 100% harmed. |
| degraded, p = 0.3 / gated | 10 | 5.0% | [5.0%, 5.0%] | Paused 10/10, median 2,199 ms; mean 2.1% of the fleet harmed. |
| degraded, p = 0.3 / timer | 10 | 100.0% | [100.0%, 100.0%] | Never paused; mean 30.0% harmed and still faulty. |
| degraded, p = 0.3 / all-at-once | 10 | 100.0% | [100.0%, 100.0%] | Never paused; mean 33.6% harmed. |
| degraded, p = 0.1 / gated | 10 | **31.0%** | [10.5%, 53.0%] | Paused 10/10, median 2,209 ms (CI 2,165 to 6,892). Exposure stayed within the 5-device first ring in only **6 of 10** runs (Wilson 95% [31%, 83%]); in the other 4 the canary contained no affected device and the gate tripped on a later ring (3 runs at 60% exposure, 1 run at 100%). Mean 4.3% harmed; 0% still faulty after rollback. |
| degraded, p = 0.1 / timer | 10 | 100.0% | [100.0%, 100.0%] | Never paused; mean 8.9% harmed and still faulty. |
| degraded, p = 0.1 / all-at-once | 10 | 100.0% | [100.0%, 100.0%] | Never paused; mean 9.2% harmed. |
| bootloop, p = 1.0 / gated | 10 | 5.0% | [5.0%, 5.0%] | Paused 10/10, median 2,199 ms; 5% harmed (and self-recovered through the simulated A/B revert). |
| bootloop, p = 1.0 / timer | 10 | 100.0% | [100.0%, 100.0%] | Never paused; every device ran the bad image once and reverted itself. |
| bootloop, p = 1.0 / all-at-once | 10 | 100.0% | [100.0%, 100.0%] | Never paused; same. |
| brick, p = 1.0 / gated | 10 | 5.0% | [5.0%, 5.0%] | Paused 10/10, median 4,556 ms (install timeout 2.5 s plus soak); **5% of the fleet stays bricked**, rollback cannot reach a hung device. |
| brick, p = 1.0 / timer | 10 | 100.0% | [100.0%, 100.0%] | Never paused; 100% bricked. |
| brick, p = 1.0 / all-at-once | 10 | 100.0% | [100.0%, 100.0%] | Never paused; 100% bricked. |
| healthy image / gated (control) | 10 | 100.0% (rollout completed) | n/a | **0 of 10 false pauses** (Wilson 95% upper bound 28%): every ring passed and the campaign completed. |

| Comparison of exposure share (gated vs baseline) | Mann-Whitney p | Holm-adjusted p (10 comparisons) | Cliff's delta | Reading |
| --- | --- | --- | --- | --- |
| each fault at p >= 0.3 or hard fault, vs all-at-once and vs timer (8 comparisons) | < 0.0001 | 0.0001 | -1.00 | Gated exposure was lower in every run of every pair. |
| degraded p = 0.1, vs all-at-once and vs timer (2 comparisons) | 0.0002 | 0.0003 | -0.90 | Large, but not total, separation: gated runs that escaped the canary reached 60% or 100%. |

Raw per-run data: `results/e1-runs.csv`; summary and parameters: `results/e1-summary.json`.

### E2. Early failure detection

**Manipulation:** 20 independent simulated fleets (seeds 1 to 20) of 200 devices each over 17 virtual days, with 10% of devices given a battery-decay fault and 10% a sensor-drift fault (onset between day 4 and day 8; hard failure 36 to 168 h later for batteries and 48 to 192 h later for sensors). The real telemetry store (rollups) and the real alert engine evaluate hourly. Detectors: the per-model anomaly rule, and static rules (battery < 20%, battery < 30%, sensor outside [10, 30]) evaluated on hourly averages.
**Measured:** lead time (hard-failure time minus first alert), recall of faulty devices with a lead of at least 24 h, device-level false-positive rate among the 160 healthy devices per fleet, precision. Replicate-level means with bootstrap CIs over the 20 fleets; lead times pooled over all faulty devices.

| Condition (detector / fault kind) | n (fleets; faulty devices pooled) | Recall with lead >= 24 h (mean) | 95% CI | Notes |
| --- | --- | --- | --- | --- |
| anomaly rule / all faults | 20; 782 detected of 800 | **97.6%** | [96.7%, 98.5%] | Recall at any positive lead 97.8%; **0 false-positive devices in 3,200 healthy device-fleets** (Wilson 95% upper bound 0.12%); precision 100%; pooled median lead **91.4 h** (CI [87.0, 95.3]). |
| anomaly rule / battery decay | 20; 382 | 95.2% | [93.5%, 97.0%] | Median lead 85.3 h (CI [79.4, 92.3]). All 19 faulty devices that were not caught with a 24 h lead are battery faults with a long time to failure (102 to 168 h): slow decays whose discharge rate stays close to the cohort's normal spread (17 never flagged, 1 flagged 10 h before failure, 1 flagged after failure). |
| anomaly rule / sensor drift | 20; 400 | 100.0% | [100.0%, 100.0%] | Median lead 95.3 h (CI [90.3, 102.1]). |
| static battery < 20% | 20; 400 | 54.8% | [50.7%, 58.7%] | 0% false positives; median lead 26.5 h (CI [23.8, 28.1]); every failing battery is eventually caught, but late. |
| static battery < 30% | 20; 400 | 84.0% | [81.0%, 86.8%] | **17.9% of healthy devices flagged** (CI [16.5%, 19.3%]); precision 41.6%; median lead 44.8 h. |
| static sensor range [10, 30] | 20; 261 detected of 400 | 21.5% | [18.2%, 25.0%] | 0% false positives; only 65.3% of drifting sensors are caught at all before failure; median lead 17.0 h. |

| Paired comparison across the 20 fleets | Wilcoxon signed-rank p | Holm-adjusted p (2 comparisons) | Cliff's delta |
| --- | --- | --- | --- |
| recall (lead >= 24 h): anomaly vs static battery < 20%, battery faults | 0.00009 | 0.00016 | 1.00 |
| recall (lead >= 24 h): anomaly vs static sensor range, sensor faults | 0.00008 | 0.00016 | 1.00 |

Raw per-device data: `results/e2-devices.csv`; per-fleet data: `results/e2-replicates.csv`; parameters and detector constants: `results/e2-summary.json`.

### E3. Ingest scaling

**Manipulation:** simulated devices connect over mTLS and publish QoS 1 telemetry (about 110 bytes) to the real ingest path (broker, validation, rate limiter set high enough never to engage, batched in-memory SQLite with rollups). The system under test is one process pinned to logical cores 4 and 5; the load generator is a separate process pinned to core 6. Three ladders, each step 3 repetitions of a 30 s window after a 3 s settle: **E3a** device count at 0.1 msg/s per device; **E3b** per-device rate at 500 devices; **E3c** the E3b ladder with the system under test pinned to one core. A step passes only if message loss <= 1%, p99 commit latency (device timestamp to committed row) <= 1,000 ms and >= 99% of devices stay connected, in **every** repetition. The default rule stops a ladder at the first failing step. A shared-machine memory guard skips (does not fail) a step whose projected memory would leave less than 3 GB available.
**Measured:** committed messages per second, loss, p99 commit latency, system-under-test CPU and memory.

Three E3 runs exist and are kept, because the harness was corrected between them:

| Run | Files | What it is |
| --- | --- | --- |
| Interrupted (2 runs) | `results/logs/e3-interrupted-run-console.log`, `e3-interrupted-run2-console.log` | Two runs that were killed before finishing (the tool shell hosting them restarted; the second also while waiting for free memory). No results from them are used. |
| Preliminary | `results/e3-prelim-*` | A complete run with an earlier harness revision (see change log). Reported below only as a run-to-run comparison. |
| **Final** | `results/e3-*` (main) and `results/e3c-full-*` (supplement) | The run whose numbers are reported as the E3 result. |

**Final run, E3a (device ladder, 0.1 msg/s per device, 2 cores for the system under test):**

| Condition | n (reps) | Result: committed msg/s; p99 commit latency per rep | 95% CI | Notes |
| --- | --- | --- | --- | --- |
| 250 devices | 3 | 25 msg/s; 63 / 63 / 63 ms | n/a (3 reps, listed) | PASS. Loss 0.00%. Latency is dominated by the 50 ms store flush interval. |
| 500 devices | 3 | 50 msg/s; 63 / 63 / 63 ms | n/a | PASS. |
| 1,000 devices | 3 | 100 msg/s; 64 / 63 / 63 ms | n/a | PASS. |
| 2,000 devices | 3 | 200 msg/s; 64 / 64 / 63 ms | n/a | PASS. |
| 4,000 devices | 3 | 400 msg/s; 64 / 64 / 64 ms | n/a | PASS. |
| **8,000 devices** | 3 | 800 msg/s; 66 / 65 / 65 ms | n/a | PASS; 8,000 of 8,000 connected; broker CPU 0.15 cores; broker RSS about 1.15 GB. **Largest step run.** |
| 16,000 devices | 0 | not run | n/a | Skipped by the memory guard (6.1 GB available, 5.5 GB projected, 3 GB reserve). **Not a software ceiling.** |

**The device-count ceiling was not reached.** Every step up to 8,000 devices passed; 16,000 and above were not attempted. The only statement supported is "at least 8,000 simulated devices connected and publishing at 0.1 msg/s each, with p99 commit latency about 65 ms, on this machine". Nothing is extrapolated to 100,000. Memory use was about 140 KB per connection in the system under test (1.15 GB at 8,000).

**Final run, E3b (rate ladder, 500 devices) and E3c (one core):**

| Condition | n (reps) | Result: committed msg/s; p99 commit latency per rep (ms) | Notes |
| --- | --- | --- | --- |
| 2 cores, 500 msg/s offered | 3 | 501; 65 / 65 / 64 | PASS |
| 2 cores, 1,000 | 3 | 1,000; 66 / 65 / 67 | PASS |
| 2 cores, 2,000 | 3 | 2,000; 66 / 65 / 66 | PASS |
| 2 cores, 4,000 | 3 | 3,993; 266 / 298 / 314 | PASS. **Largest step that passed in all repetitions.** |
| 2 cores, 8,000 | 3 | 7,991; 917 / 490 / **1,481** | FAIL by the rule: one repetition exceeded 1,000 ms. Loss 0.00%. Broker CPU 0.61 cores. |
| 1 core, 500 / 1,000 / 2,000 | 3 each | 500 / 1,000 / 1,998; p99 about 65 ms | PASS |
| 1 core, 4,000 | 3 | 3,998; 327 / 523 / 329 | PASS (main run) |
| 1 core, 8,000 | 3 | 7,952; 616 / 1,248 / 1,616 | FAIL by the rule (two repetitions over 1,000 ms) |

**E3c supplement (one core, no early stop; the machine was 64.6% busy before it started, vs 100% for the main run):**

| Offered msg/s (500 devices) | Committed msg/s | p99 per rep (ms) | Loss | Verdict |
| --- | --- | --- | --- | --- |
| 500 / 1,000 | 500 / 1,000 | 64 to 79 | 0% | PASS |
| 2,000 | 2,001 | 65 / 65 / 355 | 0% | PASS |
| 4,000 | 3,985 | 1,378 / 536 / 434 | 0% | FAIL (one 1.4 s excursion; the same step passed in the main run) |
| 8,000 | 8,016 | 534 / 332 / 346 | 0% | PASS (the same step failed in the main run) |
| 16,000 | 16,012 | 4,040 / 4,035 / 3,348 | 0% | FAIL: sustained queueing, latency of seconds |
| 32,000 | 14,749 | 35,527 / 73,751 / 78,207 | 17.2% | FAIL: collapse |

What the data supports: with the system under test on one pinned core the ingest path kept up with 8,000 msg/s offered in one run and queued badly at 16,000 and collapsed at 32,000 (committed rate about 14,700 msg/s, 17% unaccounted loss). Pass/fail at 4,000 and 8,000 msg/s flipped between runs and repetitions because of isolated latency excursions of 0.5 to 1.5 s, so **the passing boundary on this machine lies between 4,000 and 8,000 msg/s and is not sharper than that**. In every step the system under test used well under one core (at most 0.69 of the one or two it was allowed), so the limit seen is not simple saturation of the pinned cores; on this oversubscribed machine other workloads are the likely cause of the excursions, but that was not isolated.

**Preliminary run (earlier harness revision), for run-to-run comparison:** E3a passed through 4,000 devices and failed at 8,000 because one repetition had p99 1,025 ms (the other two 65 and 66 ms); E3b passed through 8,000 msg/s (p99 756 to 928 ms) and failed at 16,000 (9,780 msg/s committed, p99 14 to 31 s); E3c stopped at its first step because of a single 2.3 s excursion. The final run passed 8,000 devices and failed 8,000 msg/s: the same system gave different verdicts on neighbouring steps in different runs.

Raw per-step data: `results/e3-steps.csv`, `results/e3c-full-steps.csv`, `results/e3-prelim-steps.csv`; latency histograms (1 ms buckets) for every repetition: `results/e3*-latency-histograms.csv`; summaries: `results/e3-summary.json`, `results/e3c-full-summary.json`, `results/e3-prelim-summary.json`.

### E4. Identity overhead

**Manipulation:** the same MQTT broker accepts three kinds of connection, each repeated in 10 blocks of 200 sequential connections (2,000 per mode, blocks interleaved in a seeded random order, after 50 warm-up connections per mode): `mtls` (client certificate), `secret-tls` (TLS with server authentication only, plus username and password), `secret-plain` (no TLS, plus username and password). Broker on two pinned cores in its own process, driver on a third core, named-pipe transport.
**Measured:** time from dial to CONNACK (client side) and broker process CPU time per connection. Ratios are paired by block (10 pairs).

| Condition | n (connections) | Result: median connect time per block, mean over 10 blocks | 95% CI | Notes |
| --- | --- | --- | --- | --- |
| mTLS (client certificate) | 2,000 | 5.74 ms | [5.50, 6.01] | p95 12.2 ms; broker CPU 2,907 us per connection (CI [2,673, 3,180]). |
| TLS + shared secret | 2,000 | 3.71 ms | [3.54, 3.93] | p95 6.4 ms; broker CPU 1,875 us per connection (CI [1,727, 2,039]). |
| plain + shared secret | 2,000 | 0.64 ms | [0.58, 0.73] | p95 3.7 ms; broker CPU 789 us per connection (CI [611, 1,014]). |

| Comparison (paired by block) | Overhead | 95% CI |
| --- | --- | --- |
| mTLS vs TLS + shared secret, connect time | **+55.6%** | [+47.7%, +63.6%] |
| mTLS vs TLS + shared secret, broker CPU per connection | +56.4% | [+43.9%, +73.1%] |
| TLS + shared secret vs plain, connect time | +487% | [+448%, +523%] |
| mTLS vs plain, connect time | +818% | [+726%, +899%] |

Raw per-connection and per-block data: `results/e4-connections.csv`, `results/e4-blocks.csv`; summary: `results/e4-summary.json`.

## 8. Discussion

**H1 (gated rollouts cap exposure at the first ring): supported for faults that affect at least 30% of devices and for hard faults; not supported as worded for a low-prevalence fault.** In all 40 gated runs with degraded p = 1.0 or 0.3, bootloop or brick faults, the faulty image ran on exactly the 5 canary devices, against 100% for the all-at-once baseline and for the timer-staged baseline (Cliff's delta -1.00, Holm-adjusted p 0.0001). With a degraded fault on only 10% of devices the canary contained no affected device in 4 of 10 runs, so exposure reached 60% (3 runs) or 100% (1 run) before a later ring tripped the gate; only 6 of 10 runs stayed within the first ring. That matches simple arithmetic: a 5-device canary sees at least one affected device with probability 1 - 0.9^5 = 41% at p = 0.1 (59% observed here is within sampling noise for n = 10), 83% at p = 0.3 and 100% at p = 1. The gate still paused every one of the 10 low-prevalence runs and rolled back, and the healthy control never tripped (0 of 10), but "capped at the first ring" does not hold when faults are rare. Two further points: the timer-staged arm behaved exactly like all-at-once, so the protection comes from the health check, not from the ring structure; and time to pause (about 2.2 s, 4.6 s for a hang) is set by the configured soak and install timeout, not by detection difficulty. Rollback cannot help a device that is already hung: 5% of the fleet stayed bricked even with the gate, so the canary size is the loss budget.

**H2 (early detection): supported in the simulation, with large caveats.** The per-model anomaly rule flagged 97.6% of failing devices at least 24 h before hard failure (CI 96.7% to 98.5%, threshold 70%), with 0 false-positive devices in 3,200 healthy device-fleets (Wilson upper bound 0.12%, threshold 2%) and a median lead of 91 h. Static thresholds could meet only one side at a time: battery < 20% had no false positives but only 54.8% recall at 24 h lead, battery < 30% reached 84.0% recall but flagged 17.9% of healthy devices, and a fixed sensor range caught 21.5%. The detector's misses were all slow battery decays. This result is conditional on a simulator written by the same author, on clean Gaussian noise, and on detector constants taken from that simulator's noise scale; the first version of the sensor feature failed completely on a development fleet because a per-device daily cycle aliased into its slope, which is a reminder that design details, not the idea of peer comparison, decide the outcome. The size of the gap between detector and baselines is not transferable to real fleets; the direction (cohort comparison removes shared cycles and avoids the lead-versus-false-alarm trade-off of a fixed threshold) is plausible but is not shown by this study.

**H3 (ingest scaling): not supported as worded, and mostly not testable.** There is no worker count to vary (one single-threaded process), and pinning the system under test to one core instead of two made no measurable difference up to 4,000 msg/s (both passed; the system used under 0.7 of a core throughout), which is the opposite of near-linear scaling with cores. The "100,000 devices" clause was not reached: **8,000 simulated devices at 0.1 msg/s each passed with p99 commit latency about 65 ms, and nothing larger was run** because a shared-machine memory guard skipped the 16,000-device step; that is a limit of the test environment, not a measured software ceiling. On the rate axis, 4,000 msg/s from 500 devices passed in every repetition with p99 below about 0.3 s on two cores; 8,000 msg/s was borderline (it failed on one slow repetition in the main run, passed in the supplement); 16,000 msg/s queued for seconds and 32,000 msg/s collapsed with 17% loss on one core. The p99 < 1 s clause held up to the 4,000 msg/s boundary and, at low rates, with a wide margin. Run-to-run variation was as large as the differences between neighbouring steps, so the boundary between 4,000 and 8,000 msg/s is the most that can be said.

**H4 (identity overhead under 10%): not supported.** Mutual TLS added 55.6% to connection-setup time compared with TLS plus a shared secret (CI 47.7% to 63.6%) and 56.4% to broker CPU per connection (CI 43.9% to 73.1%); the interval excludes 10% by a wide margin. In absolute terms that is about 2 ms of connect time and 1 ms of broker CPU per connection on this machine over a named pipe. TLS itself dominates: adding TLS to the shared-secret baseline cost +487%. Whether about 2 ms matters depends on reconnect behaviour (reconnect storms after an outage are where it would), which this study did not measure. The comparison used full handshakes only; session resumption would change both TLS rows.

**What surprised me.** (1) The staged-but-ungated rollout offered no protection at all. (2) The first detector design failed outright on drifting sensors for a reason that had nothing to do with the statistics (daily-cycle aliasing). (3) The ingest path was not CPU-bound on its pinned cores yet showed 0.5 to 1.5 s latency excursions on a busy machine; the pre-registered "first failing step" rule is too brittle for such a noisy environment, which is why a supplementary no-early-stop run was added and why the two E3 runs disagree on neighbouring steps.

**Practical guidance.** Size the canary ring to the fault prevalence you want to catch (a canary of k devices detects a fault affecting a share p of devices with probability 1 - (1 - p)^k), keep a known-good image for automatic rollback, and treat hung devices as the unrecoverable cost of the canary. Do not use "staged" as a synonym for "safe": without a gate it is not. Budget for mTLS as an extra 50% of connection setup cost, and measure on the target hardware.

## 9. Threats to validity

- **Simulated faults and physics (construct validity).** Every fault, every device and every load is synthetic and was written by the same author who wrote the detector and the gates. Real failure modes are messier. The E2 detector's constants were chosen from the simulator's own noise scale, so its near-perfect separation is partly built in. Validate on field data before believing any E2 or E1 number outside this simulator.
- **Zero false positives in E2 is a property of clean synthetic noise.** Healthy devices are Gaussian around smooth cycles with no site-specific differences, no heavy tails, no firmware-induced step changes and no maintenance events. 0 flagged of 3,200 healthy device-fleets only bounds the simulated rate (Wilson upper bound 0.12%); it says nothing about real fleets.
- **Shared, saturated machine (internal validity of every timing result).** The machine was at 100% CPU busy before every run except the E3c supplement (64.6%), with three unrelated projects building and testing on it, and was short of free memory during E3 (see section 7). Affinity pinning restricts which cores a process may use but does not give it exclusive access to them: in E3 the system under test consumed well under one core of its two while latency rose, which is what contention looks like. Short stalls of 1 to 2 seconds appear in some steps at low load and were not reproducible (see the E3 preliminary run). Ceilings reported here are for this contended machine, are not upper bounds, and would change on an idle one.
- **Memory limits bound E3a.** Steps were skipped or delayed by a shared-machine memory guard, not by software failure; where that happened the table says so.
- **Transport is not TCP.** mTLS ran over a named pipe because loopback TLS is intercepted on this machine. There is no network latency, loss or reordering, and socket-layer costs differ from TCP. E4's ratios apply to the TLS handshake and MQTT CONNECT path as measured here.
- **In-memory SQLite.** No disk I/O in the measured path (all experiments). A disk-backed store would lower E3 throughput by an unmeasured amount.
- **Gate thresholds were fixed, not explored.** v0.1 promised a sensitivity analysis over gate thresholds; **none was run**. E1 varies fault prevalence but not ring sizes or thresholds, so how much the 10% bad-device threshold and the 5% canary matter is untested.
- **Small samples.** E1 has 10 runs per cell on a 100-device fleet; outcomes are discrete and proportions have wide Wilson intervals. E3 has 3 repetitions per step. E4 compares 10 blocks per mode.
- **Compressed time.** E1 uses 200 ms telemetry and a 2 s soak; time to pause scales with the configured soak and install timeout, not with fleet size or detection difficulty, and says nothing about real-world rollout durations. E2 compresses 17 days into a replay of hourly evaluations.
- **Load generator realism (E3).** A hand-built MQTT client sending QoS 1 messages of about 110 bytes, one tenant, a rate limiter set so high it never engages. Real clients, bigger payloads, retained messages, many subscribers and TLS session resumption would change the numbers.
- **E4 scope.** Sequential connections only (no concurrency), ECDSA P-256 certificates only, full handshakes with no session resumption, and the client library's own cost is included in connect time.
- **Researcher degrees of freedom.** Listed in section 4.6: one detector revision before E2 ran, E3 harness revisions after two aborted or superseded runs (the earlier complete run is kept and reported as the preliminary run), E1 prevalence conditions added, and an unreported 3-seed E2 pilot and 2-repetition E1 smoke run to check the pipelines. The stopping rule for E3 ("first failing step") is brittle under noise, which is why supplementary no-early-stop runs exist; both are reported.
- **Statistical caveats.** Bootstrap intervals on all-equal samples collapse to a point (for example 5.0% exposure in 10 of 10 runs); the tables say so rather than implying certainty. Tests on small discrete samples are conservative in resolution and their p-values are bounded below by the number of runs.

## 10. Reproducibility checklist

- [ ] Code tagged at the commit used for the reported results. _Not true: the working directory is not under version control yet, so there is no commit to tag._
- [ ] Environment pinned (lockfile and container image digest recorded). _Partly true: `package-lock.json` pins every dependency and its hash prefix is recorded in each result file, but no container image was used, so the box stays unticked._
- [x] Random seeds and run counts recorded. _E1 and E2 record seeds and run counts; E3 and E4 record repetition counts and use fixed seeds for block order. Wall-clock timing results are not bit-reproducible._
- [ ] Raw results published (CSV or Parquet) alongside the analysis notebook. _Raw CSV and JSON are in `results/` and the analysis is a script (`experiments/summarize.ts`), not a notebook; nothing has been published anywhere, so the box stays unticked._
- [x] Hardware or service configuration described in full. _CPU model, logical cores, pinning, whole-machine load before each run, memory situation, OS and Node version: section 4.2 and every summary file._
- [x] One command reproduces every figure and table. _`npm run experiments` re-runs E1 to E4 (about two to three hours on this machine) and regenerates `results/summary.md`, from which the tables below are copied; there are no figures. Timing-dependent numbers will differ on other hardware._
- [x] Data sources, licenses and versions listed. _There are no external datasets: all data is generated by seeded simulators in this repository. Software versions are in `package-lock.json`._

## 11. References

- OASIS (2019). MQTT Version 5.0. OASIS Standard.
- Cooper, D. et al. (2008). RFC 5280: Internet X.509 Public Key Infrastructure Certificate and CRL Profile. IETF.
- Moran, B., Tschofenig, H., Brown, D., Meriac, M. (2021). RFC 9019: A Firmware Update Architecture for Internet of Things. IETF.
- Moran, B., Meriac, M., Tschofenig, H., Brown, D. (2022). RFC 9124: A Manifest Information Model for Firmware Updates in Internet of Things (IoT) Devices. IETF.
- Kuppusamy, T. K., Brown, A., Awwad, S., McCoy, D., Bielawski, R., Mott, C., Lauzon, S., Weimerskirch, A., Cappos, J. (2016). Uptane: Securing Software Updates for Automobiles. escar Europe.
- Beyer, B., Jones, C., Petoff, J., Murphy, N. R. (eds.) (2016). Site Reliability Engineering. O'Reilly (release engineering and canarying).
- Leys, C., Ley, C., Klein, O., Bernard, P., Licata, L. (2013). Detecting outliers: Do not use standard deviation around the mean, use absolute deviation around the median. Journal of Experimental Social Psychology 49(4).
- Chandola, V., Banerjee, A., Kumar, V. (2009). Anomaly detection: A survey. ACM Computing Surveys 41(3).
- Efron, B., Tibshirani, R. J. (1993). An Introduction to the Bootstrap. Chapman and Hall.
- Wilson, E. B. (1927). Probable inference, the law of succession, and statistical inference. Journal of the American Statistical Association 22(158).

> Citations were written from memory and have not been verified against the original sources. Verify each one before publishing.

## 12. Change log

| Date       | Version | Change                                                                                                                                                                                                                                                                    |
| ---------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-10-21 | v0.1    | Protocol drafted; no results yet (blueprint).                                                                                                                                                                                                                             |
| 2026-10-21 | v0.2    | Prototype built and the four experiments run on simulated devices. Hypotheses unchanged. Method rewritten to describe what was run. Deviations listed in section 4.6 (H3 adaptation, E3 ladder, E4 CPU definition, one detector revision before E2 ran, added E1 prevalence conditions). Results, discussion, threats and checklist filled in. E3 was run three times (one superseded preliminary run, one final run with a supplement, plus two runs killed mid-way); reasons in section 4.6 item 6. |

---

[README](../README.md) · [Client case study](case-study.md) · [Research report](research-report.md)
