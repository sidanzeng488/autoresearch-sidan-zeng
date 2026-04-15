# Eval Criteria

## What is being evaluated

A categorizer tool that classifies infrastructure projects into 6 dimensions based on project metadata and hierarchy definitions.

**Input fields used by the LLM** (per record):
- `Project Name (PN)` / `Project Name (PN)_EN` — project name (original + English)
- `Sector or Category (SOC)` — utility-provided sector/category
- `Additional Fields (AF)2` — supplementary metadata
- `Description for EU WIN.1` / `Description for EU WIN.1_EN` — project-specific description (primary source)
- `Project Description (PD)` — utility-level programme description (used for Sector/Application/Sub-application only; **excluded** from Technology extraction to prevent hallucination)
- `utilityName` — utility company name (included for Sector/Application context)

**Excluded fields:** `AI generated description` (value is just "Yes"/"No", wastes tokens without adding classification signal).

The tool feeds project data together with hierarchy definitions into an LLM, which decides the appropriate category for each dimension.

## Categorization dimensions

There are 6 categorizations, executed in a specific pipeline order:

| Step | Dimension | Cardinality | Output Fields | Depends on |
|------|-----------|-------------|---------------|------------|
| 1 | Sector | primary + secondary | `Sector`, `Sector Confidence`, `Sector (Secondary)` | — |
| 2 | Application | primary + secondary | `Application (Primary)`, `Application (Primary) Confidence`, `Application (Secondary)` | Sector (primary + secondary) determines candidate set |
| 3 | Sub-application | primary + secondary | `Sub-application (Primary)`, `Sub-application (Primary) Confidence`, `Sub-application (Secondary)` | Application determines candidate set |
| 4 | Technology | multi-label (additive set) | `Technology Tags`, `Technology Confidence` | Independent (parallel with 5 & 6) |
| 5 | Theme | multi-label (additive set) | `Theme Tags`, `Theme Confidence` | Independent (parallel with 4 & 6) |
| 6 | Project Type | multi-label (additive set) | `Project Type Tags`, `Project Type Confidence` | Independent (parallel with 4 & 5) |

**Pipeline order:** Steps 1→2→3 run sequentially (each step's output filters the next step's candidates). Steps 4, 5, 6 run in parallel after step 3 completes.

**Sector multi-select:** Sector supports primary + secondary values. For combined sewer projects, primary/secondary assignment depends on keyword type (e.g., MW/Mischwasser → Stormwater primary; Reinwasserableitung → Wastewater primary). The Application candidate set is the union of allowed applications from both primary and secondary Sector values.

**Cascading dependency:** If Sector is wrong, Application candidates are wrong, which makes Sub-application wrong. This cascading effect is the primary cause of Sub-application errors.

**Project Type constraint:** "New" and "Retrofit" are mutually exclusive — they should never appear together. However, "Decommissioning & demolition" + "New" is valid for demolish-and-rebuild projects.

## Definitions source

Categorization hierarchy and definitions are recorded in:
`042026 latest definitions/CATEGORIZATION_Hierarchy 3 (1) (2).xlsx`

**CRITICAL — Strict hierarchy compliance:** All categorizations MUST strictly follow this hierarchy. The hierarchy defines exactly which Sub-applications belong to which parent Applications:

| Application | Valid Sub-applications |
|---|---|
| Water Networks | Water Transmission, Distribution Networks, Pump Stations |
| Wastewater Treatment | Primary Treatment, Secondary Treatment, Tertiary Treatment, Quaternary Treatment, Disinfection, General treatment, Reuse |
| Wastewater Networks | Collection Systems, Lift Stations |
| Stormwater | Green Infrastructure, Gray Infrastructure, General Stormwater, Combined Sewer |
| Water Resources | Groundwater Extraction, Dams and Reservoirs, Surface Water Intakes, Other |
| Other | Administrative / Support, Construction Budget / Unspecified Construction |
| Sludge Management | *(none — no sub-applications in hierarchy)* |
| Water Treatment | *(none)* |
| Desalination | *(none)* |

