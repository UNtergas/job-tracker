# Tracker export — instructions for Claude in AWS-prep

Put this file at the root of AWS-prep (or paste it into `CLAUDE.md`). It tells Claude how to emit
Job Tracker import bundles as part of normal work in this repo.

## Contract

- Every bundle must validate against `tracker/import.schema.json` (copy of the tracker repo's
  `docs/import.schema.json`). Validate before saving:
  `npx -y -p ajv-cli -p ajv-formats ajv validate --spec=draft2020 -c ajv-formats -s tracker/import.schema.json -d <bundle>`
- `schemaVersion` is `1`. If the schema changes, the tracker repo bumps it first; never invent fields.
- `source` is always `"aws-prep"`.
- All date-times are ISO 8601 **with offset** (`2026-10-24T09:30:00+02:00`). Date-only fields use `YYYY-MM-DD`
  (`contact.lastContactedAt`, `cvVersion.sentAt`, `letter.sentAt`, `jobPosting.postedAt`, KB periods).
  Compute offsets with `TZ=Europe/Paris date -Iseconds` (or `-d '<local time>'`), never by hand:
  Paris switches +02:00 → +01:00 on the last Sunday of October.
- The tracker stores date-times as absolute instants and displays them in Europe/Paris. The
  original offset is not kept; the moment in time is.
- `sourceCommit` must be a commit that contains the exact `.typ` used to build the PDF: commit
  the source first, then compile, then hash.

## Where bundles go

- Write to `tracker/exports/YYYY-MM-DD-<topic>.json` (one bundle per step, e.g. `2026-10-25-acme-applied.json`).
- `tracker/exports/` is in `.gitignore` of every public repo. Bundles hold real data.
- The user imports manually: `curl -F file=@<bundle> "http://localhost:8080/api/import?dryRun=true"`, then without `dryRun`.

## When to emit a bundle

Emit a small bundle (only what changed) at each of these steps:

| Step in AWS-prep | Objects to include |
|---|---|
| New job posting found / saved | `companies` (if new), `jobPostings` |
| Application sent | `applications`, `events` (`APPLIED`), `cvVersions`, `letters` |
| Outreach to a person | `contacts` (with `outreachStatus`, `lastContactedAt`) |
| Reply / rejection / ghosting decided | `events` (`REPLIED`, `REJECTED`, `GHOSTED`, …) |
| Interview booked or done | `interviews`, `events` (`INTERVIEW_SCHEDULED`) |
| Knowledge base fact written / changed | `kbEntries` (with `sourcePath`) |
| STAR story drafted / changed | `starStories` (+ any new `kbEntries` it cites) |
| A decision taken (accept, prioritise, …) on an application | `events` with `applicationRef`, `type: "DECISION"`, reason in `note` |
| Decided **not** to apply to a posting | `jobPostings` + `events` with `jobPostingRef` (no `applicationRef`), `type: "DECISION"`, reason in `note` |

Bundles may reference objects from earlier bundles by `ref`: the tracker resolves them against
its database.

## Ref rules

- Format: `<prefix>-<kebab-slug>`, lowercase. Prefixes: `co-`, `jp-`, `ct-`, `app-`, `ev-`, `iv-`, `kb-`, `star-`, `ltr-`, `cv-`.
- A ref is **permanent**. Re-emitting the same ref updates the object; never rename a ref, never reuse one for something else.
- Derive refs deterministically from the data, so re-running an export yields the same refs:

  | Entity | Rule | Example |
  |---|---|---|
  | company | `co-<company>` | `co-orus` |
  | job posting | `jp-<company>-<title-slug>` | `jp-orus-fullstack` |
  | application | `app-<company>-<title-slug>` | `app-orus-fullstack` |
  | event | `ev-<app-slug>-<type>-<yyyymmdd>-<hhmm>` | `ev-orus-fullstack-applied-20261021-0930` |
  | posting decision | `ev-jp-<jp-slug>-decision-<yyyymmdd>[-<hhmm>]` | `ev-jp-octime-backend-decision-20261015` |
  | interview | `iv-<app-slug>-<round>` | `iv-orus-fullstack-1` |
  | cv version | `cv-<app-slug>-v<n>` | `cv-orus-fullstack-v1` |
  | letter | `ltr-<app-slug>-v<n>` | `ltr-orus-fullstack-v1` |
  | contact | `ct-<first>-<last>` | `ct-jane-doe` |
  | kb entry | `kb-<topic>` | `kb-cpu-fix` |
  | star story | `star-<topic>` | `star-cpu-incident` |

