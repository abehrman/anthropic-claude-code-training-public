# SPEC · NWP-101 — Payments export: let ops choose columns and scope

> Written before any code. Generated with `/spec`, then edited by a human.
> Load it as context when you build: `@docs/specs/NWP-101-export-options.md`

**Ticket:** [NWP-101](../tickets/NWP-101.md)
**Author:** Adam Behrman
**Status:** draft

## Problem

Ops exports payments several times a day, but every file carries every column, including card last-four. Dana's team spends 3–4 hours a month hand-cleaning files before they go to merchants, and an unedited file nearly went to the wrong merchant last quarter. They need to pick columns and scope before downloading.

## Current state

All paths are under `build-battle/merchant-console/`.

- `src/app/payments/page.tsx:68-75` — Export is a plain `<a href="/api/payments/export?…">` carrying the page's query string. No dialog, no options.
- `src/app/payments/page.tsx:36-45` — the page builds `PaymentFilters` by hand (its own status list) and calls `queryPayments`, which returns `total` for the current filter.
- `src/app/api/payments/export/route.ts:11-25` — `parseFilters` → `filterPayments` → `sortPayments` over every matching row (no pagination), then `toCsv(rows)` with the default columns and `exportFilename()`.
- `src/data/queries.ts:18` `parseFilters` — the server-side allowlist for filter params. `:45` `filterPayments`, `:72` `sortPayments`: the one query builder.
- `src/lib/csv.ts:13` `EXPORT_COLUMNS` — the ten columns, `last4` included. `:58` `toCsv(payments, columns = EXPORT_COLUMNS)` already accepts a column list in order. `:69` `exportFilename(date)` returns `payments-YYYY-MM-DD.csv` (UTC, no scope).
- `src/lib/money.ts:15` `formatMoney` — `toCsv` already formats amounts with it; `currency` is already its own column.
- `src/lib/csv.test.ts` — already pins escaping, column order (`["id","amount"]`) and the filename.
- `src/components/Drawer.tsx` — the only Radix dialog primitive in the repo.

**Ticket and rules versus the code:**
- `.claude/rules/components.md` says `src/components/` has a Dialog. It does not; `Drawer` is the dialog primitive. There is no Checkbox or Radio component either.
- The ticket warns against interpolating column names "into SQL". There is no SQL; the risk here is a column name reaching `toCsv` or the `content-disposition` filename unchecked.
- The export already covers all rows, not just the visible page. The pagination warning is about not moving the export into the browser.

## Domain rules

| Rule | Source | What breaks if ignored |
| --- | --- | --- |
| "Anything from the client — column names, currencies, limits, statuses — is checked against an allowlist before it reaches a query, a filename, or the store." | `merchant-console/CLAUDE.md`, ORG-STANDARDS #7 | Arbitrary text in the CSV header or download filename |
| "Payment filtering goes through the builder behind `GET /api/payments`. A second implementation is a bug." | `merchant-console/CLAUDE.md`, ORG-STANDARDS #6 | "All payments" and "current filter" disagree on what a payment is |
| "Money is integer minor units… Format once, at the edge, next to its currency code." | `merchant-console/CLAUDE.md`, ORG-STANDARDS #1–2 | Cents drift, or symbols land in data |
| "Storage and bucketing are UTC." | `merchant-console/CLAUDE.md` | Filename date off by one near midnight |
| Card last-four is **off** by default | Ticket AC 1 | The near-miss happens again |
| "Every input has a label, the dialog has an accessible name, focus moves into it and returns on close, Escape closes it." | `.claude/rules/components.md` | Dialog unusable by keyboard |

## Approach

The server stays the only place a file is built. `src/lib/csv.ts` gains three small server-side parsers beside the existing allowlist, mirroring `parseFilters`: selected columns (allowlisted, requested order, duplicates dropped, default is every column except `last4`), scope (`filtered` by default, or `all`), and a scope-aware filename that uses only allowlisted values (the status, or `all` / `filtered`). The export route uses them: `scope=all` runs `filterPayments` with empty filters, so both scopes go through the same builder, and an empty column selection returns 400 instead of a file. On the page, the Export link becomes a button that opens a `Drawer` with a plain GET form: column checkboxes, a scope radio with the row count for each, and hidden inputs carrying the current filters. Download is disabled when no column is checked. The page passes both row counts from `queryPayments` as props, so there is no extra fetch.

**Considered and rejected:** building the CSV in the browser from rows already loaded. It would export only the current page of 20, which is the bug the ticket warns about, and it would put the column allowlist on the client, where it is not enforcement.

## File map

| File | Add or change | Why |
| --- | --- | --- |
| `src/lib/csv.ts` | Change | Default columns, column and scope parsers, scope-aware `exportFilename` |
| `src/lib/csv.test.ts` | Change | Tests for selection order, `last4` off by default, empty selection, scope and filename |
| `src/app/api/payments/export/route.ts` | Change | Use the parsers; `scope=all`; 400 on empty selection |
| `src/app/payments/export-dialog.tsx` | Add | Client component: Drawer with the options form |
| `src/app/payments/page.tsx` | Change | Swap the link for the dialog; pass the columns, defaults and both row counts |

## Plan

1. **Parsers and filename in `csv.ts`** — done when: they are callable and the existing `csv.test.ts` still passes.
2. **Route handler** — done when: `curl` on the export route with `columns=amount&columns=id` returns those two columns in that order, a request with no `columns` returns no `last4`, `columns=bogus` returns 400, and `scope=all&status=disputed` returns every payment with an `all` filename.
3. **Run `npm test`** — done when: the baseline passes.
4. **Dialog and page** — done when: in the browser, Export opens the dialog, last-four is unchecked, both row counts show, unchecking every column disables Download, and Escape closes it with focus back on Export.
5. **Tests in `csv.test.ts`** — a subset in the requested order, last-four excluded by default, an empty selection — done when: `npm test` passes with the new cases.
6. **`npm run lint` and `/ship-ready`** — done when: both are clean.

## Verification

| Acceptance criterion | How it is proven |
| --- | --- |
| Choose columns; last-four off by default | `csv.test.ts` default-columns test; dialog shows it unchecked; `curl` without `columns` has no `last4` |
| Scope: current filter (default) or all, with the row count visible | Dialog shows both counts, matching the table's total; `curl` with `scope=all` returns the full count of rows |
| Filename reflects scope and date | `csv.test.ts` filename tests; the `content-disposition` header from `curl -I` |
| Minor units inside, formatted once, currency in its own column | Unchanged `toCsv` path through `formatMoney`; the existing amount tests still pass |
| Empty selection disables Download, no empty file | Disabled button in the browser; `csv.test.ts` empty-selection test; `curl` returns 400 |

## Risks

- The page parses filters by hand (`page.tsx:36-43`) while the route uses `parseFilters`. Their counts could diverge if a filter is added to one and not the other. Today they read the same three params; I will pass only those three as hidden inputs.
- Renaming files by scope changes the existing `exportFilename` test. I will keep the current signature working so that test still passes unchanged.
- The 400 body has no existing error shape in the repo to copy. I'll use `{ message }` with a status that means what it says, and note it in the PR.

## Out of scope

- Amount sort compares strings (`queries.ts:78-80`), so `9999` sorts after `10000`. That's a real bug, but a separate ticket.
- Moving `page.tsx` onto `parseFilters`. Worth doing, not this ticket.
- Saved export presets, other formats, and persistence (NWP-203).

## Open questions

- Should a "current filter" with a merchant or search but no status be named `payments-filtered-<date>.csv`? Merchant ids and search text stay out of filenames on purpose. This spec assumes yes.