**Universal Sub-applications** (available under ALL Applications, no parent restriction): `Planning & Engineering`, `Construction Budget / Unspecified Construction`, `Administrative / Support`.

**Key constraint:** Technologies (Level 4) are NOT Sub-applications (Level 3). Sludge process types like Anaerobic Digestion, Sludge Thickening, Sludge Dewatering are Technologies, not Sub-applications.

## Ground truth

Baz-only human-reviewed classifications (201 records total):
- `042026 latest human review/projects_export_sample_ww_sludge - Baz.xlsx` (100 records)
- `042026 latest human review/projects_export_sample 2 - Baz.xlsx` (101 records)

## Scoring logic per dimension

### Primary-only dimensions: Sector, Application, Sub-application

Scoring: **exact match** after normalization (lowercase, strip whitespace, synonym mapping).

```
norm_pred = normalize(predicted_primary)
norm_gt   = normalize(ground_truth)
acceptable = {norm_gt} ∪ {normalize(tag) for tag in additional_tags}
correct = (norm_pred in acceptable)
```

Each record scores 1 (correct) or 0 (incorrect). Category accuracy = correct / total.

### Multi-label dimensions: Technology, Theme, Project Type

These dimensions can have multiple tags per record (e.g., "Pipes, Pumps"). Scoring uses an **additive/deductive** system.

Additional tags from human reviewers expand the **acceptable set** (predictions matching them are not penalized as FP) but are NOT required (missing them does not count as FN).

```
gt_set   = {normalize(x) for x in primary_column.split(',')}
pred_set = {normalize(x) for x in predicted.split(',')}

# Additional tags expand what's acceptable, but are not required
acceptable_set = gt_set ∪ {normalize(x) for x in additional_tag_columns}

TP = |pred_set ∩ acceptable_set|    # correct/acceptable tags:  +1 each
FP = |pred_set - acceptable_set|    # wrong extra tags:         −0.5 each
FN = |gt_set   - pred_set|          # missed primary tags:      −0.5 each

record_score = max((TP − 0.5 × FP − 0.5 × FN) / |gt_set|, 0.0)
```

- Maximum score per record: **1.0** (perfect match)
- Minimum score per record: **0.0** (floor, no negative scores)
- Category score = mean of all record scores

| Scenario | GT | Predicted | Acceptable | TP | FP | FN | Score |
|----------|-----|-----------|------------|----|----|-----|-------|
| Exact match | Pipes, Pumps | Pipes, Pumps | {Pipes, Pumps} | 2 | 0 | 0 | 1.0 |
| Missing tag | Pipes, Pumps | Pipes | {Pipes, Pumps} | 1 | 0 | 1 | 0.25 |
| Extra tag | Pipes | Pipes, Pumps | {Pipes} | 1 | 1 | 0 | 0.5 |
| Partial overlap | Pipes, Pumps | Pipes, Screens | {Pipes, Pumps} | 1 | 1 | 1 | 0.0 |
| No overlap | Pipes | Screens | {Pipes} | 0 | 1 | 1 | 0.0 |

**Rationale:** FP_PENALTY = FN_PENALTY = 0.5 balances precision and recall without being overly harsh. The floor of 0.0 prevents a single bad record from disproportionately dragging down category scores. Additional tags as acceptable (not required) reflects that human reviewers may add borderline tags that are correct but not mandatory.

### Overall accuracy

Mean of all per-record scores across all dimensions:

```
overall = sum(all record-dimension scores) / count(all record-dimension pairs)
```

Primary-only records contribute 0 or 1. Multi-label records contribute the normalized score (0.0 to 1.0). This single number reflects both precision and recall across the whole system.

## Experiment loop

See `../eu_project_categoriser/program.md` for the full autonomous experiment loop instructions (autoresearch-style).

Key points:
- **Only file to modify:** `categorization_service.py` (prompts, rules, few-shot examples, post-processing)
- **Evaluation script (fixed):** `eval_accuracy.py` — runs the full pipeline and auto-logs results
- **Results log:** `results.tsv` — auto-appended after each run
- **Loop:** analyze errors → hypothesize → edit → commit → eval → keep/revert → repeat

