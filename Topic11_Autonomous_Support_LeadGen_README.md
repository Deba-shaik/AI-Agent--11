# Topic 11 - Autonomous Support & Lead Generation Agent

## Project overview
This n8n workflow receives customer requests through an HTTP webhook, retrieves customer account tier information from PostgreSQL, combines it with a **mock recent transaction** record, and routes the context through an AI Agent. The agent can save a sales lead to PostgreSQL or send a high-priority support escalation to Slack. The workflow then prepares a JSON API response.

## Workflow architecture
`Webhook -> [Get_Recent_Transactions (mock) + Get_Customer_Tier (PostgreSQL)] -> Consolidate_Context -> AI Agent -> Log_Token_Usage -> Respond to Webhook`

- `Get_Customer_Tier` has an error route to `Assign_Fallback_Tier`, which feeds `Consolidate_Context`.
- The AI Agent uses an OpenAI Chat Model and two optional tools: `insert_lead_tool` (Postgres) and `Send a message in Slack`.
- Tools execute conditionally, depending on the customer request; they do not both run on every request.

## Node responsibilities
| Node | Purpose |
| --- | --- |
| Webhook | Accepts a POST JSON customer inquiry at `/support-lead-intake` (test URL used during validation). |
| Get_Recent_Transactions | Provides sample transaction context; currently manually mapped/mock, not a live transaction lookup. |
| Get_Customer_Tier | Parameterized query against `users` by email; uses PostgreSQL credentials. |
| Assign_Fallback_Tier | Provides fallback account context when the tier query errors. |
| Consolidate_Context | Combines transaction and account data using matching `email`. |
| AI Agent | Responds to customer requests and chooses CRM or Slack tools when appropriate. |
| insert_lead_tool | Inserts email, company and budget into `public.leads`. |
| Send a message in Slack | Posts high-priority support escalations to the designated Slack channel. |
| Log_Token_Usage | Preserves agent response and token fields; usage currently unavailable (`null`). |
| Respond to Webhook | Returns HTTP JSON to the caller. |

## Example request
Method: `POST`; Content-Type: `application/json`. Use the workflow's actual test/production URL; do not publish private endpoints or credentials.

```json
{
  "email": "customer@example.com",
  "company": "ABC Technologies",
  "message": "What is my current account status?",
  "budget": 5000
}
```

## Example successful response (observed structure)
```json
{
  "success": true,
  "response": "Your current account status is Active. If you have any questions or need assistance, feel free to ask!",
  "token_usage": {"input": null, "output": null, "total": null}
}
```

## PostgreSQL setup
`public.users` must contain `email`, `plan`, `status` for the tier lookup. `public.leads` was created in the test database with `id` (serial primary key), `email`, `company`, `budget`, and `created_at`. Inserted IDs and timestamps are database-generated. The AI tool populates email, company and budget. Avoid duplicate submissions: the current schema does not enforce unique emails.

Customer lookup query:
```sql
SELECT email, plan, status FROM users WHERE email = $1;
```
Use a parameterized array containing the Webhook email, not string interpolation.

## Evidence-backed tests
1. **Normal account lookup - PASS.** Customer tier query and merge succeeded; agent returned Active status.
2. **CRM lead insertion - PASS.** Postgres tool output returned a newly inserted row, id `2`, email `customer@example.com`, company `Example Inc.`, budget `5000.00`. The company differs from the example Postman payload (`ABC Technologies`); verify extraction/mapping before production.
3. **Slack escalation - PASS.** Slack bot posted a high-priority support escalation in `#logistics-critical-alerts`.
4. **Database failure fallback - PASS.** An intentionally invalid table query triggered `Assign_Fallback_Tier`, the merge and agent continued. Restore the original query after testing. The fallback response said `Offline-Backup`; this wording should be revised so it does not imply actual backups.
5. **End-to-end HTTP response - PASS.** Postman displayed HTTP `200 OK`, about `4.34 s`, and JSON with `success: true` and token fields `null`.

## Known limitations and next improvements
- Transaction context is **mock**, not a verified live transaction integration.
- Token counts are **not measured** in the returned AI Agent JSON; `null` means unavailable, not zero usage.
- Fallback response should state account data is temporarily unavailable, without claiming a backup or recovery ETA.
- AI tool can alter or infer company names: enforce exact field extraction and validation before inserting.
- The `leads` table has no unique-email constraint or duplicate check; add a deduplication strategy if required.
- A green Respond to Webhook node does not by itself prove client delivery; the separate Postman HTTP 200 screenshot confirms the observed API result.
- Use synthetic test data, limit access to customer information, and keep credentials out of exported JSON and screenshots.

## Reproduce the validation
1. Import the workflow JSON into an authorized n8n environment (export is a separate deliverable, not included here).
2. Configure PostgreSQL, OpenAI and Slack credentials; confirm `users` and `leads` tables exist.
3. Start the test webhook and send the sample JSON from Postman.
4. Verify account response and HTTP 200; inspect execution trace.
5. Test a sales inquiry and inspect the inserted row.
6. Test a critical support request and confirm the Slack notification.
7. Temporarily simulate a database error; verify fallback and restore SQL immediately.

## Submission checklist
- [x] Main workflow and normal response screenshots
- [x] CRM lead insertion screenshot
- [x] Slack escalation and channel notification screenshots
- [x] Database fallback screenshots
- [x] Postman 200 response screenshot
- [ ] Exported workflow JSON (export from n8n)
- [ ] Recorded 5-6 minute walkthrough (record separately)
- [ ] Verify sensitive values are redacted from any additional evidence
