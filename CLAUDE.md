# AI Experiments DB — Claude Code Guidelines

This is a research lab notebook. Every experiment must be documented well enough that future-me (or anyone) can understand what was done, why, and what happened — months or years later.

## Core Principles

1. **Document everything, especially failures.** A failed experiment is still valuable if it's recorded. Never skip documenting something because "it didn't work."
2. **Be specific about infrastructure.** Exact GPU model, driver version, container image, provider, cost. "A cloud GPU" is never enough.
3. **Keep the repo self-describing.** If something changes how experiments are structured or what conventions are used, update README.md immediately. Don't let documentation drift.
4. **Link experiments to each other.** If experiment 005 follows up on experiment 002, say so explicitly with a relative link.

## When creating or updating experiments

- Always use the `/new-experiment` skill to scaffold new experiment folders
- Every file in an experiment folder must be listed in the "Files" table in experiment.md
- Include reproduction steps — commands, configs, environment details
- Record costs (even rough estimates) for every experiment
- If an experiment produces insights that affect future experiments, note them in the "What to Try Next" section AND consider whether README.md needs updating

## Repo maintenance rules

- **README.md** is the entry point. If a new convention, pattern, or structural change emerges from any experiment, update README.md in the same commit or PR.
- **Numbering is sequential.** Never skip or reuse experiment numbers. Use `ls -d [0-9][0-9][0-9]-*/ | sort | tail -1` to find the latest.
- **Folder slugs are permanent.** Once an experiment folder is created and committed, don't rename it. Other experiments may link to it.
- **No large binary files.** Don't commit model checkpoints, large datasets, or files over 10MB. Instead, document where they can be found or regenerated. Use links to cloud storage, HuggingFace, or include the commands to reproduce.

## When finishing a work session

Before ending any session that touched this repo:
1. Verify every modified experiment's `experiment.md` is current (especially the Files table)
2. Check if any findings should propagate to README.md
3. If a new convention was established during this session (e.g., "we now track X metric"), add it to this CLAUDE.md file
4. Commit with a descriptive message referencing the experiment number, e.g., `001: add results.tsv and agent summary`

## Tone and style

- Write experiment.md files in a direct, technical style. Not academic, not casual — like thorough engineering notes.
- Use tables for structured data (configs, results, file manifests)
- Use prose for narrative (background, insights, what went wrong)
- Avoid filler. If a section has nothing to say, omit it rather than writing "N/A"
