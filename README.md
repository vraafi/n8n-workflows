# n8n workflows — tested, not just exported

Four n8n workflows I run myself. Each one was imported into **n8n 2.35.7** and run
against edge cases before it went in here. No credentials are included.

| Workflow | Tests passed | Demo |
|---|---|---|
| [Invoice reminder — Google Sheets + Gmail](workflows/invoice-reminder.json) | 12 cases | [video](media/demo_invoice.mp4) |
| [Low-stock digest — Google Sheets + Telegram](workflows/stok-rendah.json) | 12 cases | [video](media/demo_stok.mp4) |
| [Record sales in Airtable without duplicates](workflows/webhook-to-airtable.json) | 12 cases | [video](media/demo_webhook.mp4) |
| [Telegram class booking that cannot double-book](workflows/telegram-class-booking.json) | 11 cases | [video](media/demo_booking.mp4) |

## Also in this repo, not counted above

Two more workflows live here. Neither is in the table above, and the reason
differs, so read the note on each.

### [Order sync that survives a rate limit](workflows/retry-and-error-branch.json)

A payload is validated before the API call, sent with a fixed idempotency key,
and — when the API answers 429/408/5xx — waited on and tried once more. The
original payload is kept either way so the order can be replayed.

Its companion is [error-workflow.json](workflows/error-workflow.json): point the
main workflow's `settings.errorWorkflow` at it and any unhandled failure is
classified and turned into an alert instead of dying silently.

**The setup step the guides leave out:** the error workflow must be **active**
too. Pointing `settings.errorWorkflow` at an imported-but-inactive workflow
fails silently — n8n writes one line to its log and drops the alert:

```
Calling Error Workflow for "your-workflow". Workflow "error-workflow" is not active and cannot be executed
```

So: import it, then **publish/activate it**, then point the other workflow at it.
Both steps, or you get nothing.

Measured, with a real failure forced through a webhook:

| Failure | Alert received |
|---|---|
| Code node throws | `class: PERMANENT`, `alert_level: immediate`, `failed_node: "Gagal sengaja"`, `execution_id: 3`, `mode: webhook` |
| HTTP node gets 500 | `class: TEMPORARY`, `http_code: 500`, `alert_level: digest`, `failed_node: "HTTP gagal"` |

Two things worth knowing about how it fires:

- It only fires from the **server**, not from `n8n execute` on the command line.
  The dispatch lives in `active-workflow-manager` and `execution-lifecycle-hooks`;
  a CLI run will fail without ever calling the error workflow. If you test by
  hand, test against a running instance.
- The payload has two shapes. With an execution id you get
  `{ execution: {...}, workflow: {...} }`; without one you get
  `{ trigger: { error, mode }, workflow }`. This workflow reads the first, which
  is what a server-triggered failure always produces.

**This one was run, not just reasoned about.** n8n 2.48.6 on Node 24, against a
local endpoint I could script, with the calls timed. The 429→success case was
then repeated on **n8n 2.35.7 / Node 22** — the version named in this README's
header — with the same result (3 calls, +5.06 s and +5.04 s, `Record success`
with both identifiers filled). Both files import cleanly on both versions.

| Endpoint replies | API calls | Gaps between calls | Where it ended |
|---|---|---|---|
| 429, 429, 200 | 3 | +5.12 s, +5.06 s | `Record success`, `order_id` and `idem_key` filled |
| 500 always | 5 | +5.07 s, +5.04 s, +5.13 s, +5.05 s | `Record failure`, `attempts: 2`, `payload_kept: true` |
| 400 always | 3 | +5.12 s, +5.04 s | `Record failure`, `attempts: 1`, `payload_kept: true` |

The 5 s is `waitBetweenTries` doing what it says. The 3-then-2 pattern is
`maxTries: 3` on the first call and `maxTries: 2` on the retry.

**One honest correction to the obvious claim.** I first wrote that a 4xx which
is not 429 "is not retried". That is false, and the table shows it: the 400 run
made three calls. `retryOnFail` catches every error and cannot look at a status
code, so it burns two retries on a request that can never succeed. What this
workflow actually buys you is that a permanent error never reaches the *second,
delayed* stage — 3 calls instead of 5, and no pointless extra wait. Turning
those 3 into 1 would mean dropping `retryOnFail` and driving the retry entirely
from the graph; that is a different trade, and it is not what is shipped here.

Beyond the three runs, 24 cases exercise the Code nodes directly: payload
validation, status-code classification (429/408/500/503/0 → TEMPORARY;
400/401/404/422 → PERMANENT), the payload surviving into the failure branch, and
both entry paths into `Record failure`.

The node settings were checked against n8n 2.48.6 source rather than the docs,
which is how I found that `maxTries` is clamped to 2–5 and `waitBetweenTries` to
5000 ms. That ceiling is why the wait is a real `Wait` node and not a setting.

To reproduce: import both files, point `crm_url` in `CONFIG` at an endpoint that
returns 429, and open the execution list.

### [News to draft — Google News + OpenAI](workflows/berita-ke-draf.json)

The workflow I use for AI drafting: it reads a news feed on a schedule, drops
stories it has already seen, and asks a model for a structured draft it can put
in a sheet. Read it for the pattern, not for a test count.

The 47 cases above were run against a numbered test matrix I kept. Neither of
these two was, so counting them would overstate what is here.

## Why the tests matter

Most shared workflows are exported from a canvas that was never run. The three
failures I hit most often:

1. A retried webhook creates a duplicate row.
2. Two people acting at the same second both get a confirmation.
3. A second run re-sends everything the first run already sent.

Each workflow below says which of those it defends against, and how.

## The workflows

### Invoice reminder — Google Sheets + Gmail

Reads an invoice sheet every morning and emails only the rows that are overdue and
not yet contacted, then writes `status = done` back. Defends against (3): the
write-back is terminal, so a second run the same day is a no-op. Duplicate
`record_id` values are dropped before any send.

### Low-stock digest — Google Sheets + Telegram

Keeps only the rows at or below their reorder point, adds `days_left` from your own
usage, and merges them into **one** message. One message, not one per item, because
a burst of Telegram sends gets rate-limited.

### Record sales in Airtable without duplicates

A Gumroad sale ping is written with Airtable's `performUpsert` keyed on `sale_id`.
Defends against (1): a retried delivery updates the same row instead of adding a
second one. Failures are logged as TEMPORARY or PERMANENT so a retry is a decision,
not a guess.

### Telegram class booking that cannot double-book

Students book a seat from Telegram. A partial unique index on
`(room_id, starts_at)` plus `ON CONFLICT DO NOTHING` means the second tap returns
zero rows, and the bot says so. Defends against (2).

## Use them

Download the JSON, import it in n8n (**Workflows → Import from File**), attach your
own credentials. Each workflow has a sticky note explaining the setup inside the
canvas. The bundled versions with a written setup guide are on
[Gumroad](https://verdiawan.gumroad.com).

## Need one installed or fixed?

I install and test these on your instance, or repair a workflow that stopped
working — [details and pricing](https://verdiawan.gumroad.com).
