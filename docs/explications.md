# ScholarFund AI — Prototype Status

## Overview

ScholarFund AI helps a scholarship endowment fund, modeled on Harvard's, decide how to invest its capital. The client chooses a scenario (caution level, ethics, private equity share, excise tax, inflation, scholarship amount); the prototype:

1. retrieves public data (ETF prices, inflation, corporate ethics data);
2. simulates three investment strategies and tests whether they can pay the scholarships for 10 years (adjustable horizon), including after a crash;
3. recommends a strategy, or explains that none passes and which scholarship amount would be sustainable;
4. has an LLM (Azure OpenAI) explain the result in plain language, restricted to using only the model's figures;
5. displays all of this in the Streamlit webapp (`app.py`), where the client adjusts the scenario and can chat with the AI analyst.

**Two Harvard-specific topics** are modeled:

- **the 2026 tiered excise tax** (1.4 / 4 / 8%; Harvard is at 8%), paid on realized capital gains when the fund sells;
- **private equity liquidity**: Harvard holds more than 40% in it, which put it in difficulty; the model targets 20%, and shows what 40% would change.

**Main result** (October 2026 data, default settings: $100,000, Balanced, Moderate filter, 20% private equity, 2% tax, 4% payout, 3% inflation, 10 years): Balanced is recommended. With a 6% payout and a crash, a fund with 40% private equity misses scholarships in 9.4% of cases because the money is locked up, versus 0% at 20%.

---

## 1. Getting Started

**Installation**

```bash
pip install -r requirements.txt
```

**API keys**: copy `.env.example` to `.env` and fill in the keys. The `.env` file is excluded from git; never put a key in the code or in `.env.example`.

| Variable | Use |
|---|---|
| `FRED_API_KEY` | inflation and T-bill rates |
| `FINNHUB_API_KEY` | prices and news (not yet used by the model) |
| `WIKIRATE_API_KEY` | corporate ethics data |
| `AZURE_OPENAI_API_KEY` | LLM explanation |

The Azure endpoint, the model (`gpt-6.1-sol`) and the API version are taken from `templates/main.py`; they can be changed in `.env`.

**Useful commands**

