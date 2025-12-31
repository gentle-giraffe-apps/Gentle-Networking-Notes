# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a documentation repository containing practical networking and API design patterns for mobile engineers, written from a senior iOS perspective. There is no code to build, test, or lint—only markdown articles.

## Structure

```
01_fundamentals/      Mental models, request lifecycle, idempotency, consistency, versioning
02_rest_api_design/   Resource modeling, errors, pagination, rate limiting, currency, security
03_scalability/       Caching layers, CDN strategies, hotspot handling
04_reliability/       Circuit breakers, graceful degradation, offline-first, partial failures
05_testing/           Contract testing, mocking vs replay, integration harness, chaos testing
06_case_studies/      End-to-end API designs applying patterns from all chapters
07_interview_drills/  System design interview preparation for mobile engineers
```

## Core Mental Models

When editing or extending these notes, maintain consistency with these architectural positions:

- **Mobile as unreliable edge compute node**: Design around network loss, app suspension, token expiry, and schema drift as the default state
- **The Mobile Triangle**: Latency, Battery, Consistency—you can only optimize two
- **Offline-first as posture**: Network is a sync layer, not source of truth; UI reads from local state
- **Idempotency is non-negotiable**: Every write must be safely retryable
- **Additive API evolution**: New meaning requires new field names; prefer capability negotiation over versioning
- **Errors are product states**: Classify into transport/auth/domain/system with explicit handling rules

## Writing Style

Each article should end with a **Key Concept** block—a quotable summary suitable for articulating design decisions in interviews or architecture discussions.

## Authorship Approach

These notes use LLM-assisted drafting. The author takes responsibility for topics chosen, architectural positions, trade-offs emphasized, and practical guidance—not line-by-line prose authorship.