- `<app-slug>` = the application ref without its `app-` prefix. `<jp-slug>` = posting ref without `jp-`;
  the extra `jp-` in posting-decision refs keeps them from colliding with application events.
- `<type>` = event type lowercased, `_` → `-` (`INTERVIEW_SCHEDULED` → `interview-scheduled`).
- `<yyyymmdd>-<hhmm>` = the event's `at`, in the offset written in `at` (Paris local time).
- **Unknown time:** set `"atPrecision": "DAY"`, `at` = `T00:00:00` Paris local of that date
  (schema enforces it), and the ref drops the time: `ev-<app-slug>-<type>-<yyyymmdd>`.
  Never invent a time. Same rule for interviews (ref unaffected).
- If the exact time becomes known later, update `at` and `atPrecision` but **keep the old ref**:
  refs are ids, not data.
- Same type, same app, same day/minute (rare): append `-2`, `-3`… and record it in `refs.md` so
  the suffix stays stable on re-export.
- `<title-slug>` and `<topic>`: 3 words max (`software-engineering`, not the full posting title).
  Refs are capped at 120 characters.
- Slugs: strip accents (`développeur` → `developpeur`), lowercase, non-alphanumerics → `-`, collapse repeats.
- Homonym contacts: append company, `ct-jane-doe-acme`.
- Keep a `tracker/refs.md` index (ref → what it is) so refs stay consistent across sessions.

## Content rules

- **Never invent facts.** Every STAR story lists `evidenceKbRefs` pointing to KB entries that
  support each claim. If no KB entry supports a claim, write the KB entry first or drop the claim.
- STAR stories are always drafts on import. The user verifies them in the tracker; never claim one is verified.
- Application status is not a field: express it as an event.
- Contacts: professional data only (name, company, LinkedIn URL, role). No phone, no personal
  email, nothing private in `note`.
- Message templates stay in AWS-prep; do not export them.
- Job postings: `description` = full ad text, verbatim, only when you have it. Otherwise use
  `summary` (requirements only). Interview-process notes go to `application.note` or events;
  people named in the ad go to `contacts`. `location` = where the job is; `company.city` = HQ.
- Salary range asked: `salaryAskEur` = low end, `salaryAskMaxEur` = high end.
- `company.kind` is a judgment call: `CORPORATE` = in-house tech of a non-tech company
  (Carrefour, Decathlon Digital), `CONSULTING` = Big4 / IT consulting (EY), `ESN`, `AGENCY`, `STARTUP`, `PRODUCT`.
- Letters: `format: "PDF"` (PDF fields, uploaded separately) or `format: "TEXT"` (`body`, optional
  `sourcePath`, no PDF fields). A form with several text fields → one TEXT letter, each field under
  a `## <field label>` heading.
- Backfill only: `sourceCommit` = last commit touching the `.typ` before the APPLIED date;
  `sentAt` = APPLIED date. From now on: commit, compile, hash.
- PDFs are never embedded. Put `pdfPath` (path in this repo), `pdfSha256` (`sha256sum <file>`)
  and `sourceCommit` (`git rev-parse --short HEAD` of the commit holding the `.typ` source).
  The PDF itself is uploaded separately.
- Withdrawing an application, accepting an offer and deleting anything are the user's actions in
  the tracker UI. You may emit a `WITHDRAWN` / `ACCEPTED` event only when the user explicitly
  says it happened.

## Example

See `tracker/example.bundle.json` (fake data, validates against the schema).
