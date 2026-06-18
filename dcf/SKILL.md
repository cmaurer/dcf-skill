---
name: dcf
description: >
  Run a full Discounted Cash Flow (DCF) valuation on any publicly traded security to
  estimate intrinsic value per share. Implements a two-stage DCF model (explicit FCFe
  forecast + Gordon Growth Model terminal value) discounted at the CAPM cost of equity
  or WACC. Fetches live financial data from Massive Market Data MCP (falling back to
  AlphaVantage), auto-selects the best growth estimation method (analyst estimates →
  historical FCFe CAGR → user-specified), and always produces a sensitivity table.
  ALWAYS use this skill when the user asks to run a DCF, discounted cash flow analysis,
  intrinsic value of a ticker, DCF valuation, what a stock is worth intrinsically, fair
  value estimate, whether a ticker is undervalued, DCF model, or any request for
  fundamental cash-flow-based valuation. Also trigger when called from
  investment-scorecard, equity-analysis, or earnings-preview skills to produce a
  structured DCF_SUMMARY block.
---

# DCF Skill — Discounted Cash Flow Valuation

Estimates the intrinsic equity value of a publicly traded security using the HBS two-stage DCF model. Compares intrinsic value to current price and produces a margin of safety signal.

**Formula basis** (HBS / Srinivasan):
- `V₀ = Σ [FCFe_t / (1 + re)^t]` for t = 1 to N
- Terminal Value: `TV = FCFe_N × (1 + g) / (re - g)`  (Gordon Growth Model)
- Intrinsic Price = `(Σ PV(FCFe) + PV(TV)) / Shares Outstanding`

---

## Adaptive Output Mode

Read the conversation context before producing output:

- **Standalone** — user invoked DCF directly → produce the **Full Report** (see Report Structure)
- **Called from another skill** — investment-scorecard, equity-analysis, earnings-preview context present → produce a **short analysis** + the `DCF_SUMMARY` block
- **Quick-invoke** — `/dcf TICKER` with no other context → produce **headline numbers + sensitivity table only** + `DCF_SUMMARY` block

---

## Step 1: Gather Inputs

Identify the ticker symbol. If ambiguous, ask.

Collect from context or user if not already clear:
- **Forecast horizon** — default 5 years; ask if they want 10
- **Discount rate override** — use if provided; otherwise auto-calculate via CAPM
- **Growth rate override** — use Stage 1 / Stage 2 rates if provided; otherwise auto-estimate

---

## Step 2: Fetch Financial Data

Try **Massive Market Data MCP** first. On error or empty response, fall back to **AlphaVantage MCP**.

### Data checklist

| Data needed | Purpose |
|---|---|
| Cash Flow Statement (3–5 years) | Compute historical FCFe |
| Income Statement (3–5 years) | Net income, D&A, revenue trend |
| Balance Sheet (latest) | Debt/equity for WACC; net debt issuance |
| Current price + shares outstanding | Per-share value and margin of safety |
| Beta | CAPM cost of equity |

### FCFe Calculation

Preferred (three-component):
```
FCFe = Operating Cash Flow − Capital Expenditures + Net Debt Issuance (Repayment)
```

Simplified fallback (if net debt issuance unavailable):
```
FCFe = Operating Cash Flow − Capital Expenditures
```

**If FCFe is negative for most years**: Flag prominently. Consider using normalized FCFe (average of positive years) or ask the user whether to switch to an earnings-based approach. Do not silently proceed with a negative base FCFe into the model.

### Foreign Currency & ADR Handling

**If financials are reported in a non-USD currency** (e.g., TWD for TSM, EUR for ASML, JPY for Sony): convert all FCFe values to USD using the most recent average exchange rate before running the model. State the rate used prominently in the report.

**If the security trades as an ADR**: detect this and note the ordinary-share-to-ADR conversion ratio (e.g., TSM: 1 ADR = 5 ordinary shares). Use ADR-equivalent shares outstanding for the per-share intrinsic value calculation so the output is directly comparable to the quoted USD price. State the ratio in the report.

---

## Step 3: Estimate Growth Rates

Use this priority order — document which method was chosen and why:

1. **Analyst consensus estimates** — if MMD or AV returns forward EPS or FCF growth estimates, use them for Stage 1
2. **Historical FCFe CAGR** — calculate 3–5 year trailing CAGR from the financials you pulled
3. **User-specified** — ask if neither above is available or reliable (e.g., FCFe history is too volatile)

**Two-stage model (always):**
- **Stage 1** (Years 1–N): Explicit higher growth rate
- **Stage 2 / Terminal**: Conservative long-run rate, typically 2–4%. Must not exceed long-run nominal GDP (~3%). Cap at `re − 1%` to prevent model blow-up.

---

## Step 4: Determine Discount Rate

### Default: CAPM cost of equity

```
re = rf + β × ERP
```

| Component | Source | Default if unavailable |
|---|---|---|
| rf (risk-free rate) | Current 10-yr US Treasury yield | 4.25% |
| β (beta) | Financial data | 1.0 (flag as assumed) |
| ERP (equity risk premium) | Damodaran US ERP | 5.0% |

**Country / geopolitical risk premium**: For companies with concentrated sovereign or geopolitical risk not captured by CAPM beta, add an explicit premium to `re`. Common cases:

| Company profile | Suggested add-on |
|---|---|
| Taiwan-domiciled (e.g., TSM) | +1.0–2.0% for cross-strait risk |
| Emerging market HQ | +1.0–3.0% using Damodaran country risk premium |
| Sanction-exposed sector | +0.5–1.5% at analyst discretion |

