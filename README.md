# JG Plugins

Personal Claude Code plugin marketplace.

## Installation

Add this marketplace to Claude Code:

```
/plugin marketplace add jgatt493/jg-plugins
```

## Available Plugins

### Observational Memory

Automatic behavioral profiling for Claude Code sessions. Watches how you work, extracts patterns, and builds a portable behavioral profile that any AI agent can load.

```
/plugin install observational-memory@jg-plugins
```

### Mini TDD Harness

Automated TDD harness — write test specs, have Claude Code implement them. Includes a skill for writing specs and a Go CLI for running them.

```
/plugin install mini-tdd-harness@jg-plugins
```

### Decompose to Queue

Skill for decomposing high-level intents into atomic task files for the [`aq`](https://github.com/jgatt493/jg-agent-queue) runner. Routes prescriptive work to local pi/Qwen3, escalates judgment-heavy work back to Claude.

```
/plugin install decompose-to-queue@jg-plugins
```
