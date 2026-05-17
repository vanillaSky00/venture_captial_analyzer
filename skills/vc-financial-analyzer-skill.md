---
name: vc-financial-analyzer
description: Analyze startups and companies for venture capital evaluation using time-series financial metrics, forecasts, burn/runway, growth, efficiency, return signals, and Python chart exports. Use when asked to analyze company financials, show trends over time, predict revenue/burn/growth, build a VC dashboard, or create dynamic VC evaluation charts.
---

# VC Financial Analyzer Skill

> **Purpose:** Evaluate companies dynamically — not as a static snapshot, but as a moving story
> over time, with trend lines and forward projections. Built for analysts with limited resources
> (no Bloomberg terminal, no proprietary data feeds).
>
> **Trigger phrases:** "analyze this company financially", "show trends over time",
> "predict revenue/burn/growth", "VC evaluation", "dynamic chart", "rolling average",
> "financial dashboard python", or any request to go beyond static numbers and reveal momentum.

---

## Philosophy: Why Dynamic > Static

A static balance sheet tells you *where a company is*. A dynamic trend tells you *where it's
going and how fast*. VC mentors prioritize trajectory over position.

> A company with $500K revenue growing 25% MoM is more investable than one with $5M revenue
> flat for 12 months.

**Core principle:** Every metric must be shown as a time series, not a single number.

---

## Output Convention (v1.2+)

Each indicator is saved as its **own PNG file**, named by indicator, then all files are
**zipped into one download**. Open the zip — the files sort into logical VC evaluation order.

```
vc_analysis/
  00_vc_signal_summary.png   ← open this first: scorecard of all signals
  01_revenue_trend.png
  02_growth_rate.png
  03_burn_rate.png
  04_runway.png
  05_burn_multiple.png
  06_gross_margin.png
  07_valuation.png           ← NEW: DCF / Comparables / Risk-Adjusted side-by-side
```

**Why numbered filenames?** File explorers sort alphabetically — the `NN_` prefix keeps the
summary card first and the metrics in logical VC evaluation order (survival → efficiency).

**Text output (machine-readable for downstream agents):**

```
vc_financial_results.md     ← all computed values as structured markdown; feed this to the next skill
```

---

## Step 1 — Clarify the Analysis Target

Before writing any code, ask:

1. **What data do you have?** (CSV upload, manual table, public filings, API like yfinance?)
2. **What stage is the company?** (Pre-revenue, growth, late-stage / public)
3. **Time horizon?** (Quarterly for private; daily/weekly for public stocks)
4. **Time basis for every input?** (`monthly`, `quarterly`, or `annual`; especially for burn/cost)
5. **Prediction window?** (e.g., forecast next 3–6 periods)

> If the user has no data, generate a realistic synthetic dataset to demonstrate the workflow
> (see the full demo script at the bottom of this document).

---

## Step 2 — Data Sources by Resource Level

| Resource Level | Source | Python Tool |
|---|---|---|
| Minimal (free) | Yahoo Finance (public companies) | `yfinance` |
| Minimal (free) | Company CSV/Excel upload | `pandas.read_csv()` |
| Medium | SEC EDGAR filings (public) | `requests` + manual parse |
| Medium | Macrotrends.net scrape | `pandas.read_html()` |
| Advanced | Alpha Vantage API (free tier) | `requests` + JSON |

**For private companies:** user must supply data manually (pitch deck, data room, investor
updates). Always accept a minimal CSV with columns: `date, revenue, burn, cash`, but
require one explicit metadata value: `PERIOD = "month" | "quarter" | "year"`. Never assume
that `burn` is monthly just because the metric is called burn.

---

## Step 3 — The VC Metrics Hierarchy

Evaluate in this order (highest signal first):

### Tier 1 — Survival Metrics (always required)

| Metric | Formula | Signal |
|---|---|---|
| **Burn Rate** | Monthly cash outflow | Trend direction > absolute number |
| **Runway** | Cash ÷ Monthly Burn | Must be >12 mo; trending up = safe |
| **Revenue Growth Rate** | MoM or QoQ % change | Compounding matters more than absolute |

### Tier 2 — Efficiency Metrics (growth-stage)

| Metric | Formula | Benchmark |
|---|---|---|
| **Gross Margin %** | (Revenue − COGS) ÷ Revenue | SaaS: 70–80%+; trending up = scale effect |
| **Burn Multiple** | Net Burn ÷ Net New ARR | <1.5x healthy; trending down = efficient |
| **CAC Payback** | Months to recover customer cost | <12 mo = best-in-class |

### Tier 3 — Return Metrics (fund-level or late-stage)

| Metric | Formula | VC Target |
|---|---|---|
| **MOIC** | Total Value ÷ Capital Invested | Strong = 3x+ |
| **IRR** | Discount rate where NPV = 0 | 20–30%+ (seed: 30%+) |
| **DPI** | Cash Returned ÷ Capital Invested | >1.0x = got money back; LPs prefer over TVPI |

---

## Step 3a — Reliability Guardrail: Normalize Time Units First

Most chart bugs come from mixing monthly, quarterly, and annual units. Before any chart is
drawn, normalize derived metrics once and reuse those columns everywhere.

```python
import pandas as pd
import numpy as np

# Required metadata. Set this from the source file or user prompt.
PERIOD = "quarter"  # one of: "month", "quarter", "year"
DATE_FREQ_BY_PERIOD = {"month": "MS", "quarter": "QS", "year": "YS"}
PERIODS_PER_YEAR_BY_PERIOD = {"month": 12, "quarter": 4, "year": 1}
PERIOD_LABEL_BY_PERIOD = {"month": "month", "quarter": "qtr", "year": "yr"}

if PERIOD not in PERIODS_PER_YEAR_BY_PERIOD:
    raise ValueError("PERIOD must be one of: month, quarter, year")

PERIODS_PER_YEAR = PERIODS_PER_YEAR_BY_PERIOD[PERIOD]
MONTHS_PER_PERIOD = 12 / PERIODS_PER_YEAR
DATE_FREQ = DATE_FREQ_BY_PERIOD[PERIOD]
PERIOD_LABEL = PERIOD_LABEL_BY_PERIOD[PERIOD]

def annualize(values):
    return values * PERIODS_PER_YEAR

def period_label(ts):
    ts = pd.Timestamp(ts)
    if PERIOD == "year":
        return f"{ts.year}"
    if PERIOD == "quarter":
        return f"{ts.year}-Q{(ts.month - 1)//3 + 1}"
    return ts.strftime("%Y-%m")

def add_derived_metrics(df):
    """Create one source of truth for charts and markdown.

    Input contract:
      revenue = revenue for the period
      burn    = cash burn or cost proxy for the period, not necessarily monthly
      cash    = ending cash balance
      gross_margin = decimal (0.72 means 72%)
    """
    required = {"date", "revenue", "burn", "cash", "gross_margin"}
    missing = required - set(df.columns)
    if missing:
        raise ValueError(f"Missing required columns: {sorted(missing)}")

    df = df.copy()
    df["date"] = pd.to_datetime(df["date"])
    df = df.sort_values("date").reset_index(drop=True)

    # Runway is always months. If burn is quarterly, divide by 3; if annual, divide by 12.
    df["monthly_burn"] = df["burn"] / MONTHS_PER_PERIOD
    df["runway"] = np.where(df["monthly_burn"] > 0, df["cash"] / df["monthly_burn"], np.nan)

    # Burn multiple uses period burn divided by net new annualized revenue.
    df["annualized_revenue"] = annualize(df["revenue"])
    df["net_new_arr"] = df["annualized_revenue"].diff()
    df["burn_multiple"] = np.where(df["net_new_arr"] > 0, df["burn"] / df["net_new_arr"], np.nan)

    df["revenue_ma"] = df["revenue"].rolling(3, min_periods=1).mean()
    df["growth_rate"] = df["revenue"].pct_change() * 100
    df["growth_ma"] = df["growth_rate"].rolling(3, min_periods=1).mean()
    return df
```

**Important:** If the company is profitable and `burn` is a cost proxy such as OpEx or
COGS+OpEx, rename the chart/metric as a cost-efficiency proxy. Do not present it as cash
burn unless the input is actual net cash burn. For public annual P&L data, prefer:
`cost_efficiency_multiple = (total_cost.diff() / revenue.diff())`, and label it
`ΔTotal Cost ÷ ΔRevenue` instead of `Burn Multiple`.

---

## Step 4 — Canvas & Chart Configuration

**Set the canvas once, globally, before any `plt.plot()` calls.**

```python
import os, zipfile
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import matplotlib.font_manager as fm
from matplotlib.patches import Patch
import pandas as pd
import numpy as np

def choose_font_family():
    """Prefer CJK-capable fonts when charts include Chinese company names."""
    candidates = [
        "/System/Library/Fonts/PingFang.ttc",
        "/System/Library/Fonts/STHeiti Medium.ttc",
        "/System/Library/Fonts/Hiragino Sans GB.ttc",
        "/System/Library/Fonts/Supplemental/Arial Unicode.ttf",
    ]
    for path in candidates:
        if os.path.exists(path):
            try:
                fm.fontManager.addfont(path)
            except Exception:
                pass
    return ["PingFang TC", "STHeiti", "Hiragino Sans GB", "Arial Unicode MS", "DejaVu Sans"]

# ── CANVAS CONFIG ─────────────────────────────────────────────────────────────
plt.rcParams.update({
    "figure.facecolor":  "#0f0f14",   # dark background
    "axes.facecolor":    "#16161e",
    "axes.edgecolor":    "#2a2a3a",
    "axes.labelcolor":   "#c0c0d0",
    "text.color":        "#e0e0f0",
    "xtick.color":       "#888899",
    "ytick.color":       "#888899",
    "grid.color":        "#2a2a3a",
    "grid.linestyle":    "--",
    "grid.alpha":        0.5,
    "lines.linewidth":   2.2,
    "font.family":       choose_font_family(),
    "axes.unicode_minus": False,
    "font.size":         10,
    "axes.titlesize":    13,
    "axes.titleweight":  "bold",
    "legend.framealpha": 0.15,
    "legend.edgecolor":  "#3a3a5a",
    "legend.fontsize":   9,
})

# ── COLOR PALETTE ─────────────────────────────────────────────────────────────
COLORS = {
    "primary":   "#7b6ef6",   # purple  — main metric line
    "positive":  "#4ade80",   # green   — growth / healthy
    "warning":   "#fb923c",   # orange  — caution zone
    "danger":    "#f87171",   # red     — risk / declining
    "neutral":   "#94a3b8",   # gray    — raw / reference data
    "forecast":  "#38bdf8",   # blue    — predicted values
    "accent":    "#f472b6",   # pink    — annotations / secondary
}

FOOTER = "⚠  Forecast is directional only (linear regression). Not investment advice."
```