## Evaluation script

`eval_accuracy.py` — runs the full pipeline on ground-truth records, compares predictions, auto-appends to `results.tsv`, and auto-updates this file.

Usage:
```
python eval_accuracy.py --iteration expN --status keep --desc "short description"
```

All arguments:
- `--iteration LABEL` — experiment label (e.g. `exp31`)
- `--status {keep,discard,crash,pending}` — experiment status for results.tsv
- `--desc TEXT` — short description for results.tsv
- `--limit N` — limit records processed
- `--sample N` — random sample N records
- `--model MODEL` — model to use (default: `openai/gpt-4.1-mini`)
- `--ensemble N` — number of ensemble runs (majority vote)

Default model: `openai/gpt-4.1-mini` (via OpenRouter API).

## Accuracy target

Overall accuracy: **90%+**

<!-- AUTO-UPDATED SECTION — do not edit manually below this line -->
## Latest result (baz-ww-fixes)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 82.6% | below 90% |
| Application | 81.9% | below 90% |
| Sub-application | 69.8% | below 90% |
| Technology | 81.5% | below 90% |
| Theme | 83.3% | below 90% |
| Project Type | 94.6% | ✓ above 90% |
| **Overall** | **82.2%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (hierarchy-fix)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 83.2% | below 90% |
| Application | 81.9% | below 90% |
| Sub-application | 69.1% | below 90% |
| Technology | 81.8% | below 90% |
| Theme | 83.3% | below 90% |
| Project Type | 95.3% | ✓ above 90% |
| **Overall** | **82.4%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (manual-fixes)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 83.5% | below 90% |
| Application | 81.9% | below 90% |
| Sub-application | 68.3% | below 90% |
| Technology | 82.8% | below 90% |
| Theme | 79.2% | below 90% |
| Project Type | 95.3% | ✓ above 90% |
| **Overall** | **82.5%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp42-speed)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 92.9% | ✓ above 90% |
| Application | 90.4% | ✓ above 90% |
| Sub-application | 82.4% | below 90% |
| Technology | 87.6% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 93.8% | ✓ above 90% |
| **Overall** | **89.6%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp41-speed)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 91.7% | ✓ above 90% |
| Application | 89.2% | approaching 90% |
| Sub-application | 79.4% | below 90% |
| Technology | 85.0% | approaching 90% |
| Theme | 83.3% | below 90% |
| Project Type | 94.9% | ✓ above 90% |
| **Overall** | **88.2%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp40-speed)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.5% | ✓ above 90% |
| Application | 90.1% | ✓ above 90% |
| Sub-application | 80.9% | below 90% |
| Technology | 84.8% | below 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 94.9% | ✓ above 90% |
| **Overall** | **89.1%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (baseline)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.1% | ✓ above 90% |
| Application | 92.4% | ✓ above 90% |
| Sub-application | 80.2% | below 90% |
| Technology | 87.5% | approaching 90% |
| Theme | 87.5% | approaching 90% |
| Project Type | 94.9% | ✓ above 90% |
| **Overall** | **90.1%** | **✓ above 90% target** |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (baseline)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.8% | ✓ above 90% |
| Application | 92.4% | ✓ above 90% |
| Sub-application | 80.2% | below 90% |
| Technology | 87.7% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 94.9% | ✓ above 90% |
| **Overall** | **90.1%** | **✓ above 90% target** |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (baseline)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.8% | ✓ above 90% |
| Application | 92.1% | ✓ above 90% |
| Sub-application | 80.2% | below 90% |
| Technology | 86.4% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 93.5% | ✓ above 90% |
| **Overall** | **89.5%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (baseline)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.8% | ✓ above 90% |
| Application | 91.8% | ✓ above 90% |
| Sub-application | 80.5% | below 90% |
| Technology | 86.4% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 94.9% | ✓ above 90% |
| **Overall** | **89.8%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (baseline)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.5% | ✓ above 90% |
| Application | 91.8% | ✓ above 90% |
| Sub-application | 79.0% | below 90% |
| Technology | 86.6% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 93.8% | ✓ above 90% |
| **Overall** | **89.3%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp34-confirm)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 92.3% | ✓ above 90% |
| Application | 91.2% | ✓ above 90% |
| Sub-application | 79.0% | below 90% |
| Technology | 84.5% | below 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 91.3% | ✓ above 90% |
| **Overall** | **88.1%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp33-combined-opt2)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 92.3% | ✓ above 90% |
| Application | 90.4% | ✓ above 90% |
| Sub-application | 77.5% | below 90% |
| Technology | 85.4% | approaching 90% |
| Theme | 87.5% | approaching 90% |
| Project Type | 92.0% | ✓ above 90% |
| **Overall** | **87.9%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp32-combined-opt)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 92.3% | ✓ above 90% |
| Application | 91.2% | ✓ above 90% |
| Sub-application | 78.2% | below 90% |
| Technology | 86.2% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 92.8% | ✓ above 90% |
| **Overall** | **88.5%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp31-combined-baseline)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 89.7% | approaching 90% |
| Application | 87.7% | approaching 90% |
| Sub-application | 74.0% | below 90% |
| Technology | 86.6% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 92.0% | ✓ above 90% |
| **Overall** | **86.5%** | below 90% target |

