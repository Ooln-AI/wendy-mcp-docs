# Wendy MCP Partner Integration Guide

Distribution classification: partner-facing; share only after commercial and security review.

Document version: 1.0  
Response schema version: 1.2  
Last updated: 2026-09-29

## Purpose and scope

Wendy exposes read-only market analytics to approved partners through the Model Context Protocol
(MCP). This guide describes the public integration contract: how to connect, discover tools, make
calls, interpret results, and handle failures.

The service returns structured snapshot data. It does not place trades, select investments, or
make recommendations. Calculation methods, source code, internal prompts, model coefficients,
market-data sourcing, infrastructure, caching, operational thresholds, and security controls are
proprietary and are not part of the partner contract.

The live MCP `tools/list` response is authoritative for tool descriptions, input JSON Schemas,
enums, defaults, and bounds.

## Connection

Wendy uses MCP over Streamable HTTP.

| Item | Value |
|---|---|
| Endpoint | Supplied during onboarding; it ends in `/mcp` |
| Authentication | Credential and header scheme supplied during onboarding |
| Session model | Stateless; the client retains conversation state |
| Tool behavior | Read-only, non-destructive, and idempotent for the same data snapshot |
| Request format | MCP JSON-RPC, normally handled by an MCP SDK |

Keep credentials in a server-side secret store. Never embed them in browser or mobile code, commit
them to source control, or include them in support tickets and logs. Partners connect only through
the endpoint issued during onboarding and are not given access to Wendy's internal infrastructure.

Depending on the agreed authentication profile, the issued credential is sent in one of these
forms:

```http
Authorization: Bearer <credential>
```

or

```http
X-API-Key: <credential>
```

Use only the scheme assigned to your integration.

## Quick start with Python

Install the official MCP Python SDK, then set the endpoint and credential in the environment:

```bash
pip install mcp
export WENDY_MCP_URL="https://partner-endpoint.example/mcp"
export WENDY_MCP_TOKEN="replace-with-issued-credential"
```

This example initializes a session, discovers the current schema, and makes one tool call:

```python
import asyncio
import os
from datetime import timedelta

from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client


async def main() -> None:
    headers = {"Authorization": f"Bearer {os.environ['WENDY_MCP_TOKEN']}"}

    async with (
        streamablehttp_client(
            os.environ["WENDY_MCP_URL"],
            headers=headers,
            timeout=timedelta(seconds=30),
            sse_read_timeout=timedelta(seconds=180),
        ) as (read, write, _),
        ClientSession(
            read,
            write,
            read_timeout_seconds=timedelta(seconds=180),
        ) as session,
    ):
        await session.initialize()

        catalog = await session.list_tools()
        for tool in catalog.tools:
            print(tool.name, tool.inputSchema)

        result = await session.call_tool(
            "support_resistance",
            {
                "ticker": "SPX",
                "mode": "intraday",
                "components": ["nearest_support", "nearest_resistance"],
            },
        )
        print(result.structuredContent)


asyncio.run(main())
```

For an API-key profile, change the example header to
`{"X-API-Key": os.environ["WENDY_MCP_TOKEN"]}`.

## Recommended client flow

1. Open an MCP Streamable HTTP connection and call `initialize`.
2. Call `tools/list` and use the returned schemas for routing and validation.
3. Call `tools/call` with one public tool name and arguments that validate against its schema.
4. Read `structuredContent` from the MCP result and inspect its top-level `status`.
5. Store `request_id` with your trace so Wendy support can correlate a specific call.

Do not hard-code a private copy of the input schemas. Cache discovery briefly if needed, but
refresh it on startup or deployment so additive contract changes do not require a client release.
Ignore unrecognized response fields and branch on documented status and reason-code values.

## Tool catalog

### `support_resistance`

Locates structural support and resistance and reports the evidence and current state of requested
levels.

- Required: `ticker`
- Optional: `mode` (`intraday` or `swing`; default `intraday`), `components`
- Components: `structural_levels`, `nearest_support`, `nearest_resistance`, `level_ladder`,
  `confluence`, `broken_levels`
- Use for: price structure, nearby levels, confluence, breaks, retests, and reclaims
- Do not use for: gamma walls, forward ranges, or named technical indicators

Example arguments:

```json
{
  "ticker": "SPX",
  "mode": "intraday",
  "components": ["nearest_support", "nearest_resistance", "confluence"]
}
```

### `gamma_exposure`

Returns modeled option-chain gamma context. The results are estimates based on observable market
data and do not represent known dealer inventory.

- Required: `ticker`
- Optional: exact `expiration` in `YYYY-MM-DD`, `components`
- Components: `put_wall`, `call_wall`, `gamma_flip`, `dealer_gamma_state`, `net_gex`,
  `strike_concentrations`, `spot_context`
- Supported instruments: options-supported equities, ETFs, and supported cash indices
- Do not use for: structural price levels or statistical ranges

Example arguments:

```json
{
  "ticker": "SPX",
  "expiration": "2027-01-15",
  "components": ["put_wall", "call_wall", "gamma_flip", "spot_context"]
}
```

