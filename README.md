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

[News to draft — Google News + OpenAI](workflows/berita-ke-draf.json) is the
workflow I use for AI drafting: it reads a news feed on a schedule, drops
stories it has already seen, and asks a model for a structured draft it can put
in a sheet.

It is deliberately **not** in the table above. The 47 cases were run against a
numbered test matrix I kept; this one was not, so counting it would overstate
what is here. Read it for the pattern, not for a test count.

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
