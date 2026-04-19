# iSighted Sight Token Specification v1.1
## Published April 2026 — Updated April 2026

---

## What This Is

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
- The trust tier system (Gold / Silver / Bronze)
- The two-sided confirmation requirement
- The AI document verification standard
- The device fingerprinting requirement
- The transaction timing window
- The valid and invalid flag criteria
- The enforcement escalation framework

This document does **not** define:

- Token pricing or commercial billing models
- Subscription or pre-purchase mechanics
- Investigation fees or dispute bond amounts
- Any commercial product built on top of this standard

Those are implementation concerns. This spec is the protocol layer only.

---

## Core Principles

**1. Privacy is architectural, not a policy.**
A Sight Token proves a transaction occurred without revealing who the customer is, what they paid, or any personally identifiable information. PII used during verification is never stored — it is processed, checked, and permanently deleted.

**2. The business is legally accountable for every token it generates.**
By generating a Sight Token, a business makes a legal attestation that the described transaction occurred. Fraudulent token generation is misrepresentation under the laws of any jurisdiction.

**3. No minimum dollar amount.**
Dollar thresholds are not universal — $5 means something completely different in Sydney versus Lagos. Fraud is detected through velocity patterns, business category benchmarks, two-sided confirmation, and device fingerprinting — not transaction value.

**4. The open standard is the moat.**
This protocol is free. Any platform may implement it. Commercial tools built on top are where value is captured. This distinction never reverses.

---

## Business Verification — Required Before Token Generation

No business may generate Sight Tokens until the following verification steps are complete.

### Required verification fields

| Field | Description |
|-------|-------------|
| Legal business name | Must match government registration |
| Business registration number | ABN (Australia), EIN (USA), GST (NZ), VAT (UK/EU), or country equivalent |
| Business owner identity | Government-issued photo ID of the account owner |
| Business address | Physical address verified against registration |
| Contact email | Deliverability confirmed |
| Payment method | Valid payment method on file for token credit |

### Legal agreement

Every business must sign iSighted's terms of service before token generation is enabled. The agreement includes an explicit clause:

> "By generating a Sight Token, the business owner personally attests that the described transaction occurred between the business and a real customer on the stated date. Generating tokens for transactions that did not occur constitutes fraud and misrepresentation, and the business owner accepts personal legal liability for any such tokens."

This clause applies to the individual, not just the entity. It cannot be dissolved by company closure or restructure.

---

## Sight Token Schema

```json
{
  "spec_version": "1.1",
  "token_id": "uuid-v4",
  "token_hash": "sha256(business_id + customer_email_hash + transaction_date + transaction_type + issued_at)",
  "business_id": "string — verified business identifier",
  "customer_email_hash": "sha256(lowercase(customer_email))",
  "transaction_date": "ISO 8601 date — YYYY-MM-DD",
  "transaction_type": "string — service or product category",
  "transaction_value_band": "optional string — low | mid | high",
  "trust_tier": "string — gold | silver | bronze",
  "issued_at": "ISO 8601 datetime — UTC",
  "expires_for_review_min": "token issued_at + 60 seconds",
  "expires_for_review_max": "token issued_at + 30 days",
  "verification_url": "https://{host}/verify/{token_hash}",
  "spec_version": "1.1"
}
```

### What a Token Must Never Contain

- Customer name
- Customer email in plaintext
- Transaction amount
- Any government or national ID
- Any health record or diagnosis
- Any payment instrument detail
- Any device identifier

Implementations that include any of the above fields are non-compliant with this specification.

---

## Trust Tier System

Every Sight Token carries one of three trust tiers. The tier is determined by the level of payment evidence provided at token generation.

### Gold Tier — Payment Processor Confirmed

The transaction has been independently confirmed by a connected payment processor. The processor confirms that a payment from a customer was received by the business on the transaction date.

Supported processors include but are not limited to: Stripe, Square, PayPal, Shopify Payments, M-Pesa, Paytm, UPI-connected processors, and any processor with a compliant API.

The Gold badge is displayed on published reviews. AI search systems and other consumers may weight Gold tier reviews more heavily.

### Silver Tier — Auditable Invoice Verified

The business has uploaded a transaction document (invoice, receipt, or booking confirmation) that has been verified by the AI document verification process defined in this specification.

Supported systems include but are not limited to: Xero, MYOB, QuickBooks, FreshBooks, practice management systems, and any system producing a compliant invoice document.

### Bronze Tier — Manual Token

The token was generated manually without connected payment evidence. Bronze tokens are subject to stricter velocity caps and receive a lighter badge indicator. They are not hidden but are visually distinguished from Gold and Silver.

