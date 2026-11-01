# Fluid L2 continuation project — locked summary

**Status:** ETH has a narrow, externally replicated historical candidate. BTC
does not. Delta India is an engineering/paper-trading bridge, not yet a
validated live deployment.

## The question

Once directional aggressive flow is already observable, can the visible L2
liquidity response tell us to continue with the flow, fade it, or abstain—after
costs?

This is not a global-price-prediction or low-dimensional-representation
project. The commercial test is a selective, after-cost continuation or
reversal edge.

## Frozen ETH candidate

| Item | Locked specification |
|---|---|
| Asset / venue | ETHUSDT Binance USD-M perpetual historical data |
| Signal type | Continuation only; do not trade a fade signal |
| Horizon | 30 minutes |
| Entry | Immediately after the causal signal boundary; historical result is a **0 ms theoretical upper bound** |
| Trigger | Absolute 30-second signed taker-notional flow / causal trailing 600-second median absolute 30-second flow >= **2.6203264325** |
| Event handling | One event per 30-minute cooldown; never stack overlapping positions |
| Model | Full-state standardized Ridge regression, alpha = **10** |
| Selection | Highest predicted signed continuation edge, top **25%** |
| Exit | Fixed 30 minutes after entry |
| Costs reported | 2, 5, 10, and 20 bp round trip; primary comparison = 10 bp |

The candidate is frozen for external replication. It is not a universal ETH
law and it is not a deployable Delta India model.

## Causal inputs to the full-L2 Ridge

### Directional forcing / trade state

- Signed aggressive USD notional over 5, 10, 30, and 60 seconds.
- Gross flow over the same windows; 30-second shock magnitude relative to
  trailing activity.
- Buy/sell volume balance, trade count, mean trade size, and trade-sign
  persistence.
- Short realized volatility and trade/notional activity.

### L2 state

The historical source provided top-25 snapshots. At each causal second the
system used the last received, valid, non-crossed snapshot strictly before that
second.

- Spread in bp, spread change, and recent spread volatility.
- Cumulative bid and ask notional depth at 1, 5, 10, and 20 levels.
- Multi-level imbalance at the same levels.
- Forcing-oriented opposing-side and same-side depths: ask is opposing for a
  buy shock; bid is opposing for a sell shock.
- Positive/negative net depth changes over 5, 10, and 30 seconds. These are
  **proxies**, not a true exchange-level separation of new orders,
  cancellations, and executions.
- Replenishment proxies: positive opposing-side net-depth change relative to
  opposing-side removal.
- Recent price susceptibility (price change per signed flow) and its change.
- Explicit forcing × opposing-depth/removal/replenishment/susceptibility
  interactions.

## Training objective and protocol

For trigger direction `d = sign(signed 30s flow)`, the regression target is:

```text
Y(30m) = d × 10,000 × log(P[t + 30 minutes] / P_entry)
```

Positive `Y` means continuation in the shock direction. The Ridge model is
trained to rank this signed return; the trading rule executes only its top
quarter. It may abstain the remaining 75% of candidates.

Initial chronological design:

| Role | Dates |
|---|---|
| Train | 1 Sep–1 Dec 2020 |
| Validation / model selection | 1 Jan–1 Feb 2021 |
| Original held-out test | 1 Mar–1 Aug 2021 |
| External ETH replication | 1 Sep 2021–1 Feb 2022 |

The final frozen ETH model was fitted on the first six dates after the
validation decision. The trigger cutoff was carried forward exactly. One
protocol defect must remain visible: although an 80/90/95 threshold grid was
written down, the code used the 80th-percentile trigger rather than selecting
the grid on validation. No claim should hide that limitation.

## What worked

### ETH, original held-out test

At top-quarter continuation selection, 30 minutes, and 10 bp round-trip cost:

| Metric | Result |
|---|---:|
| Trades | 67 across 6 isolated test days |
| Gross bp/trade | +24.01 |
| Net bp/trade | +14.01 |
| Win rate | 67.2% |
| Day-block 95% CI, net | +5.04 to +20.46 bp |
| Profitable days | 5 / 6 |

Flow-only ETH at the same 30-minute/top-quarter cell was -7.26 bp net. This is
the strongest evidence that observed L2 state—not simply shock flow—added value
in that bounded sample.

### ETH, untouched external replication

The rule above was held fixed and applied to six fresh public Tardis days.

| Metric | Result |
|---|---:|
| Candidate events | 270 |
| Executed top-quarter trades | 67 |
| Gross bp/trade | +16.78 |
| Net bp/trade at 10 bp costs | **+6.78** |
| Day-block 95% CI at 10 bp | **+0.57 to +10.70 bp** |
| Win rate | 58.2% |
| Profitable days at 10 bp | 5 / 6 |
| Net bp/trade at 20 bp | -3.22 |

This confirms a **narrow historical ETH candidate under a 10-bp cost model**.
It does not prove executable profitability at nonzero latency or at 20 bp.

## What did not work

| Branch | Outcome |
|---|---|
| BTC 30-minute full-L2 Ridge | No stable net edge; external fresh result -11.02 bp/trade at 10 bp costs |
| BTC 30-minute shallow gradient boosting | Validation appeared positive, but fresh external result -11.87 bp/trade at 10 bp; 5/6 days negative |
| Stacking every qualifying second | Both BTC and ETH became strongly negative after costs; consecutive triggers are one correlated episode, not independent trades |
| ETH shorter horizons (1–15m) | No durable after-cost result comparable to 30m |
| One-hour sweep | Exploratory gross positives, but non-monotonic scores and wide intervals; not validated |
| Reversal / fade branch | Broadly negative in the original test |

## Interpretation

The evidence is consistent with a slow, selective ETH continuation state:
directional forcing plus an L2 condition sometimes identifies a 30-minute
propagation episode. It is **not** evidence that every high-flow observation
should be traded, that BTC shares the signal, or that an arbitrary nonlinear
model discovers a general crypto alpha.

Current honest classification: **2 — NARROW NET ALPHA**, conditional on the
10-bp historical cost model. It is below "robust conditional alpha" because it
has one asset, sparse isolated dates, zero-delay historical entry, a threshold
selection protocol blemish, and no Delta India validation.

## Delta India deployment boundary

The repository in `delta_india_fluid_trader/` is a paper-first collector and
execution harness. It is intentionally not armed with the Binance ETH model.

- Delta public `ob_l2` supplies top 15 levels, not the top 25 used historically.
- Delta public trades do not publish aggressor direction; the harness infers it
  causally from trade price versus the contemporaneous best bid/ask. That is
  noisier than the explicit Binance/Tardis taker-side field.
- The deployment path is therefore: capture Delta-native L2/trade data → check
  feed quality and inferred aggressor labels → train/validate a Delta-native
  model → forward paper trade → only then consider a tightly gated live trial.

Live orders remain disabled by default and require explicit environment gates,
credentials, and a Delta model marked forward-validated.

## Essential files

- `l2_conditional_alpha/results/protocol.json` — original experiment protocol.
- `l2_conditional_alpha/results/selective_alpha.csv` — original selective test
  results.
- `l2_conditional_alpha/external_replication/replication_summary.csv` — locked
  ETH external replication.
- `l2_conditional_alpha/btc_gbt_external/summary.csv` — fresh BTC nonlinear vs
  Ridge result.
- `delta_india_fluid_trader/README.md` — Delta collection/execution harness.
