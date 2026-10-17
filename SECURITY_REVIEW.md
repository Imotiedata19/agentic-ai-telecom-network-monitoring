# Security and publication review

## Original export — do not post publicly

The private n8n export includes an email recipient, an n8n Gmail credential **reference** (ID/name, not a visible access token), unique workspace Data Table IDs and links, a workflow/instance identifier, and a webhook identifier. No recognizable API secret, password, or authorization header was found during this **static JSON inspection**. This is **not** a comprehensive security audit of n8n or connected resources.

## Changes made to `telecom_ai_workflow_public_template.json`

- Replaced the original Gmail recipient with `YOUR_ALERT_EMAIL@example.com`.
- Removed the Gmail node `credentials` object; reauthorization is required after importing.
- Removed the Gmail node `webhookId` field.
- Cleared Data Table resource locator IDs (`dataTableId.value`) and removed their cached URLs while retaining non-sensitive table names for configuration instructions.
- Removed workflow-level IDs, version metadata and instance metadata.
- Set workflow `active` to `false` to avoid an activation assumption on import.
- Retained node topology, JavaScript logic, thresholds, schedule, logical table names, and expressions for inspectability.

## What a reader must do after importing

The public template is intentionally **not fully configured**. Re-select the table resources in every Data Table node, reconnect Gmail using your own credentials, and set an approved email recipient. Check the import preview and test the workflow before enabling scheduled execution.

## Before adding files, screenshots, or examples to GitHub

- Do not commit the original JSON or any `.env` file containing secrets.
- Screenshots should not reveal email addresses, account IDs, credentials, client records, sensitive tower/site data, or proprietary network information.
- Do not publish actual telecom performance information without authorization. Demonstration/test data is preferred.
- The public template's identifiers are no longer bound to the original private n8n data tables.
- If a private file was **already** publicly committed, deleting it in a later commit is not enough to remove it from Git history; rotate any exposed secrets and remediate the repository history.
