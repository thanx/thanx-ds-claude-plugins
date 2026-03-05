---
description: Diagnose a partner integration email - extract requirements, classify integration type, query Thanx docs, and generate a structured integration guide.
---

# Diagnose Partner Integration

Analyze an inbound partnership email and produce a structured integration diagnosis and guide.

## Usage

```bash
/ds:diagnose <email_text_or_description>
```

**Required:**

- `email_text_or_description` - The full partner email text, or a summary of their integration request

---

You are executing the `/ds:diagnose` command.

## Step 1: Extract Key Information

Arguments: $ARGUMENTS

Parse the email and extract the following. If any field cannot be determined, mark it as "UNCLEAR - needs follow-up".

- **Partner Name**
- **Partner Type** (POS provider, mobile app developer, kiosk vendor, ordering platform, data/analytics platform, marketing platform, other)
- **Primary Use Case** (one sentence)
- **Specific Needs** (bullet list: e.g., reward redemption, points accrual, user lookup, purchase submission, SSO, webhooks)
- **Their Existing Tech** (platforms, languages, systems mentioned)
- **Timeline/Urgency**
- **Open Questions** (anything ambiguous or missing)

## Step 2: Classify Integration Type

Map to one or more types with confidence (HIGH / MEDIUM / LOW):

1. **POS/Kiosk** (Loyalty API) - POS systems, kiosks, or digital ordering that look up loyalty accounts, display rewards, process redemptions at checkout
2. **Consumer UX** (Consumer API) - Mobile apps or web experiences embedding full Thanx loyalty: SSO, cards, rewards, points, tiers, purchases
3. **Pay-at-Table** (Consumer API subset) - QR-code-based mobile payment where guests scan, view order, pay, earn points
4. **Partner API** (Partner API) - Backend/server-side: submit purchases, manage users/subscribers, tags, feedback without consumer-facing UX

If ambiguous, list all applicable types ranked by likelihood. If too vague to classify, skip Steps 3 and 4 and instead produce a Discovery Questionnaire (see Fallback section below Step 4).

## Step 3: Query Thanx Documentation

Using the **thanx-docs** MCP server, perform these searches. This is mandatory - do NOT rely on prior knowledge alone:

1. Search for the overview/guide for the identified integration type (this often covers auth and certification)
2. Search for each specific endpoint the partner will need that was NOT already covered by the overview
3. If webhooks, data exports, or other supplementary features were mentioned, search for those too
4. Limit to 8 total doc searches. If more are needed, present what you have and ask the user whether to continue

**If the thanx-docs MCP server is unavailable, errors, or times out:**
Stop and output "Blocked by docs retrieval" followed by the Discovery
Questionnaire (see Fallback section below). Do NOT guess endpoint paths,
auth headers, or certification requirements.

## Step 4: Generate Integration Guide

Create the guide as a markdown file saved to `.context/diagnoses/[partner-name-slug]-YYYY-MM-DD.md`.
If Partner Name cannot be determined, use `unknown-partner-YYYY-MM-DD` as the slug.

Use this structure:

```markdown
# Thanx Integration Guide: [Partner Name]

## 1. Executive Summary

| Field | Value |
|-------|-------|
| Partner | [name] |
| Integration Type | [type(s)] ([confidence]) |
| Primary API | [Consumer API / Loyalty API / Partner API] |
| Estimated Complexity | [Low / Medium / High] |
| Key Deliverable | [one sentence] |

## 2. Authentication Setup

### Environments

| Environment | Base URL |
|-------------|----------|
| Sandbox | [url] |
| Production | [url] |

### Required Credentials

[List all credentials needed]

### Required Headers

| Header | Format | Purpose |
|--------|--------|---------|
| ... | ... | ... |

### Auth Flow

[Step-by-step auth flow for this integration type]

## 3. Endpoint Reference

Group by workflow stage:

**Authentication**

| Endpoint | Method | Path | Purpose |
|----------|--------|------|---------|
| ... | ... | ... | ... |

**Core Operations**

| Endpoint | Method | Path | Purpose |
|----------|--------|------|---------|
| ... | ... | ... | ... |

**Supporting Operations**

| Endpoint | Method | Path | Purpose |
|----------|--------|------|---------|
| ... | ... | ... | ... |

## 4. Recommended Implementation Workflow

1. **Phase 1: Setup & Auth** - [steps with endpoint references]
2. **Phase 2: Core Integration** - [steps]
3. **Phase 3: Supporting Features** - [steps]
4. **Phase 4: Error Handling & Edge Cases** - [steps]
5. **Phase 5: Testing & Certification** - [steps]

## 5. Certification Requirements

- What Thanx will validate
- Required demonstrations
- Expected timeline
- Contact: developer.support@thanx.com

## 6. Additional Considerations

- Rate limits
- Webhook opportunities (if applicable)
- Data export options via Thanx Connex (if applicable)
- Best practices from Thanx docs

## 7. Open Questions & Next Steps

- Items needing clarification from the partner
- Recommended follow-up questions
- Suggested scoping call agenda
- Postman collections: https://docs.thanx.com/overview/api_collections.md

## 8. Sources

| Document | URL | Retrieved |
|----------|-----|-----------|
| [doc title] | [doc URL] | [YYYY-MM-DD] |

## 9. Contacts

- Partnerships: partnerships@thanx.com
- Developer Support: developer.support@thanx.com
- Status Page: https://status.thanx.com
```

### Fallback: Discovery Questionnaire

If the email is too vague to determine an integration type, produce a "Discovery Questionnaire" instead of a full guide. The questionnaire should include:

1. What type of product/platform does the partner operate?
2. Will end-consumers interact with Thanx through the partner's UI, or is this a backend integration?
3. What specific loyalty features do they need? (reward display, redemption, points accrual, user enrollment, purchase tracking)
4. Do they need real-time data (webhooks) or batch data exports?
5. What is their tech stack?
6. What is their target launch timeline?
7. How many merchant brands do they plan to support?

Save this questionnaire to `.context/diagnoses/[partner-name-slug]-questionnaire.md`.

## Rules

1. **Always use the thanx-docs MCP server** to verify endpoint paths, headers, and auth details. Do not guess.
2. **Sanitize partner name slugs** before using in file paths: lowercase, replace spaces and
   punctuation with hyphens, remove path-unsafe characters (`/`, `\\`, `..`), strip any
   leading/trailing hyphens, collapse consecutive hyphens, and limit to 50 characters.
   Enforce with: `echo "$slug" | sed 's/[^a-z0-9-]/-/g; s/--*/-/g; s/^-//; s/-$//' | cut -c1-50`
   Example: "Bosque Brewing Co." becomes `bosque-brewing-co`.
3. If the email mentions capabilities spanning multiple APIs, design the guide with separate tracks per API.
4. Always recommend starting with sandbox credentials.
5. Flag any requirements that Thanx may not support in Additional Considerations.
6. Create the `.context/diagnoses/` directory if it doesn't exist before saving the file.
7. **Redact PII** before writing files to disk: strip personal phone numbers, home addresses, and other contact details not relevant to the integration. Keep business email addresses and company names.
8. Do not send any communication to the partner. Present the guide for human review.