| Command | What it does |
|---|---|
| `python -m pytest` | 61 offline tests, no key needed |
| `python -m scholarfund.check_connections` | checks each data source |
| `python -m scholarfund.analysis` | quantitative analysis on real data |
| `python -m scholarfund.llm_explainer` | LLM explanation, saved in `reports/` |
| `streamlit run app.py` | **launches the webapp** (http://localhost:8501) |

All commands are run from the project root.

---

## 2. Overview

```
Yahoo Finance ─┐
FRED ──────────┼─> analysis.py ─> finance.py (simulation, tests,
Wikirate ──────┘   (+ ethics.py)    recommendation)
                                        │
                                        ├─> llm_explainer.py ─> Azure OpenAI
                                        │
Streamlit webapp: app.py <── model.py (adapter) + llm.py (chat)
```

- `analysis.run_analysis(...)` runs the whole chain for a scenario and returns a single dictionary: results and data context.
- Market and FRED data are kept in memory for the day; the ethics filter is cached in `.cache/`.

---

## 3. Data

### ETF prices (Yahoo Finance, `market_data.py`)

| ETF | Content | Class in the model |
|---|---|---|
| BIL | US Treasury bills, 1-3 months | bonds and T-bills |
| BND | US bonds (government and corporate) | bonds and T-bills |
| BNDX | International bonds, currency-hedged | bonds and T-bills |
| VTI | US stocks | stocks |
| VXUS | International stocks excluding the US | stocks |
| ESGV | US stocks, ESG version (replaces VTI if VTI is excluded) | stocks |
| VSGX | International stocks, ESG version (replaces VXUS if VXUS is excluded) | stocks |

- Only adjusted daily closing prices (dividends included) are used, converted into monthly returns.
- History: **June 2007 to September 2026 (232 months)**, including the 2008 crisis. The current, incomplete month is excluded.
- VXUS has only existed since 2011: before that, its returns are replaced by those of VEU, a very similar fund (monthly correlation of 0.998).
- BNDX has only existed since 2013: before that, its returns are replaced by those of BND (correlation of 0.86, similar volatility), which is closer than unhedged international bond funds.
- ESGV and VSGX have only existed since 2018: before that, their returns are replaced by those of VTI and VXUS (correlation of 0.994).
- These data substitutions are indicated in the results.
- Private equity has no market data: it is an assumption (section 5).
- "Less risky" is a simplification: bonds can also lose value.

### Inflation and rates (FRED, `fred_data.py`)

- Default inflation assumption: 10-year average CPI inflation, **3.41%** in October 2026. If FRED does not respond, the model uses 3% and flags it. The client can also choose their own value.
- The 3-month T-bill rate is displayed for information.

### Ethics data (Wikirate, `wikirate_data.py`)

Open, community-sourced data on companies. Used by the ethics filter (section 6).

### Finnhub (`finnhub_data.py`)

Connected (prices, company profiles, news of at most one week in the free version) but not yet used by the model. ETF holdings and ESG scores are paid features: the code stops with a clear error rather than inventing data.

---

## 4. The Client's Scenario

| Parameter | Values | Default (engine) | Default (webapp) |
|---|---|---|---|
| Caution (`risk_tolerance`) | Conservative, Balanced, Growth | Balanced | Balanced |
| Ethics (`ethical_policy`) | Standard, Moderate, Strict | Standard | Moderate |
| Private equity (`pe_share`) | share of the fund; webapp: 0, 20% (target) or 40% (Harvard today) | 20% | 20% |
| Excise tax (`excise_tax_rate`) | rate on realized capital gains; webapp: switch + rate | 0 | enabled, 2% |
| Annual scholarship (`annual_payout`) | in $, indexed to inflation each year; webapp: % of the gift | $5,000 | 4% |
| Inflation (`inflation_rate`) | decimal (0.03 = 3%) | FRED average | 3% |
| Capital (`initial_capital`) | in $ | $100,000 | $100,000 |
| Horizon (`horizon_years`) | length of the simulations, in years | 10 | 10 |

---

## 5. Allocation, Reserve and Withdrawal Rules

### Fund structure

```
Fund = private equity (20%)  +  public markets (80%)
                                  ├─ bonds and T-bills (reserve included)
                                  └─ stocks
```

**Public markets**: each class always weighs **at least 30%**; the remaining 40% goes to one or the other depending on the profile.

| Profile | Bonds and T-bills | Stocks |
|---|---|---|
| Conservative | 70% | 30% |
| Balanced | 50% | 50% |
| Growth | 30% | 70% |

Within each class (mostly US, with some international diversification):

| Class | Breakdown |
|---|---|
| Bonds and T-bills | US T-bills 15% (BIL), US bonds 65% (BND), international bonds 20% (BNDX) |
| Stocks | US stocks 65% (VTI, otherwise ESGV), international stocks 35% (VXUS, otherwise VSGX) |

- The scholarship **reserve** is part of the bonds and T-bills class (it is held in BIL).
- Each class is split into **zones** (US / international). If the ethics filter excludes a zone's fund, it is **replaced by its ESG version** for the same zone: the money stays spread across zones instead of piling into a single fund. The app displays each replacement.
- If a zone has no permitted fund, its weight goes to the other zones of the class (this is flagged). If an entire class is empty, the strategy is unavailable (the 30% rule cannot be met).
- The client only sees strategies that are **no riskier than their profile**.

Example, Balanced with the Moderate filter (VXUS excluded, replaced by VSGX), $100,000: $20,000 in private equity; $12,902 in BIL (reserve included), $20,722 in BND, $6,376 in BNDX; $26,000 in VTI and $14,000 in VSGX.

### Private equity (illustrative assumption)

- **Why**: Harvard holds more than 40% of its fund in private equity, which has created liquidity problems; the goal is to return to 20%.
- **Assumptions**: fixed return of **9% per year** (below the 11-13% of the past 15 years, hence prudent), capital **locked for 7 years**: it can neither pay scholarships nor refill the reserve before then. Afterwards, it can be sold as a last resort. In the crash scenario, it loses 30%.
- **What is measured**: the share of the fund in private equity over time (it rises when public assets pay the scholarships: this is the "denominator effect"), and the share of simulations where a scholarship is missed even though the money existed but was locked up.
- **Limitation**: a fixed return makes private equity smooth and attractive; real private equity is riskier. The returns and volatilities displayed for the strategies cover public markets only; "With PE" adds the assumed private equity return.

### Excise tax (modeled on the 2026 tiered tax)

- Since 2026, the excise tax on US universities depends on the endowment per student: **1.4%, 4% or 8%**. Harvard is in the 8% tier (versus 1.4% before).
- In the model, it is paid on the **realized capital gain** each time the fund sells an asset that has appreciated (to pay a scholarship, refill the reserve or rebalance). The fund therefore sells slightly more to cover the tax. The purchase cost of each bucket is tracked to compute the gain.
- Simplification: the real tax also applies to dividends and interest.

### Buckets and withdrawal rules

The fund is divided into four buckets: **cash** (the reserve, in BIL), **less risky** (the other bonds and T-bills), **growth** (stocks) and **private equity**. Each month:

1. each bucket receives its return;
2. the scholarship is paid from the cash bucket; if it is empty, from the less risky bucket, then from stocks, then from private equity if it is no longer locked;
3. the cash bucket is refilled to cover the next 24 payments: first from stocks, unless they are down, then from the less risky bucket, then from unlocked private equity;
4. once a year, the less risky bucket and stocks are brought back to their target split, unless stocks are down. Private equity is never rebalanced.

"Down" means **more than 10% below their last peak**: stocks are then not sold, unless nothing else remains (forced sale, counted). Each sale of an asset with a gain pays the excise tax.

---

## 6. Ethics Filter (`ethics.py`, proof of concept)

Built using free data only.

**What is examined**

- **Equity funds (VTI, VXUS)**: their top 10 holdings as published by Yahoo, i.e. about 35% of VTI and 15% of VXUS. For each company: its sector (Yahoo) and its public human rights commitment (Wikirate, World Benchmarking Alliance). The climate commitment (SBTi 1.5 °C) is displayed for information only.
- **Bond funds (BIL, BND, BNDX)**: their holdings are not available for free; they are judged on their share of US government debt.

**Rules**

| Policy | Equity funds | Bond funds |
|---|---|---|
| Standard | no filter | no filter |
| Moderate | excluded if a top holding is in fossil fuels, tobacco or weapons | accepted, without company-level review |
| Strict | Moderate + no company without a human rights commitment, and at least 80% of the examined weight with a verified commitment | at least 90% US government debt |

A missing answer on Wikirate **never** counts as validated.

**Current result**

| ETF | Moderate | Strict |
|---|---|---|
| BIL | accepted | accepted (99.8% government debt) |
| BND | accepted | excluded (52% government debt) |
| BNDX | accepted | excluded (78% government debt per Yahoo) |
| VTI | accepted | excluded (Alphabet without human rights commitment) |
| VXUS | excluded (Shell, Energy sector) | excluded |
| ESGV | accepted | excluded (Alphabet without human rights commitment) |
| VSGX | accepted | excluded (Alibaba, AstraZeneca without commitment; too few verified answers) |

Under Strict, no equity fund is accepted: all strategies are unavailable, since the 30% equity rule cannot be met.

**Limitations**: only a small part of each fund is examined; company names from Yahoo and Wikirate do not always match ("not found" = not validated); Wikirate may be incomplete.

---

## 7. Simulations and Funding Tests

Each available strategy is simulated three times.

1. **Historical**: the 232 real months, with the rules from section 5 (over the entire period, not just the horizon).
2. **Long term (Monte Carlo)**: 1,000 paths of 10 years (adjustable horizon), built by randomly drawing blocks of 12 consecutive months from the history. The same draws are used for all strategies, and a fixed seed makes the results reproducible.
3. **Stress**: the same paths, with a crash in the first month: BIL 0%, BND and BNDX -10%, stocks -40%, private equity -30%. With 20% private equity and the Moderate filter, the total shock is about -20% for Conservative, -25% for Balanced and -30% for Growth.

**To pass, a strategy must:**

- pay all scholarships over the historical period;
- pay all scholarships in at least **90%** of the long-term paths;
- do the same in the stress scenario.

Indicators computed: annualized return, volatility and maximum drawdown (public markets), return with private equity, success rate, share of paths with a forced sale, final balances (5th, 50th and 95th percentiles), excise tax paid, maximum private equity share, scholarships missed due to locked-up money, and **real value preservation**: the share of paths that end with at least the starting capital in today's dollars. This last indicator is displayed but not used as a test. These are measurements of the past, not forecasts.

---

## 8. Recommendation

- We recommend the strategy that **matches the client's profile** if it passes all tests; otherwise, **the next less risky one** that passes.
- A strategy that fails a test is discarded, not penalized. The result lists the failed tests for each strategy, and gives the reason for the recommendation.
- If no strategy passes, no recommendation is forced. The model then computes the **sustainable scholarship**: for each strategy, the highest annual scholarship that would pass all tests (to within $50).

Honest wording: "the strategy passed the tested scenarios," not "the scholarships are guaranteed."

---

## 9. LLM Explanation (`llm_explainer.py`)

**What the LLM receives**: a compact version of the analysis: scenario, recommendation and reason, sustainable scholarships, allocation in dollars, indicators and failed tests per strategy (including tax paid and private equity liquidity), model rules, data sources and dates, ethics filter summary, disclaimer.

**What the prompt requires of it**

- use only these facts and figures, inventing nothing;
- never speak of a guarantee;
- state first if there is no recommendation;
- explain each rule that applied and cite the data limitations;
- present the tax as "modeled on the new 2026 tiered endowment excise tax," with its tier and cost (or say that it is disabled);
- present private equity as an illustrative assumption tied to Harvard's situation (more than 40%, target of 20%), with the rise in its share and the scholarships missed because of locked-up money;
- give no buying advice, and end with the disclaimer;
- follow a fixed outline: recommendation, why, risks and stress, excise tax, private equity and liquidity, ethics filter, what would change the result, limitations.

The language of the explanation is a parameter (English, French…).

**Number check**: `unverified_numbers()` lists the numbers in the text that match no data sent (English and French formats included). It is a safety net, not proof: it also flags calculations made by the LLM (for example 5,000 - 3,900 = $1,100), which must then be checked by hand.

---

## 10. Streamlit Webapp (`app.py`, `model.py`, `llm.py`)

The team's webapp keeps its interface and style; its calculations now go through our engine.

- **`model.py`** is an adapter: it translates the app's settings into a scenario for the engine (payout rate in % → scholarship in $, inflation, tax and private equity in % → decimal) and runs three analyses: the chosen scenario, the same without tax (or with the 8% tax if it is disabled), and the same with the other private equity share (40% if currently at 20% or 0, otherwise 20%). The Harvard facts (`HARVARD`) are carried over as is, to be verified.
- **`llm.py`** keeps the team's chat: the analyst answers from the engine's results and can **change the settings** by ending its reply with an `ACTION: {...}` line, whose values are bounded before being applied. It uses the same Azure connection as `llm_explainer.py`.
- **`app.py`**: sidebar (caution, three-level ethics, private equity 0 / 20 / 40%, gift, payout rate, excise tax switch and rate, horizon, inflation), then:

| Tab | Content |
|---|---|
| Allocation | amount per asset class (reserve and private equity included), failed tests |
| Simulation | strategies compared (including tax paid), sustainable scholarship, fund value since 2007 |
| Liquidity | chosen private equity vs. the other level: maximum PE share, scholarships missed due to locked-up money, PE share over time |
| Endowment tax | chosen tax vs. no tax (or 8%): tax paid, final value, real value |
| Ethics | filter result per ETF, companies examined |
| AI analyst | committee memo, full explanation with number check, chat |
| Data & assumptions | sources, dates, rules, returns of the last 12 months |

- At the top: a green banner (recommended strategy) or orange banner (none passes, with the sustainable scholarship), and five key indicators.
- The light theme and the crimson color are fixed in `.streamlit/config.toml`, so that the team's style also displays on a browser in dark mode.
- Each scenario takes one to a few seconds the first time, then is kept in memory.

Default settings: Balanced, Moderate filter, 20% private equity, tax enabled at 2%, 4% payout, 3% inflation (the 10-year FRED average, 3.4%, is shown next to the slider), 10 years. With these settings, Balanced is recommended.

Demo example (liquidity): raise the payout to 6%, then switch private equity to 40% ("Harvard today"). In the Liquidity tab, the private equity share climbs to about 83% of the fund, and after a crash, 9.4% of simulations miss a scholarship because the money is locked up, versus 0% at 20%. At a 7% payout, the gap is 28.1% versus 1.7%.

### User Guide

1. **Launch**: `streamlit run app.py`, then open http://localhost:8501. The first calculation takes a few seconds (downloading the day's data).
2. **Set the scenario** in the left sidebar: caution, ethics filter, private equity, gift amount, payout rate, excise tax, horizon, inflation. Everything is recalculated on each change.
3. **Read the verdict** at the top: green banner = recommended strategy; orange banner = no strategy passes, with the sustainable scholarship. The five cards give the displayed strategy, its success rates (normal and after a crash), the chance of preserving the real value of the gift, and the total scholarships to be paid over the horizon.
4. **Explore the tabs** for detail: allocation in dollars, strategy comparison, private equity liquidity, tax effect, ethics filter, sources.
5. **Use the AI analyst** ("AI analyst" tab): committee memo, full explanation with number check, or chat. In the chat, you can ask to change a setting ("lower the payout to 3.5%"): the analyst applies it and the results update.

## 11. Current Results (October 2026)

**App default settings, Growth client** (Moderate, 20% private equity, 2% tax, 4% payout, 3% inflation, 10 years)

| Strategy | Bonds / stocks (public markets) | Worst drawdown (2007-2026) | After a crash | Preserves real value | Max. PE share | Tax paid (median) |
|---|---|---|---|---|---|---|
| Conservative | 70 / 30 | -16% | 100% | 13.7% | 41% | $324 |
| Balanced | 50 / 50 | -29% | 100% | 43.6% | 37% | $370 |
| Growth | 30 / 70 | -40% | 100% | 58.5% | 34% | $398 |

→ All strategies pass over 10 years; more risk = better chances of preserving the real value of the gift, but larger drops along the way.

**Liquidity, Balanced client, after a crash**

| Payout | 20% PE: scholarships missed (locked-up money) | 40% PE |
|---|---|---|
| 6% | 0% | 9.4% |
| 7% | 1.7% | 28.1% |

---

## 12. Files

```
app.py                     Streamlit webapp (run from the root)
requirements.txt           dependencies
.env / .env.example        API keys (the real .env is not versioned)
pytest.ini                 test configuration
scholarfund/               the engine
  finance.py               allocation, private equity, excise tax, buckets,
                           simulations, tests, recommendation
  analysis.py              full analysis for a scenario
  ethics.py                ethics filter
  llm_explainer.py         Azure OpenAI explanation and number check
  config.py                reads API keys from .env
  check_connections.py     connection check
  sources/                 market_data.py (Yahoo Finance), fred_data.py,
                           finnhub_data.py, wikirate_data.py
webapp/                    model.py (webapp ↔ engine adapter),
                           llm.py (AI analyst chat)
tests/                     automated offline tests
docs/                      this document, presentation and demo scripts
templates/                 examples from the Azure OpenAI template (main.py,
                           embeddings.py, original README)
reports/                   LLM-generated explanations
.cache/                    the day's ethics filter results
```

---

## 13. Limitations and Next Steps

**Known limitations**

- The past does not predict the future; 19 years of history do not contain every possible crisis.
- Over the default 10-year horizon, the funding tests almost always pass at a 4% payout: they no longer distinguish between strategies. Real value preservation, which is only displayed, becomes the useful indicator.
- Private equity has an assumed fixed return of 9%: it looks safer and more profitable than it is. This is why a fund with 40% PE preserves its real value better in the model, even though it is less liquid.
- The excise tax applies only to realized capital gains (not to dividends and interest).
- The ethics filter examines only a small part of each fund.

**Next steps**

1. Validate the thresholds with the team: 10-year horizon, 90% success rate, size of the shocks. The crash test (-40% on stocks from the first month, 90% success required) is what makes the model prudent: it always favors strategies with few stocks.
2. Add a real (inflation-adjusted) capital preservation criterion.
3. Review explanations for other profiles and policies.
4. Verify the displayed Harvard facts (`HARVARD` in `webapp/model.py`).
5. Possibly use Finnhub (news) in the explanation.

---

> **Disclaimer** — Educational prototype only. Historical data and stress scenarios do not guarantee future results. This document is not a substitute for a full code review or professional financial advice.
