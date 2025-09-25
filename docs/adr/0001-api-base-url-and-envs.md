# ADR 0001 — API Base URL and Environment Separation

**Status:** Accepted  
**Date:** 2025-09-24

## Context
The app relies on a backend (AppRunner) exposing `/documents` and `/search`. Mixing base URLs (e.g., API Gateway vs AppRunner) caused 404/500 errors and inconsistent response shapes.

## Decision
Use a single source of truth for the base URL via `vBaseUrl` and keep environment-specific values in a single header section. Do not mix origins (AppRunner vs API Gateway) in the same deployment.

## Alternatives Considered
- Multiple base URLs chosen ad hoc → prone to drift and runtime mismatches.
- Single base URL with feature flags → simpler, but requires a reliable place to toggle.

## Consequences
- Positive: Fewer 404/500, easier debugging, predictable payloads.
- Negative: Requires disciplined config management across environments.

## References
- `HEADER` script sets `vBaseUrl`.
