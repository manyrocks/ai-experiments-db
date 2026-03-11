---
name: new-experiment
description: Scaffold a new experiment in the ai-experiments-db repo. Use when the user wants to create, log, document, or start a new experiment. Also triggers on phrases like "new experiment", "log this experiment", "scaffold experiment", "document this run", or "add experiment". Creates a numbered folder with an experiment.md template and organizes supporting files.
---

# New Experiment Scaffolding

This skill creates consistently structured experiment folders in the ai-experiments-db repository.

## Folder Naming Convention

Experiments use zero-padded three-digit sequential numbering with a kebab-case slug:

```
001-autoresearch-5090-overnight/
002-nanochat-gpt2-speedrun/
003-llm-debate-claude-vs-deepseek/
```

## How to Determine the Next Number

1. List existing experiment folders:
   ```bash
   ls -d [0-9][0-9][0-9]-*/ 2>/dev/null | sort | tail -1
   ```
2. Extract the number from the most recent folder and increment by 1
3. If no folders exist, start at `001`
4. Zero-pad to 3 digits (e.g., `001`, `012`, `123`)

## Creating a New Experiment

### Step 1: Determine the next number

```bash
# Get the latest experiment number
LAST=$(ls -d [0-9][0-9][0-9]-*/ 2>/dev/null | sort | tail -1 | grep -oP '^\d{3}')
NEXT=$(printf "%03d" $((10#${LAST:-0} + 1)))
```

### Step 2: Ask the user for a short slug

The slug should be 2-5 words in kebab-case describing the experiment. Examples:
- `autoresearch-5090-overnight`
- `nanochat-gpt2-speedrun`  
- `deepseek-vs-claude-coding`
- `sft-gemology-dataset`
- `qlora-llama4-finetuning`

### Step 3: Create the folder and experiment.md

```bash
FOLDER="${NEXT}-${SLUG}"
mkdir -p "$FOLDER"
```

### Step 4: Generate experiment.md

Create `experiment.md` inside the folder using the template below. Interview the user to fill in the details — don't leave placeholders if you can infer or ask for the information.

## experiment.md Template

```markdown
# {NUMBER} — {Title}

**Date:** {date or date range}
**Type:** {category — e.g., "Autonomous LLM training research", "Model fine-tuning", "Agent conversation experiment", "Benchmark comparison", "Prompt engineering"}
**Duration:** {how long it ran}
**Cost:** {estimated cost breakdown — compute, API calls, etc.}

## Background

{Why this experiment exists. What question are we trying to answer? What inspired it? Link to relevant repos, papers, or prior experiments.}

## Infrastructure

| Detail | Value |
|--------|-------|
| **Provider** | {cloud provider, local hardware, or API} |
| **GPU/Hardware** | {specific GPU model, memory, or "API-only"} |
| **Software** | {key frameworks, versions, models used} |
| **Agent** | {if an AI agent was involved — which one, what mode} |

## Method

{What was actually done. Step by step if relevant. Include commands, configurations, prompts used, datasets, or any details needed to reproduce.}

## Results

{What happened. Metrics, outputs, observations. Tables and numbers where applicable.}

## Key Findings

{The insights. What was learned? What was surprising? What confirmed expectations?}

## What I Learned

{Personal takeaways in plain language. This section is for future-you, not for a journal. Explain what clicked, what was confusing, what analogy helped it make sense. If this experiment changed how you think about something, capture that here. Don't just restate the metrics — explain what they mean and why they matter.}

## Files in This Folder

| File | Description |
|------|-------------|
| `experiment.md` | This file |
| {list each file} | {describe it} |

## What to Try Next

{Ideas for follow-up experiments. What would you change? What remains unanswered?}
```

## Guidelines for Writing experiment.md

- **Be specific about infrastructure.** Future-you wants to know exactly what GPU, what driver version, what container image. "A cloud GPU" is not enough.
- **Record costs.** Even rough estimates help with budgeting future experiments.
- **Link to prior experiments.** If this is a continuation, reference the folder number: "Follows up on [001](../001-autoresearch-5090-overnight/)".
- **Include reproduction steps.** Someone (probably future-you) should be able to rerun this.
- **Document failures and dead ends.** An experiment that shows something doesn't work is still valuable.
- **Write the "What I Learned" section in plain language.** Use analogies, explain what clicked, capture the aha moment. This section is the most valuable part for future reference — the metrics will mean nothing in 6 months without context for why they mattered.
- **Keep the file manifest current.** Every file in the folder should be listed and described.

## Handling Supporting Files

When the user has files to add (logs, code, configs, results):

1. Ask what files they have and what each one is
2. Copy or create them in the experiment folder
3. Add each one to the "Files in This Folder" table in experiment.md

Common file types per experiment category:

**Training runs:** results.tsv, best_train.py, training configs, loss curves, model checkpoints (or links to them)
**Agent experiments:** conversation logs, prompt templates, output samples
**Benchmarks:** raw data CSVs, comparison tables, analysis scripts
**Fine-tuning:** dataset samples, training configs, eval results, adapter weights (or links)

## Updating an Existing Experiment

If the user wants to add results or notes to an existing experiment:

1. Ask which experiment number or search by slug
2. Read the existing experiment.md
3. Update the relevant sections
4. Add new files to the manifest table