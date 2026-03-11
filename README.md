# ai-experiments-db

A personal lab notebook for AI experiments. Each folder is a self-contained experiment with an `experiment.md` that describes what was done, why, how, and what happened.

## Structure

```
ai-experiments-db/
├── README.md
├── 001-autoresearch-5090-overnight/
│   ├── experiment.md        ← start here
│   ├── results.tsv
│   ├── summary.md
│   └── best_train.py
├── 002-some-future-experiment/
│   ├── experiment.md
│   └── ...
└── ...
```

## Conventions

Each experiment folder contains:

- **experiment.md** — The meta document. Background, motivation, infrastructure details, file manifest, key findings, and what to try next. Read this first.
- **Supporting files** — Data, code, logs, configs, outputs. Varies per experiment. Everything is described in experiment.md.

Experiments are numbered sequentially (`001-`, `002-`, etc.) with a short slug describing the work. No strict schema beyond that — some experiments are overnight training runs, others might be prompt engineering tests, agent conversations, or benchmark comparisons.

## Why public?

Sharing the raw process of learning, not just polished results. If someone stumbles on this and finds it useful, great. If not, it's still a useful personal reference and backup.

## Context

These experiments are run by a software engineer exploring LLM training, agent orchestration, and AI tooling. Infrastructure varies per experiment — cloud GPUs, local hardware, API calls — and is documented in each experiment.md.
