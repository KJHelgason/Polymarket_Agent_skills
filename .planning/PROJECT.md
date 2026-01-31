# Polymarket Agent Skills

## What This Is

A comprehensive Polymarket knowledge package for Claude agents — complete API documentation, WebSocket documentation, py-clob-client library patterns, edge cases, and best practices. When users invoke these skills, Claude has deep expertise in Polymarket integration without needing to research or guess.

## Core Value

Claude knows Polymarket as well as someone who's built with it extensively — no guessing at endpoints, no common mistakes like wrong minimum order amounts, no ambiguity about what API fields actually mean.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Complete REST API documentation with all endpoints, request/response schemas, auth patterns
- [ ] Complete WebSocket documentation with connection patterns, message types, subscription handling
- [ ] py-clob-client library reference — initialization, common patterns, method signatures
- [ ] Edge cases and constraints documentation (5 share minimum, $1 minimum, etc.)
- [ ] Ambiguous field explanations (market end status, outcome resolution, etc.)
- [ ] Best practices guide — when to use REST vs WebSocket, rate limiting, error handling
- [ ] Claude skill that loads this knowledge into context
- [ ] Shareable package structure for other Claude users

### Out of Scope

- Trading bot logic — this is knowledge/documentation, not a trading system
- Account management UI — just the API knowledge layer
- Backtesting frameworks — focus is on API expertise, not strategy testing

## Context

**Why this exists:**
Claude currently makes mistakes when implementing Polymarket integrations due to lack of domain knowledge. Common issues include:
- Not knowing minimum order constraints
- Misinterpreting ambiguous JSON response fields
- Using wrong authentication patterns
- Not knowing when REST vs WebSocket is appropriate

**Target users:**
- The author (primary)
- Other Claude users who work with Polymarket (shareable)

**Key resources to research:**
- Polymarket official API documentation
- Polymarket WebSocket documentation
- py-clob-client GitHub repository (official Python client)
- Real-world usage patterns and gotchas

## Constraints

- **Format**: Must work as Claude Code skills (markdown-based, loadable via slash commands)
- **Accuracy**: Documentation must be verified against actual API behavior, not just official docs
- **Completeness**: Cover ALL endpoints and features, not just common ones
- **Shareability**: Structure must allow other users to install and use

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Skills-based approach | Allows selective loading of Polymarket knowledge when needed, shareable format | — Pending |
| Deep research before documenting | Official docs may be incomplete or ambiguous; need to understand actual behavior | — Pending |

---
*Last updated: 2025-01-31 after initialization*