---

## Step 5 — Save Helper (one PNG per indicator)

```python
OUT_DIR  = "./vc_analysis"
ZIP_PATH = "./vc_analysis_charts.zip"
os.makedirs(OUT_DIR, exist_ok=True)

def save_fig(fig, filename):
    """Add footer, save PNG at 150 dpi, close figure. Returns saved path."""
    path = os.path.join(OUT_DIR, filename)
    fig.text(0.5, -0.03, FOOTER, ha="center", fontsize=8, color="#555570")
    fig.savefig(path, dpi=150, bbox_inches="tight", facecolor=fig.get_facecolor())
    plt.close(fig)
    print(f"  ✓  {filename}")
    return path

saved_files = []   # collect all paths, zip at the end
```

---

## Step 6 — Chart Recipes by Indicator

Each chart is a standalone `fig, ax = plt.subplots(figsize=(12, 6))` block.
**Never reuse a figure across indicators.**

### 6a. `01_revenue_trend.png` — Revenue + Rolling Avg + Forecast

```python
fy_rev, _ = linear_forecast(df["revenue"], FORECAST_PERIODS)

fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  Revenue Trend + Forecast", fontsize=14, fontweight="bold", color="#e0e0f0")

ax.plot(df["date"], df["revenue"] / 1e6,    color=COLORS["neutral"], alpha=0.45, linewidth=1.5, label="Actual Revenue")
ax.plot(df["date"], df["revenue_ma"] / 1e6, color=COLORS["primary"], linewidth=2.8, label="3-period rolling avg")
ax.plot(f_dates, fy_rev / 1e6, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Linear Forecast")
ax.fill_between(f_dates, fy_rev/1e6 * 0.82, fy_rev/1e6 * 1.18,
                color=COLORS["forecast"], alpha=0.12, label="±18% Band")

ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.1fM"))
ax.set_ylabel("Revenue ($M)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper left"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "01_revenue_trend.png"))
```

### 6b. `02_growth_rate.png` — Period Growth + Momentum Zones

```python
df["growth_rate"] = df["revenue"].pct_change() * 100
df["growth_ma"]   = df["growth_rate"].rolling(3, min_periods=1).mean()
valid = df["growth_rate"].notna()

fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  Period Revenue Growth Rate (%)", fontsize=14, fontweight="bold", color="#e0e0f0")

ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] > 0), color=COLORS["positive"], alpha=0.2)
ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] < 0), color=COLORS["danger"], alpha=0.2)
ax.plot(df["date"][valid], df["growth_rate"][valid], color=COLORS["neutral"], alpha=0.5, linewidth=1.5)
ax.plot(df["date"][valid], df["growth_ma"][valid],   color=COLORS["primary"], linewidth=2.8, label="3-Period Avg")
ax.axhline(0,  color="#444455", linewidth=1.0)
ax.axhline(15, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="15% VC Benchmark")

ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylabel("Growth Rate (%)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "02_growth_rate.png"))
```

### 6c. `03_burn_rate.png` — Burn Bars (color = runway urgency) + Trend

```python
bar_colors = [COLORS["positive"] if r > 18 else COLORS["warning"] if r > 12
              else COLORS["danger"] for r in df["runway"]]
fy_burn, _ = linear_forecast(df["burn"] / 1e3, FORECAST_PERIODS)

fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  Burn Rate by Period", fontsize=14, fontweight="bold", color="#e0e0f0")

ax.bar(df["date"], df["burn"] / 1e3, color=bar_colors, width=55, alpha=0.85)
ax.plot(df["date"], df["burn"]/1e3, color=COLORS["neutral"], linewidth=1.5, linestyle=":", alpha=0.6)
ax.plot(f_dates, fy_burn, color=COLORS["forecast"], linewidth=2, linestyle="--")

legend_els = [
    Patch(facecolor=COLORS["positive"], label="Runway > 18 mo (Safe)"),
    Patch(facecolor=COLORS["warning"],  label="Runway 12–18 mo (Watch)"),
    Patch(facecolor=COLORS["danger"],   label="Runway < 12 mo (Risk)"),
    plt.Line2D([0],[0], color=COLORS["forecast"], linewidth=2, linestyle="--", label="Burn Forecast"),
]
ax.legend(handles=legend_els, loc="upper right")
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.0fK"))
ax.set_ylabel(f"Burn Rate ($K/{PERIOD_LABEL})"); ax.set_xlabel(PERIOD.title())
ax.grid(True, axis="y"); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "03_burn_rate.png"))
```

### 6d. `04_runway.png` — Runway with Threshold Bands

```python
fy_run, _ = linear_forecast(df["runway"], FORECAST_PERIODS)

fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  Cash Runway (months)", fontsize=14, fontweight="bold", color="#e0e0f0")

ax.plot(df["date"], df["runway"], color=COLORS["primary"], linewidth=2.8,
        marker="o", markersize=6, label="Runway (months)")
ax.plot(f_dates, fy_run, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_run * 0.85, fy_run * 1.15, color=COLORS["forecast"], alpha=0.1)

ax.axhline(24, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="24 Mo — Ideal")
ax.axhline(18, color=COLORS["warning"],  linewidth=1.2, linestyle=":", alpha=0.7, label="18 Mo — VC target")
ax.axhline(12, color=COLORS["danger"],   linewidth=1.2, linestyle=":", alpha=0.7, label="12 Mo — Warning")
ax.axhspan(0,  12, color=COLORS["danger"],  alpha=0.04)
ax.axhspan(12, 18, color=COLORS["warning"], alpha=0.04)

ax.set_ylabel("Runway (months)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper left"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "04_runway.png"))
```

### 6e. `05_burn_multiple.png` — Capital Efficiency + Zone Shading

```python
bm_valid = df["burn_multiple"].notna()
bm_vals  = df["burn_multiple"][bm_valid]
fy_bm, _ = linear_forecast(bm_vals.values, FORECAST_PERIODS)

fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  Burn Multiple (Capital Efficiency)", fontsize=14, fontweight="bold", color="#e0e0f0")

ax.plot(df["date"][bm_valid], bm_vals, color=COLORS["warning"], linewidth=2.8,
        marker="o", markersize=6, label="Burn Multiple")
ax.plot(f_dates, fy_bm, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_bm * 0.85, fy_bm * 1.15, color=COLORS["forecast"], alpha=0.1)

ax.axhline(1.0, color=COLORS["positive"], linewidth=1.4, linestyle="--", alpha=0.8, label="< 1x Exceptional")
ax.axhline(1.5, color=COLORS["warning"],  linewidth=1.4, linestyle="--", alpha=0.8, label="1.5x Good")
ax.axhline(2.0, color=COLORS["danger"],   linewidth=1.4, linestyle="--", alpha=0.8, label="2x Concerning")
ax.axhspan(0,   1.0, color=COLORS["positive"], alpha=0.04)
ax.axhspan(1.0, 1.5, color=COLORS["warning"],  alpha=0.04)
ax.axhspan(1.5, 4.0, color=COLORS["danger"],   alpha=0.04)
ax.fill_between(df["date"][bm_valid], bm_vals, 2.0,
                where=(bm_vals < 2.0), color=COLORS["positive"], alpha=0.08)
ax.set_ylim(0, 4)
ax.set_ylabel("Burn Multiple (×)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "05_burn_multiple.png"))
```

### 6f. `06_gross_margin.png` — Gross Margin % Trend

```python
fy_gm, _ = linear_forecast(df["gross_margin"] * 100, FORECAST_PERIODS)
gm_ma = (df["gross_margin"] * 100).rolling(3, min_periods=1).mean()

fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  Gross Margin % Trend", fontsize=14, fontweight="bold", color="#e0e0f0")

ax.plot(df["date"], df["gross_margin"] * 100, color=COLORS["neutral"], alpha=0.5, linewidth=1.5, label="Actual")
ax.plot(df["date"], gm_ma, color=COLORS["accent"], linewidth=2.8, label="3-period rolling avg")
ax.plot(f_dates, fy_gm, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_gm - 3, fy_gm + 3, color=COLORS["forecast"], alpha=0.1, label="±3pp Band")

ax.axhline(80, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="80% Top SaaS")
ax.axhline(70, color=COLORS["warning"],  linewidth=1.2, linestyle=":", alpha=0.7, label="70% SaaS min")
ax.axhline(50, color=COLORS["danger"],   linewidth=1.2, linestyle=":", alpha=0.7, label="50% Below standard")

ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylim(30, 100)
ax.set_ylabel("Gross Margin (%)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="lower right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "06_gross_margin.png"))
```

### 6g. `07_valuation.png` — Valuation: DCF · Comparables · Risk-Adjusted

Three methods, one chart. Shows each valuation estimate as a bar with an uncertainty band.
Analyst reads the **overlap zone** as the credible range.

**Three common valuation methods for startups:**

