# Daily Task Prioritization Agent
# Daily Task Prioritization Agent

Short, focused README replacing the longer version. This README explains what the agent does, how to run it, and the run-time flow.

---

## One-line summary

Reads tasks from `tasks.csv`, scores them with a simple weighted heuristic, and writes a prioritized plan to `plan.txt` and `plan.json`.

---

## Quick start

1. Install Python 3.8+.
2. Edit `tasks.csv` with your tasks.
3. Run:

```bash
python agent.py
```

Outputs: `plan.txt` (summary) and `plan.json` (detailed).

---

## Files

```
agent.py
tasks.csv
plan.json
plan.txt
README.md
```

---

## Execution flow (short)

1. Read CSV → `Task` objects
2. Parse/normalize fields (`deadline`, `effort`, `impact`, `blocked`, `tags`)
3. Compute scores with `compute_score`
4. Sort and bucket into `top3`, `next5`, `unblock`, `defer`
5. Write `plan.json` and `plan.txt` and print the summary

---

## Scoring (quick)

- Urgency: based on days to deadline (overdue = highest)
- Impact: `low=1`, `medium=2`, `high=3`
- Quick-win bonus: +1 if effort ≤ 15 min
- Blocked penalty: −5 if blocked

Formula:

```
score = (2.0 * urgency) + (3.0 * impact) + quickwin_bonus - blocked_penalty
```

Constants are at the top of `agent.py`.

---

## CSV input columns

`title,description,deadline,effort,impact,blocked,tags`

- `title` required
- `deadline` `YYYY-MM-DD` or blank
- `effort` `S`/`M`/`L` or minutes like `25m`
- `impact` `low`/`medium`/`high`
- `blocked` `yes`/`no`
- `tags` comma-separated

---

If you want a different tone or additional examples, tell me the style (short/school-style/technical) and I will regenerate.

Repo: https://github.com/Sree-8639/Ai-Agents-30-days-Learning-
3. **Title alphabetically** (final tie-break for determinism)

After sorting, tasks are split into four buckets:

| Bucket           | Contents                                                                 |
|------------------|--------------------------------------------------------------------------|
| **Top 3**        | The 3 highest-scoring unblocked tasks — do these first today             |
| **Next 5**       | The next 5 highest-scoring unblocked tasks — tackle if Top 3 finish early|
| **Unblock**      | All blocked tasks — requires removing a dependency before starting        |
| **Defer**        | Remaining unblocked tasks that have **both** low urgency and low impact   |

### Stage 5 — Serialize (`serialize`)

Each bucketed task is converted into a dictionary containing:
- Core fields: `title`, `description`, `deadline`, `effort_min`, `impact`, `blocked`, `tags`
- Computed fields: `score`, `reason` (plain-English explanation), `score_breakdown` (per-component values)

### Stage 6 — Render & Write (`render_summary` + `main`)

- `plan.json` is written via `json.dump` — full structured data for tooling or follow-up automation
- `plan.txt` is written via `render_summary` — a clean, human-readable report with sections and rankings
- The same summary is printed to the console in real time

---

## Scoring Algorithm

### Urgency Score (`urgency_score`)

Computed from days remaining until the deadline:

| Days Until Deadline | Urgency Score |
|---------------------|---------------|
| Overdue (< 0)       | 5.0           |
| Due today (0)       | 5.0           |
| Tomorrow (1)        | 4.0           |
| 2–3 days            | 3.0           |
| 4–7 days            | 2.0           |
| > 7 days            | 1.0           |
| No deadline         | 0.5 (baseline)|

### Composite Score Formula

```
score = (2.0 × urgency) + (3.0 × impact) + quickwin_bonus − blocked_penalty
```

| Component          | Weight | Notes                                      |
|--------------------|--------|--------------------------------------------|
| Urgency            | ×2.0   | Based on deadline proximity                |
| Importance/Impact  | ×3.0   | `low=1`, `medium=2`, `high=3`              |
| Quick-win bonus    | +1.0   | Applied when `effort ≤ 15 min`             |
| Blocked penalty    | −5.0   | Applied when `blocked = yes`               |

**Example:**  
A High-impact task due today with 25 min effort (not blocked):  
`score = (2.0 × 5.0) + (3.0 × 3.0) + 0 − 0 = 10 + 9 = 19.0`

---

## Configuration

All weights, thresholds, and bucket sizes are defined as constants at the top of `agent.py` and can be tuned without touching any logic:

```python
EFFORT_DEFAULTS_MIN = {"S": 15, "M": 45, "L": 90}
IMPACT_MAP = {"low": 1, "medium": 2, "high": 3}

WEIGHTS = {
    "urgency":         2.0,
    "importance":      3.0,
    "quickwin_bonus":  1.0,
    "blocked_penalty": 5.0,
}

TOP3_COUNT  = 3   # Number of tasks in the "Do first" bucket
NEXT5_COUNT = 5   # Number of tasks in the "Next" bucket
```

---

## Input Format — tasks.csv

