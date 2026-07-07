# iotops-workspace

A thin coordination repo for two independent sibling projects:

- [`IoTOps`](https://github.com/ridvanzengin/IoTOps) — the self-hosted IoT
  Operations Platform (FastAPI + React).
- [`custom-telegraf`](https://github.com/ridvanzengin/custom-telegraf) — a
  custom Telegraf build bundling IoTOps's Automater plugins.

This repo does **not** contain either project's code — clone them as
siblings of this directory:

```
personal/
├── iotops-workspace/   (this repo)
├── IoTOps/
└── custom-telegraf/
```

See [`CLAUDE.md`](CLAUDE.md) for the cross-repo routing guide (which repo
to work in for a given task, and the shared facts/contracts spanning
both) — read that first for any task touching more than one of these
repos. Each repo has its own `README.md`/`CLAUDE.md` for depth once you
know which one you're working in.

Modeled on the `agritwin` project's root-repo pattern, adapted for two
independent sibling repos rather than nested sub-repos, since neither
existing project needed to move to get the benefit (a single Claude Code
session with routing context for both).