*GT: Baz-only (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (baseline-baz-isa)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 90.3% | ✓ above 90% |
| Application | 88.3% | approaching 90% |
| Sub-application | 76.7% | below 90% |
| Technology | 84.8% | below 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 91.7% | ✓ above 90% |
| **Overall** | **86.8%** | below 90% target |

*GT: Baz+Isabelle (343 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp39)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.5% | ✓ above 90% |
| Application | 91.5% | ✓ above 90% |
| Sub-application | 88.5% | approaching 90% |
| Technology | 86.4% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 95.2% | ✓ above 90% |
| **Overall** | **91.1%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (stability-check)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.5% | ✓ above 90% |
| Application | 91.5% | ✓ above 90% |
| Sub-application | 88.5% | approaching 90% |
| Technology | 88.0% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 93.9% | ✓ above 90% |
| **Overall** | **91.4%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp38)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.0% | ✓ above 90% |
| Application | 91.0% | ✓ above 90% |
| Sub-application | 89.3% | approaching 90% |
| Technology | 86.4% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 95.2% | ✓ above 90% |
| **Overall** | **91.1%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp37)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.5% | ✓ above 90% |
| Application | 93.0% | ✓ above 90% |
| Sub-application | 88.5% | approaching 90% |
| Technology | 87.0% | approaching 90% |
| Theme | 91.7% | ✓ above 90% |
| Project Type | 93.9% | ✓ above 90% |
| **Overall** | **91.6%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp36)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.0% | ✓ above 90% |
| Application | 92.5% | ✓ above 90% |
| Sub-application | 89.3% | approaching 90% |
| Technology | 87.0% | approaching 90% |
| Theme | 87.5% | approaching 90% |
| Project Type | 95.2% | ✓ above 90% |
| **Overall** | **91.6%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (check)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.5% | ✓ above 90% |
| Application | 93.0% | ✓ above 90% |
| Sub-application | 87.0% | approaching 90% |
| Technology | 85.9% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 94.5% | ✓ above 90% |
| **Overall** | **91.4%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp35)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.5% | ✓ above 90% |
| Application | 93.0% | ✓ above 90% |
| Sub-application | 89.3% | approaching 90% |
| Technology | 85.9% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 93.9% | ✓ above 90% |
| **Overall** | **91.6%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp34)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.0% | ✓ above 90% |
| Application | 92.5% | ✓ above 90% |
| Sub-application | 86.3% | approaching 90% |
| Technology | 88.0% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 94.5% | ✓ above 90% |
| **Overall** | **91.5%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp33)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.0% | ✓ above 90% |
| Application | 92.5% | ✓ above 90% |
| Sub-application | 87.8% | approaching 90% |
| Technology | 87.5% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 94.5% | ✓ above 90% |
| **Overall** | **91.6%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp32)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.5% | ✓ above 90% |
| Application | 92.0% | ✓ above 90% |
| Sub-application | 89.3% | approaching 90% |
| Technology | 87.5% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 93.9% | ✓ above 90% |
| **Overall** | **91.5%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp31)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 93.5% | ✓ above 90% |
| Application | 92.0% | ✓ above 90% |
| Sub-application | 87.8% | approaching 90% |
| Technology | 84.7% | below 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 93.9% | ✓ above 90% |
| **Overall** | **90.7%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (baseline)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.5% | ✓ above 90% |
| Application | 93.0% | ✓ above 90% |
| Sub-application | 89.3% | approaching 90% |
| Technology | 85.3% | approaching 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 94.5% | ✓ above 90% |
| **Overall** | **91.6%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Latest result (exp30-baz-only)

