# AI Job Application Assistant (n8n)

An n8n automation that finds relevant jobs across multiple job boards, tailors an ATS-friendly CV summary and skills list for each role with Gemini, and keeps **me in the loop**: every application is approved by email before it is marked as applied. Everything is logged in Google Sheets.

> Built during my AI Engineer internship at OptimusFox.
> Status: **working prototype** — discovery, tailoring, approval and logging work end to end. Automatic submission of applications is intentionally not implemented yet (see [Known limitations](#known-limitations)).

---

## What it does

1. **Discovers jobs** from the Adzuna API (UK and other supported countries) and an Apify multi-board actor (LinkedIn, Indeed, Bayt, Talent.com and more, including Pakistan).
2. **Cleans the data** — merges both sources, normalises them into one format, keeps only relevant titles (AI, ML, software, Laravel, automation) and removes duplicates.
3. **Logs every job** in Google Sheets with company, title, platform, location, apply link and status.
4. **Tailors the CV** — Gemini reads the job and rewrites my professional summary and skills in ATS-friendly language, and lists any extra requirements it detects (transcript, cover letter, portfolio, etc.).
5. **Asks for approval** — the tailored content is emailed to me. I reply `APPROVE` or `SKIP`.
6. **Updates the sheet** — a second workflow reads my reply and sets the row to `Applied` or `Skipped`.

---

## Architecture

```
WORKFLOW 1 — Job Search Pipeline
Trigger ─┬─> Adzuna API ──┐
         ├─> Apify actor ─┴─> Merge ─> Normalize ─> Dedup Check ─> Keyword Filter ─> Append row (Google Sheets)
         └─> Get Existing Applications (Sheets) ───────────────┘ (used by Dedup Check)

WORKFLOW 2 — CV Tailoring & Approval Request
Trigger ─> Get Pending Jobs ─> Limit ─> Gemini Tailor CV ─> Parse AI Response
        ─> Build Resume HTML ─> Send Approval Email (Gmail) ─> Mark Pending Approval (Sheets)

WORKFLOW 3 — Process CV Approval
Gmail Trigger ─> Extract Decision ─> If approved?
                                       ├─ true  ─> Mark as Applied (Sheets)
                                       └─ false ─> Mark as Skipped (Sheets)
```

### Row status lifecycle

`Found` → `Pending Approval` → `Applied` / `Skipped`

### Google Sheet columns

| Date | Company | Title | Platform | Location | Apply URL | Status | CV Sent |
|---|---|---|---|---|---|---|---|

---

## Tech stack

| Area | Tool |
|---|---|
| Orchestration | n8n (self-hosted, Docker) |
| Job sources | Adzuna API, Apify `truefetch/job-search` actor |
| AI | Google Gemini API (free tier) |
| Storage / logs | Google Sheets |
| Email | Gmail (send + trigger) |

---

## Repository structure

```
.
├── README.md
├── workflows/
│   ├── 1-job-search-pipeline.json
│   ├── 2-cv-tailoring-approval.json
│   └── 3-process-cv-approval.json
└── screenshots/
```

---

## Setup

### Prerequisites
- A running n8n instance (self-hosted or cloud)
- Google account (Sheets + Gmail) connected to n8n via OAuth2
- Free [Adzuna developer](https://developer.adzuna.com) account (App ID + App Key)
- [Apify](https://apify.com) account and API token
- Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey)

### Steps
1. Create a Google Sheet with the columns listed above (first row = headers).
2. In n8n: **Workflows → Import from file**, and import the three JSON files from `/workflows`.
3. Create these credentials in n8n:

| Service | Credential type | Value |
|---|---|---|
| Adzuna | Custom Auth | `{"qs": {"app_id": "YOUR_APP_ID", "app_key": "YOUR_APP_KEY"}}` |
| Apify | Header Auth | Name `Authorization`, value `Bearer YOUR_APIFY_TOKEN` |
| Google Sheets | OAuth2 | Sign in with Google |
| Gmail | OAuth2 | Sign in with Google |

4. In the **Gemini Tailor CV** node, replace `YOUR_GEMINI_API_KEY` in the URL (or move it into a credential). Never commit a real key.
5. In every Google Sheets node, paste your own spreadsheet ID in place of `YOUR_SHEET_ID` and attach your Google Sheets credential. The exported workflows contain **no credentials**, so also attach your Apify, Adzuna, Gmail and Google Sheets credentials to the matching nodes after importing.
6. In the **Send Approval Email** node, replace `YOUR_EMAIL@example.com` with your own address. In **Build Resume HTML**, replace `YOUR_CITY`, `YOUR_COUNTRY`, `YOUR_PHONE`, `YOUR_EMAIL` and the name, links and experience with your own details.
7. In the Gmail Trigger of workflow 3, set the search to: `subject:"CV Approval Needed" in:inbox`
8. Set your own target keywords in the **Keyword Filter** node and replace the base summary/skills text inside the Gemini prompt with your own.
9. Run workflow 1, then workflow 2, reply `APPROVE` to the email, and watch workflow 3 update the sheet.

### Notes on the Apify actor
- Input fields used: `keyword`, `country`, `max_results`.
- `country` must be the **full English name** (for example `Pakistan`, `United Kingdom`), not an ISO code.

---

## Problems I ran into (and how I fixed them)

| Problem | Cause | Fix |
|---|---|---|
| Duplicates kept being logged | Adzuna adds a random tracking token to every apply link, so the same job looked new each run | De-duplicate on `title + company` instead of the URL |
| Dedup node failed with "node hasn't been executed" | Referencing a node that was not wired into the execution path | Connected the sheet-read node directly into the Dedup Check node |
| Status updates failed on `row 0` | The Gmail node output does not contain the sheet row number | Read it from the earlier node: `$('Parse AI Response').item.json.row_number` |
| Reply workflow always had `rowNumber = null` | Gmail returns `Subject` (capital S) while the code read `subject` | Read `$json.Subject \|\| $json.subject` |
| Apify run timed out / was aborted | Free monthly credit used up and a slow actor run | Smaller `max_results`; paid plan or another source needed for large runs |
| Gemini returned 404 | The model I first used had been retired | Switched to a current Gemini model |

---

## Known limitations

- **Applications are not auto-submitted.** Most boards (LinkedIn, Indeed, Bayt) use their own apply forms, and automating those can violate their terms. The final submission stays manual on purpose.
- The tailored CV is currently **text + an HTML preview**, not yet a generated PDF/DOCX attachment.
- The **Limit** node is set to `1` for testing; the planned daily cap is 50.
- Apify's free credit is limited; bulk daily searches need a paid plan or another source.

## Roadmap

- [ ] Generate a tailored PDF CV per job and attach it automatically for email-based applications
- [ ] Daily scheduler with a configurable cap (target: 50 jobs/day)
- [ ] Daily summary email of everything processed
- [ ] Attach supporting documents from Google Drive when a job asks for them
- [ ] Scholarship discovery + criteria matching using the same pipeline (SOP draft assist, manual submission)

---

## Responsible use

This project keeps a human approval step on purpose. Respect each job board's terms of service, do not mass-spam recruiters, and never commit API keys, personal documents or ID numbers to a public repository.

---

## Author

**Muhammad Mujtaba Ahmad** — Computer Engineer, AI Engineer Intern
GitHub: [engrmujtabaahmad](https://github.com/engrmujtabaahmad) · LinkedIn: [muhammad-mujtaba-ahmad](https://www.linkedin.com/in/muhammad-mujtaba-ahmad-29a3b7360)