| Method | Formula / Logic | Best For |
|---|---|---|
| **DCF** | PV of projected free cash flows at discount rate *r* | Companies with visible revenue trajectory |
| **Comparables (EV/ARR)** | Peer median multiple × current ARR | SaaS with public comps |
| **Risk-Adjusted** | DCF × (1 − failure_probability) | Seed / early-stage with high uncertainty |

```python
# ── VALUATION INPUTS (edit per company) ───────────────────────────────────────
DISCOUNT_RATE       = 0.25          # 25% — typical VC hurdle rate
TERMINAL_GROWTH     = 0.03          # 3% long-run growth after projection window
PROJECTION_YEARS    = 5             # DCF horizon
FCF_MARGIN          = 0.15          # Assumed FCF margin on projected revenue
EV_ARR_MULTIPLE     = 8.0           # Comparable median (public SaaS comps, adjust per sector)
EV_ARR_RANGE        = (5.0, 12.0)   # Low / high comparable range
FAILURE_PROBABILITY = 0.40          # Risk-adjusted discount (40% = Series A typical)

# Use last-period revenue as ARR proxy (replace with actual ARR if available)
last_rev = df["revenue"].iloc[-1]
ann_rev  = annualize(last_rev)      # annualize period revenue → ARR proxy

# ── DCF VALUATION ─────────────────────────────────────────────────────────────
avg_period_growth = df["growth_rate"].dropna().mean() / 100
annual_growth = (1 + avg_period_growth) ** PERIODS_PER_YEAR - 1
proj_revs  = [ann_rev * ((1 + annual_growth) ** yr) for yr in range(1, PROJECTION_YEARS + 1)]
proj_fcfs  = [r * FCF_MARGIN for r in proj_revs]

# Terminal value (Gordon Growth Model)
terminal_value = proj_fcfs[-1] * (1 + TERMINAL_GROWTH) / (DISCOUNT_RATE - TERMINAL_GROWTH)

dcf_value = sum(fcf / (1 + DISCOUNT_RATE) ** t
                for t, fcf in enumerate(proj_fcfs, 1))
dcf_value += terminal_value / (1 + DISCOUNT_RATE) ** PROJECTION_YEARS

# ── COMPARABLES VALUATION ─────────────────────────────────────────────────────
comp_value_mid  = ann_rev * EV_ARR_MULTIPLE
comp_value_low  = ann_rev * EV_ARR_RANGE[0]
comp_value_high = ann_rev * EV_ARR_RANGE[1]

# ── RISK-ADJUSTED VALUATION ───────────────────────────────────────────────────
risk_adj_value  = dcf_value * (1 - FAILURE_PROBABILITY)
# ±30% uncertainty band on risk-adjusted
risk_adj_low    = risk_adj_value * 0.70
risk_adj_high   = risk_adj_value * 1.30

# ── CHART ─────────────────────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 7))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  Valuation Estimate: DCF vs Comparables vs Risk-Adjusted",
             fontsize=14, fontweight="bold", color="#e0e0f0")

labels     = ["DCF\n(Intrinsic)", "Comparables\n(EV/ARR)", "Risk-Adjusted\n(DCF × Survival)"]
midpoints  = [dcf_value,      comp_value_mid,  risk_adj_value]
lows       = [dcf_value*0.75, comp_value_low,  risk_adj_low]
highs      = [dcf_value*1.25, comp_value_high, risk_adj_high]
bar_colors = [COLORS["primary"], COLORS["forecast"], COLORS["warning"]]

x = np.arange(len(labels))
bars = ax.bar(x, [m/1e6 for m in midpoints], color=bar_colors, width=0.5, alpha=0.85, zorder=3)

# Error bars = uncertainty / comparable range
yerr_low  = [(m - l)/1e6 for m, l in zip(midpoints, lows)]
yerr_high = [(h - m)/1e6 for m, h in zip(midpoints, highs)]
ax.errorbar(x, [m/1e6 for m in midpoints],
            yerr=[yerr_low, yerr_high],
            fmt="none", color="#e0e0f0", capsize=10, capthick=2, linewidth=2, zorder=4)

# Annotate bar tops
for bar, mid in zip(bars, midpoints):
    ax.text(bar.get_x() + bar.get_width()/2, bar.get_height() + max(midpoints)*0.01/1e6,
            f"${mid/1e6:.1f}M", ha="center", va="bottom", fontsize=12,
            fontweight="bold", color="#e0e0f0")

# Overlap band (credible range = max of lows → min of highs)
overlap_low  = max(lows)  / 1e6
overlap_high = min(highs) / 1e6
if overlap_low < overlap_high:
    ax.axhspan(overlap_low, overlap_high,
               color=COLORS["positive"], alpha=0.08, label=f"Credible range ${overlap_low:.1f}M–${overlap_high:.1f}M")
    ax.axhline(overlap_low,  color=COLORS["positive"], linewidth=1.2, linestyle="--", alpha=0.6)
    ax.axhline(overlap_high, color=COLORS["positive"], linewidth=1.2, linestyle="--", alpha=0.6)

# Annotation box: inputs used
param_text = (f"Inputs:  r = {DISCOUNT_RATE*100:.0f}%  |  "
              f"EV/ARR = {EV_ARR_MULTIPLE:.1f}×  |  "
              f"Failure prob = {FAILURE_PROBABILITY*100:.0f}%  |  "
              f"ARR proxy = ${ann_rev/1e6:.2f}M")
ax.text(0.5, -0.10, param_text, transform=ax.transAxes,
        ha="center", fontsize=8.5, color="#888899")

ax.set_xticks(x); ax.set_xticklabels(labels, fontsize=11)
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.0fM"))
ax.set_ylabel("Estimated Valuation ($M)")
ax.legend(loc="upper right"); ax.grid(True, axis="y", alpha=0.4); ax.set_axisbelow(True)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "07_valuation.png"))

# ── STORE VALUATION RESULTS (for vc_financial_results.md writer) ──────────────
valuation_results = {
    "dcf_value_M":        round(dcf_value / 1e6, 2),
    "dcf_low_M":          round(dcf_value * 0.75 / 1e6, 2),
    "dcf_high_M":         round(dcf_value * 1.25 / 1e6, 2),
    "comp_value_mid_M":   round(comp_value_mid / 1e6, 2),
    "comp_value_low_M":   round(comp_value_low / 1e6, 2),
    "comp_value_high_M":  round(comp_value_high / 1e6, 2),
    "risk_adj_value_M":   round(risk_adj_value / 1e6, 2),
    "risk_adj_low_M":     round(risk_adj_low / 1e6, 2),
    "risk_adj_high_M":    round(risk_adj_high / 1e6, 2),
    "overlap_low_M":      round(overlap_low, 2) if overlap_low < overlap_high else None,
    "overlap_high_M":     round(overlap_high, 2) if overlap_low < overlap_high else None,
    "arr_proxy_M":        round(ann_rev / 1e6, 2),
    "discount_rate_pct":  DISCOUNT_RATE * 100,
    "ev_arr_multiple":    EV_ARR_MULTIPLE,
    "failure_prob_pct":   FAILURE_PROBABILITY * 100,
}
```

### 6h. `00_vc_signal_summary.png` — Text Scorecard (open this first)

```python
fig, ax = plt.subplots(figsize=(10, 6))
fig.patch.set_facecolor("#0f0f14")
ax.set_facecolor("#0f0f14"); ax.axis("off")
fig.suptitle("Company  ·  VC Signal Summary", fontsize=14, fontweight="bold", color="#e0e0f0")

def signal_row(name, current, slope, good_dir="up", stable_band=0.02):
    if good_dir == "up":
        color = COLORS["positive"] if slope > stable_band else (COLORS["warning"] if slope >= -stable_band else COLORS["danger"])
        emoji = "●" if slope > stable_band else ("◑" if slope >= -stable_band else "○")
    else:
        color = COLORS["positive"] if slope < -stable_band else (COLORS["warning"] if slope <= stable_band else COLORS["danger"])
        emoji = "●" if slope < -stable_band else ("◑" if slope <= stable_band else "○")
    return name, current, f"{slope:+.2f}", color, emoji

# Build one signal_row() call per metric
rows = [
    signal_row("Revenue Growth",      "display_value", slope_rev,  "up",   0.02),
    signal_row("Period Growth Rate",  "display_value", slope_gr,   "up",   2.0),  # percentage points / period
    signal_row("Burn Rate",           "display_value", slope_burn, "down", 0.02),
    signal_row("Runway",              "display_value", slope_run,  "up",   1.0),  # months / period
    signal_row("Burn Multiple",       "display_value", slope_bm,   "down", 0.10),
    signal_row("Gross Margin",        "display_value", slope_gm,   "up",   1.0),  # percentage points / period
]

header_y = 0.88
for label, x in [("Indicator",0.04),("Latest",0.42),("Trend Slope",0.62),("Signal",0.80)]:
    ax.text(x, header_y, label, fontsize=10, color="#888899", fontweight="bold",
            transform=ax.transAxes)
ax.plot([0.04,0.96],[header_y-0.04,header_y-0.04], color="#2a2a3a",
        linewidth=1, transform=ax.transAxes)

for i, (name, current, slope, color, emoji) in enumerate(rows):
    y = 0.76 - i * 0.12
    ax.text(0.04, y, name,    fontsize=11, color="#e0e0f0", transform=ax.transAxes)
    ax.text(0.42, y, current, fontsize=11, color="#c0c0d0", transform=ax.transAxes)
    ax.text(0.62, y, slope,   fontsize=11, color=color,     transform=ax.transAxes)
    ax.text(0.80, y, emoji,   fontsize=18, color=color,     transform=ax.transAxes)

plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "00_vc_signal_summary.png"))
```

---



