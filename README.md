# stockflow

Multi-store inventory management and redistribution for retail cooperatives.

**Status: design stage. Nothing is implemented.**

## What this is

Most open-source inventory systems solve the bookkeeping problem — what is where, and how
much of it. This one is aimed at the decision problem underneath: given demand across many
stores, what should be bought, and what should be moved between stores instead of bought.

## Why it is not a CRUD app

The interesting part decomposes into four problems with genuinely different mathematics:

| Problem | Class | Tractability |
|---|---|---|
| Store-to-store reallocation | Minimum-cost flow on a network | Polynomial; LP relaxation is integral |
| Reallocation under shared vehicle capacity | Integer multi-commodity flow | NP-hard |
| Replenishment / buying | Stochastic inventory control | Policy-dependent |
| Cost basis across transfers | Cost-layer bookkeeping | Not an optimization problem, and usually got wrong |

The first is the spine. Because the constraint matrix of a single-commodity network flow
problem is totally unimodular, its linear relaxation yields integral solutions without
branch-and-bound — whole units of stock fall out of a continuous solver for free. That
property is what makes the problem tractable at realistic scale, and knowing precisely which
modelling additions destroy it is most of the engineering.

## Stack

Next.js and TypeScript on Vercel; Supabase Postgres. Free tiers throughout.

## Design

Domain language is defined in `CONTEXT.md`. Decisions and their rejected alternatives are
recorded in `docs/adr/`. Neither exists yet.
