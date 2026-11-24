# ReviewGuard Test Results

The results below summarize the manual tests performed in the local self-hosted n8n environment using synthetic chat messages and a simulated Google reply node.

| Test | Observed result | Interpretation |
|---|---|---|
| One-star review | Gmail manual-review alert received; automatic reply path not used | Manual escalation path worked in the test workflow. |
| Clean five-star review | Guardrails passed; mock mode returned simulated HTTP 200; Slack notification received | Simulated-success route worked. |
| Outbound PII test | Synthetic email/phone values triggered Guardrails; Gmail security alert received; no mock reply or Slack success | Outbound guardrail fail path worked. |
| Mock API failure | Mock node returned simulated HTTP 503; Gmail failure alert received; Slack success notification did not execute | API-failure route worked. |
| Mock API success restored | Mock mode set back to `success`; Slack notification received and no failure email was sent | Normal test path was restored. |
| Duplicate review ID | First `TEST-002` submission inserted one row; second submission stopped at duplicate lookup; Data Table contained one `TEST-002` row | Sequential duplicate detection worked. |

## Test constraints

- Google responses were simulated; no public reply was posted.
- Synthetic test IDs and review text were used.
- Duplicate detection was tested sequentially, not under simultaneous/concurrent arrivals.
- The Data Table status remains `RECEIVED`; lifecycle transitions are not implemented.
- Results are manual development checks, not a formal load, security, or production-readiness certification.