```python
with zipfile.ZipFile(ZIP_PATH, "w", zipfile.ZIP_DEFLATED) as zf:
    for fpath in sorted(saved_files):
        arcname = os.path.join("vc_analysis", os.path.basename(fpath))
        zf.write(fpath, arcname)

print(f"✅  Zip ready: {ZIP_PATH}")
with zipfile.ZipFile(ZIP_PATH, "r") as zf:
    for name in sorted(zf.namelist()):
        print(f"  {name}  ({zf.getinfo(name).file_size//1024} KB)")
```

---

## Step 7a — Zip All Charts

```python
with zipfile.ZipFile(ZIP_PATH, "w", zipfile.ZIP_DEFLATED) as zf:
    for fpath in sorted(saved_files):
        arcname = os.path.join("vc_analysis", os.path.basename(fpath))
        zf.write(fpath, arcname)

print(f"✅  Zip ready: {ZIP_PATH}")
with zipfile.ZipFile(ZIP_PATH, "r") as zf:
    for name in sorted(zf.namelist()):
        print(f"  {name}  ({zf.getinfo(name).file_size//1024} KB)")
```

---

## Step 7b — Write `vc_financial_results.md` (Machine-Readable Handoff)

> **Why?** The `.md` file captures every computed value as clean structured text so a
> downstream agent (report writer, memo generator, investment committee summarizer) can
> read it as a string without parsing binary chart files.
>
> **Rule:** Write this file **after all charts are generated**, so all values are final.

```python
import datetime

MD_PATH = "./vc_financial_results.md"

# ── COLLECT SIGNAL ROWS (reuse from signal_summary logic) ─────────────────────
bm_all = df["burn_multiple"].dropna()
latest_bm = bm_all.iloc[-1] if not bm_all.empty else np.nan
latest_bm_label = f"{latest_bm:.2f}×" if not pd.isna(latest_bm) else "n/a"

def slope_label(slope, good_dir="up", stable_band=0.02):
    if good_dir == "up":
        return "↑ Positive" if slope > stable_band else ("→ Stable" if slope >= -stable_band else "↓ Declining")
    else:
        return "↓ Improving" if slope < -stable_band else ("→ Stable" if slope <= stable_band else "↑ Worsening")

def signal_emoji(slope, good_dir="up", stable_band=0.02):
    if good_dir == "up":
        return "🟢" if slope > stable_band else ("🟡" if slope >= -stable_band else "🔴")
    else:
        return "🟢" if slope < -stable_band else ("🟡" if slope <= stable_band else "🔴")

# Compute slopes (use variables already produced by chart sections)
slope_rev  = safe_slope(df["revenue"], df["revenue"].mean())
slope_gr   = safe_slope(df["growth_rate"])
slope_burn = safe_slope(df["burn"], df["burn"].mean())
slope_run  = safe_slope(df["runway"])
slope_bm   = safe_slope(bm_all)
slope_gm   = safe_slope(df["gross_margin"] * 100)

fy_rev_end = linear_forecast(df["revenue"], FORECAST_PERIODS)[0][-1]
fy_run_end = linear_forecast(df["runway"],  FORECAST_PERIODS)[0][-1]
fy_gm_end  = linear_forecast(df["gross_margin"] * 100, FORECAST_PERIODS)[0][-1]
fy_bm_end  = linear_forecast(bm_all.values, FORECAST_PERIODS)[0][-1]

# ── WRITE MARKDOWN ─────────────────────────────────────────────────────────────
lines = []
lines.append(f"# VC Financial Results")
lines.append(f"\n> Generated: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M')}  ")
lines.append(f"> Company: **{COMPANY_NAME}**  ")
lines.append(f"> Data range: {period_label(df['date'].iloc[0])} → {period_label(df['date'].iloc[-1])}  ")
lines.append(f"> Forecast window: {FORECAST_PERIODS} {PERIOD}s forward\n")

# ── SECTION 1: SIGNAL SUMMARY ─────────────────────────────────────────────────
lines.append("## 1. VC Signal Summary\n")
lines.append("| Metric | Latest Value | Trend Slope | Direction | Signal |")
lines.append("|--------|-------------|-------------|-----------|--------|")

signal_table = [
    ("Revenue (Latest Period)", f"${df['revenue'].iloc[-1]/1e6:.2f}M",    slope_rev,  "up",   0.02),
    ("Period Growth Rate",      f"{df['growth_rate'].dropna().iloc[-1]:.1f}%", slope_gr, "up", 2.0),
    ("Burn Rate",               f"${df['burn'].iloc[-1]/1e3:.0f}K/{PERIOD_LABEL}", slope_burn, "down", 0.02),
    ("Runway",                  f"{df['runway'].iloc[-1]:.1f} months",    slope_run,  "up",   1.0),
    ("Burn Multiple",           latest_bm_label,                          slope_bm,   "down", 0.10),
    ("Gross Margin",            f"{df['gross_margin'].iloc[-1]*100:.0f}%", slope_gm, "up",   1.0),
]

for name, val, slope, gdir, stable_band in signal_table:
    lines.append(f"| {name} | {val} | {slope:+.3f} | {slope_label(slope, gdir, stable_band)} | {signal_emoji(slope, gdir, stable_band)} |")

# ── SECTION 2: TIME-SERIES DATA ───────────────────────────────────────────────
lines.append(f"\n## 2. Historical Data (all {PERIOD}s)\n")
lines.append(f"| Period | Revenue ($K) | Burn ($K/{PERIOD_LABEL}) | Cash ($K) | Runway (mo) | Burn Mult | Gross Margin % |")
lines.append("|---------|-------------|----------|----------|-------------|-----------|----------------|")
for _, row in df.iterrows():
    qtr = period_label(row["date"])
    bm  = f"{row['burn_multiple']:.2f}" if not pd.isna(row["burn_multiple"]) else "—"
    lines.append(f"| {qtr} | {row['revenue']/1e3:.0f} | {row['burn']/1e3:.0f} | "
                 f"{row['cash']/1e3:.0f} | {row['runway']:.1f} | {bm} | {row['gross_margin']*100:.1f}% |")

# ── SECTION 3: FORECASTS ──────────────────────────────────────────────────────
lines.append(f"\n## 3. Linear Forecast (next {PERIOD} projections)\n")
lines.append("| Metric | Forecast End-Value | Method |")
lines.append("|--------|--------------------|--------|")
lines.append(f"| Revenue | ${fy_rev_end/1e6:.2f}M / {PERIOD_LABEL} | Linear regression |")
lines.append(f"| Runway | {fy_run_end:.1f} months | Linear regression |")
lines.append(f"| Gross Margin | {fy_gm_end:.1f}% | Linear regression |")
lines.append(f"| Burn Multiple | {fy_bm_end:.2f}× | Linear regression |")

# ── SECTION 4: VALUATION ──────────────────────────────────────────────────────
lines.append("\n## 4. Valuation Estimates\n")
lines.append(f"ARR Proxy (annualized last quarter): **${valuation_results['arr_proxy_M']:.2f}M**\n")
lines.append("| Method | Low ($M) | Mid ($M) | High ($M) | Notes |")
lines.append("|--------|---------|---------|---------|-------|")
lines.append(
    f"| DCF (Intrinsic) | ${valuation_results['dcf_low_M']} | "
    f"${valuation_results['dcf_value_M']} | ${valuation_results['dcf_high_M']} | "
    f"r={valuation_results['discount_rate_pct']:.0f}%, {PROJECTION_YEARS}yr horizon |"
)
lines.append(
    f"| Comparables (EV/ARR) | ${valuation_results['comp_value_low_M']} | "
    f"${valuation_results['comp_value_mid_M']} | ${valuation_results['comp_value_high_M']} | "
    f"{EV_ARR_RANGE[0]}–{EV_ARR_RANGE[1]}× EV/ARR multiple |"
)
lines.append(
    f"| Risk-Adjusted | ${valuation_results['risk_adj_low_M']} | "
    f"${valuation_results['risk_adj_value_M']} | ${valuation_results['risk_adj_high_M']} | "
    f"DCF × (1 − {valuation_results['failure_prob_pct']:.0f}% failure) |"
)

if valuation_results["overlap_low_M"] is not None:
    lines.append(
        f"\n**Credible range (overlap of all methods):** "
        f"${valuation_results['overlap_low_M']}M — ${valuation_results['overlap_high_M']}M"
    )

# ── SECTION 5: KEY TAKEAWAYS (auto-generated flags) ───────────────────────────
lines.append("\n## 5. Automated Flags\n")

flags = []
if df["runway"].iloc[-1] < 12:
    flags.append("🔴 **RUNWAY CRITICAL** — less than 12 months; fundraising urgency high.")
elif df["runway"].iloc[-1] < 18:
    flags.append(f"🟡 **RUNWAY WATCH** — 12–18 months; plan next round within 2 {PERIOD}s.")

if not pd.isna(latest_bm):
    if latest_bm > 2.0:
        flags.append(f"🔴 **HIGH BURN MULTIPLE** — {latest_bm:.2f}× (target <1.5×); capital efficiency needs attention.")
    elif latest_bm > 1.5:
        flags.append(f"🟡 **BURN MULTIPLE ELEVATED** — {latest_bm:.2f}× (target <1.5×).")

if df["gross_margin"].iloc[-1] < 0.50:
    flags.append(f"🔴 **LOW GROSS MARGIN** — {df['gross_margin'].iloc[-1]*100:.0f}% (SaaS floor 70%).")
elif df["gross_margin"].iloc[-1] < 0.70:
    flags.append(f"🟡 **GROSS MARGIN BELOW BENCHMARK** — {df['gross_margin'].iloc[-1]*100:.0f}% (target 70–80%).")

if slope_gr > 2.0:
    flags.append("🟢 **GROWTH ACCELERATING** — period growth rate trend is positive.")
elif slope_gr < -2.0:
    flags.append("🔴 **GROWTH DECELERATING** — period growth rate is declining.")

if not flags:
    flags.append("🟢 All monitored metrics within acceptable ranges.")

for f in flags:
    lines.append(f"- {f}")

lines.append(f"\n---\n*Auto-generated by vc-financial-analyzer skill v1.2. "
             f"Not investment advice.*\n")

# ── SAVE ──────────────────────────────────────────────────────────────────────
with open(MD_PATH, "w", encoding="utf-8") as fh:
    fh.write("\n".join(lines))
print(f"📄  Results written → {MD_PATH}")
```

