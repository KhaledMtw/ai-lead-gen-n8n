# AI Lead Generation & Outreach — n8n Workflow

An n8n workflow that turns a plain-English search phrase (`"roofing companies in Garland, Texas"`)
into a spreadsheet of scored local-business leads, each qualified one carrying a ready-to-send
cold email — in a single execution, with no manual research step.

It scrapes Google Maps, normalizes the results, has an LLM score every lead against a
money-signal rubric, and drafts a personalized email for the ones that pass the threshold.
Every intermediate stage is written to its own spreadsheet tab, so nothing fails silently.

---

## Pipeline

```
Manual Trigger
  │
  └─▶ Search Params ............ niche, maxResults, scoreThreshold, signature config
        │
        └─▶ Apify Google Maps ... POST run-sync-get-dataset-items (retry 5× / 3s)
              │
              └─▶ Normalize ..... map raw scrape → 18 flat fields, junk-email filter
                    │
                    ├─▶ Append to 1_Raw ──────────────────── CHECKPOINT 1
                    │
                    └─▶ Qualify ... LLM score 1–10 + one-line reason (structured output)
                          │
                          └─▶ Merge Score .... lead + score + deterministic `qualified` flag
                                │
                                ├─▶ Append to 2_Scored ───── CHECKPOINT 2
                                │
                                └─▶ Qualified?  (IF)
                                      │
                                      ├── true ──▶ Draft Email ──▶ Compose Draft ─┐
                                      │                                            │
                                      └── false ─▶ No Draft ──────────────────────┤
                                                                                   │
                                                        Append to 3_Final ◀────────┘
                                                              CHECKPOINT 3 — deliverable
```

17 nodes, one execution, append-only.

---

## Stages

### 1. Find

`Search Params` holds every knob in one place — the search niche, result cap, score threshold,
and the email signature settings. Nothing else in the workflow is hard-coded.

`Apify Google Maps` calls the `compass/crawler-google-places` actor synchronously
(`run-sync-get-dataset-items`) with `scrapeContacts: true`, a 180s timeout and 5 retries.

`Normalize` flattens the scrape into one item per lead with 18 fields. Two things it does that
matter:

- **Missing values become `—`, never `null` or `""`.** Downstream prompts are told explicitly
  that `—` means *unknown*, not *absent* — this is what stops the LLM from inventing deficiencies.
- **`emailValid` junk filter.** Apify regularly returns placeholder scrape artifacts
  (`email@gmail.com`, `test@example.com`, Wix/Sentry tracking addresses) that pass a naive
  presence check. `isUsableEmail()` rejects those by local-part and domain blocklist while
  deliberately *keeping* real generic business inboxes (`info@`, `support@`, `sales@`).

### 2. Qualify

An LLM scores each lead **1–10** with a one-line reason, returned as strict JSON through a
Structured Output Parser with **autoFix enabled** (the model is wired to the parser as well as
the chain, so malformed JSON gets repaired instead of killing the run).

The rubric looks for businesses **already spending money to get customers, inefficiently** —
not struggling ones. A business with no budget cannot buy; money already in motion is the signal.

| Signal | Effect |
|---|---|
| Occupying a paid Maps ad slot | +3 |
| 25+ listing photos / 10–24 | +2 / +1 |
| Real custom branded domain | +2 |
| Rating ≥ 4.0 with 40+ reviews | +1 |
| Active social profiles | +1 |
| **No online estimates/booking** (the gap being sold) | +2 |
| Under 15 reviews | cap at 2 |
| 500+ reviews (likely in-house marketing team) | cap at 4 |
| No reachable email | cap at 3 |
| Rating below 3.8 (service problem, not lead-flow problem) | cap at 2 |
| No real website | cap at 3 |
| Unclaimed listing | −2 |

Two guards are built into the prompt:

- **Unknown ≠ negative.** `isAdvertisement: "no"` means *the scrape didn't see them in an ad
  slot*, which is not evidence they aren't advertising. The prompt forbids treating it as a gap.
- **No invented data.** Review recency, owner replies, company size, franchise status and
  website features are all explicitly off-limits — the workflow was producing confident,
  unverifiable claims before this was locked down.

`qualified` is **not** left to the model. `Merge Score` computes it deterministically:

```js
score >= scoreThreshold && emailValid === 'yes'
```

### 3. Draft

Qualified leads go to `Draft Email`, which writes a subject plus a 60–90 word casual body,
signed with the identity configured in `Search Params`. It carries an **accuracy guard** — the
email goes to a real stranger, so every claim has to be checkable from the scraped data:

- Never assert what the business *lacks* (no website, no ads, no online quotes) — that data
  wasn't supplied.
- Never describe their website; it was never visited.
- If `onlineEstimates` is `yes`, pitch the follow-up layer behind the form instead of claiming
  they have no form.
- If the scoring reason is vague or mentions ads, ignore it rather than assert a flaw.