Bronze tokens are permitted to support:
- Cash-only businesses globally
- Businesses in markets without digital payment infrastructure
- Service categories where payment processing is non-standard

Bronze tokens require enhanced two-sided confirmation before publication.

---

## Token Hash Algorithm

```
token_hash = SHA-256(
  business_id
  + "|"
  + sha256(lowercase(customer_email))
  + "|"
  + transaction_date
  + "|"
  + transaction_type
  + "|"
  + issued_at
)
```

Pipe characters (`|`) are used as delimiters. All string values are UTF-8 encoded. The outer hash is hex-encoded lowercase.

### Reference Implementation (Node.js)

```javascript
const crypto = require('crypto')

function generateSightToken({ businessId, customerEmail, transactionDate, transactionType, trustTier }) {
  const issuedAt = new Date().toISOString()
  const emailHash = crypto
    .createHash('sha256')
    .update(customerEmail.toLowerCase().trim())
    .digest('hex')

  const hashInput = [businessId, emailHash, transactionDate, transactionType, issuedAt].join('|')
  const tokenHash = crypto.createHash('sha256').update(hashInput).digest('hex')

  return {
    spec_version: '1.1',
    token_id: crypto.randomUUID(),
    token_hash: tokenHash,
    business_id: businessId,
    customer_email_hash: emailHash,
    transaction_date: transactionDate,
    transaction_type: transactionType,
    trust_tier: trustTier,
    issued_at: issuedAt,
    expires_for_review_min: new Date(Date.parse(issuedAt) + 60000).toISOString(),
    expires_for_review_max: new Date(Date.parse(issuedAt) + 30 * 24 * 60 * 60 * 1000).toISOString(),
    verification_url: `https://isighted.io/verify/${tokenHash}`,
  }
}
```

---

## AI Document Verification Standard (Silver Tier)

When a business submits a document to elevate a token to Silver tier, the following process must be followed by any compliant implementation.

### Verification flow

1. Business uploads document (invoice, receipt, booking confirmation)
2. Document is encrypted in transit using TLS 1.3 minimum
3. AI system extracts the following fields:
   - Business name
   - Business registration number
   - Transaction date
   - Invoice or reference number
   - Customer name or identifier (used for matching only)
   - Invoice amount (used for fraud pattern detection only — not stored)
4. Extracted fields are cross-referenced against the token data
5. Document is permanently and irrecoverably deleted within 60 seconds of verification
6. Only the verification result is stored: passed / failed / timestamp

### Verification pass criteria

All of the following must be true for a document to pass:

| Check | Requirement |
|-------|-------------|
| Business name | Must match registered business name on file (fuzzy match permitted for trading names) |
| Registration number | Must exactly match registration number submitted at signup |
| Transaction date | Must be within 72 hours of token generation timestamp |
| Invoice number | Must be unique — same invoice cannot verify two tokens |
| Customer presence | Customer name or identifier must be present in document |

### Verification fail criteria

Any of the following causes automatic rejection:

- Business name does not match registered name
- Registration number mismatch
- Transaction date more than 72 hours before or after token generation
- Duplicate invoice number detected
- Document metadata inconsistent with claimed creation date
- Document shows signs of digital manipulation

### Privacy commitment

The customer's name appears in the document for verification purposes only. It is never stored, indexed, logged, or associated with the token record. The verification system processes PII but retains none. This is enforced architecturally, not by policy.

### Language support

Implementations must support document verification in any language. The AI verification layer must be capable of extracting required fields from documents in non-Latin scripts including but not limited to Arabic, Chinese, Japanese, Korean, Hindi, and Cyrillic.

---

## Two-Sided Confirmation Requirement

A review may not be published based on business-side token generation alone. The customer must independently confirm the transaction.

### Confirmation step

When a customer opens the review submission form (via QR, NFC, SMS, email, or any other delivery method), they must answer the following before the rating interface is displayed:

> "Did you visit or transact with [Business Name] on or around [Transaction Date]?"
> 
> **Yes, I did** / **No, I did not**

If the customer selects **No** — the review form does not open. The token is flagged. The business is notified. The flag is logged for pattern analysis.

If the customer selects **Yes** — the review form opens and submission proceeds.

### Why this matters

A review farm operating on behalf of a business must now actively lie on the confirmation step. This converts what would be a terms of service violation into active fraud by a third party. The legal exposure shifts.

A legitimate customer who genuinely did not transact with the business cannot accidentally submit a review — the confirmation gate prevents it.

---

## Transaction Timing Window

### Minimum submission delay

A review may not be submitted until at least **60 seconds** after the token was generated.

This minimum catches automated bot submissions which typically complete in under 30 seconds while adding zero friction to genuine customers submitting via NFC, SMS, or QR in real time.

### Maximum submission window

A review may not be submitted more than **30 days** after the token was generated.

Tokens expire after 30 days. This prevents tokens being stockpiled and deployed strategically. A genuine customer has 30 days to submit — sufficient for any real-world scenario.

### Velocity fraud signal

The timing window is not the primary fraud signal. Pattern detection across tokens is.

Implementations must flag the following patterns for review:

| Pattern | Signal |
|---------|--------|
| Review submitted within 90 seconds of token generation | Possible automated submission |
| 10+ reviews for one business submitted within 60 minutes | Coordinated activity |
| Zero timing variance across 5+ submissions (e.g. all arrive exactly 4m 12s after token) | Bot signature |
| Average rating drops more than 1.5 stars in 48 hours | Coordinated attack |
| 3+ reviews from same device fingerprint in 24 hours | Single-device farming |

---

## Device Fingerprinting Requirement

All implementations must capture a device fingerprint at the point of review submission.

### Required fingerprint components

| Signal | Description |
|--------|-------------|
| IP address | Captured at submission. Not stored permanently. Used for velocity check only. |
| User agent string | Browser, OS, version combination |
| Browser fingerprint | Screen resolution, timezone, language, canvas rendering hash |
| Persistent session token | Written to device storage on form open. Detects repeat submissions from same device. |
| Form interaction signature | Time spent on form, field completion pattern, scroll behaviour |

### NFC-specific signal

For reviews submitted via NFC tap, implementations should additionally capture the NFC chip identifier from the submitting device where the platform permits. This identifier is hardware-level and not subject to MAC address randomisation. It provides a stronger device-level signal than browser fingerprinting alone.

### What device fingerprinting must never do

- Store the fingerprint linked to a customer identity
- Share device data with third parties
- Use device data for any purpose other than fraud detection
- Retain raw fingerprint data beyond 90 days

### Fingerprint collision rules

| Situation | Action |
|-----------|--------|
| Same device fingerprint, different businesses, 24 hours | Permitted. Flag for monitoring. |
| Same device fingerprint, same business, 24 hours | Hold second review pending investigation. |
| Same device fingerprint, 3+ reviews, 24 hours | Auto-hold all. Escalate to fraud review. |
| Headless browser detected | Auto-reject. Log attempt. |

---

## Review Submission Contract

A review may only be published if all of the following are satisfied:

1. Token hash matches an issued, active token
2. Submitter email hash matches token customer email hash
3. Two-sided customer confirmation completed with "Yes"
4. Submission timestamp is within the valid timing window
5. Device fingerprint does not trigger collision rules
6. Rating is an integer between 1 and 5 inclusive
7. Transaction type tag matches token transaction type
8. Token has not already been used for a published review

### Rejection is silent on mismatch reason

When rejecting due to checks 1 or 2, implementations must return a generic validation failure. The reason for rejection must not be disclosed. This prevents token enumeration attacks.

### Submission payload

```json
{
  "token_hash": "string",
  "submitter_email_hash": "sha256(lowercase(submitter_email))",
  "customer_confirmation": "yes",
  "rating": "integer 1–5",
  "transaction_type_tag": "string",
  "comment": "string | null",
  "submitted_at": "ISO 8601 datetime UTC",
  "device_fingerprint_hash": "sha256 of fingerprint composite"
}
```

---

## Flag and Dispute Framework

### Valid flag reasons — investigated

A review may be flagged for investigation only on the following grounds:

| Reason | Evidence required |
|--------|-------------------|
| Token hash not in public verification record | Token hash value |
| Submission outside valid timing window | Timestamp evidence |
| Device fingerprint matches 3+ reviews in 24 hours | System-detected |
| Statutory declaration of no transaction | Legally witnessed signed document |
| Specific identified witness claim | Named person with verifiable connection to business |

### Invalid flag reasons — automatically rejected

The following flag reasons are automatically rejected without investigation:

- Star rating is too low
- Business disagrees with written content of review
- Anonymous complaint with no supporting evidence
- Competitor business flagging without transaction evidence
- Repeated flags from same source on same business (throttled after 3 dismissed flags)
- Review contains negative sentiment only

### Business dispute process

A business wishing to dispute a published review must submit a statutory declaration — a legally witnessed signed document — stating that no transaction with that customer occurred on the stated date.

The statutory declaration must:
- Be signed by the business owner personally (not an employee or representative)
- Be witnessed by a legally recognised authority (Justice of the Peace, Notary Public, or country equivalent)
- State specifically: the token hash in dispute, the transaction date, and the basis for claiming no transaction occurred

A false statutory declaration constitutes perjury in all jurisdictions. The legal exposure is personal, not corporate.

### Consumer flag process

A consumer wishing to flag a review as fraudulent must:

1. Identify their specific relationship to the business or transaction
2. Provide specific verifiable evidence (not opinion)
3. Confirm their identity to iSighted (not published publicly)
4. Acknowledge that false fraud reports violate iSighted terms

Anonymous flags receive lower investigation priority. Identified flags with specific evidence are investigated within 14 business days.

---

## Velocity Limits — Per Business Category

Implementations must apply token generation velocity limits by business category. These limits exist because legitimate businesses have natural transaction ceilings determined by physical and operational constraints.

| Category | Maximum tokens per day |
|----------|----------------------|
| Solo practitioner (dentist, GP, lawyer) | 40 |
| Small practice (2-5 chair dental, small clinic) | 80 |
| Retail (single location) | 200 |
| Restaurant / café | 300 |
| Solo tradesperson | 12 |
| Small trades business (2-5 staff) | 40 |
| eCommerce (online only) | 500 |
| Enterprise / multi-location | Custom — reviewed on application |

Tokens generated above these limits are automatically held pending manual review. The business is notified. Repeated velocity breaches trigger account investigation.

---

## Enforcement Escalation

### Stage 1 — Freeze

Triggered by: velocity breach, device fingerprint collision, flag threshold reached, or anomaly detection.

Actions:
- All pending unverified reviews held — not published
- Only previously verified reviews remain visible
- Business notified with specific reason
- 14-day window to respond with evidence

### Stage 2 — Investigate

Triggered by: business fails to respond within 14 days, or response is insufficient.

Actions:
- Business must provide transaction evidence for all flagged tokens
- iSighted conducts AI-assisted document review
- Business account access restricted to read-only
- All new token generation suspended

### Stage 3 — Enforce

Triggered by: investigation concludes fraud is confirmed, or business refuses to cooperate.

Actions:
- Account permanently deactivated
- All fraudulent reviews removed from public record
- Legitimate reviews preserved with notation of account status
- Legal action initiated where evidence supports it
- Business name added to internal fraud registry (not published)

---

## Public Verification Endpoint

Every published review must carry a `verification_url`. That URL must return the following to any caller without authentication.

```json
{
  "verified": true,
  "business_name": "string",
  "transaction_date": "YYYY-MM-DD",
  "transaction_type": "string",
  "trust_tier": "gold | silver | bronze",
  "token_hash": "string",
  "hash_algorithm": "SHA-256",
  "review_submitted_at": "ISO 8601 datetime",
  "spec_version": "1.1"
}
```

### HTTP requirements

- Unauthenticated GET requests
- Content-Type: application/json
- HTTP 200 for verified tokens
- HTTP 404 for unknown hashes
- CORS: Access-Control-Allow-Origin: *

---

## Schema.org Alignment

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
  "datePublished": "2026-04-19",
  "description": "Review comment text",
  "additionalProperty": [
    {
      "@type": "PropertyValue",
      "name": "SightToken",
      "value": "a7f3d2e1b9c04821",
      "propertyID": "https://isighted.io/verify/a7f3d2e1b9c04821"
    },
    {
      "@type": "PropertyValue",
      "name": "TrustTier",
      "value": "gold"
    }
  ]
}
```

