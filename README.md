# Ai-ops
Versioned AI-Ops

# ByteLens infra-clearance POC

Goal: when an Airtel Money database incident happens, find the infrastructure root cause (or clear infra) in 5-10 minutes
instead of about 30, by correlating the monitoring tools of every domain with one ML pipeline.

This repo builds that in small steps. Real Airtel data comes later; until then everything runs on a lab you control
and on generated (dummy) data. **Nothing here is Airtel data.**

## Project layout

| Folder | What it is |
| --- | --- |
| `lab/` | **Step 1.** A small real data source: Prometheus, node_exporter, Postgres and a load generator in Docker, plus a script that injects a disk-I/O fault and a script that checks the data is flowing. |
| `demo/` | The finished-looking dummy version: data generator, ML models, FastAPI backend and a web UI. Use it to show the idea and to see where the steps are heading. |

## Progress (tick as you go)

- [ ] **Step 1: data source.** `lab/` running, `check_data.py` shows at least 4 KPIs, and an injected I/O fault is visible.
- [ ] **Step 2: collector.** Pull KPIs from Prometheus into one table (time, component, domain, signal, value).
- [ ] **Step 3: baseline and anomaly detection** on one KPI (disk I/O wait). Catches the injected fault, quiet otherwise.
- [ ] **Step 4: score it.** Repeat the fault 10+ times at different strengths and times; record detection rate and false alarms against `faults.log`.
- [ ] **Step 5: second domain and correlation.** Add the database signals; the disk fault should show first, the database slowdown after.
- [ ] **Step 6: root cause score with confidence**, compared with the cause you injected.
- [ ] **Step 7: UI.** Reuse or adapt `demo/static/index.html`.

## Step 1: run the lab

Needs Docker (on Windows: Docker Desktop) and Python 3 with `pip install requests`.

```
cd lab
docker compose up -d
```

Wait about 3 minutes, then:

```
python check_data.py
```

You should see data for at least 4 of 6 KPIs, with a small trend line each. Prometheus itself is at http://localhost:9090.

Inject the fault (a heavy disk writer on the same disk as the database, the lab version of a backup in business hours):

| Where | Start (10 minutes) | Stop |
| --- | --- | --- |
| Windows PowerShell | `.\scripts\fault_io.ps1 start 600` | `.\scripts\fault_io.ps1 stop` |
| Linux, macOS, WSL, Git Bash | `scripts/fault_io.sh start 600` | `scripts/fault_io.sh stop` |

Run `python check_data.py` again while it runs: I/O wait, disk busy and write latency should jump.
Every start and stop is appended to `lab/faults.log` in UTC. That file is the answer key for scoring your models in step 4.

Note for Docker Desktop on Windows or macOS: the host metrics describe Docker's Linux VM, not Windows itself.
That is fine for the lab, because Postgres and the fault run in the same VM.

Step 1 is done when the check shows data and the fault is visible in the numbers.

## The demo (optional, to see the end state)

```
cd demo
pip install fastapi uvicorn numpy pandas scikit-learn
uvicorn app:app --port 8765
```

Open http://localhost:8765. Five scenarios: backup on a shared volume (default), ESXi overload, network latency,
infra clear, and a stale storage feed (verdict Inconclusive). It shows the verdict with confidence, the root-cause chain,
all eleven domains, the disk to VM to volume map, noisy alerts removed, and KPI charts against their baselines.

How the demo reasons, which you will rebuild step by step:

1. **Baseline and anomaly detection.** Median and MAD per 15-minute bucket of the day from 6 days of history; a KPI is
   deviating when it is more than 5 robust deviations and 25% off baseline for 4 of 5 minutes. An IsolationForest on the
   golden signals gives a second opinion.
2. **Evidence collector.** Drops flapping, off-path and below-significance alerts.
3. **Correlation.** Lead-lag cross-correlation along the Airtel Money dependency chain.
4. **Root cause with confidence.** Weighted score of earliest onset, depth in the stack, linked change or job, and how much
   of the rest it explains; a softmax turns the scores into a confidence.
5. **Verdict.** Infra Suspect, Infra Clear or Inconclusive. Missing or stale data is never reported as healthy.

To move the demo to real data, replace `build_scenario()` in `demo/simulate.py` with loaders that return the same frame
(columns = KPI ids, 1-minute index), the events list and the stale-feed map. `models.py` and the UI stay as they are.

## Troubleshooting

- `check_data.py` shows NO DATA for the Postgres KPIs: the load generator needs a few minutes to create its tables. Wait and rerun.
- No data at all: open http://localhost:9090/targets and check that both targets are UP; then `docker compose logs <service>`.
- The fault shows no effect: lower-end disks can absorb it; raise the duration, or run the fault script twice in parallel.