The agent reads from `tasks.csv` in the same directory. The file must have a header row with these columns:

| Column        | Required | Format / Accepted Values                                      |
|---------------|----------|---------------------------------------------------------------|
| `title`       | Yes      | Free text — rows with blank titles are skipped                |
| `description` | No       | Free text summary of the task                                 |
| `deadline`    | No       | `YYYY-MM-DD` or empty for no deadline                         |
| `effort`      | No       | `S` (15 min), `M` (45 min), `L` (90 min), `"25m"`, or `"25"` |
| `impact`      | No       | `low`, `medium`, `high` — defaults to `medium`               |
| `blocked`     | No       | `yes`/`no`, `true`/`false`, `1`/`0` — defaults to `no`        |
| `tags`        | No       | Comma-separated labels, e.g. `work,urgent`                    |

**Example `tasks.csv`:**

```csv
title,description,deadline,effort,impact,blocked,tags
Send proposal to client,Finalize and email proposal,2026-02-25,25m,high,no,work
Pay electricity bill,Online payment,2026-02-25,10m,medium,no,personal
Book dentist appointment,Call dentist office,,10m,low,no,personal
Prepare meeting agenda,Outline topics and decisions,2026-02-26,30m,high,no,work
Fix login bug,Reproduce and patch issue,2026-02-27,L,high,no,work
Wait for design assets,Need final banner files,2026-02-25,15m,high,yes,work
Clean downloads folder,Organize files,,S,low,no,personal
```

---

## Output Files

### plan.txt (Human-readable)

```
Daily Task Prioritization Plan (2026-02-24)
=============================================

TOP 3 (Do these first)
----------------------
1. Send proposal to client  | deadline: 2026-02-25 | effort: 25m | score: 19.0
   Why: Due soon, High impact
2. Prepare meeting agenda  | deadline: 2026-02-26 | effort: 30m | score: 19.0
   Why: Due soon, High impact
3. Fix login bug  | deadline: 2026-02-27 | effort: 90m | score: 19.0
   Why: Due soon, High impact

NEXT 5
------
1. Pay electricity bill  | deadline: 2026-02-25 | effort: 10m | score: 15.0
   Why: Due soon, Medium impact, Quick win

UNBLOCK (Blocked tasks)
-----------------------
1. Wait for design assets  | deadline: 2026-02-25 | effort: 15m | score: 10.0
   Why: Due soon, High impact, Quick win, Blocked (needs unblock step)

DEFER (Low urgency/impact)
--------------------------
1. Book dentist appointment | deadline: none | effort: 10m | score: 5.0
   Why: No deadline, Quick win
```

### plan.json (Machine-readable)

Contains `top3`, `next5`, `unblock`, `defer` arrays plus an `assumptions` block with all weights and config used for that run. Every task entry includes `score`, `reason`, and a full `score_breakdown` dictionary.

---

## Getting Started

### Prerequisites

- Python 3.8 or higher
- No external libraries required (standard library only)

### Run

```bash
# Clone the repository
git clone https://github.com/Sree-8639/Ai-Agents-30-days-Learning-.git
cd "Ai-Agents-30-days-Learning-/Daily Task Prioritization Agent"

# Edit tasks.csv with your actual tasks, then run:
python agent.py
```

The agent will print the prioritized plan to the console and save `plan.txt` and `plan.json` in the same folder.

---

## Sample Run

**Input (`tasks.csv`)** — 7 tasks across work and personal categories.

**Console Output:**

```
Daily Task Prioritization Plan (2026-02-24)
=============================================

TOP 3 (Do these first)
----------------------
1. Send proposal to client  | deadline: 2025-12-22 | effort: 25m | score: 19.0
   Why: Overdue, High impact
2. Prepare meeting agenda  | deadline: 2025-12-23 | effort: 30m | score: 19.0
   Why: Overdue, High impact
3. Fix login bug  | deadline: 2025-12-24 | effort: 90m | score: 19.0
   Why: Overdue, High impact

NEXT 5
------
1. Pay electricity bill  | deadline: 2025-12-22 | effort: 10m | score: 17.0
   Why: Overdue, Medium impact, Quick win
2. Book dentist appointment  | deadline: none | effort: 10m | score: 5.0
   Why: No deadline, Quick win
3. Clean downloads folder  | deadline: none | effort: 15m | score: 5.0
   Why: No deadline, Quick win

UNBLOCK (Blocked tasks)
-----------------------
1. Wait for design assets  | deadline: 2025-12-22 | effort: 15m | score: 15.0
   Why: Overdue, High impact, Quick win, Blocked (needs unblock step)

DEFER (Low urgency/impact)
--------------------------
  (none)

Saved: plan.json and plan.txt
```

**Outputs saved:** `plan.json` and `plan.txt`

---

> Part of the **30 Days of AI Agents Learning** series.  
> Repository: [github.com/Sree-8639/Ai-Agents-30-days-Learning-](https://github.com/Sree-8639/Ai-Agents-30-days-Learning-)
