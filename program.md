# Project Categorizer — Autonomous Experiment Loop

This is an autoresearch-style experiment for improving the infrastructure project categorizer. The categorizer classifies procurement projects into 6 dimensions using an LLM. Your job: make it more accurate.

## Architecture overview

The categorizer pipeline processes records through 6 sequential/parallel steps:

1. **Sector** (primary + secondary) — e.g., `Wastewater` primary, `Stormwater` secondary for combined sewer projects
2. **Application** (primary + secondary) — candidate set determined by union of Sector primary + secondary
3. **Sub-application** (primary + secondary) — candidate set determined by Application
4. **Technology Tags** (multi-label) — runs in parallel with 5 & 6; uses `exclude_utility_description=True` to prevent hallucination from utility-level descriptions. Technology names are full names from Excel hierarchy (e.g., "Pipes", "Pumps"), not codes.
5. **Theme Tags** (multi-label) — runs in parallel with 4 & 6
6. **Project Type Tags** (multi-label) — runs in parallel with 4 & 5

Key architectural elements:
- **Input field selection** via `_slim_project()`: controls which fields are sent to the LLM. The `exclude_utility_description` parameter filters out `Project Description (PD)` (utility-level) and only keeps `Description for EU WIN.1` (project-specific) — currently enabled only for Technology extraction.
- **Field name helper** `_get_project_name()`: handles both raw field names (`Project Name (PN)`) and normalized names (`project_name`) in post-processing rules.
- **Post-processing rules**: regex-based hardcoded overrides per dimension that run AFTER LLM classification. These handle known patterns (e.g., Regenbauwerke → Stormwater, serbatoio → Water Resources, UWWTD stripping from non-qualifying projects).
- **Batch processing**: records are processed in batches of 20 per API call for efficiency (configurable via `BATCH_SIZE` env var).
- **Disk-based LLM cache** (`.llm_cache/`): identical prompt+model+temperature+seed combinations return cached results, ensuring 100% deterministic output when cache is preserved. Only clear cache when prompts or model change.
- **Excluded fields**: `AI generated description` is excluded from LLM input globally (saves tokens without affecting accuracy).

**CRITICAL — Strict hierarchy compliance:**
All categorizations MUST strictly follow the hierarchy defined in `042026 latest definitions/CATEGORIZATION_Hierarchy 3 (1) (2).xlsx`. The hierarchy is the single source of truth. Key rules:
- **Sub-applications are filtered by parent Application.** Each Application has a specific set of valid Sub-applications. A Sub-application from one Application MUST NOT appear under a different Application.
- **Applications without Sub-applications:** Sludge Management, Water Treatment, and Desalination have NO sub-applications defined in the hierarchy. They only get universal subs (Planning & Engineering, Construction Budget / Unspecified Construction, Administrative / Support). Do NOT invent sub-applications for these (e.g., "Anaerobic Digestion" is a Technology, NOT a Sub-application).
- **Universal Sub-applications** (available under ALL Applications, no parent restriction): `Planning & Engineering`, `Construction Budget / Unspecified Construction`, `Administrative / Support`.
- **Technologies are NOT Sub-applications.** Sludge-related process types (Anaerobic Digestion, Sludge Thickening, Sludge Dewatering, etc.) are Level 4 Technologies, not Level 3 Sub-applications.

## Setup

To set up a new experiment run, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `apr10`). The branch `categorizer/<tag>` must not already exist.
2. **Create the branch**: `git checkout -b categorizer/<tag>` from the current best commit.
3. **Read the in-scope files** for full context:
   - `eval.md` — evaluation criteria, scoring logic, current best result. Auto-updated after each run.
   - `../eu_project_categoriser/categorization_service.py` — **the file you modify**. Prompts, rules, few-shot examples, post-processing.
   - `../eu_project_categoriser/eval_accuracy.py` — fixed evaluation script. Do not modify.
4. **Run baseline**: `python eval_accuracy.py --iteration baseline` to establish starting accuracy.
5. **Confirm and go**: Confirm setup looks good, then begin the loop.

## Experimentation

Each experiment runs the categorization pipeline on **201 ground-truth records** (~25-50 seconds wall clock with batch size 20). You launch it as:

```
python eval_accuracy.py --iteration expN --status pending --desc "short description" 2>run.log
```

Run this from the `../eu_project_categoriser/` directory.

**What you CAN do:**
- Modify `categorization_service.py` — this is the only file you edit. Everything is fair game: system prompts, CoT reasoning templates, few-shot examples, keyword hints, heuristic rules, post-processing logic, confidence thresholds, `_slim_project()` field selection, `_get_project_name()` helper.

**What you CANNOT do:**
- Modify `eval_accuracy.py`. It is read-only. It contains the fixed evaluation harness.
- Modify ground truth data files or hierarchy definitions.
- Modify `openrouter_service.py`, `hierarchy_loader.py`, or `prompts_config.py`.
- Install new packages or add dependencies.

**The goal is simple: get the highest `overall_accuracy`.** Target: **90%+** (already achieved at 91.3%; push higher). The 6 dimensions are: Sector, Application, Sub-application, Technology, Theme, Project Type. Overall accuracy = mean of all per-record scores across all dimensions.

