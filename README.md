# ICE-NAV AI

**AI-Enabled Antarctic Sea-Ice, Iceberg Trajectory, and Navigation Decision Support System**

**SIH26059 — Ministry of Earth Sciences**

**Team:Daemon(id-22)**

Repository: <https://github.com/whytaytho/icenav-ai>

---

## Problem statement

Antarctic navigation is not a shortest-path problem. Sea ice evolves, icebergs
drift, and a route that is safe at departure can be dangerous several hours
later. A road network is static and only its traffic changes; here the
navigable surface itself changes, so cells that were passable become forbidden
and the set of feasible routes at T+6h differs from the set at T+0.

## Our solution

ICE-NAV AI combines sea-ice, iceberg and environmental data into a
**time-dependent risk surface**, plans routes across it with a deterministic
A\* search, detects when a committed route is predicted to become unsafe, and
explains the change factor by factor.

The division of responsibility is deliberate, and is the answer to
"where is the AI?":

| Layer | Nature | What it does |
|---|---|---|
| Forecast | statistical / physical, uncertain | predicts iceberg drift and sea-ice advection |
| Risk | deterministic, configurable | turns the environment into a 0–100 risk surface |
| Decision | deterministic, auditable | A\* routing, hazard detection, explanation |

**AI forecasts the environment. It never chooses the route.** Route selection
stays inspectable, reproducible and explainable.

---

## Demo sequence

The timed three-minute script is in [`docs/demo-script.md`](docs/demo-script.md).
Every figure below is produced by the committed scenario and reproduced by
`python scripts/demo_check.py`.

1. **Compare all modes** at T+0 — three strategies over one risk surface.
2. **Commit the balanced route** — planned safety **89.8 / 100**.
3. **Move the forecast slider to +6h** — IB-04 drifts into the direct lead.
4. **PREDICTED ROUTE CONFLICT** fires — safety falls **89.8 → 43.4**, first
   conflict at waypoint 10 of 23, about 10 hours into the voyage.
5. **AUTO REROUTE** — an alternate route scoring **87.5**, costing
   **+12.0 km**, **+0.5 h**, **+12.0 ice-adjusted km**.
6. **Explainability panel** — the risk change attributed by factor, summing
   arithmetically to the reported total.

### Measured demo figures

| Quantity | Value |
|---|---:|
| Planned balanced route safety at T+0 | 89.8 |
| Same route re-scored against the +6h forecast | 43.4 |
| Alternate route safety at +6h | 87.5 |
| Additional distance / time / fuel index | +12.0 km / +0.5 h / +12.0 |
| First conflict | waypoint 10 of 23, hour 10.0 |
| Responsible iceberg | IB-04 |

### T+0 route comparison

Straight-line reference: **344.4 km**.

| Mode | Alpha | Distance | ETA | Safety | Est. Fuel Index |
|---|---:|---:|---:|---:|---:|
| Fastest | 0.0 | 350.9 km | 15.8 h | 82.5 | 350.9 |
| Balanced | 0.6 | 350.9 km | 15.8 h | 82.5 | 350.9 |
| Safest | 4.0 | 368.4 km | 16.6 h | 90.6 | 368.4 |

Fastest and Balanced resolve to the same path on this scenario; Safest buys
**8.1 safety points for 17.5 km** by taking the wider eastern lead.

Because no candidate route crosses ice above the 0.20 concentration at which
the resistance factor rises above 1.0, the Estimated Fuel Index equals route
distance here. That is the correct output of the model, not a defect — the
index exceeds distance only where a route is forced through heavier ice. The
scenario was not reshaped to manufacture a fuel difference.

---

## System architecture