### `technical_indicators`

Calculates one or more explicitly requested technical indicators. The caller supplies the
indicator type and customer-selected parameters; Wendy does not infer missing choices.

- Required: `ticker`, `indicators` (one through twelve indicator requests)
- Optional: `include_partial`, `session` (`regular` or `extended`), `adjusted`, `as_of`
- Indicator types: `ema`, `sma`, `rsi`, `atr`, `atr_state`, `adx`, `macd`, `bollinger`,
  `squeeze_momentum`, `stochastic`, `stoch_rsi`, `obv`, `mfi`, `cci`, `williams_r`,
  `parabolic_sar`, `vwap`, `volume_profile`, `relative_strength`, `crossover`,
  `reference_levels`, `historical_gaps`, `ichimoku`, `realized_volatility`, and
  `fibonacci_retracement`
- Use for: a calculation the user named, with explicit parameters
- Do not use for: general market conditions, structural levels, gamma, or forward ranges

Each member of `indicators` is selected by its `type` discriminator. Read its exact required and
optional fields from `tools/list`.

Example arguments:

```json
{
  "ticker": "NVDA",
  "indicators": [
    {"type": "ema", "period": 20, "timeframe": "1d"},
    {"type": "rsi", "period": 14, "timeframe": "1d"}
  ],
  "include_partial": true,
  "session": "regular",
  "adjusted": true
}
```

Supported indicator timeframes are `1m`, `2m`, `3m`, `5m`, `10m`, `15m`, `30m`, `45m`, `1h`,
`2h`, `3h`, `4h`, `1d`, `1w`, `2w`, and `1M`. Timeframes are case-sensitive: `1m` is one minute;
`1M` is one calendar month.

### `market_regime`

Describes current market conditions and can compare them with the previous or similar sessions.
It distinguishes the latest tape state from the shape of the full session.

- Required: `ticker`
- Optional: `components`, `comparison_alignment`, `as_of`, `similarity_lookback_sessions`
- Comparison alignment: `same_elapsed_session_time` (default) or
  `completed_previous_session`
- Components: `current_regime`, `previous_regime`, `comparison`, `session_evidence`,
  `previous_session_evidence`, `trend`, `strength`, `volatility`, `momentum`, `participation`,
  `timeframe_alignment`, `similar_sessions`, `realized_move_context`
- Use for: trending/ranging conditions, session character, current-session VWAP, today's gap,
  cross-timeframe agreement, and historical comparisons

Example arguments:

```json
{
  "ticker": "NVDA",
  "components": ["current_regime", "session_evidence", "timeframe_alignment"],
  "comparison_alignment": "same_elapsed_session_time"
}
```

### `expected_range`

Returns symmetric one- and two-standard-deviation statistical bounds for the remaining session or
a multi-session horizon. It expresses magnitude, not direction, a price target, or the probability
that a bound will be reached.

- Required: `ticker`
- Optional: `mode`, `swing_horizon`, `include_path`, `supplied_volatility`, `components`
- Modes: `intraday` (default) or `swing`
- Components: `remaining_session_range` for intraday, `multi_session_range` for swing, and
  optional `range_path` for either mode
- `supplied_volatility`, when used, is annualized decimal volatility (`0.25` means 25%)

Example arguments:

```json
{
  "ticker": "NVDA",
  "mode": "swing",
  "swing_horizon": 10,
  "include_path": false,
  "components": ["multi_session_range"]
}
```

### `price_action`

Reports confirmed price-structure events and associated origin zones, fair-value gaps, and
liquidity references. Events are confirmed on completed bars.

- Required: `ticker`
- Optional: `timeframe`, `as_of`, `components`
- Timeframes: `15m`, `30m`, `1h` (default), `4h`, `1d`
- Components: `structure_events`, `origin_zones`, `fair_value_gaps`, `liquidity_references`
- Use for: confirmed BOS/CHoCH events, zone state, gap-fill state, and equal highs/lows
- Do not use for: a consolidated support-and-resistance map

Example arguments:

```json
{
  "ticker": "BTC/USD",
  "timeframe": "4h",
  "components": ["structure_events", "fair_value_gaps"]
}
```

### `strike_comparison`

Places one or two customer-selected option contracts in market context. It describes only the
contracts supplied by the caller and never searches, ranks, scores, or recommends contracts.

- Required: `ticker`, `contracts` (one or two exact contracts)
- Each contract requires: `expiration` (`YYYY-MM-DD`), `option_type` (`call` or `put`), and a
  positive `strike`
- Supported instruments: options-supported equities, ETFs, and supported cash indices
- Two-contract results express numeric differences in the order supplied, without a preference

Example arguments:

```json
{
  "ticker": "NVDA",
  "contracts": [
    {"expiration": "2027-01-15", "option_type": "call", "strike": 650}
  ]
}
```

## Instruments and input conventions

The general market tools accept equities, ETFs, supported cash indices, crypto pairs, and FX
pairs. Option-dependent tools require a supported listed option chain. Futures are not part of the
public contract. An otherwise valid component may be unavailable or inapplicable for a particular
asset—for example, a cash index does not have trade volume.

