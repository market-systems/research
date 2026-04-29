# Research

Research notes and system map for a low-latency onchain execution engine built
for Base.

This repository is designed to document the engine as a market system rather
than a collection of isolated protocol notes. The focus is execution:

- how market data is ingested
- how local pool state is maintained
- how opportunities are detected and simulated
- how risk is enforced
- how transactions are built, submitted, and evaluated

## Scope

The underlying engine targets execution workloads that depend on both canonical
chain state and pre-confirmation signals. In practical terms, that means:

- monitoring Base node streams
- consuming Base Flashblocks-style pre-confirmation updates
- maintaining engine-owned views of V2 and V3 pool state
- scanning for route-based opportunities
- simulating and risk-checking candidate trades
- executing through an onchain router with atomic settlement constraints

This repository exists to explain those moving parts, clarify the current
implementation boundary, and identify the most valuable directions for
extension.

## System Overview

The engine can be understood as a pipeline with six layers.

### 1. Ingest

The runtime consumes logs, blocks, transactions, and pre-confirmation data.
The objective of this layer is not just connectivity, but normalization:
external payloads must be converted into internal events that can drive the
rest of the system consistently.

Key concerns:

- source latency
- duplicate handling
- pending vs canonical state
- event completeness and decode quality

### 2. Market State

The engine maintains its own view of pools instead of depending on fresh RPC
queries for every decision. This local state is what makes hot-path decisions
possible.

For V2-style pools, the essential state is:

- token pair
- reserves
- fee
- last update context

For V3-style pools, the essential state is:

- token pair
- fee tier
- tick spacing
- `sqrtPriceX96`
- active liquidity
- current tick

The quality of this layer determines whether the engine is merely fast or both
fast and trustworthy.

### 3. Opportunity Detection

Once the state is live, the engine enumerates candidate routes and ranks them
by expected value. This is where market structure becomes strategy.

The current center of gravity is route-based arbitrage, especially cyclic
paths that begin and end in the same settlement asset.

### 4. Simulation

A route that looks profitable in theory still needs to survive execution-aware
validation. The engine uses deterministic local quote math for hot-path
filtering and can optionally fall back to slower RPC-based preflight checks.

Important questions in this layer:

- when local simulation is sufficient
- when slow-path validation is worth the latency
- how pending-state simulation should evolve on Base

### 5. Risk

A useful execution engine is not only selective about profit, but disciplined
about survival. Risk controls exist to prevent the engine from turning local
model error, market churn, or infrastructure failures into repeated losses.

Typical controls include:

- minimum profit thresholds
- slippage ceilings
- token allowlists and denylists
- per-trade and rolling notional caps
- kill switches
- consecutive-revert circuit breakers

### 6. Execution and Evaluation

After a candidate clears risk, it becomes an execution problem:

- build calldata
- prepare signer, gas, and nonce state
- submit through the chosen transport
- observe inclusion, revert, drop, or replacement
- attribute realized surplus and fee cost

This is also where the off-chain system meets the on-chain router and its
invariants.

## Protocol Coverage

The research surface is driven by what the engine must model and execute.

### Uniswap V2

This is the simplest and currently most direct trading primitive in the stack.
It matters because reserve-based exact-input quote math is cheap, deterministic,
and well suited to hot-path opportunity detection.

Topics worth covering deeply:

- constant product math
- fee-adjusted quoting
- reserve updates from logs
- route construction from known pairs
- liquidity quality filters

### Uniswap V3

V3 matters because much of the next technical depth of the engine lives here.
Concentrated liquidity introduces richer opportunity classes but also makes
local modeling harder.

Topics worth covering deeply:

- `sqrtPriceX96`
- active liquidity
- tick spacing
- in-range quote math
- initialized tick liquidity
- swap behavior across tick crossings

### Aerodrome

Aerodrome is strategically important on Base because it expands venue coverage
while still fitting familiar abstractions:

- V2-like semantics for some pools
- V3-like semantics for concentrated liquidity paths

Research here should focus less on generic protocol theory and more on what
must change in discovery, quoting, and routing to support the venue well.

### Balancer Flash Loans

Flash-loan-backed execution matters because it changes capital requirements and
execution shape. The interesting questions are not just how flash loans work,
but when they are operationally superior to self-funded routes and which
failure modes they introduce.

## Base-Specific Infra

Base pre-confirmation infrastructure is one of the strongest reasons to study
this engine as a distinct system rather than a generic EVM bot.

Flashblocks-style feeds can provide:

- sub-block state updates
- pending logs
- pre-confirmation transaction visibility
- pending-state simulation opportunities

That creates a different design space for latency-sensitive strategies,
especially reactive execution. It also introduces new reconciliation problems:
pending-state information is useful precisely because it arrives before full
finality.

## Current Implementation Boundary

At the time of writing, the active slice of the engine is best described as:

- low-latency ingest from canonical sources plus Base pre-confirmation signals
- discovery and hydration of supported pools
- local V2 and partial V3 market state maintenance
- two-leg V2 loop detection
- deterministic local simulation with optional RPC validation
- configurable risk gates
- atomic multi-venue execution through a router contract

The most important partial or missing areas are:

- richer pending transaction understanding
- complete V3 quote support across initialized ticks
- opportunity classes beyond simple two-leg loops
- replay and attribution tooling
- more production-grade operator and signer workflows

## Research Priorities

If the goal is to understand and extend the engine in the most leverageful
order, the best sequence is:

1. Understand the ingest-to-execution pipeline end to end.
2. Master V2 state and quote math.
3. Learn V3 active-state modeling and its current limitations.
4. Study Base pre-confirmation infrastructure and pending-state simulation.
5. Analyze why profitable local candidates fail or succeed in realized execution.

The most valuable extension paths are likely:

- Base-aware pending transaction decoding
- complete V3 state and quote support
- execution replay and attribution tooling
- richer opportunity classes such as backrun and triangle routing
- solver or filler-style infrastructure built on top of the execution core

## Publishing Standard

This repository should remain publishable and self-contained.

That means:

- no private operational details
- no credentials, endpoints, or production secrets
- no internal conversation transcripts or chat-style notes
- no assumptions that the reader has access to unpublished context

The README should stand on its own as a clear public description of the
research surface, the system boundary, and the roadmap.
