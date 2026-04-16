# EU Project Categoriser — Program Guide

## Overview

An AI-powered tool that classifies European water/wastewater infrastructure projects into a 6-dimensional hierarchy using LLM-based categorization. The system processes project metadata (names, descriptions, SOC fields) in multiple European languages and outputs structured classifications.

Project code location: `../eu_project_categoriser/`

## Architecture

```
Input (JSON) → Pipeline → Output (JSON)
                 │
                 ├─ Step 1: Sector (primary + secondary)
                 ├─ Step 2: Application (filtered by Sector)
                 ├─ Step 3: Sub-application (filtered by Application)
                 └─ Steps 4-6 (parallel): Technology, Theme, Project Type
```

### Core Files

| File | Role | Editable? |
|------|------|-----------|
| `categorization_service.py` | Main classification engine: prompts, rules, few-shot examples, batch processing, post-processing | **Yes — primary** |
| `prompts_config.py` | Fallback rubric descriptions, technology definitions, config loading from Excel | **Yes** |
| `042026 latest definitions/CATEGORIZATION_Hierarchy 3 (1) (2).xlsx` | Hierarchy source: rubrics, technology names/descriptions, application-to-technology mappings | **Yes — primary source** |
| `openrouter_service.py` | OpenRouter API client with caching | Rarely |
| `hierarchy_loader.py` | Loads hierarchy from Excel into runtime config | Rarely |
| `eval_accuracy.py` | Evaluation script — runs pipeline on ground truth, scores results | Fixed |

### Runner Scripts

| Script | Purpose |
|--------|---------|
| `_run_sample20_desc.py` | Quick iteration: 20 records with descriptions, configurable seed |
| `_run_sample100.py` | Larger sample: 100 records for broader coverage testing |
| `_run_baz_ww.py` / `_run_baz_s2.py` | Run on human-reviewed ground truth sets |

## Classification Pipeline

### Step 1: Sector Classification
- **Categories:** Water, Wastewater, Stormwater, Other
- **Output:** Primary + optional Secondary
- **Key rules:**
  - Combined sewer suffixes (MW, MWK, SW/RW) → Wastewater primary + Stormwater secondary
  - Utility name does NOT determine secondary — only the project's own content does
  - Multi-domain project descriptions (e.g., "idrici e fognari") → add secondary

### Step 2: Application Classification
- **Filtered by:** Sector (primary + secondary)
- **Categories:** Water Networks, Water Treatment, Water Resources, Desalination, Wastewater Treatment, Wastewater Networks, Sludge Management, Stormwater, Other
- **Key rules:**
  - Application rubric includes options from both primary and secondary Sectors
  - BHKW/CHP/biogas projects → Sludge Management (not Wastewater Treatment)
  - Network/pipe/connection projects → never get "Wastewater Treatment" as secondary
  - Secondary Application must represent genuinely different scope in the project

### Step 3: Sub-application Classification
- **Filtered by:** Application
- **Key rules:**
  - SOC field treatment stage keywords override vague project names: "terziario" → Tertiary Treatment, "preliminare" → Primary Treatment
  - Multi-language detection: Italian (terziario, secondario), French (tertiaire), German (tertiär)
  - Lift Stations vs Collection Systems: pump keywords (Pumpwerk, EBAR, gemaal) → Lift Stations; pipe keywords (Kanal, condotte) → Collection Systems
  - "General treatment" only when no specific treatment stage can be determined

### Steps 4-6: Technology, Theme, Project Type (Parallel)
- **Technology:** Multi-label tags from a defined list (loaded from Excel hierarchy). Includes "Biogas processing" for BHKW/CHP/Klärgas. Keyword hints in prompt drive assignment.
- **Theme:** Multi-label (e.g., Energy Efficiency, UWWTD compliance)
- **Project Type:** New, Retrofit, Planning & Engineering, Decommissioning & demolition

## Experiment Loop

### Quick Iteration Cycle

```
1. Identify error → review misclassified projects
2. Hypothesize fix → modify prompts/rules/examples
3. Test with sample20 → python _run_sample20_desc.py (change SEED for new sample)
4. Review results → manually inspect all 20 classifications
5. If good → run sample100 or eval_accuracy.py for formal scoring
6. Commit & push if improvement confirmed
```

### Formal Evaluation

```bash
python eval_accuracy.py --iteration expN --status keep --desc "description"
```

Runs full pipeline on 201 Baz-reviewed ground truth records. Auto-appends results to `results.tsv` and updates `eval.md`.

### Key Principles

- **One change at a time:** Isolate variables to understand what helps
- **Cache awareness:** LLM cache (`.llm_cache/`) makes re-runs deterministic. Clear cache when prompts change.
- **False positive penalty:** Technology scoring penalizes extra tags (FP) at -0.5 each. Prefer fewer, more precise tags.
- **Cascading impact:** Sector errors cascade to Application and Sub-application. Fix upstream first.

## Configuration

### Environment Variables (`.env`)
- `OPENROUTER_API_KEY` — API key for OpenRouter
- `BATCH_SIZE` — records per API call (default: 20)
- `MAX_WORKERS` — concurrent API threads (default: 12)

### Model
Default: `openai/gpt-4.1-mini` via OpenRouter. Configurable per-step.

## Recent Improvements (April 2026)

1. **Tertiary Treatment multi-language detection:** SOC fields with "terziario", "tertiaire", "tertiary" now correctly map to Tertiary Treatment instead of General treatment
2. **Biogas Processing technology:** New tag covering BHKW/CHP/Gasdruckerhöhung/Klärgas projects. Sludge Management application for biogas energy projects.
3. **Cross-sector secondary Applications:** Application rubric merges primary + secondary Sector options. Validation permits secondary Applications from secondary Sector.
4. **Network ≠ Treatment secondary:** Pure pipe/connection projects no longer get false "Wastewater Treatment" secondary
5. **Utility name guard:** Multi-domain utility names (e.g., "Water and Sewerage Co") no longer trigger false secondary Sector/Application tags
6. **Lift Stations vs Collection Systems:** Expanded keyword lists and explicit distinction rules

## Evaluation Criteria

See `eval.md` for full scoring methodology, ground truth details, accuracy targets, and historical results.