---

## Transparency Reporting

All compliant iSighted implementations must publish a quarterly transparency report containing:

- Total flags received in period
- Flags upheld (fraud confirmed) — count and percentage
- Flags dismissed — count and percentage
- Business accounts deactivated — count
- Reviews removed — count
- Reviews restored after investigation — count

No business names are published in the transparency report. The report is public and machine-readable.

---

## Versioning

- **Patch (1.1.x):** Clarifications only. No schema changes.
- **Minor (1.x.0):** Additive backwards-compatible changes.
- **Major (x.0.0):** Breaking changes. Published as separate specification.

---

## What Is Deliberately Out of Scope for v1.1

The following are real and important. They are excluded because they are unproven at scale:

- Media attachments (photos/videos anchored to tokens)
- Reviewer reputation profiles and tier system
- Zero-knowledge identity proofs
- B2B transaction trust
- Agent-to-agent transaction verification
- Neural interface review data

---

## Reference Implementation

`https://github.com/isightedit/isighted-token-spec`

MIT licensed. This specification is CC BY 4.0.

---

## Contributing

Maintained by iSighted. Community proposals via GitHub Issues.

Before proposing: *is this proven in production, or speculative?* Speculative features belong in discussion threads, not the specification.

---

*iSighted Sight Token Specification v1.1*  
*Originally published April 19, 2026*  
*Updated April 19, 2026*  
*CC BY 4.0 — https://creativecommons.org/licenses/by/4.0/*