Non-qualified leads go to `No Draft`, which still writes them to `3_Final` with a **specific**
skip reason — `score 4 below threshold 6`, `no email address found`, or
`email found but not usable (email@gmail.com)` — so a low-scoring row is auditable rather than
just absent.

---

## Design decisions

**Append-only, no row matching.** Every sheet node appends; nothing looks up or updates an
existing row. Row-matching is the largest silent-failure surface in Sheets automations — a
mismatched key writes to the wrong row or nothing at all, and neither is visible. Appending
cannot half-succeed.

**Three tabs, not one.** `1_Raw`, `2_Scored`, `3_Final` are checkpoints. If output looks wrong,
the tab where it stopped being right identifies the broken stage immediately.

**All rows are kept, not just the winners.** Low scores land in `3_Final` marked
`qualified: false` with their reason. Reviewing the rejects is how the rubric gets tuned —
and it makes the filtering visible rather than a black box.

**Retries on every external call.** Apify, both LLM chains, and all three Sheets appends carry
`retryOnFail` 5× / 3s. `Normalize` uses `onError: continueRegularOutput` so one malformed
scrape record can't abort a whole run.

**Token caps on both model nodes.** Uncapped requests were being billed against the full context
window and returning HTTP 402 on a low balance. `Qualify` is capped at 1500 tokens, `Draft Email`
at 4000.

**Separate models per stage.** Scoring runs on a fast cheap model (20 leads × 1 call each);
drafting runs on a stronger one, since only the qualified subset reaches it. Both are single
dropdown swaps in their respective model nodes.

---

## Setup

### Requirements

- A running n8n instance (self-hosted or cloud)
- An [Apify](https://apify.com) account with API token
- An [OpenRouter](https://openrouter.ai) account with credits
- A Google account for Sheets

### 1. Prepare the spreadsheet

Create a Google Sheet with three tabs — `1_Raw`, `2_Scored`, `3_Final` — with these header rows:

**`1_Raw`**
```
name · website · email · phone · address · city · category · rating · reviews ·
mapsUrl · placeId · onlineEstimates · description · socials · isAdvertisement ·
imagesCount · listingClaimed
```

**`2_Scored`** — the `1_Raw` columns, plus:
```
emailValid · score · reason · qualified
```

**`3_Final`**
```
name · website · email · phone · city · category · rating · reviews ·
onlineEstimates · score · reason · qualified · subject · body
```

### 2. Import the workflow

In n8n: **Workflows → Import from File →** [`workflows/lead-gen-outreach.json`](workflows/lead-gen-outreach.json)

### 3. Create the three credentials

| Credential type | Used by | How |
|---|---|---|
| Header Auth | `Apify Google Maps` | Name `Authorization`, value `Bearer <your-apify-token>` |
| OpenRouter API | `Qualify Model`, `Draft Model` | Your OpenRouter API key |
| Google Sheets OAuth2 | all three append nodes | Standard n8n Google OAuth flow |

The workflow ships with credential *names* but no IDs, so n8n will prompt you to map each one
on import. **No keys are stored in this repository.**

### 4. Point it at your sheet

All three Google Sheets nodes have `documentId` set to the placeholder `YOUR_GOOGLE_SHEET_ID`.
Replace it in each with your own sheet ID — the part of the URL between `/d/` and `/edit`:

```
https://docs.google.com/spreadsheets/d/[THIS PART]/edit
```

### 5. Configure and run

Open `Search Params` and set:

| Field | Meaning |
|---|---|
| `niche` | Google Maps search phrase, e.g. `plumbers in Austin, Texas` |
| `maxResults` | Leads per run (20–25 keeps Apify under ~90s) |
| `scoreThreshold` | Minimum score to earn a drafted email |
| `agencyName` | Your company name, used in the emails |
| `senderName` | Sign-off name |
| `cta` | The single call to action in every email |

Then hit **Execute Workflow** and watch the three tabs fill in order.

---

## Retargeting to another niche

Change `niche` in `Search Params` — that's the whole change for a new city or vertical. The
rubric is written in general local-business terms (spend signals, review volume, booking gap),
not roofing-specific ones.

Worth knowing when you do: the score *distribution* shifts by market. In a dense major metro,
established operators are uniformly healthy and nearly everything clears the threshold — there,
the score *tier* is your shortlist rather than the boolean `qualified` flag. In smaller
markets the threshold separates leads properly on its own. Check the spread in `2_Scored` after
the first run of a new niche and move `scoreThreshold` to match.

---

## Cost per run

At 20 leads: Apify Google Maps ≈ free-to-cents, one scoring call per lead on a cheap model, and
one drafting call per *qualified* lead only. A 20-lead run typically lands in the low single-digit
cents of LLM spend.

---

## Repository layout

```
.
├── workflows/
│   └── lead-gen-outreach.json    the workflow, sanitized for import
├── .gitignore
├── LICENSE
└── README.md
```
