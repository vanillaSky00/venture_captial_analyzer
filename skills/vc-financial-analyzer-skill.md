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

## Output Convention (v1.1+)

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
```

**Why numbered filenames?** File explorers sort alphabetically — the `NN_` prefix keeps the
summary card first and the metrics in logical VC evaluation order (survival → efficiency).

---

## Step 1 — Clarify the Analysis Target

Before writing any code, ask:

1. **What data do you have?** (CSV upload, manual table, public filings, API like yfinance?)
2. **What stage is the company?** (Pre-revenue, growth, late-stage / public)
3. **Time horizon?** (Quarterly for private; daily/weekly for public stocks)
4. **Prediction window?** (e.g., forecast next 3–6 quarters)

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
updates). Always accept a minimal CSV with columns: `date, revenue, burn, cash`.

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

## Step 4 — Canvas & Chart Configuration

**Set the canvas once, globally, before any `plt.plot()` calls.**

```python
import os, zipfile
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
from matplotlib.patches import Patch
import pandas as pd
import numpy as np

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
    "font.family":       "DejaVu Sans",
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
ax.plot(df["date"], df["revenue_ma"] / 1e6, color=COLORS["primary"], linewidth=2.8, label="3-Qtr Rolling Avg")
ax.plot(f_dates, fy_rev / 1e6, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Linear Forecast")
ax.fill_between(f_dates, fy_rev/1e6 * 0.82, fy_rev/1e6 * 1.18,
                color=COLORS["forecast"], alpha=0.12, label="±18% Band")

ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.1fM"))
ax.set_ylabel("Revenue ($M)"); ax.set_xlabel("Quarter")
ax.legend(loc="upper left"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "01_revenue_trend.png"))
```

### 6b. `02_growth_rate.png` — QoQ Growth + Momentum Zones

```python
df["growth_rate"] = df["revenue"].pct_change() * 100
df["growth_ma"]   = df["growth_rate"].rolling(3, min_periods=1).mean()
valid = df["growth_rate"].notna()

fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("Company  ·  QoQ Revenue Growth Rate (%)", fontsize=14, fontweight="bold", color="#e0e0f0")

ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] > 0), color=COLORS["positive"], alpha=0.2)
ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] < 0), color=COLORS["danger"], alpha=0.2)
ax.plot(df["date"][valid], df["growth_rate"][valid], color=COLORS["neutral"], alpha=0.5, linewidth=1.5)
ax.plot(df["date"][valid], df["growth_ma"][valid],   color=COLORS["primary"], linewidth=2.8, label="3-Qtr Avg")
ax.axhline(0,  color="#444455", linewidth=1.0)
ax.axhline(15, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="15% VC Benchmark")

ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylabel("Growth Rate (%)"); ax.set_xlabel("Quarter")
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
fig.suptitle("Company  ·  Burn Rate by Quarter", fontsize=14, fontweight="bold", color="#e0e0f0")

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
ax.set_ylabel("Burn Rate ($K/qtr)"); ax.set_xlabel("Quarter")
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

ax.set_ylabel("Runway (months)"); ax.set_xlabel("Quarter")
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
ax.set_ylabel("Burn Multiple (×)"); ax.set_xlabel("Quarter")
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
ax.plot(df["date"], gm_ma, color=COLORS["accent"], linewidth=2.8, label="3-Qtr Rolling Avg")
ax.plot(f_dates, fy_gm, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_gm - 3, fy_gm + 3, color=COLORS["forecast"], alpha=0.1, label="±3pp Band")