| Dimension | Accuracy | Status |
|-----------|----------|--------|
| Sector | 94.0% | ✓ above 90% |
| Application | 92.5% | ✓ above 90% |
| Sub-application | 88.6% | approaching 90% |
| Technology | 85.3% | below 90% |
| Theme | 95.8% | ✓ above 90% |
| Project Type | 94.5% | ✓ above 90% |
| **Overall** | **91.3%** | **✓ above 90% target** |

*GT: Baz-only (201 records) · Model: openai/gpt-4.1-mini · Scoring: additive (TP − 0.5×FP − 0.5×FN)/|GT| floor 0*

## Known issues

1. **LLM non-determinism:** Same code, same data, temperature=0 + seed=42 produces ±2-3% variance across runs. **Mitigation:** disk-based LLM cache (`.llm_cache/`) guarantees 100% deterministic results when cache is preserved. Only clear cache when prompts/model change. Small improvements (<3%) may be noise when cache is cleared.
2. **Cascading errors:** Application misclassification directly causes Sub-application failure because sub-app candidates are filtered by detected Application.
3. **Sub-application remains the weakest link:** "General treatment" is over-used as a catch-all. Context-dependent rules help but are fragile.
4. **Technology accuracy:** "Pipes" assignment is the main challenge — balancing liberal assignment for sewer/network projects vs conservative for treatment plant contexts. Technology extraction now excludes utility-level descriptions to prevent hallucination.
5. **Multi-lingual input:** Descriptions come in German, Italian, French, Dutch, etc. The model handles this well but occasional mistranslation-driven errors occur.
6. **Field name inconsistency:** Raw data uses `Project Name (PN)` while output uses `project_name`. The `_get_project_name()` helper handles both conventions in post-processing.
7. **UWWTD theme:** Previously over-applied to ~55% of records. Now tightened to only tag when there is specific UWWTD compliance evidence (new sewer connections, capacity upgrades). Routine maintenance/renovation does not qualify.
8. **Sector primary/secondary for combined sewer:** Determined by keyword type. MW-Kanal, Mischwasser, MWK, sfioratore, RÜB, Regenbauwerke → `Sector=Stormwater` + `Sector (Secondary)=Wastewater`. Reinwasserableitung, Fremdwassersanierung → `Sector=Wastewater` + `Sector (Secondary)=Stormwater`. Application and Sub-application follow the Sector hierarchy.
9. **Strict hierarchy compliance (enforced):** Sub-applications must match their parent Application per the hierarchy spreadsheet. Applications without hierarchy-defined sub-apps (Sludge Management, Water Treatment, Desalination) only get universal subs (Planning & Engineering, Construction Budget / Unspecified Construction, Administrative / Support). Technologies (Anaerobic Digestion, Sludge Dewatering, etc.) must NOT appear as Sub-applications.

## Levers (what can be changed)

- Prompts and classification rules in `categorization_service.py`
- Chain-of-Thought instructions
- Few-shot examples
- Post-processing rules (hardcoded regex-based overrides per dimension)
- Model selection (per step or globally)
- Pipeline structure (e.g., adding verification steps)
- Batch size and concurrency settings (default batch size: 20 records per API call)
- Input field selection via `_slim_project()` (which fields are sent to LLM, with `exclude_utility_description` for Technology; `AI generated description` excluded globally)
- `_get_project_name()` helper for field name compatibility