```text
        sea ice · icebergs · wind · ocean current
                          |
                  generate_scenario.py
              (deterministic committed scenario)
                          |
        +-----------------+------------------+
        |                                    |
  iceberg.py (persistence | free drift | ML) |
  seaice.py  (semi-Lagrangian advection)     |
        |                                    |
        +----------> api/forecast.py <-------+
                 (T+0/3/6/12/24 cached)
                          |
                       risk.py
              two-tier risk surface, 0-100
                          |
                     routing.py
          time-expanded A*, admissible heuristic
                          |
        +-----------+-----+------+------------+
        |           |            |            |
    fuel.py   comparison.py  hazard.py   explain.py
                          |
                       FastAPI
                          |
                       api.js
                          |
                    Dashboard.jsx
      map · comparison · alert · explainability
```

A fuller walkthrough is in [`docs/architecture.md`](docs/architecture.md).

The calculation modules are pure: no HTTP, no file loading, no global state.
FastAPI validates the whole YAML configuration once at startup, builds the
forecast cache once, and passes plain data into the engines. That purity is
what lets the forecast layer reuse the risk and routing code unchanged.

## API

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | per-component readiness (scenario, forecast cache, model, risk config) |
| GET | `/environment/current` | committed scenario |
| GET | `/environment/risk` | risk surface for a horizon |
| GET | `/config/risk` | the active, configurable risk weights |
| GET | `/scenarios` | available scenarios |
| GET | `/forecast`, `/forecast/horizons` | forecast environment and horizon list |
| GET | `/icebergs/trajectory` | predicted positions of one berg |
| POST | `/route` | one route in one mode |
| POST | `/routes/compare` | all three modes against one risk surface |
| POST | `/route/reroute` | evaluate committed route, detect hazard, propose alternate, explain |
| GET | `/validation/backtest`, `/validation/options`, `/validation/times` | historical iceberg backtesting |

Interactive documentation at <http://localhost:8000/docs>. Request and response
shapes are frozen in [`docs/data-contract.md`](docs/data-contract.md).

## Risk model

Two tiers, and the distinction matters.

**Hard constraints** remove a cell from the navigable world entirely — land,
ice concentration at or above 0.80, and anything within an iceberg's
`radius_km + safety_buffer_km`. These cells are absent from the search graph,
so no weighting can ever route a vessel through an iceberg.

**Soft risk** scores the remaining cells 0–100:

```text
risk = 0.50 x sea_ice + 0.25 x iceberg + 0.15 x wind + 0.10 x current
```

Route-level scores combine average and worst-case exposure, so a route that is
mostly clear but touches one severe cell cannot hide behind its mean:

```text
risk_score   = 0.5 x mean_cell_risk + 0.5 x max_cell_risk
safety_score = 100 - risk_score
```

Every active constant lives in `backend/config/weights.yaml` and is served live
at `/config/risk`. They are configurable engineering-demo values, **not**
scientifically calibrated operating limits.

## Navigation algorithm

A\* over navigable cells, 8-connected, with no diagonal passage between two
blocked orthogonal cells, and deterministic tie-breaking so the same request
always returns the same route.

```text
risk_norm = mean(risk_from, risk_to) / 100
step_cost = segment_km x (1 + alpha x risk_norm)
```

The cost is **multiplicative and in kilometre-equivalent units**, which matters
for two reasons. It keeps distance and risk dimensionally coherent instead of
adding kilometres to risk points; and because the edge multiplier can never
fall below 1.0, the straight-line haversine heuristic can never overestimate
the remaining cost. The heuristic is therefore admissible and A\* returns a
genuinely optimal path — a property the test suite verifies by comparing A\*
against Dijkstra over the same cost function.

`alpha` is the single risk-aversion knob: **Fastest 0.0, Balanced 0.6,
Safest 4.0**.

Routing is **time-expanded**: node identity is `(row, col, arrival_time_bucket)`
and each cell is costed against the forecast snapshot nearest the hour the
vessel actually reaches it. A voyage of roughly 16 hours is therefore scored
against several different states of the world rather than one frozen snapshot.

## Estimated Fuel Index

```text
fuel contribution = segment_km x resistance_factor(mean_segment_ice)
```