ax.axhline(80, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="80% Top SaaS")
ax.axhline(70, color=COLORS["warning"],  linewidth=1.2, linestyle=":", alpha=0.7, label="70% SaaS min")
ax.axhline(50, color=COLORS["danger"],   linewidth=1.2, linestyle=":", alpha=0.7, label="50% Below standard")

ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylim(30, 100)
ax.set_ylabel("Gross Margin (%)"); ax.set_xlabel("Quarter")
ax.legend(loc="lower right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "06_gross_margin.png"))
```

### 6g. `00_vc_signal_summary.png` — Text Scorecard (open this first)

```python
fig, ax = plt.subplots(figsize=(10, 6))
fig.patch.set_facecolor("#0f0f14")
ax.set_facecolor("#0f0f14"); ax.axis("off")
fig.suptitle("Company  ·  VC Signal Summary", fontsize=14, fontweight="bold", color="#e0e0f0")

def signal_row(name, current, slope, good_dir="up"):
    if good_dir == "up":
        color = COLORS["positive"] if slope > 0.05 else (COLORS["warning"] if slope > -0.02 else COLORS["danger"])
        emoji = "●" if slope > 0.05 else ("◑" if slope > -0.02 else "○")
    else:
        color = COLORS["positive"] if slope < -0.02 else (COLORS["warning"] if slope < 0.05 else COLORS["danger"])
        emoji = "●" if slope < -0.02 else ("◑" if slope < 0.05 else "○")
    return name, current, f"{slope:+.2f}", color, emoji

# Build one signal_row() call per metric
rows = [
    signal_row("Revenue Growth",  "display_value", slope_rev,  "up"),
    signal_row("QoQ Growth Rate", "display_value", slope_gr,   "up"),
    signal_row("Burn Rate",       "display_value", slope_burn, "down"),
    signal_row("Runway",          "display_value", slope_run,  "up"),
    signal_row("Burn Multiple",   "display_value", slope_bm,   "down"),
    signal_row("Gross Margin",    "display_value", slope_gm,   "up"),
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

## Step 7 — Zip All Charts (always last)

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

## Step 8 — Prediction Methods (Tiered by Data Availability)

> **Rule:** Use the simplest model that answers the question.

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

- [ ] Canvas `plt.rcParams.update()` set once at top, before any plot calls
- [ ] Each indicator has its own `fig, ax = plt.subplots(figsize=(12, 6))`
- [ ] Every metric shown as trend over time (no isolated single bars)
- [ ] Rolling average (window=3) on all noisy series
- [ ] Forward forecast + confidence band on all continuous metrics
- [ ] Bar charts color-coded green/orange/red by threshold
- [ ] Summary card `00_vc_signal_summary.png` saved last
- [ ] All filenames prefixed `NN_` for correct directory sort order
- [ ] `save_fig()` called for every chart (adds footer, closes figure)
- [ ] `zipfile.ZipFile` wraps everything into `vc_analysis/` folder

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
vc_charts.py  —  VC Financial Analyzer: Individual Chart Exports
Saves one PNG per indicator -> zips into vc_analysis_charts.zip

Output:
  vc_analysis/
    00_vc_signal_summary.png
    01_revenue_trend.png
    02_growth_rate.png
    03_burn_rate.png
    04_runway.png
    05_burn_multiple.png
    06_gross_margin.png
"""

import os, zipfile
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
from matplotlib.patches import Patch
import warnings
warnings.filterwarnings("ignore")

# ── PATHS ─────────────────────────────────────────────────────────────────────
OUT_DIR  = "./vc_analysis"
ZIP_PATH = "./vc_analysis_charts.zip"
os.makedirs(OUT_DIR, exist_ok=True)

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
    "font.family":       "DejaVu Sans",
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

# ── SYNTHETIC DATA (replace with real df for actual company) ──────────────────
np.random.seed(42)
n        = 12
quarters = pd.date_range(start="2022-01-01", periods=n, freq="QS")
rev_base     = [200_000 * (1.18 ** i) for i in range(n)]
revenue      = np.array([r * np.random.uniform(0.92, 1.08) for r in rev_base])
new_arr      = revenue * np.random.uniform(0.25, 0.45, n)
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
    "new_arr":       new_arr,
    "runway":        cash / burn,
    "burn_multiple": burn / np.where(new_arr > 0, new_arr, np.nan),
    "gross_margin":  gross_margin,
})
df["revenue_ma"]  = df["revenue"].rolling(3, min_periods=1).mean()
df["growth_rate"] = df["revenue"].pct_change() * 100
df["growth_ma"]   = df["growth_rate"].rolling(3, min_periods=1).mean()

FORECAST_PERIODS = 4
f_dates = pd.date_range(df["date"].iloc[-1], periods=FORECAST_PERIODS + 1, freq="QS")[1:]

def linear_forecast(values, n_periods):
    x = np.arange(len(values))
    z = np.polyfit(x, values, 1)
    p = np.poly1d(z)
    return p(np.arange(len(values), len(values) + n_periods)), z[0]

saved_files = []
print("\n📊  Generating charts...\n")

# ── CHART 1: Revenue Trend ────────────────────────────────────────────────────
fy_rev, slope_rev = linear_forecast(df["revenue"], FORECAST_PERIODS)
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Revenue Trend + Forecast", fontsize=14,
             fontweight="bold", color="#e0e0f0")
ax.plot(df["date"], df["revenue"]/1e6,    color=COLORS["neutral"], alpha=0.45, linewidth=1.5, label="Actual Revenue")
ax.plot(df["date"], df["revenue_ma"]/1e6, color=COLORS["primary"], linewidth=2.8, label="3-Qtr Rolling Avg")
ax.plot(f_dates, fy_rev/1e6, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Linear Forecast")
ax.fill_between(f_dates, fy_rev/1e6*0.82, fy_rev/1e6*1.18, color=COLORS["forecast"], alpha=0.12, label="±18% Band")
ax.axvline(df["date"].iloc[5], color=COLORS["accent"], linewidth=1.4, linestyle=":", alpha=0.8)
ax.text(df["date"].iloc[5], df["revenue"].max()/1e6*0.42, "  Series A\n  $6M raised",
        color=COLORS["accent"], fontsize=9)
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("$%.1fM"))
ax.set_ylabel("Revenue ($M)"); ax.set_xlabel("Quarter")
ax.legend(loc="upper left"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "01_revenue_trend.png"))

# ── CHART 2: Growth Rate ──────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  QoQ Revenue Growth Rate (%)", fontsize=14,
             fontweight="bold", color="#e0e0f0")
valid = df["growth_rate"].notna()
ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] > 0), color=COLORS["positive"], alpha=0.2, label="Positive Zone")
ax.fill_between(df["date"][valid], 0, df["growth_rate"][valid],
                where=(df["growth_rate"][valid] < 0), color=COLORS["danger"], alpha=0.2, label="Negative Zone")
ax.plot(df["date"][valid], df["growth_rate"][valid], color=COLORS["neutral"], alpha=0.5, linewidth=1.5, label="Raw QoQ")
ax.plot(df["date"][valid], df["growth_ma"][valid],   color=COLORS["primary"], linewidth=2.8, label="3-Qtr Avg")
ax.axhline(0,  color="#444455", linewidth=1.0)
ax.axhline(15, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="15% VC Benchmark")
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylabel("Growth Rate (%)"); ax.set_xlabel("Quarter")
ax.legend(loc="upper right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "02_growth_rate.png"))

# ── CHART 3: Burn Rate ────────────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(12, 6))
fig.patch.set_facecolor("#0f0f14")
fig.suptitle("NovaSaaS Inc.  ·  Burn Rate by Quarter", fontsize=14,
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
ax.set_ylabel("Burn Rate ($K/qtr)"); ax.set_xlabel("Quarter")
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
ax.set_ylabel("Runway (months)"); ax.set_xlabel("Quarter")
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
ax.set_ylabel("Burn Multiple (×)"); ax.set_xlabel("Quarter")
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
ax.plot(df["date"], gm_ma, color=COLORS["accent"], linewidth=2.8, label="3-Qtr Rolling Avg")
ax.plot(f_dates, fy_gm, color=COLORS["forecast"], linewidth=2.2, linestyle="--", label="Forecast")
ax.fill_between(f_dates, fy_gm-3, fy_gm+3, color=COLORS["forecast"], alpha=0.1, label="±3pp Band")
ax.axhline(80, color=COLORS["positive"], linewidth=1.2, linestyle=":", alpha=0.7, label="80% Top SaaS")
ax.axhline(70, color=COLORS["warning"],  linewidth=1.2, linestyle=":", alpha=0.7, label="70% SaaS min")
ax.axhline(50, color=COLORS["danger"],   linewidth=1.2, linestyle=":", alpha=0.7, label="50% Below standard")
ax.yaxis.set_major_formatter(mticker.FormatStrFormatter("%.0f%%"))
ax.set_ylim(30, 100)
ax.set_ylabel("Gross Margin (%)"); ax.set_xlabel("Quarter")
ax.legend(loc="lower right"); ax.grid(True); ax.tick_params(axis="x", rotation=30)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "06_gross_margin.png"))

# ── CHART 0: VC Signal Summary ────────────────────────────────────────────────
fig, ax = plt.subplots(figsize=(10, 6))
fig.patch.set_facecolor("#0f0f14")
ax.set_facecolor("#0f0f14"); ax.axis("off")
fig.suptitle("NovaSaaS Inc.  ·  VC Signal Summary", fontsize=14,
             fontweight="bold", color="#e0e0f0")

def signal_row(name, current, slope, good_dir="up"):
    if good_dir == "up":
        color = COLORS["positive"] if slope > 0.05 else (COLORS["warning"] if slope > -0.02 else COLORS["danger"])
        emoji = "●" if slope > 0.05 else ("◑" if slope > -0.02 else "○")
    else:
        color = COLORS["positive"] if slope < -0.02 else (COLORS["warning"] if slope < 0.05 else COLORS["danger"])
        emoji = "●" if slope < -0.02 else ("◑" if slope < 0.05 else "○")
    return name, current, f"{slope:+.2f}", color, emoji

bm_all = df["burn_multiple"].dropna()
rows = [
    signal_row("Revenue Growth",  f"${df['revenue'].iloc[-1]/1e6:.2f}M / qtr",
               np.polyfit(np.arange(n), df["revenue"], 1)[0] / df["revenue"].mean(), "up"),
    signal_row("QoQ Growth Rate", f"{df['growth_rate'].iloc[-1]:.1f}%",
               np.polyfit(np.arange(n-1), df["growth_rate"].dropna(), 1)[0], "up"),
    signal_row("Burn Rate",       f"${df['burn'].iloc[-1]/1e3:.0f}K / qtr",
               np.polyfit(np.arange(n), df["burn"], 1)[0] / df["burn"].mean(), "down"),
    signal_row("Runway",          f"{df['runway'].iloc[-1]:.1f} months",
               np.polyfit(np.arange(n), df["runway"], 1)[0], "up"),
    signal_row("Burn Multiple",   f"{bm_all.iloc[-1]:.2f}×",
               np.polyfit(np.arange(len(bm_all)), bm_all, 1)[0], "down"),
    signal_row("Gross Margin",    f"{df['gross_margin'].iloc[-1]*100:.0f}%",
               np.polyfit(np.arange(n), df["gross_margin"], 1)[0], "up"),
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
        f"3-Qtr Avg Growth: {df['growth_ma'].iloc[-1]:.1f}%  ·  Latest QoQ: {df['growth_rate'].iloc[-1]:.1f}%")
ax.text(0.04, 0.04, note, fontsize=8.5, color="#555570", transform=ax.transAxes)
plt.tight_layout(pad=2.5)
saved_files.append(save_fig(fig, "00_vc_signal_summary.png"))

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
```

---

## Iteration Log

| Version | Date | Change |
|---|---|---|
| v1.0 | 2026-05-17 | Initial skill — 4-panel dashboard, linear forecast, VC signal layer |
| v1.1 | 2026-05-17 | **Output format changed** — one PNG per indicator named by metric, zipped for download. No dashboard. Added `00_vc_signal_summary.png` scorecard. |
| v1.x | — | *(your next iteration here)* |

---

*End of skill document. Edit freely — this is your living reference.*
