# iSighted Sight Token Specification v1.0

A minimal open standard for anchoring reviews to verified transactions.

The problem with every existing review platform is structural: nothing stops a review from being written by someone who never had the transaction. This spec defines a **Sight Token** — a cryptographically signed record generated at the moment of transaction — that a review must be anchored to before it can be published.

No token. No review. That's the entire idea.

---

## Scope of This Specification

This document defines:

- The Sight Token data schema
- The token generation algorithm
- The review submission contract
- The public verification endpoint contract
- The privacy requirements

This document does **not** define:

- How tokens are delivered to customers (QR, email, SMS, NFC — implementation choice)
- Where reviews are stored or displayed
- Business dashboard or analytics features
- Any commercial product built on top of this standard

Those are implementation concerns. This spec is the protocol layer only.

---

## Core Principle

**Privacy is architectural, not a policy.**

A Sight Token proves a transaction occurred without revealing who the customer is, what they paid, or any personally identifiable information. This is not a setting — it is enforced by what the token schema is permitted to contain.

---

## Sight Token Schema

```json
{
  "spec_version": "1.0",
  "token_id": "uuid-v4",
  "token_hash": "sha256(business_id + customer_email_hash + transaction_date + transaction_type + issued_at)",
  "business_id": "string — verified business identifier",
  "customer_email_hash": "sha256(lowercase(customer_email))",
  "transaction_date": "ISO 8601 date — YYYY-MM-DD",
  "transaction_type": "string — service or product category",
  "transaction_value_band": "optional string — low | mid | high",
  "issued_at": "ISO 8601 datetime — UTC",
  "expires_for_review": "ISO 8601 datetime | null — null means no expiry",
  "verification_url": "https://{host}/verify/{token_hash}",
  "schema_version": "1.0"
}
```

### Field Rules

| Field | Required | Notes |
|-------|----------|-------|
| `spec_version` | Yes | Must be "1.0" for this version |
| `token_id` | Yes | UUID v4. Generated fresh per token. |
| `token_hash` | Yes | SHA-256. See algorithm below. |
| `business_id` | Yes | Defined by the implementing platform. Must be stable and unique. |
| `customer_email_hash` | Yes | SHA-256 of lowercased email. Never store plaintext email in token. |
| `transaction_date` | Yes | Date only — not datetime. Protects appointment time privacy. |
| `transaction_type` | Yes | Human-readable category. E.g. "dental_implant", "legal_consultation". |
| `transaction_value_band` | No | Optional broad band. Never exact amount. |
| `issued_at` | Yes | UTC datetime of token generation. |
| `expires_for_review` | No | Null means the token never expires for review purposes. |
| `verification_url` | Yes | Publicly accessible URL returning verification response (see below). |

### What a Token Must Never Contain

- Customer name
- Customer email in plaintext
- Transaction amount
- Any government or national ID
- Any health record or diagnosis
- Any payment instrument detail

Implementations that include any of the above fields are non-compliant with this specification.

---

## Token Hash Algorithm

```
token_hash = SHA-256(
  business_id
  + "|"
  + sha256(lowercase(customer_email))
  + "|"
  + transaction_date         // YYYY-MM-DD
  + "|"
  + transaction_type
  + "|"
  + issued_at                // ISO 8601 UTC
)
```

Pipe characters (`|`) are used as delimiters. All string values are UTF-8 encoded before hashing. The outer hash is hex-encoded lowercase.

### Reference Implementation (Node.js)