**Important post-processing patterns to be aware of:**
- Sector: Combined sewer keywords split by type — MW/Mischwasser/sfioratore/RÜB → Stormwater primary + Wastewater secondary; Reinwasserableitung/Fremdwassersanierung → Wastewater primary + Stormwater secondary. Regenbauwerke → Stormwater. Klärschlamm/centrifug → Wastewater (empty fallback). serbatoio → Water. Duplicate secondary removal (secondary ≠ primary).
- Application: Solar at WWTP → Other; serbatoio → Water Resources; ACQUEDOTTO → Water Networks; fanghi/Klärschlamm/boues/sludge → Sludge Management (not WW Treatment). Duplicate secondary removal.
- Sub-application: Regenbauwerke → Gray Infrastructure; serbatoio → Dams & Reservoirs; ACQUEDOTTO + canale principale/adduttrice → Water Transmission; ACQUEDOTTO (general) → Distribution Networks. Pressure pipe keywords (PL/persleiding) clarified in prompt as Collection Systems.
- Technology: ACQUEDOTTO/condotta → Pipes (fallback); utility descriptions excluded via `_slim_project(exclude_utility_description=True)`
- Theme: UWWTD stripped from Stormwater/Water projects; stripped from routine WW Treatment maintenance; ACQUEDOTTO → no UWWTD. EE prompt clarifies: structural covers/roofing ≠ energy efficiency; "efficientamento" (Italian) = operational optimization ≠ EE; but process equipment replacement (centrifuges, blowers) at treatment plants = EE.
- Project Type: "Retrofit" + "New" mutually exclusive. "Decommissioning & demolition" + "New" allowed (demolish-and-rebuild).

**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Removing a rule and getting equal or better results is a great outcome. When evaluating whether to keep a change, weigh the complexity cost against the improvement magnitude.

**The first run**: Your very first run should always be to establish the baseline, so you will run the eval script as is.

## Output format

The eval script prints a machine-readable summary on stdout:

```
---
overall_accuracy:      0.913000
sector_accuracy:       0.940000
application_accuracy:  0.925000
sub_application_accuracy: 0.886000
project_type_accuracy: 0.945000
technology_accuracy:   0.853000
theme_accuracy:        0.958000
total_seconds:         49.7
per_record_seconds:    0.2
num_records:           201
```

Extract the key metric: `overall_accuracy`. Detailed error breakdowns are printed to stderr.

## Logging results

Results are **auto-appended** to `results.tsv` by `eval_accuracy.py`. The TSV has these columns:

```
commit	overall	sector	application	sub_app	technology	theme	project_type	seconds	status	description
```

- `commit`: auto-detected from git HEAD (7 chars).
- `overall` through `project_type`: accuracy per dimension.
- `seconds`: total wall-clock seconds.
- `status`: set via `--status` arg. Use `pending` during the run, then the agent decides `keep` or `discard`.
- `description`: set via `--desc` arg.

NOTE: do not commit the `results.tsv` file — leave it untracked by git.

## The experiment loop

The experiment runs on a dedicated branch (e.g. `categorizer/apr10`).

LOOP FOREVER:

1. Look at the latest eval results and top errors (from stderr / run.log). Identify the highest-leverage failure — which dimension is weakest? What error pattern repeats?
2. Propose ONE targeted change to `categorization_service.py`. Explain what you expect it to fix.
3. Implement the change.
4. `git add categorization_service.py` then `git commit -m "expN: short description"`
5. Run the experiment: `python eval_accuracy.py --iteration expN --status pending --desc "short description" 2>run.log`
6. Read out results: check stdout for `overall_accuracy` and per-dimension scores. If stdout is empty, the run crashed — read `run.log` tail and attempt a fix.
7. If `overall_accuracy` improved (higher): **KEEP**. This is the new baseline. Advance the branch.
8. If `overall_accuracy` is equal or worse: **DISCARD**. Run `git reset --hard HEAD~1` to revert.
9. Repeat from step 1.

**Diagnosis priority**: Sector > Application > Sub-application > Technology > Theme > Project Type. Upstream dimensions cascade — fixing Sector fixes downstream errors for free. Note: Sector now supports primary + secondary (e.g., Wastewater + Stormwater for combined sewer), so Application candidates come from the union of both.

**Timeout**: Each experiment should take ~50-60 seconds. If a run exceeds 3 minutes, kill it and treat it as a failure.

**Crashes**: If it's a typo or easy fix, fix and re-run. If the idea is fundamentally broken, revert and move on.

**LLM non-determinism**: temperature=0 + seed=42 still produces ±2-3% variance when cache is cleared. The disk cache (`.llm_cache/`) ensures 100% deterministic results when preserved. Only clear cache when prompts or model change. A change of <2% may be noise. Run twice if unsure before deciding to keep or discard.

**NEVER STOP**: Once the experiment loop has begun (after the initial setup), do NOT pause to ask the human if you should continue. Do NOT ask "should I keep going?" or "is this a good stopping point?". The human might be asleep, or gone from a computer and expects you to continue working *indefinitely* until you are manually stopped. You are autonomous. If you run out of ideas, think harder — re-read the error patterns, try combining previous near-misses, try more radical prompt restructuring, read the hierarchy definitions for new angles. The loop runs until the human interrupts you, period.

As an example use case, a user might leave you running while they sleep. Each experiment takes ~1 minute so you can run ~60/hour, ~500 overnight. The user wakes up to a log of experiments and a better categorizer.