> **Note for agents consuming `vc_financial_results.md`:** All numeric values are embedded
> inline in markdown tables. Parse with any markdown-aware tool or read as plain text.
> Section headers (`## 1.`, `## 2.` …) act as anchors for targeted extraction.

---

## Step 8 — Prediction Methods (Tiered by Data Availability)

| Method | When to Use | Min Periods | Python |
|---|---|---|---|
| **Linear regression** | Stable, consistent growth | 4+ | `np.polyfit` |
| **Exponential smoothing** | Seasonal or cyclical | 8+ | `statsmodels.tsa.holtwinters` |
| **ARIMA** | Long history, mature company | 20+ | `statsmodels.tsa.arima.model` |
| **Prophet** | Seasonality + limited data | 8+ | `prophet.Prophet` |

**Default for VC use:** Linear regression + ±15% confidence band.

```python
def linear_forecast(values, n_periods):
    """Returns (forecast_array, slope). Slope used for signal_row()."""
    x = np.arange(len(values))
    z = np.polyfit(x, values, 1)
    p = np.poly1d(z)
    return p(np.arange(len(values), len(values) + n_periods)), z[0]
```

---

## Step 9 — Trend Signal Quick Reference

| Metric | 🟢 Green | 🟡 Yellow | 🔴 Red |
|---|---|---|---|
| Revenue Growth (MoM) | > 15% | 5–15% | < 5% |
| Gross Margin trend | Improving | Stable | Declining |
| Burn Multiple | < 1.5x | 1.5–2.5x | > 2.5x |
| Runway | > 18 mo | 12–18 mo | < 12 mo |
| NRR | > 110% | 90–110% | < 90% |
| CAC Payback | < 12 mo | 12–24 mo | > 24 mo |
| DPI | > 1.5x | 1.0–1.5x | < 1.0x |

---

## Step 10 — Final Checklist

- [ ] `COMPANY_NAME` variable set at top of script
- [ ] Canvas `plt.rcParams.update()` set once at top, before any plot calls
- [ ] Each indicator has its own `fig, ax = plt.subplots(figsize=(12, 6))`
- [ ] Every metric shown as trend over time (no isolated single bars)
- [ ] Rolling average (window=3) on all noisy series
- [ ] Forward forecast + confidence band on all continuous metrics
- [ ] Bar charts color-coded green/orange/red by threshold
- [ ] **Valuation chart `07_valuation.png`** includes DCF, Comparables, and Risk-Adjusted bars with error bands
- [ ] **`valuation_results` dict** populated before writing the `.md` file
- [ ] Summary card `00_vc_signal_summary.png` saved last (before zip)
- [ ] All filenames prefixed `NN_` for correct directory sort order
- [ ] `save_fig()` called for every chart (adds footer, closes figure)
- [ ] `zipfile.ZipFile` wraps everything into `vc_analysis/` folder (Step 7a)
- [ ] **`vc_financial_results.md` written** with all 5 sections: signals, history, forecasts, valuation, flags (Step 7b)

---

## Metrics Glossary

### Return Metrics (Fund-Level)

**IRR** — Time-adjusted annualized return. VC target: 20–30%+. Weakness: assumes reinvestment at same rate.

**MOIC** — Total value (realized + unrealized) ÷ capital. Doesn't account for time. Strong = 3x+.

**TVPI** — DPI + RVPI. Total fund performance including unrealized. Above 2.0 = strong.

**DPI** — Cash actually returned. 1.0x = got money back. 60% of LPs now prioritize over TVPI.

**RVPI** — Unrealized portfolio value ÷ invested capital. Future potential not yet realized.

### Company-Level Metrics

**ARR / MRR** — Annual / Monthly Recurring Revenue. Gold standard for SaaS predictability.

**Burn Rate** — Monthly cash outflow. Rising burn + rising revenue = OK. Rising burn + flat revenue = danger.

**Runway** — Cash ÷ Monthly Burn. <6 months = fundraising emergency. Ideal post-round: 18–24 months.

**Burn Multiple** — Net Burn ÷ Net New ARR.
`< 1.0x` Exceptional · `1.0–1.5x` Good · `1.5–2.0x` Fair · `> 2.0x` Concerning · `> 3.0x` Unsustainable

**Gross Margin %** — (Revenue − COGS) ÷ Revenue. SaaS benchmark: 70–80%+. Improves with scale.

**CAC Payback** — Months to recover acquisition cost. Best-in-class: <12 months.

**LTV / CAC** — Above 3x = healthy unit economics. Below 1x = losing money per customer.

**NRR** — Net Revenue Retention. Above 100% = customers spend more over time. Top SaaS: 120%+.

---

## Full Demo Script (copy-paste ready)

