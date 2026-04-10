# Eval Criteria

## What is being evaluated

A categorizer tool that classifies EU TED infrastructure projects into 6 dimensions based on project metadata (project_name, project_name_en, description_english, description_eu_win_en) and hierarchy definitions.

The tool feeds project data together with hierarchy definitions into an LLM, which decides the appropriate category for each dimension.

## Categorization dimensions

There are 6 categorizations, executed in a specific pipeline order:

| Step | Dimension | Cardinality | Depends on |
|------|-----------|-------------|------------|
| 1 | Sector (Service) | one-to-one | — |
| 2 | Application | many-to-many | Sector determines candidate set |
| 3 | Sub-application | one-to-one | Application determines candidate set |
| 4 | Technology | many-to-many | Independent (runs in parallel with 5 & 6) |
| 5 | Theme | many-to-many | Independent (runs in parallel with 4 & 6) |
| 6 | Project Type | many-to-many | Independent (runs in parallel with 4 & 5) | new and retrofit shouldn't show up together

**Pipeline order:** Steps 1→2→3 run sequentially (each step's output filters the next step's candidates). Steps 4, 5, 6 run in parallel after step 3 completes.

**Cascading dependency:** If Sector is wrong, Application candidates are wrong, which makes Sub-application wrong. This cascading effect is the primary cause of Sub-application errors.

For dimensions 4 and 5 (Technology, Theme), there is no hierarchy — they are flat multi-label classifications.

## Definitions source

Categorization hierarchy and definitions are recorded in:
`042026 latest definitions/CATEGORIZATION_Hierarchy 3 (1) (2).xlsx`

## Ground truth

Human-reviewed classifications in:
- `042026 latest human review/projects_export_sample 2.xlsx`
- `042026 latest human review/projects_export_sample_ww_sludge.xlsx`

## Scoring logic per dimension

### One-to-one dimensions: Sector, Sub-application

Scoring: **exact match** after normalization (lowercase, strip whitespace).

```
norm_pred = normalize(predicted)
norm_gt = normalize(ground_truth)
correct = (norm_pred == norm_gt)
```

If the ground truth record has "Additional tags" columns (alternative acceptable answers from human reviewers), those are also accepted:

```
acceptable = {norm_gt} ∪ {normalize(tag) for tag in additional_tags}
correct = (norm_pred in acceptable)
```

Each record scores 1 (correct) or 0 (incorrect). Category accuracy = correct / total.

### Many-to-many dimensions: Application, Technology, Theme, Project Type

These dimensions can have multiple tags per record (e.g., "Pipes, Pumps"). Scoring uses an **additive/deductive** system that rewards correct predictions and penalizes false positives and false negatives.

The ground truth set is built from the primary column AND the "Additional tag" columns. The additional tags are NOT alternatives — they are part of the complete correct answer.

```
# Build full ground truth set
gt_set  = {normalize(x) for x in primary_column.split(',')}
full_gt = gt_set ∪ {normalize(x) for x in additional_tag_columns}

pred_set = {normalize(x) for x in predicted.split(',')}

TP = |pred_set ∩ full_gt|       # correct tags:      +1 each
FP = |pred_set - full_gt|       # extra wrong tags:   -0.8 each
FN = |full_gt - pred_set|       # missed tags:        -0.8 each

record_score = (TP - 0.8 * FP - 0.8 * FN) / |full_gt|
```

- Maximum score per record: **1.0** (perfect match, no extras, no misses)
- Score **can be negative** (heavily penalizes over-prediction and wrong predictions)
- Category score = mean of all record scores

| Scenario | GT | Predicted | TP | FP | FN | Score |
|----------|-----|-----------|----|----|-----|-------|
| Exact match | Pipes, Pumps | Pipes, Pumps | 2 | 0 | 0 | +1.0 |
| Missing tag | Pipes, Pumps | Pipes | 1 | 0 | 1 | +0.1 |
| Extra tag | Pipes | Pipes, Pumps | 1 | 1 | 0 | +0.2 |
| Partial overlap | Pipes, Pumps | Pipes, Screens | 1 | 1 | 1 | -0.3 |
| No overlap | Pipes | Screens | 0 | 1 | 1 | -1.6 |

**Rationale:** This scoring incentivizes the model to be precise — only predict tags it is confident about. A false positive is equally as bad as a missed tag, so the model cannot game the score by over-predicting.

**Project Type constraint:** "New" and "Retrofit" are mutually exclusive — they should never appear together in predictions.

### Overall accuracy

Mean of all per-record scores across all dimensions:

```
overall = sum(all record-dimension scores) / count(all record-dimension pairs)
```

One-to-one records contribute 0 or 1. Many-to-many records contribute the normalized score (can be negative). This single number reflects both precision and recall across the whole system.

## Evaluation script

`eval_accuracy.py` — runs the full pipeline on ground-truth records and compares predictions.

Usage:
```
python eval_accuracy.py [--limit N] [--model MODEL]
```

Default model: `openai/gpt-4.1-mini` (via OpenRouter API).

## Accuracy target

Overall accuracy: **90%+**

## Current best result (exp2, commit e3fb5e1)

Note: These numbers were measured with the old recall-only scoring for Technology/Theme. After switching to set-based F1 scoring, Technology and Theme accuracy will likely decrease.

| Dimension | Accuracy (old scoring) | Status |
|-----------|----------------------|--------|
| Sector | 98.5% | ✓ above 90% |
| Application | 90.0% | ✓ at 90% |
| Sub-application | 74.5% | ✗ largest gap |
| Technology | 84.4% (recall-only) | ✗ needs re-baseline with F1 |
| Theme | 92.6% (recall-only) | ✗ needs re-baseline with F1 |
| Project Type | 88.0% | ✗ needs +2.0% |
| **Overall** | **88.3%** | needs re-baseline |

## Known issues

1. **LLM non-determinism:** Same code, same data, temperature=0 produces ±3-4% variance across runs. Small improvements (<3%) may be noise.
2. **Cascading errors:** Application misclassification directly causes Sub-application failure because sub-app candidates are filtered by detected Application.
3. **Sub-application bottleneck:** "General treatment" is over-used as a catch-all. Model confuses project phase (design/planning) with facility type (pump station/treatment plant).
4. **Technology false positives:** Vague project descriptions get assigned specific technologies when "No Specific Technology" is correct.
5. **Project Type confusion:** "Retrofit" vs "New" — model misclassifies "installation of X at existing plant" as "New" instead of "Retrofit".

## Levers (what can be changed)

- Prompts and classification rules in `categorization_service.py`
- Chain-of-Thought instructions
- Few-shot examples
- Model selection (per step or globally)
- Pipeline structure (e.g., adding verification steps)
- Batch size and concurrency settings
