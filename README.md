# Project service due dates across a compressor house

A harbor-format benchmark task, authored for the AfterQuery Experts programme as data
annotation for AI evaluation. Task slug `afterquery/project-compressor-services` — **Operations / Plant Maintenance**.

An AI agent gets a container, this brief and nothing else. It works alone for
4 hours. Then a sealed verifier it never sees scores the result `0` or `1`.

## What it asks for

Service due dates for the compressor house

The full brief is in [`instruction.md`](instruction.md).

| | |
| --- | --- |
| **Deliverable** | `/app/service_due.csv` |
| **Agent budget** | 4 hours (14400s) |
| **Expert estimate** | 4 hours |
| **Machine** | 1 CPU, 2048 MB RAM, 0 GPU |
| **Verifier budget** | 300s |
| **Tags** | `maintenance-planning`, `compressed-air`, `current-monitoring`, `data-cleaning`, `python` |

## Why it is hard

Projecting service dates from logged use is routine work for a maintenance planner or reliability engineer looking after a compressed-air plant: screws are serviced by running hours, pistons by starts, and both come off the power monitor's current log. The data is synthetic and was made truth first. Eight compressors were run for a fortnight, 120960 ten-second lines, each started and stopped by the same sequencer, and each machine's running hours, starts and due date were taken from what it really did.

<details>
<summary>Full note</summary>

The monitor's log was written afterwards, channel by channel. The sealed plan is exact and nothing is computed at marking time. The planner's note gives every rule the marking uses, threshold and rounding included. Read literally it gets 2 of 8, because three channels are plainly not logged as it says: one is in milliamps, one carries the current transformer's secondary amps and needs its ratio, one reads negative because its transformer faces the wrong way. Each is obvious on sight and each spoils that compressor's line, where every line has to be right. What decides the task is the wiring. The schedule says channel n is compressor n; on 4 channels it is not, two pairs of current transformers having been landed on each other's terminals when the monitor was fitted. One of the four channels also has an odd notation; the other three are logged exactly as the note says. The scheduled reading fails silently - every compressor still gets believable hours, starts and a date - and gets 4 of the 8 lines wrong. Nothing first-order shows it. Each crossed pair is two machines of the same rating and full-load amps, so the 20 per cent threshold and the running level are the same for both; every machine is started and stopped by one sequencer, so starts per running hour are alike for the two kinds; and no two machines share a pattern that could be matched. What differs is second order. A piston compressor's current pulses with the strokes and a screw's does not, so set against the two lines either side, a running piston's current is ragged and a screw's is smooth: the smoothest piston channel is 6.0 times as ragged as the roughest screw channel. Four channels show the other kind of machine from the one scheduled, and within a rating that settles which is which. The register's kind column is needed anyway, because the kind decides whether hours or starts count; the note never says what a kind of machine does to its current. tests/house_check.py measures all of this against the sealed plan and refuses to pass otherwise: of 16 readings the right one gets 8 and the best other 7.

</details>

## How it is verified

The verifier reads /app/service_due.csv with the csv module and compares each compressor's running hours within 0.2, its starts exactly and its due date within a day with /tests/sealed.json; all 8 lines must be right. None of the agent's code is imported or run. The sealed plan exists only in the verifier image, /tests is mode 700 there, and the agent's container never mounts it.

<details>
<summary>Full note</summary>

Exact starts are fair because the note fixes the threshold and the logged current is either under a fiftieth or over a third of full-load amps on every line, so no line sits near the fifth. The two machines of a crossed pair differ by more than a running hour, by several starts and by months of due date, so the scheduled reading gets neither right by chance. Nothing in the agent's container attests any figure and nothing has to balance; a plan made on the wiring schedule is as self-consistent as the right one. The reward is binary and is written to /logs/verifier/reward.txt on every path; an empty container scores 0 and the reference scores 1. The generator is seeded and shipped.

</details>

<details>
<summary>Reference approach (spoiler)</summary>

solution/service_plan.py reads only /app and writes /app/service_due.csv with the standard library. It brings each channel to primary amps - dividing the milliamp channel by a thousand, passing the secondary-amp channel through its ratio, taking the size of the negative one. For each channel it measures the middle distance of a running line's current from the mean of the lines either side, as a share of the running level, and calls the channel piston or screw accordingly; a channel that shows the other kind from the machine scheduled on it is given to the other such machine of the same full-load amps. Running lines are those at or above a fifth of the machine's full-load amps, a start is a running line after a line that was not, and the due date follows the note's projection. In the task's own container it gets 8 of 8.

</details>

## Layout

```
instruction.md          the brief - the only thing the agent is told
task.toml               artifacts, timeouts, resources, metadata
environment/            Dockerfile and data for the agent's container
solution/solve.sh       reference solution; scores 1 by construction
tests/                  verifier image, grading logic and ground truth
```

## Running it

With [harbor](https://github.com/laude-institute/harbor) installed and Docker running:

```bash
uv tool install harbor

harbor run -p . -a oracle -e docker   # reference solution - must score 1
harbor run -p . -a nop    -e docker   # nothing happens    - must score 0
```

Those two runs are the calibration the bundle has to satisfy: the oracle run substitutes
`solution/solve.sh` for the agent and must score `1`, the nop run does nothing and must
score `0`. Anything in between means the verifier is measuring the wrong thing.

## Reading this

`tests/` and `solution/` hold the answers. To try the task honestly, read only
`instruction.md` and `environment/`.

---

Part of a set of 38 tasks indexed at
[afterquery-data-annotation](https://github.com/helexdev1004/afterquery-data-annotation).