When applying a country risk premium, state it explicitly in the report and explain the rationale. Use the midpoint of the range as default unless the user specifies otherwise.

### WACC (use when user requests it or company carries significant leverage)

```
WACC = (E/V × re) + (D/V × rd × (1 − tax rate))
```
- E = market cap, D = total debt, V = E + D
- rd = interest expense / total debt (pre-tax)
- tax rate = effective tax rate from income statement

Show the rate calculation transparently in the report.

---

## Step 5: Run the DCF Model

**Project FCFe** for each year, growing at Stage 1 rate:
```
FCFe_t = FCFe_0 × (1 + g1)^t
```

**Discount each year:**
```
PV(FCFe_t) = FCFe_t / (1 + re)^t
```

**Terminal Value** (end of forecast horizon):
```
TV = FCFe_N × (1 + g_terminal) / (re − g_terminal)
PV(TV) = TV / (1 + re)^N
```

**Sum to intrinsic equity value:**
```
V₀ = Σ PV(FCFe_t) + PV(TV)
Intrinsic Price = V₀ / Shares Outstanding
```

**Margin of Safety:**
```
MoS = (Intrinsic Price − Current Price) / Intrinsic Price × 100%
```
- Positive → trading below intrinsic value (undervalued)
- Negative → trading above intrinsic value (overvalued)

**Signal thresholds:**
- MoS > +15% → UNDERVALUED
- MoS between −15% and +15% → FAIRLY VALUED
- MoS < −15% → OVERVALUED

---

## Step 6: Sensitivity Analysis

Always produce this table — it's the most honest part of any DCF:

**Rows**: Discount rate (re − 2%, re − 1%, base, re + 1%, re + 2%)  
**Columns**: Terminal growth rate (g − 1%, g − 0.5%, base, g + 0.5%, g + 1%)  

Show intrinsic price per share at each intersection. Mark the base case clearly.

---

## Report Structure (Full / Standalone Mode)

```
# DCF Analysis: [COMPANY] ([TICKER])
Analysis Date: [DATE]  |  Data Source: [MMD / AlphaVantage]

## 1. Company Snapshot
Current Price | Market Cap | Sector | Industry

## 2. Historical Free Cash Flow to Equity
[Table: Year | Op CF | CapEx | Net Debt Issuance | FCFe | YoY Growth%]
Trailing FCFe CAGR (3yr / 5yr): X% / X%

## 3. Growth Assumptions
Method: [analyst estimates / historical CAGR / user-specified]
Stage 1 growth (Years 1–5): X%
Terminal growth rate: X%
Rationale: [2–3 sentences]

## 4. Discount Rate
rf: X%  |  Beta: X  |  ERP: 5.0%  |  re: X%
[Country risk premium if applied]
[WACC breakdown if applicable]

## 5. DCF Model
[Table: Year | FCFe ($M) | Discount Factor | PV ($M)]
Sum PV(FCFe Years 1–N): $XB
Terminal Value: $XB
PV(Terminal Value): $XB
──────────────────────────
Total Intrinsic Equity Value: $XB
Shares Outstanding: XB
Intrinsic Value Per Share: $X.XX

## 6. Valuation vs. Market Price
Current Price:      $X.XX
Intrinsic Value:    $X.XX
Margin of Safety:   X.X%
Signal:             [UNDERVALUED / FAIRLY VALUED / OVERVALUED]

## 7. Sensitivity Table
[Discount Rate × Terminal Growth Rate grid — intrinsic price per share]

## 8. Key Assumptions & Risks
- What would need to be true for this valuation to hold
- Downside risks to the model (macro, competitive, execution)
- Model limitations (e.g., FCFe volatility, limited history, negative FCF years)

## 9. Verdict
[3–4 sentence synthesis: intrinsic value, confidence, what to watch]
```

---

## DCF_SUMMARY Block (inter-skill / quick mode)

Append this structured block whenever called from another skill or in quick-invoke mode. Keep the key names consistent — downstream skills (investment-scorecard, equity-analysis) can parse this block from conversation context.

```
DCF_SUMMARY:
  ticker: [TICKER]
  intrinsic_value_per_share: $X.XX
  current_price: $X.XX
  margin_of_safety_pct: X.X
  signal: [UNDERVALUED | FAIRLY_VALUED | OVERVALUED]
  assumptions:
    discount_rate_pct: X.X
    terminal_growth_pct: X.X
    forecast_horizon_years: 5
    stage1_growth_pct: X.X
    growth_method: [analyst_estimates | historical_cagr | user_specified]
  sensitivity_range:
    bear_case_intrinsic: $X.XX    # re+2%, g-1%
    bull_case_intrinsic: $X.XX    # re-2%, g+1%
  data_source: [massive_market_data | alphavantage | mixed]
  confidence: [high | medium | low]
  run_date: [DATE]
```

**Confidence levels:**
- **High** — analyst estimates available + stable, positive FCFe history (low CV)
- **Medium** — historical CAGR used; FCFe positive but somewhat volatile
- **Low** — negative FCFe present, user-specified growth, or limited data coverage

---

## Edge Cases

| Situation | Handling |
|---|---|
| Negative FCFe | Flag prominently; use normalized FCFe (avg of positive years) or ask user |
| High-growth company where g ≈ re | Cap g at re − 1%; note model constraint |
| Financial company (bank, insurer) | Note FCFe is less appropriate; suggest DDM |
| Missing beta | Default to 1.0; flag as assumed in output |
| Missing shares outstanding | Derive from market cap ÷ current price |
| Very high terminal value (>80% of total) | Flag — model is highly sensitive to terminal assumptions; widen sensitivity table |
