# ScholarFund AI

ScholarFund AI helps a scholarship endowment, modeled on Harvard's, decide how to invest its capital. The user picks a scenario (risk profile, ethics policy, private equity share, excise tax, inflation, payout) and the app:

1. fetches public data (ETF prices, inflation, corporate ethics data);
2. simulates three strategies (Conservative / Balanced / Growth) and tests whether each can pay scholarships over 10 years, including after a market crash;
3. recommends a strategy, or explains that none passes and what payout would be sustainable;
4. has an LLM (Azure OpenAI) explain the result in plain language, using only the model's numbers;
5. shows everything in a Streamlit webapp with an AI analyst chat.

**Two Harvard-specific features are modeled:**
- **2026 tiered endowment excise tax** (1.4 / 4 / 8%), applied to realized capital gains.
- **Private equity liquidity:** Harvard holds 40%+ in PE. The model targets 20% and shows what 40% would change.

**Headline result** (default settings: $100k, Balanced, Moderate ethics filter, 20% PE, 2% tax, 4% payout, 3% inflation, 10 years): Balanced is recommended. At a 6% payout after a crash, a 40% PE fund misses scholarships in 9.4% of simulations because money is locked up, versus 0% at 20% PE.

## Quick start

```bash
pip install -r requirements.txt
cp .env.example .env   # fill in your API keys
streamlit run app.py   # http://localhost:8501
```

| Variable | Used for |
|---|---|
| `FRED_API_KEY` | inflation, T-bill rates |
| `WIKIRATE_API_KEY` | company ethics data |
| `AZURE_OPENAI_API_KEY` | LLM explanation |
| `FINNHUB_API_KEY` | connected, not yet used by the model |

Other commands: `python -m pytest` (61 offline tests, no keys needed), `python -m scholarfund.analysis` (full analysis on real data), `python -m scholarfund.check_connections` (check data sources).

## How it works

```
Yahoo Finance ─┐
FRED ──────────┼─> analysis.py ─> finance.py (simulation, tests, recommendation)
Wikirate ──────┘   (+ ethics.py)        └─> llm_explainer.py ─> Azure OpenAI

Streamlit: app.py <── webapp/model.py (adapter) + webapp/llm.py (chat)
```

**Data**
- ETF prices from Yahoo Finance (BIL, BND, BNDX, VTI, VXUS, plus ESG versions ESGV, VSGX), monthly returns from June 2007 to September 2026 (232 months, includes 2008). Younger funds are backfilled with close proxies, and this is flagged in the results.
- Inflation and T-bill rates from FRED (default: 10-year average CPI, falls back to 3%).
- Company ethics data from Wikirate.
- Private equity has no market data, so it is an assumption.

**Allocation**
- The fund is split into private equity (default 20%) and public markets (80%). Public markets always hold at least 30% bonds/T-bills and 30% stocks. Conservative is 70/30, Balanced 50/50 and Growth 30/70.
- Within each class, the split is mostly US with some international exposure.
- Users only see strategies no riskier than their profile.
- If the ethics filter excludes a fund, it is replaced by its ESG equivalent.
- A cash reserve (in BIL) covers the next 24 scholarship payments.

**Private equity (illustrative assumption)**
- Fixed 9% annual return (conservative vs. the 11-13% of recent years), locked for 7 years, and -30% in a crash.
- The model tracks PE's share of the fund over time (it grows as public assets pay scholarships, the "denominator effect") and scholarships missed because money was locked.
- Limitation: a fixed return makes PE look smoother and safer than it really is.

**Excise tax**
- Paid on realized capital gains whenever the fund sells appreciated assets (to pay scholarships, refill the reserve or rebalance). Dividends and interest are not taxed in the model.

**Withdrawal rules**
- Scholarships are paid from cash first, then less risky bonds, then stocks, then unlocked PE.
- Stocks are not sold when more than 10% below their peak, unless nothing else remains (counted as a forced sale).

**Ethics filter (proof of concept, free data only)**
- Standard: no filter.
- Moderate: excludes equity funds whose top 10 holdings include fossil fuels, tobacco or weapons.
- Strict: Moderate, plus a human-rights commitment is required (Wikirate), and bond funds must be 90%+ US government debt. In practice this excludes all equity funds, so no strategy is available.
- Only each fund's top holdings are examined, and company name matching is imperfect.

**Simulations and tests**
Each strategy is run three ways: historical (all 232 months), Monte Carlo (1,000 ten-year paths from 12-month blocks, fixed seed), and stress (same paths with a first-month crash: stocks -40%, bonds -10%, PE -30%).

A strategy passes if it pays all scholarships historically and in at least 90% of Monte Carlo and stress paths. Real value preservation (ending with at least the starting capital in today's dollars) is shown but is not a pass/fail test.

**Recommendation**
- Pick the strategy matching the user's profile if it passes. Otherwise pick the next less risky one that passes.
- If none passes, no recommendation is forced. The model reports the highest sustainable annual scholarship per strategy.
- Wording is "passed the tested scenarios", never "guaranteed".

**LLM explanation**
The LLM receives a compact summary of the analysis and is instructed to use only those figures, never claim guarantees, give no investment advice, and end with the disclaimer. `unverified_numbers()` flags numbers in the text that don't match the data (including LLM-made calculations, which need manual checking). The explanation language is configurable.

## Webapp

`app.py` is the Streamlit UI, `webapp/model.py` translates UI settings into engine scenarios (and runs comparison scenarios for the tax and PE tabs), and `webapp/llm.py` powers the analyst chat. The chat can change settings by returning an `ACTION: {...}` line, with values bounded before being applied.

Tabs: Allocation, Simulation, Liquidity, Endowment tax, Ethics, AI analyst, Data & assumptions.

**Demo tip:** set the payout to 6%, then switch PE to 40% ("Harvard today"). PE grows to about 83% of the fund, and 9.4% of post-crash simulations miss a scholarship (vs. 0% at 20%).

## Project structure

```
app.py              Streamlit webapp
scholarfund/        engine (finance, analysis, ethics, llm_explainer, sources/)
webapp/             adapter and chat
tests/              offline tests
reports/            generated LLM explanations
```

## Limitations

- Past data (19 years) doesn't cover every possible crisis.
- Over 10 years at a 4% payout, nearly every strategy passes the funding tests, so real value preservation is the more informative metric.
- Fixed-return PE makes a 40% PE fund look better than it should.
- Tax and ethics modeling are simplified (see above).
- Harvard facts in `webapp/model.py` (`HARVARD`) still need verification.

**Next steps:** validate thresholds (horizon, 90% pass rate, shock sizes), add an inflation-adjusted capital preservation test, and use Finnhub news in the explanation.

---

> **Disclaimer:** Educational prototype only. Historical data and stress scenarios do not guarantee future results. Not professional financial advice.

