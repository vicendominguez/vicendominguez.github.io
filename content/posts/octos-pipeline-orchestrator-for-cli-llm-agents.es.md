---
title: "Octos: un orquestador minimalista de pipelines para agentes LLM de CLI"
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

Llevo usando agentes CLI (kiro, claude, goose...) a diario y me cansé de hacer siempre lo mismo: lanzar un prompt, mirar el output, copiar lo relevante, meterlo en el siguiente prompt. Repetir. Cuando algo peta en el paso 5, a empezar de cero.

Así que escribí [Octos](https://github.com/vicendominguez/octos). Es un runner de pipelines para agentes CLI. Describes los pasos en YAML y los encadena. Básicamente eso.

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

Cualquier binario que acepte un prompt y escriba en stdout funciona. Sin SDK, sin adapters. Puedes incluso cambiar de agente entre pasos — modelo barato para el análisis aburrido y el caro solo para el paso que lo necesita.

Lo importante del contexto: cada paso solo recibe lo que tú le das explícitamente. Hay un bloque `context` global en el YAML (rol, reglas, lo que quieras) que todos los pasos reciben, pero el output de pasos anteriores NO se incluye automáticamente. Lo referencies explícitamente con `{{stepname.output}}` o lo guardas en un artefacto y lo cargas en un paso posterior con `load_from`. Así no estás quemando tokens metiéndole todo el historial de la conversación a cada paso.

## Cómo lo uso en la práctica

Los YAMLs de las pipelines los meto dentro del propio repo del proyecto. Cada uno ataca un concern concreto: buscar code smells, revisar patrones de seguridad, comprobar cobertura de tests. Luego los engancho en mise:

```toml
# .mise.toml
[tasks.review]
run = "octos pipelines/code-review.yaml"

[tasks.security]
run = "octos pipelines/security-audit.yaml"
```

Mi workflow de mantenimiento del repo es simplemente `mise run review`. La pipeline se ejecuta, el agente encuentra problemas, crea issues en Gitea. La próxima vez que abro el proyecto, las issues están ahí. Los prompts están versionados con el código, cualquiera puede tocarlos.

## Algunos detalles internos

Escrito en Go, binario único. El TUI usa [Bubble Tea](https://github.com/charmbracelet/bubbletea) v2 — muestra streaming en directo del agente, cambios en ficheros por paso, y la rama de git actual (que se refresca en cada paso completado, porque los agentes a veces crean ramas).

El executor lanza el agente con `CommandContext` y lo desacopla de la terminal controladora para evitar SIGTTIN/SIGTTOU cuando los procesos hijo intentan leer de stdin. Hace streaming de stdout/stderr línea a línea. Si un paso falla, puedes configurar `on_failure: retry` y re-ejecuta inyectando el error anterior en el prompt para que el agente sepa qué fue mal. O `on_failure: skip` para seguir adelante. O el default `fail_fast` y luego `--resume` para retomar donde se rompió.

Los prompts soportan expansión de `${ENV_VAR:-default}` (antes del parseo YAML), `prompt_file` para cargar desde ficheros markdown externos, y condiciones `when` para pasos condicionales.

## Instalación

```bash
brew install vicendominguez/tap/octos
```

O pilla el binario de [releases](https://github.com/vicendominguez/octos/releases). O `go build -o octos`.

Repo: [github.com/vicendominguez/octos](https://github.com/vicendominguez/octos)
