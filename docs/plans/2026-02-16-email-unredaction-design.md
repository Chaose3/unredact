# Email Unredaction via Known Email Dictionary

## Problem

Many redactions in the Epstein case files are email addresses, typically enclosed in `< >` angle brackets. Some have a visible `@` splitting the redaction into username/domain sub-gaps. The current solver supports name and word matching but has no email-specific mode.

## Solution

Add a known-email dictionary mode to the solver. Given a pixel-width gap and font/size, scan a list of known Epstein-network email addresses and return any that fit the gap within tolerance, using kerning-aware width computation with the surrounding context characters (typically `<` and `>`).

## Data Sources

Build `unredact/data/emails.txt` (one email per line, lowercased, deduplicated, sorted) from:

1. **Hugging Face `notesbymuneeb/epstein-emails`** — 5,082 email threads (16,447 messages) with sender/recipient fields. Extract unique email addresses from `sender` and `recipients` fields. Parquet format, free download.

2. **Epstein Exposed Black Book API** (`epsteinexposed.com/api/v1/persons`) — ~395 email addresses from the seized address book. Free REST API, no auth, 60 req/min rate limit.

3. **rhowardstone `extracted_entities_filtered.json`** — 357 emails from DOJ document entity extraction. Already used by the project for associates data.

4. **Court-verified (hardcoded)** — 8 addresses from primary court documents:
   - `jeevacation@gmail.com` (Epstein, Giuffre v. Maxwell Exhibit J)
   - `jeeproject@gmail.com` (State Dept FOIA)
   - `jeeproject@hotmail.com` (State Dept FOIA)
   - `jeeholidays@gmail.com` (State Dept FOIA)
   - `jeeproject@yahoo.com` (FOIA + Bryant v. Indyke)
   - `jeffreye@mindspring.com` (Bryant v. Indyke)
   - `zorroranch@aol.com` (NM State Land Office)
   - `gmax1@mindspring.com` (Maxwell, court testimony)

### Build Script

`scripts/build_emails.py` — fetches from sources, merges, deduplicates, writes `emails.txt`. Added as `make build-emails` Makefile target.

## Integration

### Backend (`app.py` / `solver.py`)

When solve mode is `"emails"`:
- Load `emails.txt` into memory (lazy, cached like other wordlists)
- For each email, compute kerning-aware width: `font.getlength(left_ctx + email + right_ctx) - left_width - right_width`
- Return matches within tolerance, sorted by width error

This is the same linear-scan approach `dictionary.py` already uses. The left/right context (typically `<` / `>`) is already sent in `SolveRequest`.

### Frontend (`index.html`)

Add `"emails"` option to the solve mode dropdown, alongside `"enumerate"`, `"dictionary"`, and `"both"`.

### No changes needed to:
- Rust solver (not involved in dictionary-style scans)
- Width table construction
- Charset constraints / state machines
- Font detection pipeline
- OCR pipeline

## Key Sources

- smw.ai/epstein-files/epstein-email-accounts — verified Epstein email addresses
- epsteinexposed.com — Black Book API with contact emails
- github.com/notesbymuneeb/epstein-emails — Hugging Face email threads dataset
- github.com/rhowardstone/Epstein-research-data — extracted entities
