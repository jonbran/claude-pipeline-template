---
name: jira
description: Fetch a Jira ticket by ID and start the full development pipeline
argument-hint: [ticket-id]
allowed-tools: Bash, Read, Agent
---

# Jira Ticket Pipeline

You have been given Jira ticket ID: **$ARGUMENTS**

## Step 1 — Check Environment Variables

The following environment variables are required:

| Variable | Description |
|---|---|
| `JIRA_EMAIL` | Your Jira account email address |
| `JIRA_API_TOKEN` | Your Jira API token (create at https://id.atlassian.com/manage-profile/security/api-tokens) |
| `JIRA_DOMAIN` | Your Jira domain, e.g. `yourcompany.atlassian.net` |

Check they are set:
```bash
echo "Email: $JIRA_EMAIL"
echo "Domain: $JIRA_DOMAIN"
echo "Token set: $([ -n "$JIRA_API_TOKEN" ] && echo yes || echo NO - MISSING)"
```

If any are missing, stop and tell the user exactly which variables need to be set,
and how to set them (e.g. `export JIRA_API_TOKEN=your_token_here`).

## Step 2 — Fetch the Ticket

```bash
curl -s -w "\nHTTP_STATUS:%{http_code}" \
  -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$ARGUMENTS"
```

Check the `HTTP_STATUS:` line at the end of the output.
If the status is not `200`, report the error clearly (401 = bad credentials,
403 = no access, 404 = ticket not found) and stop.
If the status is `200`, pass the body through `python3 -m json.tool` for formatting.

## Step 3 — Extract and Format

From the JSON response, extract:
- **Summary** (title)
- **Description** (full text)
- **Acceptance Criteria** (if present in description or a custom field)
- **Priority**, **Story Points**, **Labels** (if present)

Format this into a clean task brief.

## Step 4 — Hand Off to Orchestrator

Pass the formatted task brief to the **orchestrator** agent to begin the
full development pipeline (branch → BA → Architect → Dev → Review → Test → Push).
