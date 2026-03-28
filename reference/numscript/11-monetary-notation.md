# Unambiguous Monetary Notation (UMN)

> Source: <https://docs.formance.com/resources/monetary-notation>

## Overview

The Formance Platform employs a standardized approach called Unambiguous Monetary Notation
(UMN) for representing monetary values safely across all services. While custom assets
following the pattern `[A-Z]{1,16}(/\d{1,6})` are supported, UMN is strongly recommended,
particularly for ISO-4217 standard currencies.

## Format Specification

```text
[ASSET/SCALE AMOUNT]
```

### Components

- **ASSET**: One to sixteen uppercase letters denoting the currency code (standardized or
  custom)
- **SCALE**: Negative power of ten applied to the amount for converting to decimal value
- **AMOUNT**: Unsigned integer value

### Example

`[USD/2 30]` equals USD 30 x 10^-2, or USD 0.30 (thirty cents).

### Scale Omission Rule

For values where the amount already represents the full amount of said asset, a scale of zero
should not be represented. Example: `[JPY 100]` instead of `[JPY/0 100]`.

## Reference Table

| UMN Notation | Decimal Equivalent | Currency |
|---|---|---|
| `[USD/2 30]` | $0.30 | US Dollar |
| `[JPY 100]` | 100 yen | Japanese Yen |
| `[BTC/8 100000000]` | 1 BTC | Bitcoin |
| `[GBP/2 100]` | 1.00 GBP | British Pound |
| `[EUR/2 100]` | 1.00 EUR | Euro |
| `[INR/2 100]` | 1.00 INR | Indian Rupee |
| `[CNY/2 100]` | 1.00 CNY | Chinese Yuan |
| `[CAD/2 100]` | 1.00 CAD | Canadian Dollar |

## Common Scales

| Currency | Typical Scale | Meaning |
|----------|--------------|---------|
| USD | `/2` | Cents (10^-2) |
| EUR | `/2` | Cents (10^-2) |
| GBP | `/2` | Pence (10^-2) |
| JPY | (none) | Yen (no subdivision) |
| BTC | `/8` | Satoshis (10^-8) |
| ETH | `/18` | Wei (10^-18) |

## Precision Handling

UMN doesn't mandate specific precision requirements beyond requiring unsigned integer
representation. Implementation decisions regarding precision belong to third-party
implementers. Formance Stack components internally utilize arbitrary precision unsigned
integers.

## Flexibility

Organizations may employ non-standard scales (e.g., `USD/4`, `JPY/2`) when business logic
requires representing subdivisions or when amounts will be rounded later in processing.

## Problem Addressed

Unscaled currency representations like plain `USD` create interpretive ambiguity. Payment
processors often return responses like:

```json
{
  "amount": 100,
  "currency": "USD"
}
```

This leaves unclear whether the value represents cents or dollars -- a critical distinction
at scale.

## Solution

By explicitly encoding scale within the notation itself, UMN eliminates ambiguity by design.
This approach becomes increasingly valuable as organizations integrate multiple payment
service providers using inconsistent formatting standards, reducing catastrophic
misinterpretation risks inherent in converting between disparate systems.

## Custom Assets

Custom asset types are supported for non-currency values:

```numscript
send [COIN 100] (
  source = @world
  destination = @users:001
)

send [GEM/4 5000] (
  source = @treasury
  destination = @users:001
)
```