```javascript
const crypto = require('crypto')

function generateTrustToken({ businessId, customerEmail, transactionDate, transactionType }) {
  const issuedAt = new Date().toISOString()
  const emailHash = crypto
    .createHash('sha256')
    .update(customerEmail.toLowerCase().trim())
    .digest('hex')

  const hashInput = [businessId, emailHash, transactionDate, transactionType, issuedAt].join('|')
  const tokenHash = crypto.createHash('sha256').update(hashInput).digest('hex')

  return {
    spec_version: '1.0',
    token_id: crypto.randomUUID(),
    token_hash: tokenHash,
    business_id: businessId,
    customer_email_hash: emailHash,
    transaction_date: transactionDate,
    transaction_type: transactionType,
    issued_at: issuedAt,
    expires_for_review: null,
    verification_url: `https://isighted.io/verify/${tokenHash}`,
    schema_version: '1.0',
  }
}
```

---

## Review Submission Contract

A review may only be published if it is anchored to a valid Sight Token.

### Submission Payload

```json
{
  "token_hash": "string — must match an issued, unused token",
  "submitter_email_hash": "sha256(lowercase(submitter_email))",
  "rating": "integer 1–5",
  "transaction_type_tag": "string — must match token.transaction_type",
  "comment": "string | null — optional free text",
  "submitted_at": "ISO 8601 datetime — UTC"
}
```

### Validation Rules

A submission is **rejected** if any of the following are true:

1. `token_hash` does not match any issued token
2. `submitter_email_hash` does not match `token.customer_email_hash`
3. `rating` is not an integer between 1 and 5 inclusive
4. `transaction_type_tag` does not match `token.transaction_type`
5. The token has already been used for a published review (one token, one review)
6. The token has expired (if `expires_for_review` is set and is in the past)

### Rejection Is Silent on Mismatch Reason

When rejecting due to rules 1 or 2, implementations must not reveal whether the token exists or whether the email matched. Return a generic validation failure. This prevents token enumeration attacks.

---

## Public Verification Endpoint

Every published review must carry a `verification_url`. That URL must return the following response to any caller without authentication.

### Response Schema

```json
{
  "verified": true,
  "business_name": "string",
  "transaction_date": "YYYY-MM-DD",
  "transaction_type": "string",
  "token_hash": "string",
  "hash_algorithm": "SHA-256",
  "review_submitted_at": "ISO 8601 datetime",
  "spec_version": "1.0"
}
```

### What the Verification Endpoint Must Not Return

- Customer email or email hash
- Customer name or any PII
- Transaction amount or value
- Business internal identifiers beyond `business_name`

### HTTP Requirements

- Must respond to unauthenticated `GET` requests
- Must return `Content-Type: application/json`
- Must return HTTP 200 for verified tokens
- Must return HTTP 404 for unknown token hashes
- Should support CORS (`Access-Control-Allow-Origin: *`) to allow browser-based verification

---

## Anti-Gaming Requirements

Implementations claiming compliance with this specification must enforce the following at minimum:

| Rule | Requirement |
|------|-------------|
| One review per token | A token may only be used to submit one review. Reuse must be rejected. |
| Email binding | The submitter's email hash must match the token's `customer_email_hash`. |
| Token integrity | The published `token_hash` must be verifiable by recomputing from available fields. |
| No business deletion | A published review may not be deleted by the business. The business may flag a review for dispute, but may not unilaterally remove it. |
| Immutability | Once published, a review's rating, comment, and token hash must not be editable by any party. |

Platforms that permit businesses to pay for review removal, suppression, or reordering are non-compliant with this specification.

---

## Schema.org Alignment

Sight Token verification data maps to `schema.org/Review` as follows:

```json
{
  "@context": "https://schema.org",
  "@type": "Review",
  "reviewRating": {
    "@type": "Rating",
    "ratingValue": 5,
    "bestRating": 5
  },
  "itemReviewed": {
    "@type": "LocalBusiness",
    "name": "A2Z Dental"
  },
  "datePublished": "2026-04-14",
  "description": "Optional review comment text",
  "additionalProperty": {
    "@type": "PropertyValue",
    "name": "TrustToken",
    "value": "a7f3d2e1b9c04821",
    "propertyID": "https://isighted.io/verify/a7f3d2e1b9c04821"
  }
}
```

This structured data allows AI systems, search engines, and other consumers to identify and cite transaction-verified reviews.

---

## Versioning

This specification uses semantic versioning.

- **Patch versions (1.0.x):** Clarifications and editorial corrections. No schema changes.
- **Minor versions (1.x.0):** Additive, backwards-compatible changes. New optional fields may be added.
- **Major versions (x.0.0):** Breaking changes. New major versions are published as separate specifications.

Implementations must include `spec_version` in every token so that verification systems can handle multiple versions simultaneously.

---

## What Is Deliberately Out of Scope

The following are real and important problems. They are not addressed in v1.0 because they are unproven at scale. They will be considered for future versions when there is production evidence to inform the design.

- **Media attachments** — anchoring photos or videos to a Sight Token
- **Reviewer reputation profiles** — building trust scores for reviewers across transactions
- **Zero-knowledge identity proofs** — proving attributes (local resident, verified professional) without revealing identity
- **B2B transaction trust** — business-to-business review anchoring
- **Agent-to-agent transaction verification** — autonomous AI commerce

Publishing unproven features in an open standard creates false expectations and fragmented implementations. v1.0 is the minimum spec that is genuinely defensible.

---

## Reference Implementation

A reference implementation of token generation, submission validation, and the verification endpoint is available at:

`https://github.com/isighted/trust-token-spec`

The reference implementation is MIT licensed. The specification itself is published under CC BY 4.0.

---

## Contributing

This specification is maintained by iSighted. Community proposals are welcome via GitHub Issues.

Before proposing an addition, ask: *is this proven in production, or is it speculative?* Speculative features belong in a discussion thread, not a specification.

---

*iSighted Sight Token Specification v1.0*
*Published April 2026*
*CC BY 4.0 — https://creativecommons.org/licenses/by/4.0/*
