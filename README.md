# Stars Agentic System

An agentic AI system for Medicare Star Ratings optimization — from forecasting
and 10,000-run simulations to closed-loop intervention, vendor orchestration,
and provider escalation, with humans setting policy and holding approval gates.

- **Plan:** Meridian Health *(fictional)*
- **Timeline:** build in 2027 → live for measurement year 2028 → Stars 2029
- **Goal:** 70% of members in 4-star-plus contracts

> **All data in this repo is illustrative / synthetic.** Contract IDs, member
> IDs, costs, lift estimates, and ratings are placeholders so the system can be
> designed and demoed end-to-end. Replace with real data before production use
> (see `inputs/stars_inputs.yaml` → `open_questions`).

## Repo structure

```
stars-agentic/
├── README.md                     # you are here
├── ui/
│   └── stars-demo.html           # interactive two-act demo (open in a browser)
├── inputs/
│   ├── stars_inputs.yaml         # input spec: constraints, levers, files, models
│   └── placeholders/             # synthetic CSVs matching the spec's schemas
│       ├── contract_measure_ratings_5yr.csv
│       ├── cutpoint_forecast.csv
│       ├── measure_rating_forecast.csv
│       ├── montecarlo_contract_stars.csv
│       ├── program_catalog.csv
│       ├── member_gap_snapshot.csv
│       ├── vendor_master.csv
│       └── provider_attribution.csv
└── .gitignore
```

## The demo (`ui/stars-demo.html`)

Open the file in any modern browser — no server needed.

- **Act 1 — "The War Room" (March 2028):** adjust constraints, toggle levers,
  run the 10,000-future simulation swarm, compare scenarios
  (*Efficiency Play*, *Provider-Led Push*, *Triage & Focus*, *Moonshot*),
  inspect each scenario's "things that must go right" checklist, re-run after
  changing levers, then approve the plan.
- **Act 2 — "The Machine at Work" (July 2028):** plan-vs-actuals trajectories
  per contract (dashed blue = projected, solid = actuals), a live agent
  activity feed (vendor file exchange, SLA breach, causal-inference budget
  reallocation, zip-level vendor research, provider escalation draft, cut-point
  alert), with human approval gates on every consequential action.

Color language: **green** = on track · **amber** = needs a human decision ·
**red** = breached/escalated · **blue (dashed)** = simulated/projected ·
**gray** = pending.

## Inputs (`inputs/stars_inputs.yaml`)

The single source of truth for the system. Five sections:

1. **constraints** — financial, operational, regulatory, data/time boundaries
   enforced by both the simulation engine and the agents.
2. **levers** — 11 levers, each declaring cost, expected lift + evidence basis,
   target measures, and consumed constraints. Costs/lifts are illustrative
   until replaced by the real program catalog.
3. **input_files** — the 9 data files/interfaces the system needs, tagged
   `HAVE` / `CAN_GET` / `NEED`, with grain, key columns, and consumers.
4. **model_interfaces** — the contract between existing models (cut-point
   range model, measure forecast, Monte Carlo) and the agents, plus the one
   model still to build: **uplift**.
5. **open_questions** — decisions to resolve before production.

## 2027 build sequencing (first cut)

- **Q1 — Foundations.** Stand up the data contracts in `input_files`
  (member gap snapshot pipeline first — targeting is guesswork without it).
  Build the program catalog with evidence tags per lever. Backtest harness on
  the 5-year history: "would our 2024 plan have predicted 2025?"
- **Q2 — Forecasting core.** Wire the cut-point range model and measure
  forecast model into the Forecasting Agent (member → measure → contract
  roll-up). Promote the Monte Carlo into the simulation engine; levers enter
  as shifts to its input distributions.
- **Q3 — Agentic layer.** Intervention Agent (targeting + vendor
  orchestration), Provider Watchdog, Cut-Point Watchtower, Research Agent.
  Human approval gates on money movement, vendor PHI transfers, and
  escalations. Act 1 War Room running on real data.
- **Q4 — Closed loop.** Build the uplift model; causal inference on program
  effectiveness with holdouts; Act 2 operations board live; pilot on 2–3
  contracts. Ready for measurement year 2028.

## Making it real — checklist

- [ ] Replace every `ILLUSTRATIVE` value in `stars_inputs.yaml`
- [ ] Land the `NEED` input files (member gap snapshot is highest priority)
- [ ] Bless program-catalog lift estimates (actuarial/clinical sign-off, not
      vendor-reported alone) — and make them **uplift**, not raw compliance
- [ ] Build the uplift model (the one model you don't have yet)
- [ ] Define the PHI/compliance sign-off workflow for vendor file transfers
- [ ] Confirm real budget + who can reallocate mid-year
- [ ] Answer `open_questions` in `stars_inputs.yaml`

## Disclaimer

Fictional plan, fictional data, illustrative demo. Nothing here is medical
advice, and no real member data is included or required to run the demo.
