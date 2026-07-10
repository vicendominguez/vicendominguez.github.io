---
title: "Octos: a minimalist pipeline orchestrator for CLI LLM agents"
date: 2026-02-06T10:45:00+01:00
draft: false
toc: false
images:
tags:
  - llm
  - ai
  - agents
  - cli
  - golang
  - automation
  - pipelines
---

I've been using CLI agents (kiro, claude, goose...) daily and I got tired of doing the same thing over and over: run a prompt, look at the output, copy what matters, feed it to the next prompt. Repeat. When something breaks at step 5, start over.

So I wrote [Octos](https://github.com/vicendominguez/octos). It's a pipeline runner for CLI agents. You describe steps in YAML and it chains them together. That's basically it.

```yaml
agent:
  cmd: "kiro-cli"
  args: ["chat", "--no-interactive", "--trust-all-tools"]

steps:
  - name: analyze
    prompt: "Analyze this codebase and list the main issues"
    save_to: analysis.txt

  - name: fix
    load_from: analysis.txt
    prompt: |
      Fix the issues found:
      {{artifact.analysis}}
```

Any binary that takes a prompt and writes to stdout works. No SDK, no adapters. You can even swap agents between steps — use a cheap model for the boring analysis and an expensive one only for the step that needs it.

The key thing about context: each step only gets what you explicitly give it. There's a global `context` block in the YAML (role, rules, whatever you want) that every step receives, but the output of previous steps is NOT automatically included. You reference it explicitly with `{{stepname.output}}` or save it to an artifact and load it in a later step with `load_from`. This way you're not burning tokens feeding the entire conversation history into every step.

## How I use it in practice

I keep the pipeline YAMLs inside the project repo. Each one targets a specific concern: find code smells, check security patterns, review test coverage. Then I hook them up in mise:

```toml
# .mise.toml
[tasks.review]
run = "octos pipelines/code-review.yaml"

[tasks.security]
run = "octos pipelines/security-audit.yaml"
```

So my repo maintenance workflow is just `mise run review`. The pipeline runs, the agent finds problems, creates issues in Gitea. Next time I open the project, the issues are there. The prompts are versioned with the code, anyone can tweak them.

## Some internals

Written in Go, single binary. The TUI is built with [Bubble Tea](https://github.com/charmbracelet/bubbletea) v2 — it shows live streaming from the agent, file changes per step, and the current git branch (which refreshes on each step completion, since agents sometimes create branches).

The executor spawns the agent with `CommandContext` and detaches it from the controlling terminal to avoid SIGTTIN/SIGTTOU when child processes try to read stdin. Streams stdout/stderr line by line. If a step fails, you can configure `on_failure: retry` and it will re-run injecting the previous error into the prompt so the agent knows what went wrong. Or `on_failure: skip` to just move on. Or the default `fail_fast` and then `--resume` later from where it broke.

Prompts support `${ENV_VAR:-default}` expansion (before YAML parsing), `prompt_file` to load from external markdown files, and `when` conditions for conditional steps.

## Install

```bash
brew install vicendominguez/tap/octos
```

Or grab the binary from [releases](https://github.com/vicendominguez/octos/releases). Or `go build -o octos`.

Repo: [github.com/vicendominguez/octos](https://github.com/vicendominguez/octos)
