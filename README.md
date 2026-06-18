# DCF Skill for Claude Code

A Claude Code skill that runs a full **Discounted Cash Flow (DCF) valuation** on any publicly traded security using the HBS two-stage model.

## What It Does

- Fetches live financial data (Cash Flow, Income Statement, Balance Sheet) from Massive Market Data MCP, with AlphaVantage as fallback
- Auto-selects the best growth estimation method: analyst consensus → historical FCFe CAGR → user-specified
- Computes CAPM cost of equity (or WACC on request), with optional country/geopolitical risk premiums
- Runs a two-stage DCF: explicit FCFe forecast + Gordon Growth Model terminal value
- Always produces a sensitivity table (discount rate × terminal growth rate)
- Handles foreign currencies, ADRs, negative FCFe, and missing data gracefully
- Outputs a structured `DCF_SUMMARY` block for inter-skill consumption (investment-scorecard, equity-analysis, earnings-preview)

## Usage

Invoke the skill by asking Claude to run a DCF:

```
/dcf AAPL
Run a DCF on Tesla
What is Microsoft's intrinsic value?
Is NVDA undervalued based on discounted cash flows?
```

The skill is also triggered automatically when called from `investment-scorecard`, `equity-analysis`, or `earnings-preview` skills.

### Output Modes

| Mode | Trigger | Output |
|---|---|---|
| **Full report** | Standalone DCF request | 9-section report + sensitivity table |
| **Quick invoke** | `/dcf TICKER` with no context | Headline numbers + sensitivity table + `DCF_SUMMARY` block |
| **Inter-skill** | Called from another skill | Short analysis + `DCF_SUMMARY` block |

### Optional Parameters

You can override any assumption in natural language:

- **Forecast horizon**: "Run a 10-year DCF on AAPL" (default: 5 years)
- **Discount rate**: "Use a 9% discount rate"
- **Growth rates**: "Assume 12% Stage 1 growth and 2.5% terminal growth"
- **WACC**: "Use WACC instead of cost of equity"

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI or desktop app
- At least one data MCP connected:
  - **Massive Market Data MCP** (primary) — [setup instructions](https://massivemarketdata.com)
  - **AlphaVantage MCP** (fallback) — [setup instructions](https://www.alphavantage.co/support/#api-key)

### Install the Skill

Claude Code skills are loaded from a directory referenced in your Claude Code configuration. There are two ways to install:

**Option 1 — Clone into your skills directory:**

```bash
# Create a skills directory if you don't have one
mkdir -p ~/.claude/skills

# Clone the repo
git clone https://github.com/christianmaurer/dcf-skill ~/.claude/skills/dcf-skill
```

Then register the skill path in your `~/.claude/settings.json`:

```json
{
  "skills": [
    "~/.claude/skills/dcf-skill"
  ]
}
```

**Option 2 — Reference the repo directly in settings:**

```json
{
  "skills": [
    "/absolute/path/to/dcf-skill"
  ]
}
```

After adding the path, restart Claude Code. The skill will appear as `/dcf` in the slash command list.

## Skill Package Format

Claude Code skills are a directory containing a `SKILL.md` file at the root (or in a named subdirectory). The `SKILL.md` uses YAML frontmatter to register the skill:

```
skill-directory/
└── <skill-name>/
    └── SKILL.md        ← required; frontmatter declares name, description, triggers
```

**Frontmatter fields:**

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Slash-command name (e.g., `dcf` → `/dcf`) |
| `description` | Yes | Natural-language trigger description; Claude uses this to auto-invoke the skill |

The `description` field is important — Claude reads it to decide when to invoke the skill automatically, without the user typing the slash command explicitly.

## Model

**Formula** (HBS / Srinivasan two-stage DCF):

```
V₀ = Σ [FCFe_t / (1 + re)^t]  for t = 1..N
   + FCFe_N × (1 + g) / (re − g) / (1 + re)^N

Intrinsic Price = V₀ / Shares Outstanding
Margin of Safety = (Intrinsic Price − Current Price) / Intrinsic Price
```

**FCFe:**
```
FCFe = Operating Cash Flow − CapEx + Net Debt Issuance
```

**CAPM cost of equity:**
```
re = rf + β × ERP
```

Default parameters: rf = current 10-yr Treasury yield (fallback 4.25%), ERP = 5.0% (Damodaran US).

## License

MIT