```python
"""
vc_charts.py  —  VC Financial Analyzer v1.2: Individual Chart Exports + MD Results
Saves one PNG per indicator -> zips into vc_analysis_charts.zip
Also writes vc_financial_results.md for downstream agent handoff.

Output:
  vc_analysis/
    00_vc_signal_summary.png
    01_revenue_trend.png
    02_growth_rate.png
    03_burn_rate.png
    04_runway.png
    05_burn_multiple.png
    06_gross_margin.png
    07_valuation.png
  vc_analysis_charts.zip
  vc_financial_results.md
"""

import os, zipfile
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import matplotlib.font_manager as fm
from matplotlib.patches import Patch
import warnings
warnings.filterwarnings("ignore")

def choose_font_family():
    candidates = [
        "/System/Library/Fonts/PingFang.ttc",
        "/System/Library/Fonts/STHeiti Medium.ttc",
        "/System/Library/Fonts/Hiragino Sans GB.ttc",
        "/System/Library/Fonts/Supplemental/Arial Unicode.ttf",
    ]
    for path in candidates:
        if os.path.exists(path):
            try:
                fm.fontManager.addfont(path)
            except Exception:
                pass
    return ["PingFang TC", "STHeiti", "Hiragino Sans GB", "Arial Unicode MS", "DejaVu Sans"]

# ── PATHS & CONFIG ────────────────────────────────────────────────────────────
COMPANY_NAME = "NovaSaaS Inc."           # ← change per analysis
OUT_DIR      = "./vc_analysis"
ZIP_PATH     = "./vc_analysis_charts.zip"
MD_PATH      = "./vc_financial_results.md"
os.makedirs(OUT_DIR, exist_ok=True)

# ── TIME BASIS (must match source data) ───────────────────────────────────────
PERIOD = "quarter"  # one of: "month", "quarter", "year"
DATE_FREQ_BY_PERIOD = {"month": "MS", "quarter": "QS", "year": "YS"}
PERIODS_PER_YEAR_BY_PERIOD = {"month": 12, "quarter": 4, "year": 1}
PERIOD_LABEL_BY_PERIOD = {"month": "month", "quarter": "qtr", "year": "yr"}
if PERIOD not in PERIODS_PER_YEAR_BY_PERIOD:
    raise ValueError("PERIOD must be one of: month, quarter, year")
DATE_FREQ = DATE_FREQ_BY_PERIOD[PERIOD]
PERIODS_PER_YEAR = PERIODS_PER_YEAR_BY_PERIOD[PERIOD]
MONTHS_PER_PERIOD = 12 / PERIODS_PER_YEAR
PERIOD_LABEL = PERIOD_LABEL_BY_PERIOD[PERIOD]

def annualize(values):
    return values * PERIODS_PER_YEAR

def period_label(ts):
    ts = pd.Timestamp(ts)
    if PERIOD == "year":
        return f"{ts.year}"
    if PERIOD == "quarter":
        return f"{ts.year}-Q{(ts.month - 1)//3 + 1}"
    return ts.strftime("%Y-%m")

# ── CANVAS CONFIG ─────────────────────────────────────────────────────────────
plt.rcParams.update({
    "figure.facecolor":  "#0f0f14",
    "axes.facecolor":    "#16161e",
    "axes.edgecolor":    "#2a2a3a",
    "axes.labelcolor":   "#c0c0d0",
    "text.color":        "#e0e0f0",
    "xtick.color":       "#888899",
    "ytick.color":       "#888899",
    "grid.color":        "#2a2a3a",
    "grid.linestyle":    "--",
    "grid.alpha":        0.5,
    "lines.linewidth":   2.2,
    "font.family":       choose_font_family(),
    "axes.unicode_minus": False,
    "font.size":         10,
    "axes.titlesize":    13,
    "axes.titleweight":  "bold",
    "legend.framealpha": 0.15,
    "legend.edgecolor":  "#3a3a5a",
    "legend.fontsize":   9,
})
COLORS = {
    "primary":   "#7b6ef6",
    "positive":  "#4ade80",
    "warning":   "#fb923c",
    "danger":    "#f87171",
    "neutral":   "#94a3b8",
    "forecast":  "#38bdf8",
    "accent":    "#f472b6",
}
FOOTER = "⚠  Forecast is directional only (linear regression). Not investment advice."

def save_fig(fig, filename):
    path = os.path.join(OUT_DIR, filename)
    fig.text(0.5, -0.03, FOOTER, ha="center", fontsize=8, color="#555570")
    fig.savefig(path, dpi=150, bbox_inches="tight", facecolor=fig.get_facecolor())
    plt.close(fig)
    print(f"  ✓  {filename}")
    return path

def add_derived_metrics(df):
    required = {"date", "revenue", "burn", "cash", "gross_margin"}
    missing = required - set(df.columns)
    if missing:
        raise ValueError(f"Missing required columns: {sorted(missing)}")

    df = df.copy()
    df["date"] = pd.to_datetime(df["date"])
    df = df.sort_values("date").reset_index(drop=True)
    df["monthly_burn"] = df["burn"] / MONTHS_PER_PERIOD
    df["runway"] = np.where(df["monthly_burn"] > 0, df["cash"] / df["monthly_burn"], np.nan)
    df["annualized_revenue"] = annualize(df["revenue"])
    df["net_new_arr"] = df["annualized_revenue"].diff()
    df["burn_multiple"] = np.where(df["net_new_arr"] > 0, df["burn"] / df["net_new_arr"], np.nan)
    df["revenue_ma"] = df["revenue"].rolling(3, min_periods=1).mean()
    df["growth_rate"] = df["revenue"].pct_change() * 100
    df["growth_ma"] = df["growth_rate"].rolling(3, min_periods=1).mean()
    return df

def linear_forecast(values, n_periods):
    values = pd.Series(values).dropna().astype(float).to_numpy()
    if len(values) == 0:
        return np.full(n_periods, np.nan), 0.0
    if len(values) == 1:
        return np.repeat(values[-1], n_periods), 0.0
    x = np.arange(len(values))
    z = np.polyfit(x, values, 1)
    p = np.poly1d(z)
    return p(np.arange(len(values), len(values) + n_periods)), z[0]

def safe_slope(values, normalize_by=None):
    values = pd.Series(values).dropna().astype(float).to_numpy()
    if len(values) < 2:
        return 0.0
    slope = np.polyfit(np.arange(len(values)), values, 1)[0]
    if normalize_by is not None and normalize_by != 0:
        slope = slope / normalize_by
    return slope

# ── SYNTHETIC DATA (replace with real df for actual company) ──────────────────
np.random.seed(42)
n        = 12
quarters = pd.date_range(start="2022-01-01", periods=n, freq=DATE_FREQ)
rev_base     = [200_000 * (1.18 ** i) for i in range(n)]
revenue      = np.array([r * np.random.uniform(0.92, 1.08) for r in rev_base])
burn_base    = [320_000 - (i * 8_000) for i in range(n)]
burn         = np.clip([b * np.random.uniform(0.95, 1.05) for b in burn_base], 80_000, 400_000)
gross_margin = np.linspace(0.52, 0.74, n) + np.random.uniform(-0.02, 0.02, n)
cash, c = [], 3_000_000
for i in range(n):
    if i == 5: c += 6_000_000
    c = c - burn[i] + revenue[i]
    cash.append(max(c, 0))
cash = np.array(cash)

df = pd.DataFrame({
    "date":          quarters,
    "revenue":       revenue,
    "burn":          burn,
    "cash":          cash,
    "gross_margin":  gross_margin,
})
df = add_derived_metrics(df)

FORECAST_PERIODS = 4
f_dates = pd.date_range(df["date"].iloc[-1], periods=FORECAST_PERIODS + 1, freq=DATE_FREQ)[1:]

saved_files = []
print("\n📊  Generating charts...\n")

# ── CHART 1: Revenue Trend ────────────────────────────────────────────────────
fy_rev, slope_rev = linear_forecast(df["revenue"], FORECAST_PERIODS)
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Revenue Trend + Forecast", fontsize=14,
             fontweight="bold", color="#e0e0f0")
ax.plot(df["date"], df["revenue"]/1e6,    color=COLORS["neutral"], alpha=0.45, linewidth=1.5, label="Actual Revenue")
ax.plot(df["date"], df["revenue_ma"]/1e6, color=COLORS["primary"], linewidth=2.8, label="3-period rolling avg")
ax.plot(f_dates, fy_rev/1e6, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Linear Forecast")
ax.fill_between(f_dates, fy_rev/1e6*0.82, fy_rev/1e6*1.18, color=COLORS["forecast"], alpha=0.12, label="±18% Band")
ax.axvline(df["date"].iloc[5], color=COLORS["accent"], linewidth=1.4, linestyle=":", alpha=0.8)
ax.text(df["date"].iloc[5], df["revenue"].max()/1e6*0.42, "  Series A\n  $6M raised",
        color=COLORS["accent"], fontsize=9)
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.1fM"))
ax.set_ylabel("Revenue ($M)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper left"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "01_revenue_trend.png"))

# ── CHART 2: Growth Rate ──────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Period Revenue Growth Rate (%)", fontsize=14,
             fontweight="bold", color="#e0e0f0")
valid = df["growth_rate"].notna()
ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] > 0), color=COLORS["positive"], alpha=0.2, label="Positive Zone")
ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] < 0), color=COLORS["danger"], alpha=0.2, label="Negative Zone")
ax.plot(df["date"][valid], df["growth_rate"][valid], color=COLORS["neutral"], alpha=0.5, linewidth=1.5, label="Raw period growth")
ax.plot(df["date"][valid], df["growth_ma"][valid],   color=COLORS["primary"], linewidth=2.8, label="3-Period Avg")
ax.axhline(0,  color="#444455", linewidth=1.0)
ax.axhline(15, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="15% VC Benchmark")
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylabel("Growth Rate (%)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "02_growth_rate.png"))

# ── CHART 3: Burn Rate ────────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Burn Rate by Period", fontsize=14,
             fontweight="bold", color="#e0e0f0")
bar_colors = [COLORS["positive"] if r > 18 else COLORS["warning"] if r > 12
              else COLORS["danger"] for r in df["runway"]]
ax.bar(df["date"], df["burn"]/1e3, color=bar_colors, width=55, alpha=0.85)
fy_burn, _ = linear_forecast(df["burn"]/1e3, FORECAST_PERIODS)
ax.plot(df["date"], df["burn"]/1e3, color=COLORS["neutral"], linewidth=1.5, linestyle=":", alpha=0.6)
ax.plot(f_dates, fy_burn, color=COLORS["forecast"], linewidth=2, linestyle="--")
legend_els = [
    Patch(facecolor=COLORS["positive"], label="Runway > 18 mo (Safe)"),
    Patch(facecolor=COLORS["warning"],  label="Runway 12–18 mo (Watch)"),
    Patch(facecolor=COLORS["danger"],   label="Runway < 12 mo (Risk)"),
    plt.Line2D([0],[0], color=COLORS["forecast"], linewidth=2, linestyle="--", label="Burn Forecast"),
]
ax.legend(handles=legend_els, loc="upper right")
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.0fK"))
ax.set_ylabel(f"Burn Rate ($K/{PERIOD_LABEL})"); ax.set_xlabel(PERIOD.title())
ax.grid(True, axis="y"); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "03_burn_rate.png"))

# ── CHART 4: Runway ───────────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Cash Runway (months)", fontsize=14,
             fontweight="bold", color="#e0e0f0")
fy_run, slope_run = linear_forecast(df["runway"], FORECAST_PERIODS)
ax.plot(df["date"], df["runway"], color=COLORS["primary"], linewidth=2.8,
        marker="o", markersize=6, label="Runway (months)")
ax.plot(f_dates, fy_run, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_run*0.85, fy_run*1.15, color=COLORS["forecast"], alpha=0.1)
ax.axhline(24, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="24 Mo — Ideal")
ax.axhline(18, color=COLORS["warning"],  linewidth=1.2, linestyle=":", alpha=0.7, label="18 Mo — VC target")
ax.axhline(12, color=COLORS["danger"],   linewidth=1.2, linestyle=":", alpha=0.7, label="12 Mo — Warning")
ax.axhspan(0,  12, color=COLORS["danger"],  alpha=0.04)
ax.axhspan(12, 18, color=COLORS["warning"], alpha=0.04)
ax.set_ylabel("Runway (months)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper left"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "04_runway.png"))

# ── CHART 5: Burn Multiple ────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Burn Multiple (Capital Efficiency)", fontsize=14,
             fontweight="bold", color="#e0e0f0")
bm_valid = df["burn_multiple"].notna()
bm_vals  = df["burn_multiple"][bm_valid]
bm_dates = df["date"][bm_valid]
fy_bm, slope_bm = linear_forecast(bm_vals.values, FORECAST_PERIODS)
ax.plot(bm_dates, bm_vals, color=COLORS["warning"], linewidth=2.8, marker="o", markersize=6, label="Burn Multiple")
ax.plot(f_dates, fy_bm, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_bm*0.85, fy_bm*1.15, color=COLORS["forecast"], alpha=0.1)
ax.axhline(1.0, color=COLORS["positive"], linewidth=1.4, linestyle="--", alpha=0.8, label="< 1x Exceptional")
ax.axhline(1.5, color=COLORS["warning"],  linewidth=1.4, linestyle="--", alpha=0.8, label="1.5x Good")
ax.axhline(2.0, color=COLORS["danger"],   linewidth=1.4, linestyle="--", alpha=0.8, label="2x Concerning")
ax.axhspan(0,   1.0, color=COLORS["positive"], alpha=0.04)
ax.axhspan(1.0, 1.5, color=COLORS["warning"],  alpha=0.04)
ax.axhspan(1.5, 4.0, color=COLORS["danger"],   alpha=0.04)
ax.fill_between(bm_dates, bm_vals, 2.0, where=(bm_vals < 2.0), color=COLORS["positive"], alpha=0.08)
ax.set_ylim(0, 4)
ax.set_ylabel("Burn Multiple (×)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="upper right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "05_burn_multiple.png"))

# ── CHART 6: Gross Margin ─────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Gross Margin % Trend", fontsize=14,
             fontweight="bold", color="#e0e0f0")
fy_gm, slope_gm = linear_forecast(df["gross_margin"]*100, FORECAST_PERIODS)
gm_ma = (df["gross_margin"]*100).rolling(3, min_periods=1).mean()
ax.plot(df["date"], df["gross_margin"]*100, color=COLORS["neutral"], alpha=0.5, linewidth=1.5, label="Actual")
ax.plot(df["date"], gm_ma, color=COLORS["accent"], linewidth=2.8, label="3-period rolling avg")
ax.plot(f_dates, fy_gm, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_gm-3, fy_gm+3, color=COLORS["forecast"], alpha=0.1, label="±3pp Band")
ax.axhline(80, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="80% Top SaaS")
ax.axhline(70, color=COLORS["warning"],  linewidth=1.2, linestyle=":", alpha=0.7, label="70% SaaS min")
ax.axhline(50, color=COLORS["danger"],   linewidth=1.2, linestyle=":", alpha=0.7, label="50% Below standard")
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylim(30, 100)
ax.set_ylabel("Gross Margin (%)"); ax.set_xlabel(PERIOD.title())
ax.legend(loc="lower right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "06_gross_margin.png"))

# ── CHART 0: VC Signal Summary ────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(10, 6))
fig.patch.set_facecolor("#0f0f14")
ax.set_facecolor("#0f0f14"); ax.axis("off")
fig.suptitle("NovaSaaS Inc.  ·  VC Signal Summary", fontsize=14,
             fontweight="bold", color="#e0e0f0")

def signal_row(name, current, slope, good_dir="up", stable_band=0.02):
    if good_dir == "up":
        color = COLORS["positive"] if slope > stable_band else (COLORS["warning"] if slope >= -stable_band else COLORS["danger"])
        emoji = "●" if slope > stable_band else ("◑" if slope >= -stable_band else "○")
    else:
        color = COLORS["positive"] if slope < -stable_band else (COLORS["warning"] if slope <= stable_band else COLORS["danger"])
        emoji = "●" if slope < -stable_band else ("◑" if slope <= stable_band else "○")
    return name, current, f"{slope:+.2f}", color, emoji

bm_all = df["burn_multiple"].dropna()
latest_bm = bm_all.iloc[-1] if not bm_all.empty else np.nan
latest_bm_label = f"{latest_bm:.2f}×" if not pd.isna(latest_bm) else "n/a"
rows = [
    signal_row("Revenue Growth",  f"${df['revenue'].iloc[-1]/1e6:.2f}M / {PERIOD_LABEL}",
               safe_slope(df["revenue"], df["revenue"].mean()), "up", 0.02),
    signal_row("Period Growth Rate", f"{df['growth_rate'].dropna().iloc[-1]:.1f}%",
               safe_slope(df["growth_rate"]), "up", 2.0),
    signal_row("Burn Rate",       f"${df['burn'].iloc[-1]/1e3:.0f}K / {PERIOD_LABEL}",
               safe_slope(df["burn"], df["burn"].mean()), "down", 0.02),
    signal_row("Runway",          f"{df['runway'].iloc[-1]:.1f} months",
               safe_slope(df["runway"]), "up", 1.0),
    signal_row("Burn Multiple",   latest_bm_label,
               safe_slope(bm_all), "down", 0.10),
    signal_row("Gross Margin",    f"{df['gross_margin'].iloc[-1]*100:.0f}%",
               safe_slope(df["gross_margin"] * 100), "up", 1.0),
]

header_y = 0.88
for label, x in [("Indicator",0.04),("Latest",0.42),("Trend Slope",0.62),("Signal",0.80)]:
    ax.text(x, header_y, label, fontsize=10, color="#888899",
            fontweight="bold", transform=ax.transAxes)
ax.plot([0.04,0.96],[header_y-0.04,header_y-0.04], color="#2a2a3a",
        linewidth=1, transform=ax.transAxes)

for i, (name, current, slope, color, emoji) in enumerate(rows):
    y = 0.76 - i * 0.12
    ax.text(0.04, y, name,    fontsize=11, color="#e0e0f0", transform=ax.transAxes)
    ax.text(0.42, y, current, fontsize=11, color="#c0c0d0", transform=ax.transAxes)
    ax.text(0.62, y, slope,   fontsize=11, color=color,     transform=ax.transAxes)
    ax.text(0.80, y, emoji,   fontsize=18, color=color,     transform=ax.transAxes)

note = (f"Forecast Rev (end): ${linear_forecast(df['revenue'],FORECAST_PERIODS)[0][-1]/1e6:.2f}M  ·  "
        f"3-period Avg Growth: {df['growth_ma'].iloc[-1]:.1f}%  ·  Latest period growth: {df['growth_rate'].iloc[-1]:.1f}%")
ax.text(0.04, 0.04, note, fontsize=8.5, color="#555570", transform=ax.transAxes)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "00_vc_signal_summary.png"))

# ── CHART 7: VALUATION ────────────────────────────────────────────────────────
DISCOUNT_RATE       = 0.25
TERMINAL_GROWTH     = 0.03
PROJECTION_YEARS    = 5
FCF_MARGIN          = 0.15
EV_ARR_MULTIPLE     = 8.0
EV_ARR_RANGE        = (5.0, 12.0)
FAILURE_PROBABILITY = 0.40

ann_rev    = df["annualized_revenue"].iloc[-1]
avg_period_growth = df["growth_rate"].dropna().mean() / 100
annual_growth = (1 + avg_period_growth) ** PERIODS_PER_YEAR - 1
proj_revs  = [ann_rev * ((1 + annual_growth) ** yr) for yr in range(1, PROJECTION_YEARS + 1)]
proj_fcfs  = [r * FCF_MARGIN for r in proj_revs]
terminal_v = proj_fcfs[-1] * (1 + TERMINAL_GROWTH) / (DISCOUNT_RATE - TERMINAL_GROWTH)
dcf_value  = sum(f / (1+DISCOUNT_RATE)**t for t,f in enumerate(proj_fcfs,1))
dcf_value += terminal_v / (1+DISCOUNT_RATE)**PROJECTION_YEARS

comp_value_mid  = ann_rev * EV_ARR_MULTIPLE
comp_value_low  = ann_rev * EV_ARR_RANGE[0]
comp_value_high = ann_rev * EV_ARR_RANGE[1]
risk_adj_value  = dcf_value * (1 - FAILURE_PROBABILITY)
risk_adj_low    = risk_adj_value * 0.70
risk_adj_high   = risk_adj_value * 1.30

fig, ax = plt.subplots(figsize=(12, 7))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle(f"{COMPANY_NAME}  ·  Valuation: DCF vs Comparables vs Risk-Adjusted",
             fontsize=14, fontweight="bold", color="#e0e0f0")

labels    = ["DCF\n(Intrinsic)", "Comparables\n(EV/ARR)", "Risk-Adjusted\n(DCF × Survival)"]
midpoints = [dcf_value, comp_value_mid, risk_adj_value]
lows      = [dcf_value*0.75, comp_value_low,  risk_adj_low]
highs     = [dcf_value*1.25, comp_value_high, risk_adj_high]
bcolors   = [COLORS["primary"], COLORS["forecast"], COLORS["warning"]]

x = np.arange(len(labels))
bars = ax.bar(x, [m/1e6 for m in midpoints], color=bcolors, width=0.5, alpha=0.85, zorder=3)
ax.errorbar(x, [m/1e6 for m in midpoints],
            yerr=[[(m-l)/1e6 for m,l in zip(midpoints,lows)],
                  [(h-m)/1e6 for m,h in zip(midpoints,highs)]],
            fmt="none", color="#e0e0f0", capsize=10, capthick=2, linewidth=2, zorder=4)
for bar, mid in zip(bars, midpoints):
    ax.text(bar.get_x()+bar.get_width()/2, bar.get_height()+max(midpoints)*0.01/1e6,
            f"${mid/1e6:.1f}M", ha="center", va="bottom", fontsize=12,
            fontweight="bold", color="#e0e0f0")

overlap_low  = max(lows)  / 1e6
overlap_high = min(highs) / 1e6
if overlap_low < overlap_high:
    ax.axhspan(overlap_low, overlap_high, color=COLORS["positive"], alpha=0.08,
               label=f"Credible range ${overlap_low:.1f}M–${overlap_high:.1f}M")
    ax.axhline(overlap_low,  color=COLORS["positive"], linewidth=1.2, linestyle="--", alpha=0.6)
    ax.axhline(overlap_high, color=COLORS["positive"], linewidth=1.2, linestyle="--", alpha=0.6)

ax.text(0.5, -0.10,
        f"r={DISCOUNT_RATE*100:.0f}%  |  EV/ARR={EV_ARR_MULTIPLE:.1f}×  |  "
        f"Failure prob={FAILURE_PROBABILITY*100:.0f}%  |  ARR proxy=${ann_rev/1e6:.2f}M",
        transform=ax.transAxes, ha="center", fontsize=8.5, color="#888899")
ax.set_xticks(x); ax.set_xticklabels(labels, fontsize=11)
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.0fM"))
ax.set_ylabel("Estimated Valuation ($M)")
ax.legend(loc="upper right"); ax.grid(True, axis="y", alpha=0.4); ax.set_axisbelow(True)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "07_valuation.png"))

valuation_results = {
    "dcf_value_M":       round(dcf_value/1e6, 2),
    "dcf_low_M":         round(dcf_value*0.75/1e6, 2),
    "dcf_high_M":        round(dcf_value*1.25/1e6, 2),
    "comp_value_mid_M":  round(comp_value_mid/1e6, 2),
    "comp_value_low_M":  round(comp_value_low/1e6, 2),
    "comp_value_high_M": round(comp_value_high/1e6, 2),
    "risk_adj_value_M":  round(risk_adj_value/1e6, 2),
    "risk_adj_low_M":    round(risk_adj_low/1e6, 2),
    "risk_adj_high_M":   round(risk_adj_high/1e6, 2),
    "overlap_low_M":     round(overlap_low, 2) if overlap_low < overlap_high else None,
    "overlap_high_M":    round(overlap_high, 2) if overlap_low < overlap_high else None,
    "arr_proxy_M":       round(ann_rev/1e6, 2),
    "discount_rate_pct": DISCOUNT_RATE*100,
    "ev_arr_multiple":   EV_ARR_MULTIPLE,
    "failure_prob_pct":  FAILURE_PROBABILITY*100,
}

# ── ZIP ───────────────────────────────────────────────────────────────────────
print(f"\n📦  Zipping {len(saved_files)} charts...")
with zipfile.ZipFile(ZIP_PATH, "w", zipfile.ZIP_DEFLATED) as zf:
    for fpath in sorted(saved_files):
        arcname = os.path.join("vc_analysis", os.path.basename(fpath))
        zf.write(fpath, arcname)
print(f"✅  Done → {ZIP_PATH}")
with zipfile.ZipFile(ZIP_PATH, "r") as zf:
    for name in sorted(zf.namelist()):
        print(f"  {name}  ({zf.getinfo(name).file_size//1024} KB)")

# ── WRITE vc_financial_results.md ─────────────────────────────────────────────
import datetime

def _slope_label(slope, good_dir="up", stable_band=0.02):
    if good_dir == "up":
        return "↑ Positive" if slope > stable_band else ("→ Stable" if slope >= -stable_band else "↓ Declining")
    return "↓ Improving" if slope < -stable_band else ("→ Stable" if slope <= stable_band else "↑ Worsening")

def _sig_emoji(slope, good_dir="up", stable_band=0.02):
    if good_dir == "up":
        return "🟢" if slope > stable_band else ("🟡" if slope >= -stable_band else "🔴")
    return "🟢" if slope < -stable_band else ("🟡" if slope <= stable_band else "🔴")

bm_all   = df["burn_multiple"].dropna()
latest_bm = bm_all.iloc[-1] if not bm_all.empty else np.nan
latest_bm_label = f"{latest_bm:.2f}×" if not pd.isna(latest_bm) else "n/a"
s_rev    = safe_slope(df["revenue"], df["revenue"].mean())
s_gr     = safe_slope(df["growth_rate"])
s_burn   = safe_slope(df["burn"], df["burn"].mean())
s_run    = safe_slope(df["runway"])
s_bm     = safe_slope(bm_all)
s_gm     = safe_slope(df["gross_margin"]*100)

fy_rev_e = linear_forecast(df["revenue"], FORECAST_PERIODS)[0][-1]
fy_run_e = linear_forecast(df["runway"],  FORECAST_PERIODS)[0][-1]
fy_gm_e  = linear_forecast(df["gross_margin"]*100, FORECAST_PERIODS)[0][-1]
fy_bm_e  = linear_forecast(bm_all.values, FORECAST_PERIODS)[0][-1]

lines = []
lines.append("# VC Financial Results")
lines.append(f"\n> Generated: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M')}  ")
lines.append(f"> Company: **{COMPANY_NAME}**  ")
start_q = df['date'].iloc[0];  end_q = df['date'].iloc[-1]
lines.append(f"> Data range: {period_label(start_q)} → {period_label(end_q)}  ")
lines.append(f"> Forecast window: {FORECAST_PERIODS} {PERIOD}s forward\n")

lines.append("## 1. VC Signal Summary\n")
lines.append("| Metric | Latest Value | Trend Slope | Direction | Signal |")
lines.append("|--------|-------------|-------------|-----------|--------|")
for name, val, slope, gdir, stable_band in [
    ("Revenue (Latest Period)", f"${df['revenue'].iloc[-1]/1e6:.2f}M",    s_rev,  "up",   0.02),
    ("Period Growth Rate",      f"{df['growth_rate'].dropna().iloc[-1]:.1f}%", s_gr, "up", 2.0),
    ("Burn Rate",               f"${df['burn'].iloc[-1]/1e3:.0f}K/{PERIOD_LABEL}", s_burn, "down", 0.02),
    ("Runway",                  f"{df['runway'].iloc[-1]:.1f} months",    s_run,  "up",   1.0),
    ("Burn Multiple",           latest_bm_label,                          s_bm,   "down", 0.10),
    ("Gross Margin",            f"{df['gross_margin'].iloc[-1]*100:.0f}%", s_gm,  "up",   1.0),
]:
    lines.append(f"| {name} | {val} | {slope:+.3f} | {_slope_label(slope,gdir,stable_band)} | {_sig_emoji(slope,gdir,stable_band)} |")

lines.append(f"\n## 2. Historical Data (all {PERIOD}s)\n")
lines.append(f"| Period | Revenue ($K) | Burn ($K/{PERIOD_LABEL}) | Cash ($K) | Runway (mo) | Burn Mult | Gross Margin % |")
lines.append("|---------|-------------|----------|----------|-------------|-----------|----------------|")
for _, row in df.iterrows():
    qtr = period_label(row["date"])
    bm  = f"{row['burn_multiple']:.2f}" if not pd.isna(row["burn_multiple"]) else "—"
    lines.append(f"| {qtr} | {row['revenue']/1e3:.0f} | {row['burn']/1e3:.0f} | "
                 f"{row['cash']/1e3:.0f} | {row['runway']:.1f} | {bm} | {row['gross_margin']*100:.1f}% |")

lines.append("\n## 3. Linear Forecast (end of projection window)\n")
lines.append("| Metric | Forecast End-Value | Method |")
lines.append("|--------|--------------------|--------|")
lines.append(f"| Revenue | ${fy_rev_e/1e6:.2f}M / {PERIOD_LABEL} | Linear regression |")
lines.append(f"| Runway | {fy_run_e:.1f} months | Linear regression |")
lines.append(f"| Gross Margin | {fy_gm_e:.1f}% | Linear regression |")
lines.append(f"| Burn Multiple | {fy_bm_e:.2f}× | Linear regression |")

lines.append("\n## 4. Valuation Estimates\n")
lines.append(f"ARR Proxy (annualized last quarter): **${valuation_results['arr_proxy_M']:.2f}M**\n")
lines.append("| Method | Low ($M) | Mid ($M) | High ($M) | Notes |")
lines.append("|--------|---------|---------|---------|-------|")
lines.append(f"| DCF (Intrinsic) | ${valuation_results['dcf_low_M']} | "
             f"${valuation_results['dcf_value_M']} | ${valuation_results['dcf_high_M']} | "
             f"r={valuation_results['discount_rate_pct']:.0f}%, {PROJECTION_YEARS}yr |")
lines.append(f"| Comparables (EV/ARR) | ${valuation_results['comp_value_low_M']} | "
             f"${valuation_results['comp_value_mid_M']} | ${valuation_results['comp_value_high_M']} | "
             f"{EV_ARR_RANGE[0]}–{EV_ARR_RANGE[1]}× multiple |")
lines.append(f"| Risk-Adjusted | ${valuation_results['risk_adj_low_M']} | "
             f"${valuation_results['risk_adj_value_M']} | ${valuation_results['risk_adj_high_M']} | "
             f"DCF × (1−{valuation_results['failure_prob_pct']:.0f}%) |")
if valuation_results["overlap_low_M"] is not None:
    lines.append(f"\n**Credible range:** ${valuation_results['overlap_low_M']}M — "
                 f"${valuation_results['overlap_high_M']}M")

lines.append("\n## 5. Automated Flags\n")
flags = []
if df["runway"].iloc[-1] < 12:
    flags.append("🔴 **RUNWAY CRITICAL** — less than 12 months; fundraising urgency high.")
elif df["runway"].iloc[-1] < 18:
    flags.append(f"🟡 **RUNWAY WATCH** — 12–18 months; plan next round within 2 {PERIOD}s.")
if not pd.isna(latest_bm):
    if latest_bm > 2.0:
        flags.append(f"🔴 **HIGH BURN MULTIPLE** — {latest_bm:.2f}× (target <1.5×).")
    elif latest_bm > 1.5:
        flags.append(f"🟡 **BURN MULTIPLE ELEVATED** — {latest_bm:.2f}× (target <1.5×).")
if df["gross_margin"].iloc[-1] < 0.50:
    flags.append(f"🔴 **LOW GROSS MARGIN** — {df['gross_margin'].iloc[-1]*100:.0f}% (SaaS floor 70%).")
elif df["gross_margin"].iloc[-1] < 0.70:
    flags.append(f"🟡 **GROSS MARGIN BELOW BENCHMARK** — {df['gross_margin'].iloc[-1]*100:.0f}%.")
if s_gr > 2.0:
    flags.append("🟢 **GROWTH ACCELERATING** — period growth rate trend is positive.")
elif s_gr < -2.0:
    flags.append("🔴 **GROWTH DECELERATING** — period growth rate is declining.")
if not flags:
    flags.append("🟢 All monitored metrics within acceptable ranges.")
for f in flags:
    lines.append(f"- {f}")

lines.append(f"\n---\n*Auto-generated by vc-financial-analyzer v1.2. Not investment advice.*\n")

with open(MD_PATH, "w", encoding="utf-8") as fh:
    fh.write("\n".join(lines))
print(f"📄  Results written → {MD_PATH}")
```

---

## Iteration Log

| Version | Date | Change |
|---|---|---|
| v1.0 | 2026-05-17 | Initial skill — 4-panel dashboard, linear forecast, VC signal layer |
| v1.1 | 2026-05-17 | **Output format changed** — one PNG per indicator named by metric, zipped for download. No dashboard. Added `00_vc_signal_summary.png` scorecard. |
| v1.2 | 2026-05-17 | **Valuation module added** — `07_valuation.png` with DCF, Comparables (EV/ARR), Risk-Adjusted bars + uncertainty bands. **`vc_financial_results.md` writer added** — structured markdown handoff for downstream agents (5 sections: signals, history, forecasts, valuation, auto-flags). `COMPANY_NAME` variable. |
| v1.x | — | *(your next iteration here)* |

---

*End of skill document. Edit freely — this is your living reference.*
