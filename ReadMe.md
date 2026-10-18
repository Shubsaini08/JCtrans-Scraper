# JCtrans Scraper — Complete Guide

A two-stage, resumable Selenium + BeautifulSoup tool that signs in to JCtrans, walks the **Company Directory**, opens **every company profile**, and saves company details plus **every visible contact** (name, title, emails, all phone variants) to TXT, JSON and CSV files.

Script file: `jctrans_scraper.py` (you run it as `jct.py` — the name does not matter, use whichever filename you saved).

---

## 1. Contents

1. [Quick start](#2-quick-start)
2. [Installation](#3-installation)
3. [Signing in](#4-signing-in)
4. [How it works (workflow)](#5-how-it-works-workflow)
5. [Command-line options](#6-command-line-options)
6. [Command cookbook](#7-command-cookbook)
7. [Selecting pages](#8-selecting-pages--important-behaviours)
8. [Output files](#9-output-files)
9. [Data model](#10-data-model)
10. [Resume, retry and fault tolerance](#11-resume-retry-and-fault-tolerance)
11. [What was verified, and what was not](#12-what-was-verified-and-what-was-not)
12. [Change log](#13-change-log--what-changed-over-time)
13. [Troubleshooting](#14-troubleshooting)
14. [Limitations](#15-limitations)
15. [Responsible use](#16-responsible-use)

---

## 2. Quick start

```powershell
cd D:\logi\jc
python -m pip install selenium webdriver-manager beautifulsoup4

# set credentials for this terminal session only
$env:JCTRANS_USER = "your_user_id"
$env:JCTRANS_PASS = "your_password"

# small test: page 1, first 3 companies
python jct.py --stage all --max-pages 1 --max-companies 3 --debug-html --user-data-dir chrome_profile
```

Then open `output\jctrans_all_contacts.txt` and check the contacts.

---

## 3. Installation

Requirements: Python 3.9 or newer (the script uses `argparse.BooleanOptionalAction`), Google Chrome installed, internet access.

```powershell
python -m pip install selenium webdriver-manager beautifulsoup4
```

macOS / Linux:

```bash
python3 -m pip install selenium webdriver-manager beautifulsoup4
```

`webdriver-manager` downloads a matching ChromeDriver automatically on first run.

**Offline self-test (no browser, no login):** parses a saved HTML dump such as `jc.txt`.

```powershell
python jct.py --offline-test jc.txt
```

---

## 4. Signing in

The script checks whether you are signed in, using markers confirmed in your HTML dump:

| State | Marker in the page |
|---|---|
| Signed in | `div.sign-out-btn` inside `div.user-drop-down` (a *hidden* dropdown) |
| Signed out | `button.login-btn` with the text `SIGN IN` |

If you are signed out, the script tries, in order:

1. **Automated login** (if a user ID and password are supplied) — clicks `SIGN IN`, finds the username/password fields (also inside iframes), tries the password/account tab if the box opens on QR code, fills the credentials, ticks a terms checkbox if one is shown, and submits.
2. **Pause for CAPTCHA / verification code** — if one appears, it stops and asks you to complete it in Chrome, then press ENTER. It never tries to get around these.
3. **Manual sign-in** — if automation fails, you sign in by hand and press ENTER.

If your session expires in the middle of a run, the same process repeats and the run continues.

### Supplying credentials

Credentials are **never stored in the script, logs or output files**. They are held in memory only.

PowerShell:

```powershell
$env:JCTRANS_USER = "your_user_id"
$env:JCTRANS_PASS = "your_password"
```

CMD:

```bat
set JCTRANS_USER=your_user_id
set JCTRANS_PASS=your_password
```

macOS / Linux:

```bash
export JCTRANS_USER="your_user_id"
export JCTRANS_PASS="your_password"
```

Other ways:

- `--username your_user_id` — takes the user ID from the command line. If the password variable is not set, the script prompts for it with hidden input.
- `--manual-login` — skips automation entirely.

### Keep the session between runs

Always add `--user-data-dir chrome_profile`. Chrome stores the session in that folder, so later runs usually say `[AUTH] Signed-in session detected` and skip login completely. Headless mode (`--headless`) only works when that folder already holds a valid login, because a manual sign-in is impossible without a visible window.

> If a password has ever been pasted into a chat, email or ticket, change it on JCtrans.

---

## 5. How it works (workflow)

```
┌─────────────────────────────────────────────────────────────────┐
│ START → open Chrome → open jctrans.com/en/ → check sign-in      │
│         (already signed in? → continue. Else auto/manual login) │
└───────────────┬─────────────────────────────────────────────────┘
                ▼
   STAGE A — DISCOVER  (--stage discover | all)
   • open /en/company/, read the pager: total companies, last page,
     and the page-URL pattern (…/en/company-page-{n})
   • for each page from --start-page to min(--max-pages, last page):
       load page → wait for rows → confirm the active page number
       → collect 20 company links (id, name, location, logo)
       → dedupe → checkpoint to disk
   • stops safely after 3 consecutive empty/failed pages
                ▼
   STAGE B — PROFILES  (--stage profiles | all)
   • queue = discovered companies that are not yet completed
   • for each company:
       open its profile URL → verify it is that company (no login/
       error/redirect page) → scroll to load lazy content → click only
       safe "View More" expanders → parse company details →
       parse EVERY contact card → classify emails/phones → dedupe
       → store → checkpoint every 5 profiles
   • failures retry up to 3 times, then are recorded and skipped
                ▼
   FINISH (also on Ctrl+C or crash): write all TXT / JSON / CSV
   outputs, close Chrome.
```

Key design points:

- **Two stages** so a long run never loses progress: discovered links are saved first, profiles are processed from that saved list.
- **No guessing**: company URLs and IDs come from the real directory links. Missing values stay empty; no emails, countries or WhatsApp labels are invented.
- **Stops, does not evade**: on a rate-limit, block, or CAPTCHA page the script pauses for you instead of working around it.
- **Never clicks** inquiry, chat, contact, message or submit buttons.

---

## 6. Command-line options

| Option | Default | Meaning |
|---|---|---|
| `--stage {discover,profiles,all}` | `all` | `discover` = collect links only; `profiles` = extract from saved links; `all` = both |
| `--start-page N` | `1` | First directory page to process |
| `--max-pages N` | `840` | **Last** directory page to process (inclusive). The real page count is read from the site and the lower value wins |
| `--max-companies N` | `0` | Max profiles to process in this run. `0` = no limit |
| `--output-dir DIR` | `output` | Where results are written |
| `--delay SECONDS` | `2.0` | Pause between page loads (keep it conservative) |
| `--page-timeout SECONDS` | `30` | Timeout for page loads and waits (min 5) |
| `--profile-details` / `--no-profile-details` | on | Deep profile extraction: scroll fully and click safe "View More" expanders |
| `--resume` | off | Reuse saved state in the output folder (see below) |
| `--retry-failed` | off | Also retry profiles that failed earlier |
| `--debug-html` | off | Save rendered HTML/text of the first 3 profiles, plus any profile whose contact section is present but empty, plus failures |
| `--headless` | off | Run Chrome without a window (needs a saved login in `--user-data-dir`) |
| `--user-data-dir DIR` | none | Persistent Chrome profile folder (keeps your login) |
| `--username ID` | env `JCTRANS_USER` | JCtrans user ID |
| `--manual-login` | off | Skip automated login |
| `--offline-test FILE` | none | Parse a saved HTML dump and print results; no browser |

Arguments are validated before the browser starts (for example `--start-page` cannot exceed `--max-pages`).

---

## 7. Command cookbook

All examples use PowerShell and `jct.py`; on macOS/Linux use `python3` and your filename.

**Small test**

1st:
python jct.py --stage all --max-pages 1 --max-companies 3 --debug-html --user-data-dir chrome_profile

```powershell
python jct.py --stage all --max-pages 1 --max-companies 3 --debug-html --user-data-dir chrome_profile
```

**A single page (page 5) — see the caveat in section 8**

```powershell
python jct.py --stage all --start-page 5 --max-pages 5 --user-data-dir chrome_profile --resume
```

**A range (pages 10–20)**

```powershell
python jct.py --stage all --start-page 10 --max-pages 20 --user-data-dir chrome_profile --resume
```

**Next range, continuing the same dataset**

```powershell
python jct.py --stage all --start-page 21 --max-pages 50 --user-data-dir chrome_profile --resume
```

**Discover links first, extract profiles later (recommended for large runs)**

```powershell
# 1) links only (fast)
python jct.py --stage discover --start-page 1 --max-pages 100 --user-data-dir chrome_profile --resume

# 2) extract profiles in chunks of 500
python jct.py --stage profiles --max-companies 500 --user-data-dir chrome_profile --resume
```

**Continue after an interruption (Ctrl+C, crash, power cut)** — rerun the same command with `--resume`:

```powershell
python jct.py --stage all --start-page 1 --max-pages 100 --user-data-dir chrome_profile --resume
```

**Retry only failures**

```powershell
python jct.py --stage profiles --retry-failed --user-data-dir chrome_profile --resume
```

**Different output folder / slower pacing**

```powershell
python jct.py --stage all --max-pages 10 --output-dir batch_01 --delay 3 --user-data-dir chrome_profile
```

**Headless (after you have signed in once with the same profile folder)**

```powershell
python jct.py --stage profiles --headless --max-companies 200 --user-data-dir chrome_profile --resume
```

**Skip automation, sign in by hand**

```powershell
python jct.py --stage all --max-pages 1 --manual-login --user-data-dir chrome_profile
```

---

## 8. Selecting pages — important behaviours

- **Pick a range with `--start-page` and `--max-pages`.** `--max-pages` is the *last page*, not "how many pages". `--start-page 5 --max-pages 5` is page 5 only.
- **Use `--resume` when running ranges one after another.** Without it, the script moves your previous state into a `state_backup_<timestamp>` folder (and renames the earlier `jctrans_*` result files with a `.bak_<timestamp>` suffix) and starts a fresh dataset. With it, new pages and profiles are added to the existing data.
- **`--max-companies 0` means no cap.** Your test runs used `20`, which stops after 20 profiles.
- **Stage B works from the whole saved queue, not just the pages in the current command.** The profile queue is "every discovered company that is not yet completed". So `--stage all --start-page 21 --max-pages 50 --resume` will also process any companies left over from earlier discovery. To keep a run limited, either set `--max-companies`, or run `--stage discover` for the pages you want and then `--stage profiles --max-companies N`.
- **`--stage profiles` always loads the saved state** (it behaves as if `--resume` were set) and exits with an error if nothing has been discovered yet.
- **Real size of your account's directory** (from your live run): 89,029 companies over 4,452 pages, 20 per page. `--max-pages 840` is simply a ceiling; the script never loops past the real last page.
- **Time estimate (not measured):** each profile involves a page load, scrolling and the `--delay` pause, so expect several seconds per company. A full 89,029-profile run is on the order of days of continuous running. Work in batches.

---

## 9. Output files

All files are in `--output-dir` (default `output/`), UTF-8 encoded.

| File | Content |
|---|---|
| `jctrans_all_contacts.txt` | **Main deliverable.** One block per company, one `CONTACT n` section per person: name, designation, department, email(s), `Phone 1…n`, mobile/office/WhatsApp/fax (only if explicitly labelled), other channels, raw contact text. Companies with no contacts get a status record. |
| `jctrans_contacts_table.txt` | Sorted one-row-per-contact table (Company, ID, Country, City, Contact, Designation, Email, Phones, URL) with computed column widths |
| `jctrans_contacts_table.tsv` | Same table, tab-delimited, **nothing truncated** |
| `jctrans_companies_summary.txt` | One section per company: location, address, website, description, services, transport modes, membership, establishment year, employee size, certifications, contact count and status, errors |
| `jctrans_company_directory.txt` | Discovered companies: page, name, ID, URL, directory location, status |
| `jctrans_failed_profiles.txt` | Failed profiles: ID, URL, page, stage, error type/message, retry count, safe-to-retry |
| `jctrans_progress.txt` | Run start/update time, last page, counts, current URL, resume state |
| `jctrans_companies.json` | Full company records with nested contacts (Unicode preserved) |
| `jctrans_contacts.csv` | One row per contact. Phone lists are JSON arrays in quoted strings so Excel keeps them as text (opens correctly with the UTF-8 BOM) |
| `jctrans_scraper.log` | Detailed log (credentials are never written) |
| `state/discovered.json`, `state/profiles.json` | Resume state, written atomically |
| `debug_html/<id>.html` / `.txt` | Only with `--debug-html` |

### `jctrans_all_contacts.txt` format

```
================================================================================
COMPANY: Bondex Logistics Co.,Ltd.(Chengdu Branch)
COMPANY ID: 07610y1
LOCATION: Chengdu, China
PROFILE: https://www.jctrans.com/en/company/07610y1/

CONTACT 1
Name: Jackson Ren
Designation: Oversea sales manager
Department:
Email: jackson@bondex.com.cn
Phone 1: +86 17381089707
Phone 2: 17381089707
Phone 3: +8617381089707
Mobile:
Office Phone:
WhatsApp:
Fax:
Other Contact Channels: {"last_active_displayed": "(Active 18 hours ago)"}
Raw Contact Text: ...

--------------------------------------------------------------------------------
CONTACT 2
...
```

(Taken from the supplied dump; that profile has 4 contacts.)

---

## 10. Data model

One company object with a nested list of contacts:

```json
{
  "company_id": "07610y1",
  "company_name": "Bondex Logistics Co.,Ltd.(Chengdu Branch)",
  "profile_url": "https://www.jctrans.com/en/company/07610y1/",
  "location": {"city": "Chengdu", "region": "", "country": "China",
               "full_address": "No. 500 Hangshu Avenue, ...", "postal_code": ""},
  "company_website": "https://www.bondex.com.cn/",
  "company_description": "...",
  "services": ["FCL", "LCL", "Air Freight", "Warehousing", "Truck"],
  "transport_modes": ["FCL", "LCL", "Air Freight", "Truck"],
  "membership_level": "JC Premium",
  "membership_tenure": "1-Year",
  "establishment_year": "2006",
  "employee_size": "101-500",
  "certifications": ["CIFFA", "NVOCC", "IATA", "FIATA", "AEO", "ISO", "..."],
  "other_company_details": {"Advantageous Ports": ["CTU", "PVG", "CAN"], "...": "..."},
  "contacts": [
    {
      "name": "Jackson Ren",
      "designation": "Oversea sales manager",
      "email": "jackson@bondex.com.cn",
      "email_normalized": "jackson@bondex.com.cn",
      "emails_all": ["jackson@bondex.com.cn"],
      "phone_numbers_raw": ["+86 17381089707", "17381089707", "+8617381089707"],
      "phone_numbers_normalized": ["+8617381089707"],
      "mobile": "", "office_phone": "", "whatsapp": "", "fax": "",
      "other_contact_channels": {"last_active_displayed": "(Active 18 hours ago)"},
      "raw_contact_text": "..."
    }
  ],
  "contact_count": 4,
  "contact_section_present": true,
  "contact_extraction_status": "found",
  "contact_extraction_notes": "contact cards via selector '#contactUs .contactCard'",
  "source_directory_page": 1,
  "extracted_at": "2026-10-11T12:00:00+05:30",
  "extraction_status": "success"
}
```

Rules the script follows:

- **Phones are strings, never numbers.** Every raw variant is kept exactly as displayed (`+86 17381089707`, `17381089707`, `+8617381089707`). `phone_numbers_normalized` only contains numbers with an explicit `+` or `00` prefix; nothing is guessed.
- **Source text is preserved**, including typos (for example the title "Manger").
- **Contact status values:** `found`, `none_visible`, `restricted` (masked values or "sign in to view" text), `failed`.
- **When zero contacts are found**, the notes say why: no contact section, section present but empty, login/session problem, or selector mismatch.
- **Contact de-duplication** merges only strong matches (same name + email, or same name + identical phone set). Two people sharing a phone or a job title are never merged.
- **Failure fields** (`error_stage`, `error_type`, `error_message`, `retry_count`) appear on failed records.

---

## 11. Resume, retry and fault tolerance

- **Checkpoints:** discovery state is saved after every directory page; profile state every 5 profiles and at the end. JSON writes are atomic (temporary file, then replace).
- **Ctrl+C:** caught; progress is saved, outputs are regenerated, Chrome is closed.
- **Chrome crash / dead session:** Chrome is recreated and the run continues.
- **Expired login:** automatic re-login, with manual fallback.
- **Retries:** up to 3 attempts per directory page and per profile for timeouts and browser errors. Error/not-found pages are not retried (they are not temporary).
- **Restrictions:** on a rate-limit, block or CAPTCHA page, the script pauses for you (or stops when headless). It does not try to evade.
- **On resume:** completed profiles are skipped; failed profiles are skipped unless `--retry-failed` is given; discovered pages already finished are skipped.
- **Fresh start:** run without `--resume`. Old state and result files are archived, not deleted.

---

## 12. What was verified, and what was not

**Verified against the supplied `jc.txt` dump (offline):**

- Company list rows (`div.membership-list-content-center-list-item`, 20 per page), links, names, locations
- Pager: `span.el-pagination__total`, `div.el-pager a.number[.active]`, `a.btn-next`, page URLs `.../en/company-page-N`
- Profile: `h1` name; `p.desc-box` label/value pairs (Location, Address, Establishment Time, Enterprise Size, Website); `h2` sections (Service Scope, Company Profile, Certificates, Contact Us); membership header
- Contacts: `#contactUs .contactCard`, `.im-info-box` (name), next `div` (title), `div.content` chips (emails/phones) — extracted all 4 Bondex contacts with all phone variants
- Login state markers (`.sign-out-btn` vs `button.login-btn`)

**Confirmed by your live runs:**

- Signed-in session detected, directory loaded, pager read (89,029 companies / 4,452 pages)
- Pagination to a chosen page and profile extraction worked: 3-profile test (8 contacts) and 20-profile test (44 contacts), 0 failures

**Not verified (no HTML available):**

- The login form on `passport.jctrans.com` — automated login uses generic detection. If it does not work, the script falls back to manual sign-in.
- Directory pages beyond page 1 in bulk
- Fields that appear only for some company types or account levels
- Labels for mobile / fax / WhatsApp — the contact chips carry no text labels, so these fields stay empty unless the page states them explicitly (they are still captured in the raw phone list)

---

## 13. Change log — what changed over time

**v1 — initial build**
- Two-stage architecture (discover, then profiles), resume support, atomic JSON checkpoints, retry logic, graceful Ctrl+C.
- TXT outputs (contacts, summary, directory, failed, progress, table + TSV), JSON and CSV.
- Phone numbers stored as strings with all raw variants kept; conservative normalisation; conservative contact de-duplication.
- Selectors derived from the supplied HTML dump, plus fallbacks. Added `--offline-test` to parse the dump without a browser, and `--user-data-dir` to keep the login.

**Findings from analysing your dump**
- "840" in the original brief is the company count of that snapshot (20 per page ≈ 42 pages), not 840 pages. The script reads the real page count at run time. (Your live account later showed 89,029 companies / 4,452 pages.)
- The profile in the dump is `07610y1` (Bondex, 4 contacts), not `0m9kw9e`.
- Directory-home cards (`directory-home-card`) belong to `/en/directory/`; the main list uses a different row class. Both are supported.

**v2 — authentication**
- **Bug fix:** the sign-in check used visible text, but the "Sign Out" marker lives in a hidden dropdown, so the script always believed you were signed out. It now checks the DOM markers listed in section 4.
- Added automated login (`JCTRANS_USER` / `JCTRANS_PASS`, `--username`, hidden password prompt), CAPTCHA/verification hand-over, manual fallback, and automatic re-login on session expiry. Added `--manual-login`. Credentials are never saved or logged.

**Live results after v2:** sign-in detected, 20 companies discovered from page 1, 20/20 profiles extracted with 44 contacts and no failures.

---

## 14. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `can't open file '...jctrans_scraper.py'` | The file is saved under another name (e.g. `jct.py`). Use the real filename. |
| `unrecognized arguments` / `--profile-details` error | Python older than 3.9. Upgrade Python. |
| Asks to sign in every run | Add `--user-data-dir chrome_profile` and keep using the same folder. |
| `[AUTH] Login form not found automatically` | Login page layout differs from the generic detection. Sign in manually when prompted (the session is then saved). To get automation fixed, send the log line and a saved copy of the login page HTML. |
| Automated login keeps failing | Check the credentials; a CAPTCHA or code may be required (complete it when prompted); try `--manual-login` once to store the session. |
| Run only did 20 companies | `--max-companies 20` was set. Use `--max-companies 0`. |
| Previous data seems gone | You ran without `--resume`. Look for `state_backup_<timestamp>` and `*.bak_<timestamp>*` files in the output folder. |
| A page-range run also processes older companies | Stage B uses the whole unfinished queue. See section 8. |
| Many `none_visible` contacts | Open `debug_html/<id>.txt`; the `contact_extraction_notes` field says whether the section was absent, empty, or unmatched. Send me a sample file if it looks like a selector problem. |
| `[PAUSE]` message | The site showed a restriction/verification page. Resolve it in the browser (or stop the run), then press ENTER. Slow down with a larger `--delay`. |
| Headless run asks for login | The profile folder holds no valid session. Sign in once with a visible window using the same `--user-data-dir`. |
| Excel shows phone numbers oddly | Open the CSV via Data → From Text/CSV and keep the phone columns as text; the JSON-array format is designed to stay text. |

Logs: `output/jctrans_scraper.log`. Send that file (and any `debug_html` samples) when asking for help.

---

## 15. Limitations

- Only information visible to your signed-in account is collected. Masked values are reported as `restricted` and are not unmasked.
- Mobile / office / WhatsApp / fax fields are filled only when the page explicitly labels them.
- The site can change its HTML; selectors are centralised in the `SELECTORS` block at the top of the script for easy updates.
- Very long runs depend on your session, network and the site's tolerance for load; use batches and keep the delay conservative.
- Establishment year, membership dates and descriptions are copied as displayed. The script does not calculate or correct them (for example a description may state a founding year different from the "Establishment Time" field; both are kept as shown).

---

## 16. Responsible use

Use the tool only with your own authorised account and only for business purposes that comply with JCtrans' terms of service and applicable privacy and anti-spam laws. The script does not bypass login, CAPTCHA, MFA or rate limits, does not submit forms or send messages, and collects only business contact details that are visible to your account. Keep the output files secure, since they contain personal business contact data.