Follow these conventions:

- Treat all enums and symbol formats as case-sensitive unless the live schema says otherwise.
- Use ISO `YYYY-MM-DD` for expirations.
- Use an ISO date/time with an explicit UTC offset, or epoch time, for historical `as_of` calls.
- Do not send undeclared properties; nested input objects use strict schemas.
- Do not send `rh_user_id` or another customer identifier. The partner gateway derives attribution
  from the authenticated identity.
- Do not put natural-language questions in tool arguments. The partner application or agent routes
  the question to a tool and sends structured arguments.

## Response contract

A successful MCP transport call returns a structured Wendy response in `structuredContent`.
Clients should parse the envelope before interpreting component-specific values:

```json
{
  "schema_version": "1.2",
  "disclosure": "Snapshot data only, not investment advice or a recommendation. You decide.",
  "request_id": "correlation-id",
  "received_at": "2026-09-29T14:30:00Z",
  "completed_at": "2026-09-29T14:30:01Z",
  "duration_ms": 1000,
  "status": "complete",
  "symbol": "NVDA",
  "skills": ["technical_indicators"],
  "completed": [
    {
      "skill": "technical_indicators",
      "component": "ema",
      "value": {"...": "component-specific structured data"},
      "observed_at": {"bars": "source observation timestamp"},
      "parameters": {"period": 20, "timeframe": "1d"},
      "bar_status": "completed",
      "window_status": null
    }
  ],
  "unavailable": [],
  "skipped": [],
  "unsupported": [],
  "clarification": null
}
```

The example above illustrates the envelope, not a quoted market result. Fields inside `value` vary
by component. Preserve `observed_at`, `bar_status`, and `window_status` when presenting a result so
users can understand freshness and whether a forming bar contributed.

Top-level statuses:

| Status | Meaning | Client action |
|---|---|---|
| `complete` | All requested, applicable components completed | Present the result |
| `partial` | Some components completed and others did not | Present completed data and disclose gaps |
| `clarification_required` | A customer-selected input is missing or invalid | Ask for the fields in `clarification.missing`, then make a new call |
| `unsupported` | The request or all requested components are outside the contract | Do not retry unchanged |
| `unavailable` | No requested component could complete at this time | Inspect gap metadata and retry only when allowed |

Every item in `unavailable`, `skipped`, or `unsupported` includes a machine-readable
`reason_code`, an `error_class`, and a `retryable` flag. Use those fields instead of matching human
text:

| `error_class` | Meaning | Client action |
|---|---|---|
| `needs_input` | The caller must supply or correct an argument | Correct the request; do not auto-fill a customer choice |
| `permanent` | The same request is not expected to work | Change the request or explain the limitation |
| `transient` | A temporary dependency, capacity, or timing condition occurred | Retry only when `retryable` is `true` |

When `clarification_required` is returned, `clarification.example_args` is a syntax example only.
Do not execute it automatically or treat its values as user intent.

## HTTP and MCP errors

Application-level statuses above can arrive in a successful MCP call and are separate from
transport or protocol failures.

| Signal | Meaning | Client action |
|---|---|---|
| HTTP `401` | Missing, expired, or invalid credential | Refresh or correct credentials; do not retry in a tight loop |
| HTTP `429` | Request rate limit exceeded | Honor `Retry-After`, add jitter, and retry within the agreed policy |
| HTTP `5xx` or connection timeout | Temporary service or network failure | Retry idempotently with capped exponential backoff |
| MCP `isError: true` | Protocol, tool name, or schema validation failure | Log the sanitized error and correct the call |
| Wendy `status: unavailable` or `partial` | Tool executed but data or a component was unavailable | Follow per-gap `retryable` metadata |

Recommended retry behavior is exponential backoff with jitter and a fixed attempt cap. Never retry
`needs_input` or `permanent` results unchanged. Because calls are read-only and do not create
orders or resources, retrying a transient failure is safe.

## Partner integration requirements

- Keep credentials server-side and rotate them through the agreed onboarding channel.
- Validate tool arguments against the live schema before calling.
- Preserve Wendy's disclosure when showing data to an end user.
- Do not describe modeled gamma as observed dealer inventory.
- Do not turn expected-range bounds into directional targets or guaranteed probabilities.
- Do not use `strike_comparison` to imply ranking, screening, or a recommendation.
- Retain `request_id`, tool name, top-level status, and timestamps in sanitized diagnostics.
- Do not log credentials or introduce personal information into tool arguments.
- Agree on quotas, allowed environments, support contacts, and credential rotation separately; they
  are intentionally not encoded in this public interface guide.

## Support information to provide

For an integration issue, provide:

- environment name, without the credential or private endpoint query parameters;
- UTC timestamp;
- `request_id`, if one was returned;
- MCP method or tool name;
- HTTP status, MCP `isError`, Wendy top-level `status`, and `reason_code` values;
- a redacted argument shape and the client SDK name/version.

Never send access tokens, API keys, raw authorization headers, or customer personal information.
