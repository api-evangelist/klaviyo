---
name: Create and send a Klaviyo email campaign
description: Build a template, create a campaign against an audience, attach the template to its message, estimate recipients, then send and track the send job.
api: openapi/klaviyo-campaigns-api-openapi.yml, openapi/klaviyo-templates-api-openapi.yml
generated: '2026-08-13'
method: generated
source: Grounded in openapi/*.yml (revision 2026-04-15) + conventions/klaviyo-conventions.yml
operations:
  - create_template
  - get_template
  - render_template
  - create_campaign
  - get_messages_for_campaign
  - assign_template_to_campaign_message
  - update_campaign_message
  - refresh_campaign_recipient_estimation
  - get_campaign_recipient_estimation
  - send_campaign
  - get_campaign_send_job
  - cancel_campaign_send
scopes:
  - templates:write
  - templates:read
  - campaigns:write
  - campaigns:read
---

# Create and send an email campaign

A campaign send is a five-call sequence, and the ordering is not optional — the campaign
must exist before its message exists, and the message must exist before you can attach a
template to it.

## Step 1 — Create the template

`create_template` (`POST /api/templates`) with your HTML. Klaviyo returns the template id.

Check your rendering before you commit to it: `render_template`
(`POST /api/template-render`) takes the template plus a context object and returns the
rendered output. Use this to catch a broken personalisation tag before it reaches an
inbox rather than after.

## Step 2 — Create the campaign

`create_campaign` (`POST /api/campaigns`). The body names the channel, the audience
(the list or segment ids to include and exclude), and the send strategy.

Klaviyo creates the campaign's **message** as a side effect. You do not create it
directly — there is no `create_campaign_message` operation.

## Step 3 — Find the message id

`get_messages_for_campaign` (`GET /api/campaigns/{id}/campaign-messages`). Take the id
of the message for the channel you are sending.

## Step 4 — Attach the template

`assign_template_to_campaign_message` (`POST /api/campaign-message-assign-template`),
passing the message id and the template id.

This **clones** the template onto the message. Later edits to the original template do
not propagate to a campaign that already has it attached. If you need to change the
content after this point, either re-assign or edit the message directly with
`update_campaign_message` (`PATCH /api/campaign-messages/{id}`).

Subject line, preview text and sender identity live on the message, not the template —
set them via `update_campaign_message`.

## Step 5 — Check who you are about to mail

Before sending, know the size of the blast:

1. `refresh_campaign_recipient_estimation`
   (`POST /api/campaign-recipient-estimation-jobs`) — kicks off the calculation.
2. `get_campaign_recipient_estimation`
   (`GET /api/campaign-recipient-estimations/{id}`) — reads the result.

For an agent operating on a real account this step is the guardrail. Confirm the number
against what a human expected before you proceed to step 6.

## Step 6 — Send

`send_campaign` (`POST /api/campaign-send-jobs`) with the campaign id.

**This is irreversible in effect.** It dispatches real messages to real people and spends
real send budget. It is the single most consequential operation in Klaviyo's API. Treat
it as human-in-the-loop: an agent should confirm before calling it, never call it
speculatively, and never call it as part of a retry loop.

Note that Klaviyo's published MCP tool set deliberately does **not** expose
`send_campaign` — an agent using the MCP server can build a campaign but cannot send it.
That is a designed safety boundary, not an oversight. If you are calling REST directly,
you have removed that boundary yourself; put it back.

## Step 7 — Track the send

`get_campaign_send_job` (`GET /api/campaign-send-jobs/{id}`) reports status.

`cancel_campaign_send` (`PATCH /api/campaign-send-jobs/{id}`) can cancel a send that is
scheduled or still in progress. It cannot recall messages already delivered.

## Errors

- `409` on send usually means the campaign is not in a sendable state — it may be
  missing a template, a sender identity, or an audience.
- `403` means the key lacks `campaigns:write`.
- Do not retry a `send_campaign` that returned a `5xx` without first checking
  `get_campaign_send_job` — the send may have been accepted before the error surfaced,
  and a blind retry mails everyone twice.

See `errors/klaviyo-problem-types.yml` and `conventions/klaviyo-conventions.yml`.
