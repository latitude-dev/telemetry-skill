# Latitude Telemetry Skill

An AI agent skill that guides you through integrating [Latitude](https://app.latitude.so) telemetry into any backend project for LLM observability.

## Installation

```
npx skills add latitude-dev/telemetry-skill
```

Or install manually by copying the skill folder into your agent's skill directory:

**Cursor**

```bash
cp -r latitude-telemetry .cursor/skills/
```

**Claude Code**

```bash
cp -r latitude-telemetry .claude/skills/
```

**Codex CLI**

```bash
cp -r latitude-telemetry .agents/skills/
```

## Usage

Once installed, ask your AI agent to add Latitude telemetry to your project. The skill will determine the best approach for your stack and walk you through the implementation.

**Examples:**

```
Add Latitude telemetry to my project
```

```
I want to trace my LLM calls with Latitude
```

```
Set up observability for my OpenAI calls
```

## What It Does

When activated, the skill walks you through choosing and implementing the right telemetry approach for your stack. It covers four integration paths:

| Approach              | Languages          | Complexity | Observability                |
| --------------------- | ------------------ | ---------- | ---------------------------- |
| **Gateway Mode**      | Any                | Minimal    | Full (automatic)             |
| **SDK Telemetry**     | Python, TypeScript | Low        | Full (tokens, cost, latency) |
| **Raw OpenTelemetry** | Any (with OTel)    | High       | Full (tokens, cost, latency) |
| **Create Log API**    | Any (with HTTP)    | Very low   | Conversation logs only       |

## Prerequisites

- A [Latitude](https://app.latitude.so) account and project
- Your Latitude API key (found in Settings)
- Your project ID (the number in the project URL)

## Skill Structure

- `SKILL.md` -- Instructions for the agent
- `references/` -- Detailed integration guides
  - `gateway.md` -- Latitude as LLM gateway
  - `sdk-telemetry.md` -- Python/TypeScript SDK instrumentation
  - `raw-opentelemetry.md` -- Manual OTLP span export
  - `log-api.md` -- Simple HTTP log endpoint
  - `span-attributes.md` -- Full span attribute reference

## License

MIT
