# ADR-0009: LLM usage and API key policy

- **Status:** Accepted
- **Date:** 2026-11-18
- **Related:** Synthesis TD-7, R-2; ADR-0005; AI architecture research §7

## Context
The budget is near zero, but research shows:
- Gemini's free tier uses submitted content for product improvement and allows human review; Google limits EEA/UK/CH users to paid services.
- Groq's free plan doesn't train on data and allows serving end users, but forbids use "to avoid incurring Fees" (no pooling of free accounts) and has tight token/minute limits.
- Gemini 3.x Flash paid prices double on 2027-01-01.

## Decision
- **Development:** free Gemini (Flash-Lite and others) and free Groq, with **synthetic or our own non-sensitive data only**. Tests use recorded fixtures, not live calls.
- **Production:** each tenant uses **their own keys (BYOK)** or **paid platform keys** with per-tenant caps. No pooling of free accounts.
- **Free-tier keys** supplied by tenants are flagged; the product warns about training/review terms and **refuses** them for EEA/UK/CH tenants and for workflows that touch email or personal data.
- **Cost per user per month** is a tracked non-functional requirement; routing prefers the cheapest model that passes the step's evals.

## Consequences
- Key storage, key validation and key-tier detection are MVP features.
- Unit economics are modelled in the requirements phase before pricing is fixed.

## Revisit when
Provider terms or prices change materially (check quarterly).
