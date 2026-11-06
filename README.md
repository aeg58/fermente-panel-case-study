<div align="center">

# Fermente Paneli

**Sales, profit, payout-audit and customer-question dashboard for a brand selling on Trendyol and Hepsiburada**

![Status](https://img.shields.io/badge/status-on%20its%20own%20server%20·%20handover%20in%20preparation-2e7d32)
![Release](https://img.shields.io/badge/release-v0.24.0--rc.1%20·%205%20Nov%202026-455a64)
![Next.js](https://img.shields.io/badge/Next.js-16-000000?logo=nextdotjs)
![React](https://img.shields.io/badge/React-19-61dafb?logo=react&logoColor=black)
![Prisma](https://img.shields.io/badge/Prisma-7-2d3748?logo=prisma)
![SQLite](https://img.shields.io/badge/SQLite%20·%20libSQL-003b57?logo=sqlite&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-GHCR-2496ed?logo=docker&logoColor=white)

Black Wool Project · **Project Lead & Developer** · [Türkçe](README.tr.md)

<img src="docs/screenshots/01-overview-masked.jpg" alt="Overview" width="900">

</div>

> **About this repository.** Fermente Paneli is built at Black Wool for **Fermente Mutfağım**, a
> fermented-food and natural-care brand that sells on Trendyol and Hepsiburada. Its source code is
> private; this is a case study. Screenshots were taken from the running system on 28 Oct 2026;
> revenue, amounts and order numbers are covered with solid boxes.

---

## At a glance

| | |
|---|---|
| **Client** | Fermente Mutfağım — fermented foods and natural care products, sold on two marketplaces |
| **What it is** | A reporting panel that **never changes prices, stock or listings** on either marketplace. Its only write is an answer to a Hepsiburada customer question, sent after a preview and a second click |
| **Screens** | Overview · Orders & returns · Products · Finance (payout audit, payments, advertising, rules) · Customer questions |
| **Data in** | Hepsiburada API every 15 minutes · Trendyol daily Excel exports · product cost template |
| **Scale** | This month 601 orders; payout audit checks every order line against the marketplace's own deductions — 614 orders in October, 29 flagged |
| **Codebase** | 28 models · ~6,600 lines of TypeScript in `src` · 524 commits on the main branch · 21 test files / ~130 test cases, including 11 browser tests |

<sub>Codebase figures measured from the repository on 6 Nov 2026 (first published 28 Oct: v0.10.7-rc.1). Order figures read from the running panel on 28 Oct 2026.</sub>

---

## The problem

The brand's owner sells the same products on two marketplaces, each with its own seller portal,
its own reports and its own way of deducting commission, shipping, service fees and advertising
from the payout. Answering simple questions — *How much did we sell yesterday? What will actually
land in the bank? Did the marketplace deduct what it should have? Is there a customer question
waiting?* — meant logging into both portals and downloading spreadsheets.

The owner needed **one screen for both marketplaces**, and a way to catch payout deductions that do
not match the marketplace's own rules — without giving a new tool any power to change prices, stock
or listings in the shops.

## What I built

| Screen | What it shows |
|---|---|
| **Overview** | Revenue, orders, units and basket size for a chosen day, a 7-day revenue chart split by marketplace, a live "today" box, and an attention list (e.g. *product costs not entered*) |
| **Orders** | Every order from both marketplaces with products, city, carrier and status; returns in a separate tab; Excel export |
| **Products** | Products and bundles with their contents, and a downloadable cost template so profit can be calculated |
| **Finance · payout audit** | Each deduction the marketplace made is compared with what its own rules predict; mismatching orders are flagged with the reason |
| **Finance · payments & ads** | When money will be paid, and advertising cost per period |
| **Customer questions** | Questions from both marketplaces in one inbox, median answer time, questions that expired unanswered, and questions **flagged as health-related** so the owner answers those personally. Hepsiburada questions can be answered from the panel |
| **Users & roles** | Order, inventory and finance roles that open only their own sections; one person can hold several roles. A step-by-step guided tour of every screen, offered once on first login |

### In progress

| Feature | Status |
|---|---|
| **Answering Trendyol questions** | Hepsiburada answering is done (see decision 2). Trendyol uses the same safeguards and waits for the brand's Trendyol API key |
| **Morning summary e-mail** | Planned as the third part of the system next to the panel and the collector: every morning at 08:00 the owner receives yesterday's summary, with numbers identical to the panel's; the admin copy carries a health line (collector's last run, orders and questions received) |

---

## Screens

<table>
<tr>
<td width="50%"><img src="docs/screenshots/03-payout-audit-masked.jpg" alt="Payout audit"><br><sub><b>Payout audit.</b> 614 orders checked this month: 475 match every rule, 36 are explained (return, cancellation), 74 are waiting for the marketplace's record, 29 need attention. Amounts and reasons masked.</sub></td>
<td width="50%"><img src="docs/screenshots/04-customer-questions.jpg" alt="Customer questions"><br><sub><b>Customer questions.</b> One inbox for both marketplaces, with the data freshness of each source shown at the top; health-related questions are flagged for a personal answer.</sub></td>
</tr>
<tr>
<td colspan="2"><img src="docs/screenshots/02-orders-masked.jpg" alt="Orders"><br><sub><b>Orders.</b> Both marketplaces in one list with status counts; order numbers and amounts masked. No customer names are stored — see below.</sub></td>
</tr>
</table>

---

## Architecture

```mermaid
flowchart LR
    HB[Hepsiburada API] -->|GET only · every 15 min| T[Collector service]
    TYX[Trendyol Excel exports] -->|upload| X[Excel reader<br/>allow-listed columns]
    T --> DB[(SQLite / libSQL)]
    X --> DB
    DB --> P[Next.js 16 panel]
    P -->|HTTPS via Caddy| O[Owner's browser]
    P -.->|one write: question answer<br/>after preview + second click| HB
    subgraph Server[Own Docker containers on a Linux server]
      T
      X
      DB
      P
    end
```

The panel and the collector are **two services built from the same image**: the collector pulls
data, the panel reads the database. The only call the panel makes to a marketplace is the answer to
a Hepsiburada question. A nightly backup job runs on the server.

---

## Engineering decisions

### 1. Read-only by construction, with one deliberate exception

The collector calls only `GET` endpoints of Hepsiburada's order, accounting and question services.
Response bodies are never logged — the log holds only counts, endpoint names and HTTP status — and
API keys come from the environment and are never printed. The panel's single write, answering a
customer question, is off by default: it is switched on by a server setting, only after the brand's
side approved it, and it is the subject of the next decision.

### 2. An answer that cannot be taken back

Hepsiburada's API has no endpoint to edit or delete an answer, so every safeguard runs **before**
the request: a preview shows the customer's question and the answer exactly as the customer will see
it (with a red reminder on health-related questions), and only a second click sends it. A database
lock is written *before* the request, so two clicks or two tabs cannot send twice. After sending, the
panel reads the question back from Hepsiburada and compares the text. There are three outcomes:
*sent* (read back, identical), *not sent* (rejected, no answer on Hepsiburada — the user can retry)
and *unknown* (could not be read back — the panel never sends again; someone checks on Hepsiburada).
The whole path was tested end-to-end against Hepsiburada's sandbox with a test question before it
was switched on.

### 3. Security tested from outside, not assumed

Before handover I added the tests that had never been run. An independent review found that a
crafted Excel file could exhaust the server's memory; a test reproduced it — a file under 1 KB made
Node fill 4 GB and crash — and the reader now caps rows, columns, archive entries and the bytes
actually unpacked. Security headers (CSP, HSTS, frame and referrer rules) were added and checked; an
OWASP ZAP baseline scan of the server reported 0 failures; 11 Playwright browser tests click through
every section, including uploading a malicious Excel file; a light load test (10 clients for 30 s, 1,842
requests, all successful, median 108 ms) left the other sites on the same server unaffected.

### 4. No customer personal data, enforced at two doors

The data model's first rule is that **buyer personal data never enters the database**. The collector
keeps only non-personal fields from each order. The Excel reader writes only columns that appear in
an explicit allow-list — if an export ever contains an unexpected column, the import **stops**
instead of storing it, so a changed or broken export cannot leak a personal field into the panel.
Imports run in a single transaction: all rows or none.

### 5. Profit uses the marketplace's numbers, not ours

Profit = what the marketplace will pay − product cost − shipping. Deductions are **not
recomputed** from rules; they come from the marketplace's own records (Trendyol's order ledger,
Hepsiburada's accounting service). Until Hepsiburada posts its record, the payout is estimated from
the order's commission and shown as *not final*. If a product has no cost entered, the order shows
*no cost* rather than a made-up profit. Where Hepsiburada already includes a shipping share in its
deduction, the manual shipping cost is not applied again — measured on 3,070 Hepsiburada orders.

### 6. The audit explains, it does not accuse

Every deduction is compared with the amount the marketplace's own rules predict. Orders split into
*matches*, *explained* (return or cancellation), *waiting for the record* and *needs attention*;
each flagged order carries the reason in words. The panel only points; disputes are filed in the
marketplace's own portal.

### 7. Advertising cost from cumulative snapshots

Trendyol's ad report is cumulative from the campaign's start to the moment of download, and its
date filter does not split the figure by period (measured). The panel therefore stores each upload
as a snapshot and computes a period's cost as *last snapshot before the period ends − last snapshot
before it starts*.

### 8. A pinned database engine

Prisma downloads its schema engine from its own server, outside the npm supply chain. A wrapper
script verifies the engine's SHA-256 against a pinned value and **stops** on mismatch; any further
download attempt is pointed at an unreachable address so it fails loudly. Passwords are hashed with
Node's built-in scrypt — no extra dependency.

---

## Tech stack

| Layer | Technology |
|---|---|
| App | Next.js 16, React 19, TypeScript, Prisma ORM 7, SQLite via libSQL |
| Data in | Node collector (Hepsiburada REST, GET only), Python Excel reader with allow-list · one write: Hepsiburada question answer |
| Quality | Node test runner (20 test files), Python unit tests for the Excel reader, Playwright browser tests, OWASP ZAP baseline scan |
| Delivery | Docker, GitHub Actions → GHCR, Caddy, nightly backup |

---

## My role

I built the panel at Black Wool, which manages the brand's marketplace operations: requirements
with the owner, screen designs approved before building, the data model, the collector and Excel
reader, the profit and payout-audit logic, and deployment to the server in stages, each released as
a tagged build.

## What is not in this repository

Source code, API keys, the panel's address, revenue, amounts, order numbers and any customer data.
