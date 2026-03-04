# Model Matchmaker

**Stop paying Opus prices to rename files.**

A local hook for [Cursor](https://cursor.com) and [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that classifies every prompt before it's sent and tells you which model to use. No proxy, no API calls, no dependencies. Three files, two minutes to set up.

## What It Does

Before each prompt is sent, Model Matchmaker reads what you're asking and which model you're on, then makes a call:

- **Step-down**: You're on Opus asking to `git commit`? Blocked. "Switch to Haiku, same result, 90% cheaper."
- **Step-up**: You're on Sonnet asking about architecture tradeoffs? Blocked. "Switch to Opus, you need the horsepower."
- **Pass-through**: You're on the right model? Prompt goes through instantly.

Override anytime by prefixing your prompt with `!`.

## How It Works

```mermaid
flowchart TD
    SessionStart["Session starts"] --> InjectContext["sessionStart hook injects model guidance into system context"]
    UserSend["User hits Send"] --> ReadInput["beforeSubmitPrompt hook reads prompt + model"]
    ReadInput --> CheckOverride{"Prompt starts with '!'?"}
    CheckOverride -->|"Yes"| Allow["Allow immediately"]
    CheckOverride -->|"No"| Classify["Classify task via keyword matching"]
    Classify --> StepDown{"Overpaying? Opus/Sonnet for trivial task"}
    StepDown -->|"Yes"| Block["Block + recommend cheaper model"]
    StepDown -->|"No"| StepUp{"Underpowered? Sonnet/Haiku for complex task"}
    StepUp -->|"Yes"| Nudge["Block + recommend Opus"]
    StepUp -->|"No"| Allow
```

Two layers work together:

1. **`session-init.sh`** runs at session start and injects model-awareness context so the AI itself knows when to suggest switching
2. **`model-advisor.sh`** runs before every prompt, classifies the task, and blocks with a recommendation when you're on the wrong model

## Quick Setup

```bash
# 1. Clone this repo (or just copy the files)
git clone https://github.com/coyvalyss1/model-matchmaker.git

# 2. Copy files to your Cursor config
cp model-matchmaker/hooks.json ~/.cursor/
mkdir -p ~/.cursor/hooks
cp model-matchmaker/hooks/*.sh ~/.cursor/hooks/

# 3. Make scripts executable
chmod +x ~/.cursor/hooks/session-init.sh ~/.cursor/hooks/model-advisor.sh

# 4. Restart Cursor (or Claude Code)
```

That's it. No packages, no build step, no config files to edit.

## What Gets Routed Where

| Model | Task Type | Patterns |
|-------|-----------|----------|
| **Haiku** | Mechanical, simple | `git commit`, `git push`, `rename`, `reorder`, `move file`, `delete file`, `add import`, `format`, `lint`, `prettier`, `eslint` |
| **Sonnet** | Implementation | `build`, `implement`, `create`, `fix`, `debug`, `add feature`, `write`, `component`, `service`, `page`, `deploy`, `test`, `refactor` |
| **Opus** | Architecture, analysis | `architect`, `evaluate`, `tradeoff`, `strategy`, `deep dive`, `redesign`, `across the codebase`, `multi-system`, `analyze`, `rethink` |

Opus is also recommended for prompts over 200 words or analytical questions over 100 words.

The classifier is **conservative**: it only blocks when confidence is high. A false allow (wasting some money) is always better than a false block (interrupting your flow with a wrong recommendation).

## Sample Log Output

Every decision is logged to `~/.cursor/hooks/model-advisor.log`:

```
[2026-03-03 14:22:01] model=claude-4-opus rec=haiku action=BLOCK prompt="git commit all chang..."
[2026-03-03 14:23:15] model=claude-4-opus rec=sonnet action=BLOCK prompt="build a new componen..."
[2026-03-03 14:25:44] model=claude-4-sonnet rec=opus action=BLOCK prompt="evaluate the tradeof..."
[2026-03-03 14:30:02] model=claude-4-sonnet rec=uncertain action=ALLOW prompt="what time zone is Ne..."
```

The log only captures the first 20 characters of each prompt (for privacy) plus the model, recommendation, and whether it blocked or allowed. Useful for tuning the patterns to your workflow.

## Override

Prefix any prompt with `!` to bypass the advisor entirely:

```
! just do it on Opus, I know what I'm doing
```

The hook returns immediately with no classification.

## Why Not Just Use Cursor's Auto Mode?

Cursor's Auto mode runs server-side and picks from a curated shortlist (GPT-4.1, Claude 4 Sonnet, Gemini 2.5 Pro). A few limitations:

- It doesn't include Opus or Haiku, so it can't route to the cheapest or most powerful option
- It doesn't show you which model it picked
- Independent testing shows it mostly routes to Sonnet regardless of task complexity
- It optimizes for Cursor's infrastructure costs, not necessarily your output quality

Model Matchmaker doesn't replace Auto mode. It's a complementary local layer that works on top of whatever model you've selected, nudging you in both directions: down when you're overpaying, up when you're underpowered.

## Why Not a Proxy?

Proxy-based routing (custom proxy servers, ClawRouter, etc.) introduces real risks:

- 91,000+ attack sessions targeting LLM proxy endpoints were detected between Oct 2025 and Jan 2026
- API keys can leak via DNS exfiltration before HTTP-layer tools even see them
- A proxy crash means zero AI access until restarted
- You lose Cursor's built-in streaming, caching, and error handling

Model Matchmaker runs entirely locally. No network calls, no proxy, no attack surface.

## Design Decisions

- **Pure bash + python3** for JSON parsing. No external dependencies. python3 is pre-installed on macOS and most Linux.
- **2-second timeout**. If the script hangs, Cursor proceeds normally (fail-open). You're never locked out.
- **Local logging only**. Timestamp, model, recommendation, and a 20-char prompt snippet. No full prompts stored.
- **No LLM calls for classification**. Instant, free, deterministic. Keyword matching is fast and predictable.
- **No network calls**. Everything is local string matching. Nothing leaves your machine.

## Results

After a week of daily use building two products ([DoMoreWorld](https://domoreworld.com) and [Art Ping Pong](https://artpingpong.com)):

- ~40-50% of prompts get downgraded to Haiku or Sonnet with no quality loss
- Complex architecture prompts get upgraded when I forget I'm still on Sonnet
- The log file reveals usage patterns I didn't expect (most "build" prompts don't need Opus)

## Contributing

Open an issue or PR if you want to add patterns, tune the classifier, or add support for other editors. The keyword lists in `model-advisor.sh` are the main thing to tweak.

## License

MIT
