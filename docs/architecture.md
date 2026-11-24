# ReviewGuard Architecture

## Overview

ReviewGuard is an n8n workflow that receives synthetic review text through the Chat Trigger, asks a local Gemma 4 12B model served by Ollama to classify the review and draft a response, validates the model's JSON, applies a deterministic rating policy, records a synthetic review ID, and routes the result through notification and mock-response branches.

## Processing sequence

1. **Test Review Input** receives a message containing a synthetic review ID, star rating, and comment.
2. **ReviewGuard AI Agent** uses the configured Ollama chat model (`gemma4:12b`) and a calculator test tool.
3. **Parse Review JSON** parses the model's response, validates a rating from 1 to 5, and enforces the workflow decision: rating 1 is manual review; ratings 2–5 are eligible for the simulated automatic-reply path.
4. **Assign Review ID** extracts `Review ID:` from the original chat message and adds `RECEIVED`, rating, timestamp, and `CHAT_TEST` source metadata.
5. **Check Duplicate Review ID** uses the `review_id` column in the `reviewguard_processed_reviews` Data Table. Only IDs not already present continue.
6. **Register New Review** inserts the new review ID and metadata into the Data Table.
7. **Restore Review Context** merges the inserted row with the original AI analysis so downstream routing still has the response and decision fields.
8. **Rating Decision Router** sends one-star reviews to a Gmail manual-review alert and ratings 2–5 to Public Reply Guardrails.
9. **Public Reply Guardrails** checks the generated reply for PII, secret keys, and URLs. A failed check sends a Gmail security alert.
10. **Mock Google Reply API** validates the review and guardrail result, then simulates either HTTP 200 or HTTP 503 based on `MOCK_MODE`.
11. **Google API Success Check** routes successful simulations to Slack and failed simulations to a Gmail alert.

## Data Table schema

Table: `reviewguard_processed_reviews`

| Column | Type | Meaning |
|---|---|---|
| `review_id` | String | Synthetic unique ID used in duplicate lookup |
| `status` | String | Initial value is `RECEIVED` |
| `rating` | Number | Star rating from 1 to 5 |
| `first_seen_at` | Date | Timestamp assigned when the ID is extracted |
| `source` | String | `CHAT_TEST` for the current prototype |

n8n manages the row ID and created/updated timestamps.

## Trust boundaries

- Review text and AI-generated response content are treated as untrusted input.
- The AI Agent drafts and classifies; the Parse Review JSON node validates the rating and enforces the routing policy.
- Public Reply Guardrails checks the outbound response before the mock API node.
- The Mock Google Reply API node is a local simulation only. It does not send network requests to Google.
- Gmail and Slack credentials are configured in each user's n8n instance and are not shipped in the repository.

## Current design limitations

- The duplicate lookup is after LLM inference.
- The check-then-insert sequence is not atomic for concurrent events.
- Status is inserted as `RECEIVED` and is not updated after downstream outcomes.
- The current Guardrails node checks the generated reply; incoming prompt injection is not separately screened.
- There is no real Google Business Profile event trigger, API client, publication, or safe retry loop.
