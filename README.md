# Measure Agent Skills

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugins for instrumenting your app with [Measure](https://measure.dev) — the AI-native metrics platform.

> **Early access.** The Measure SDK (`@measuremetrics/measure-sdk`) is in private beta. [Request access](https://measure.dev) to get started.

## Plugins

### sdk-instrumenter

Instruments a Next.js B2B SaaS app with the Measure SDK (`@measuremetrics/measure-sdk`). Planning-first workflow:

1. **Discovers** your business model by reading the codebase (schema, server actions, webhooks, middleware)
2. **Plans** metrics using Kimball-style dimensional modeling — traces every metric back to a code path, then reviews the plan adversarially before writing any code
3. **Provisions** Measure entities and metrics (walks you through account setup if needed)
4. **Instruments** business processes with non-blocking `after()` calls
5. **Seeds** baseline data for existing customers

## Install

```
claude install measuremetrics/agent-skills
```

Then trigger the skill:

```
/instrument
```

Or describe what you want: *"instrument this app with Measure"*, *"add metrics tracking"*, *"add Measure SDK"*.

## Requirements

- A Next.js app with a B2B SaaS business model (subscriptions, multi-tenant)
- A [Measure](https://measure.dev) account (the skill walks you through setup if you don't have one)

## License

MIT
