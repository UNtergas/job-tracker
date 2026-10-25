# Job Tracker — Decisions & Roadmap

> Outcome of the grilling sessions of 2026-10-22 → 2026-10-24.
> Each entry: **decision**, alternatives rejected, **reason**. Change a decision only by adding a new
> dated entry that supersedes it; never edit history.
> Companion files: `docs/import.schema.json` (import contract), `docs/TRACKER_EXPORT.md` (AWS-prep side).

---

## 0. What this project is

A private, single-user tracker for my job search. It **stores and enforces**; it does not think.
Claude (Desktop + Code) is the intelligent client, talking to it through MCP. AWS-prep stays my
knowledge/drafting repo and exports data into the tracker as JSON bundles.

```
AWS-prep (git, drafts, .typ CVs)  ──JSON bundles──►  Tracker (Spring Boot + Postgres)  ◄──MCP──  Claude Desktop / Code
                                                         ▲
                                                         └── Thymeleaf forms (me, CRUD + human-only actions)
```

Goals, in order:
1. Credible `Java / Spring Boot / JPA / PostgreSQL` line on my CV, backed by a public repo and blog posts.
2. A tool I use every day.
3. Understand what Spring hides: terminal first, docs first, debugger.

---

## 1. Rules for building (carried into every wise-code session)

1. **Applications first.** 10 applications + referral messages per week happen before coding time.
2. **Time cap:** weekdays 1 h, weekends ~8 h total.
3. **No AI codegen for the app.** Docs → debugger → Stack Overflow. Stuck > 45 min → ask for a *hint*, never a fix.
4. **wise-code questions me, it does not write my code.** Contract/spec files (`import.schema.json`) are the only exception.
5. **Terminal only** for running and building (`./mvnw`). The editor reads, navigates, lints, attaches the debugger.
6. **No stack switching** until P4 is done.
7. **Real data never in git.** The repo holds fake seed data only. Exports and dumps live outside the repo or in ignored paths.
8. **Done = used daily**, not perfect architecture.
9. Write a concept note in `docs/NOTES.md` (by hand) for each new Spring concept met.

---

## 2. Decisions

### D1 — Language: Java, not C#/.NET (2026-10-24)
- **Decision:** Java / Spring Boot.
- **Rejected:** ASP.NET Core (first plan, 2026-10-22).
- **Reason:** Java is the largest junior pool in Paris (Indeed, Oct 2026: Java ~180 vs C# ~161). I already shipped Spring Boot 3.0 code on Coffwok (2023), so this builds on real experience instead of starting cold. Locked: no further switch.

### D2 — Versions: Java 25 LTS + Spring Boot 4.1.x (2026-10-24)
- **Decision:** Java 25 (SDKMAN, Temurin), Spring Boot latest 4.1.x patch from start.spring.io on day 1, Spring AI 2.x for the MCP server.
- **Rejected:** Boot 3.5 ("more tutorials"), Boot 4.0, Java 21.
- **Reason:** Boot 3.5 / Framework 6.2 / Spring Security 6.5 lost OSS support on 2026-06-30; Boot 4.0 loses it 2026-12-31. Spring AI 1.x is EOL with 3.5 and Spring AI 2.0 requires Boot 4. Starting on a dead line means migrating twice. Books written for Boot 3 stay valid for concepts (IoC, JPA, transactions); every 3 → 4 difference I hit goes into `NOTES.md` (interview + blog material).
- **Check on day 1:** Spring AI 2.x version compatible with the chosen Boot 4.1.x.

### D3 — Build & web layer (2026-10-24)
- **Decision:** Maven + wrapper. Spring Web MVC with `@RestController` + Bean Validation, `@ControllerAdvice` for errors. springdoc-openapi for the API docs (tailnet only).
- **Reason:** Maven and controllers are what French ESN codebases use; controllers map to my TSOA background.

### D4 — Persistence (2026-10-24)
- **Decision:** Spring Data JPA (Hibernate) + PostgreSQL + Flyway. `spring.jpa.hibernate.ddl-auto=validate`; schema changes only through Flyway migrations.
- **Reason:** Exactly the gap Coffwok (MongoDB) left. Flyway migrations are also what makes local → VPS migration a plain `pg_dump`/`pg_restore`.

### D5 — Code structure (2026-10-24)
- **Decision:** packages by feature (`application/`, `contact/`, `kb/`, `story/`, `importer/`, `mcp/`…). Plain `@Service` classes; an interface only when there is a second implementation or a real test seam.
- **DTOs:** Java records. Manual mapping (`from()` / `toEntity()`); no MapStruct.
- **Entities:** plain classes, getters/setters/`equals`/`hashCode` written by hand. **No Lombok** until P3 is done, so I learn what it hides (JPA identity, *Effective Java* items 10–11).
- **Reason:** 10 entities do not justify ceremony; interviews will ask why, and "one implementation, no interface" is a defensible answer.

### D6 — UI: Thymeleaf, unstyled (2026-10-24)
- **Decision:** server-rendered Thymeleaf with plain HTML forms, no CSS framework. React only after P4.
- **Rejected:** React + TanStack + shadcn + OpenAPI codegen in the MVP; Swagger UI as the daily UI.
- **Reason:** React is what I already know, so it costs hours without teaching Java. Thymeleaf teaches Spring MVC and is common in ESN legacy code. The UI exists for CRUD without Postman and for human-only actions (D9).

### D7 — No LLM inside the tracker (2026-10-24)
- **Decision:** the tracker holds logic and data only. Generation, critique and fabrication hints come from Claude through MCP, using my subscription.
- **Rejected:** Laya (classifier; base checkpoints score below majority-class baseline without fine-tuning, ~421M params, Python service); AwanLLM (2024-era models, anonymous operator, no GDPR/DPA terms: would ship my KB and third parties' data to an unknown provider); local Ollama (deferred indefinitely).
- **Reason:** no extra stack, no RAM pressure on the VPS, no private data leaving my machines through a third API.

### D8 — MCP (2026-10-24)
- **Decision:** MCP server inside the Spring app (Spring AI 2.x, HTTP transport). Clients: Claude Code and Claude Desktop on my laptop only (no phone). Claude Code registers it with **user scope**, so it works from AWS-prep and any repo.
- **Tools (P2):** `get_today`, `list_applications(status?)`, `get_application(id)` (with events, contacts, interviews), `add_application`, `add_event(applicationId, type, at, atPrecision, note)`, `upsert_contact`, `upsert_job_posting`.
- **Tools (P3):** `upsert_kb_entry`, `save_star_story`, `link_evidence`, `save_letter_meta`, `save_cv_version_meta`.
- **Never:** `delete_*` tools, or any tool that performs a human-only transition (D9).
- **Bulk data does not go through MCP:** JSON bundles and PDFs go through HTTP endpoints (D12, D13), because pushing them through tool arguments burns Claude context.
- **Open (R2, before P4):** can Claude Desktop reach an HTTP MCP server inside the tailnet (directly, or via a local stdio bridge)? Claude Code can.

### D9 — Human-only actions (2026-10-24)
- **Decision:** only I, in the Thymeleaf UI, can: delete anything; move a `StarStory` from `DRAFT` to `VERIFIED`; mark an application `WITHDRAWN` or `ACCEPTED`. Claude creates, updates drafts, adds events.
- **Reason:** irreversible or identity-defining states stay human. Same principle everywhere: Claude proposes, I commit.
- **Enforcement:** in the service layer (not just "the MCP tool doesn't exist"), with a test per rule.

### D10 — Data model (2026-10-24)
| Entity | Notes |
|---|---|
| `Company` | name, website, kind, city |
| `JobPosting` | company, title, url, full JD text (copied, postings vanish), contract, remote, salary band, stack, postedAt |
| `Application` | job posting, channel, salary ask, note; **M:N `Contact`**; `currentStatus` denormalized |
| `ApplicationEvent` | append-only timeline: type, `at` (Instant), `atPrecision` (`DAY`/`MINUTE`), note. `DECISION` type records decisions and their reason |
| `Contact` | name, company, LinkedIn URL, role, outreachStatus, lastContactedAt (date), short professional note. **No phone, no personal email** |
| `Interview` | application, round, kind, `at` + `atPrecision`, outcome, notes; M:N contacts, M:N star stories used |
| `KbEntry` | facts about my own experience: kind, title, body, period, tags, sourcePath (AWS-prep) |
| `StarStory` | S/T/A/R text, status `DRAFT`/`VERIFIED`; **M:N `KbEntry` evidence, at least 1**; M:N applications (reusable) |
| `Letter` | application, PDF (`bytea`), SHA-256, source commit, sentAt (date). Append-only |
| `CvVersion` | application, PDF (`bytea`), SHA-256, source commit, sentAt (date). Append-only |

- **Status:** `ApplicationEvent` is the truth; `Application.currentStatus` is updated in the same `@Transactional` as the event insert, for cheap list queries. Funnel stats and ghost detection read events.
- **Import ids:** every imported entity has `external_ref varchar(120) unique`; the DB keeps its own PK.
- **Out of the tracker:** message templates, `.typ` sources, drafts (all stay in AWS-prep).
- **Typst:** compiled in AWS-prep. The tracker stores only the PDF actually sent, plus the git commit of its source, so "what did I send" is exact and "what changed" is `git diff` between commits.

### D11 — Time handling (2026-10-24)
- **Decision:** date-times stored as `Instant`, displayed in `Europe/Paris`. `atPrecision = DAY` means only the date is known: stored as 00:00 Paris local, shown as a date. Pure dates (`lastContactedAt`, `sentAt`, `postedAt`) are `LocalDate`.
- **Reason:** most of my history is date-only; inventing times would be fake data.
- **Gotcha to test:** day-based rules (stale > 10 days) compare **dates in Europe/Paris**, not raw instants, or DAY events shift by up to a day, and DST changes (last Sunday of March/October) break naive arithmetic.

### D12 — Import from AWS-prep (2026-10-24)
- **Decision:** contract = `docs/import.schema.json` (JSON Schema 2020-12, `schemaVersion: 1`). Endpoint `POST /api/import?dryRun=true|false`, JSON body.
  - Idempotent **upsert by `ref`**; refs resolve against the same bundle or the DB.
  - **One transaction per bundle:** one invalid object → nothing imported.
  - `dryRun=true` returns counts (created / updated / unchanged per entity) and every validation error, writes nothing.
  - Unknown `schemaVersion` → rejected.
  - Bean Validation on import DTOs too (never trust input, even my own).
  - Imported `StarStory` is always `DRAFT` (D9).
- **PDFs:** separate multipart upload (`POST /api/applications/{ref}/cv-versions`, `/letters`), matched by SHA-256 to the metadata in the bundle.
- **Ref rules and examples:** `docs/TRACKER_EXPORT.md`.
- **Reason:** I will run the import many times while fixing the mapping; idempotency and dry-run make that safe. The schema lets Claude in AWS-prep validate before I ever import.

### D13 — Source of truth (2026-10-24)
- **Decision:** pipeline data (companies, postings, applications, events, contacts, interviews, KB, stories, sent PDFs) is owned by the tracker DB once imported. AWS-prep keeps knowledge drafts and emits a small bundle at each step (`TRACKER_EXPORT.md`).
- **Local → VPS (P4):** `pg_dump` / `pg_restore` of the local DB, **not** re-importing JSON (re-import would lose everything added after the import).
- **JSON reuse:** test fixtures and the V2 guest seed.

### D14 — No scheduler in V1 (2026-10-24)
- **Decision:** no `@Scheduled`. The digest is pull-based: `/today` page + `get_today` MCP tool. Ghost/stale applications are derived at query time from events.
- **Rejected:** `@Scheduled` digest presented as the "no overlapping runs" story.
- **Reason:** a single-instance `@Scheduled` task never overlaps by construction, so it would prove nothing. The real CPU-incident story comes from Snowpack. Overlap protection (ShedLock / Postgres advisory lock) only when a second instance exists. Mail digest: later.

### D15 — Hosting & network (2026-10-24)
- **Decision:** one OVH VPS (shared with future projects) + one domain.
  - **Public:** `blog.<domain>` (Hugo, static) via Caddy on 80/443. Nothing else public.
  - **Private:** tracker reachable only through Tailscale (`tailscale serve` → `127.0.0.1:8080`). No public DNS record for it.
  - **SSH:** tailnet only; password auth inside the tailnet (my choice; keys remain the better option). OVH KVM console = emergency access. No fail2ban (nothing public to brute-force).
- **Rejected:** Render free + Neon (sleeping instance, slow JVM cold start on ~0.1 CPU, MCP timeouts).
- **Non-negotiables:**
  - Docker ports bound to loopback (`127.0.0.1:8080:8080`); Postgres without `ports:`. Docker bypasses `ufw`.
  - `server.address=127.0.0.1`.
  - Actuator: only `health` exposed (`management.endpoints.web.exposure.include=health`).
  - MFA on the Tailscale identity provider; Tailscale ACLs limit the VPS to my devices.
  - Close public port 22 only after tailnet SSH is confirmed in a second session.
  - After every Compose change, from outside the tailnet: `nmap -Pn -p 22,5432,8080 <public-ip>` → all filtered/closed.
- **No login in V1:** Tailscale is the authentication. Spring Security arrives in V2 with the demo.

### D16 — Backups, by phase (2026-10-24)
- **RPO:** 1 day. **RTO:** hours.
- P1–P3 (laptop): `scripts/backup.sh` → `pg_dump -Fc` to a folder outside the repo, run **before every Flyway migration** and weekly. One restore test into a throwaway container in P2.
- P4 (VPS): nightly `pg_dump` on the VPS (keep 7) + laptop **pulls** over the tailnet (keep 7 daily + 4 weekly). Pull, so a compromised VPS cannot delete the laptop copies. Restore test at deploy, then monthly.
- P5: encrypted off-site copy (restic) at a provider other than OVH → 3-2-1 complete.
- OVH snapshots are not backups (same provider, same blast radius).

### D17 — Repos, blog, demo (2026-10-24)
- **Tracker repo:** public. Fake seed data only (`docs/example.bundle.json` style), loaded in the `dev` profile. Never in prod.
- **Blog:** Hugo, separate repo, public. I write online and locally; wise-code critiques posts (local draft or published URL).
- **Demo:** V2 only — login + guest mode on a separate instance with a separate seed-only DB, behind `demo.<domain>`. Never the real DB.

### D18 — Testing (2026-10-24)
- JUnit 5; `@WebMvcTest` for controllers, `@DataJpaTest` + Testcontainers Postgres (`@ServiceConnection`) for repositories, `@SpringBootTest` + Testcontainers for the importer and human-only rules.
- Must-have tests: importer idempotency (import twice → same state), all-or-nothing on a bad bundle, D9 rules, D11 date logic around DST.
- No frontend tests.

### D19 — Deploy (2026-10-24)
- P4: manual — SSH over the tailnet, `git pull`, `docker compose up -d --build`. CI = `./mvnw verify` on GitHub Actions only.
- P5: GitHub Actions builds the image → GHCR → joins the tailnet (ephemeral tagged key, ACL) → redeploy script.

### D20 — Import contract amendments after backfill (2026-10-24, amends D10/D12)
- **Letters:** `format = PDF | TEXT`. 5 of 9 letters were pasted into forms, never PDFs. `Letter` gets `format`, nullable `pdf`/`pdf_sha256`, nullable `body`; CHECK constraint: exactly the fields of its format. Multi-field forms → one TEXT body with `## <field>` headings.
- **Decisions on postings not applied to:** an event belongs to an application **or** a job posting (exactly one; posting-level events are `DECISION` only). DB: `application_event` gets nullable `application_id` + nullable `job_posting_id` + CHECK exactly one. Gives a real top of funnel (seen → skipped / applied).
- **JobPosting:** `description` = full ad text when available; new `summary` (my summary) and `location` (job city; `company.city` = HQ).
- **Company kind:** add `CONSULTING`, `CORPORATE`. **Contact role:** add `FOUNDER`.
- **Salary range:** `salaryAskEur` (low) + `salaryAskMaxEur` (high).
- **Backfill `sourceCommit`:** last commit touching the `.typ` up to 23:59:59 Paris on the APPLIED day; none → field left out (no false link; `pdfSha256` still identifies the file).
- Still `schemaVersion: 1`: nothing imported yet. Next change after the importer exists → bump.

---

## 3. Superseded (kept for the interview story)

| Was | Now | Why |
|---|---|---|
| ASP.NET Core | Spring Boot (D1) | Larger junior market, prior Spring experience |
| Boot 3.x first, "map to 4 later" | Boot 4.1 (D2) | 3.x OSS EOL since 2026-06-30 |
| Render free + Neon | OVH VPS + Tailscale (D15) | Sleep kills jobs/MCP, slow JVM boot |
| React MVP | Thymeleaf (D6) | Time goes to Java, not to what I already know |
| `@Scheduled` "no overlap" story | Pull digest (D14) | Single instance can't overlap; story would be fake |
| Laya / AwanLLM / internal LLM | Claude via MCP only (D7) | Accuracy, weight, privacy |
| Typst stored in tracker | PDF + commit SHA (D10) | Tracker records what was sent; git records how it changed |
| `-0000` / invented times | `atPrecision` (D11) | No fake data |
| fail2ban + public SSH | Tailnet-only SSH (D15) | Nothing public to attack |

---

## 4. Roadmap

Each phase ends with an **exit check**. Don't start the next phase before it passes.

### P0 — Contract (done 2026-10-24)
- [x] Import contract `docs/import.schema.json`, AWS-prep instructions `docs/TRACKER_EXPORT.md`, fake example bundle.
- [x] AWS-prep adopted it; backfill of ~13 applications + 3 contacts ready.
- [ ] Ask AWS-prep Claude which source fields it had to drop or squeeze into `note` (schema coverage check).

### P1 — Spring core, local (laptop)
Learn: IoC/beans, auto-configuration, JPA entity states, transactions, Flyway.
- [ ] SDKMAN Java 25; generate the project with `curl start.spring.io` (no IDE); read `pom.xml` line by line.
- [ ] `docker-compose.yml`: Postgres only, port bound to `127.0.0.1`.
- [ ] "What Spring hides" exercises: `--debug` conditions report, Actuator `beans`/`conditions`/`mappings` (local only), SQL logging, fat-jar inspection.
- [ ] Flyway V1: `company`, `job_posting`, `application`, `application_event`, `contact`, `application_contact`, `interview`.
- [ ] Entities + repositories + services by feature; `currentStatus` updated with event insert.
- [ ] Importer (D12) for P1 entities: upsert by ref, one transaction, dry-run, Bean Validation.
- [ ] `/today`: follow-ups due, upcoming interviews, stale (> 10 days without event, Paris dates).
- [ ] Thymeleaf: list + create/edit forms for applications, contacts; add-event form; `/today` page.
- [ ] `scripts/backup.sh`.
- [ ] First blog post (Hugo, local is fine): "What Spring Boot 4 hides, made visible."
- **Exit:** AWS-prep backfill imports with `dryRun`, then for real; importing it twice changes nothing; `/today` shows my real pipeline.

### P2 — MCP, daily use starts (laptop)
Learn: Spring AI MCP server, tool design, debugger on a live request.
- [ ] MCP server with the P2 tools (D8) on `localhost`.
- [ ] Register in Claude Code (user scope) and Claude Desktop.
- [ ] Debugger: attach on 5005, breakpoint inside an MCP tool call.
- [ ] Testcontainers integration tests: importer idempotency, all-or-nothing.
- [ ] Restore test of a `pg_dump` into a throwaway container.
- **Exit:** one full week where every application/reply is recorded through Claude or the UI, not by hand in AWS-prep.

### P3 — Knowledge, stories, documents (laptop)
Learn: M:N mappings, `bytea`/multipart, service-level rules, N+1.
- [ ] Flyway: `kb_entry`, `star_story`, `star_story_evidence`, `star_story_application`, `interview_story`, `cv_version`, `letter`.
- [ ] Import support for these + multipart PDF upload with SHA-256 check.
- [ ] D9 rules in services + tests (verify story, withdraw/accept, deletes only via UI).
- [ ] P3 MCP tools.
- [ ] Thymeleaf: story next to its evidence, verify button.
- [ ] Hunt N+1 in the SQL logs (Vlad Mihalcea's articles).
- **Exit:** Claude drafts a STAR story from KB entries via MCP, I verify it in the UI; an application shows the exact CV PDF sent.

### P4 — VPS (OVH)
Learn: Linux hardening, Docker networking, Caddy, Tailscale.
- [ ] Buy domain; DNS `A`/`AAAA` for the blog only.
- [ ] Hardening: non-root sudo user, SSH config, unattended upgrades, ufw (80/443 public).
- [ ] Tailscale + MFA + ACLs; SSH over tailnet confirmed; close public 22.
- [ ] Caddy + Hugo blog live.
- [ ] Multi-stage Dockerfile (Maven build → slim JRE 25); Compose with loopback bindings; `tailscale serve`.
- [ ] `pg_dump` laptop → `pg_restore` VPS; laptop becomes a client.
- [ ] Nightly dump + laptop pull + restore test.
- [ ] R2 research; point Claude Code/Desktop MCP at the tailnet URL.
- [ ] `nmap` check from outside the tailnet.
- [ ] CV line: `Java / Spring Boot / JPA / PostgreSQL — projet personnel (repo + blog)`.
- **Exit:** tracker used daily from the VPS; external `nmap` shows only 80/443.

### P5 — Ops polish
- [ ] restic encrypted off-site backups (3-2-1).
- [ ] CI deploy over the tailnet (D19).
- [ ] Mail digest (`@Scheduled`, then ShedLock if a second instance appears).
- [ ] Typst compile in-app (optional, Java process management practice).

### V2
- [ ] Spring Security (session login), demo instance with guest seed (`demo.<domain>`, separate DB).
- [ ] React frontend (springdoc → typed client).

---

## 5. Open items

| Id | Question | When |
|---|---|---|
| R2 | Claude Desktop → HTTP MCP inside the tailnet: direct, or local bridge? | Before P4 |
| R3 | Exact Spring AI 2.x version for Boot 4.1.x; MCP server starter name | P1 day 1 |
| C1 | Fields AWS-prep cannot fit in the schema | Before P1 importer |
| — | Java 21 → 25 language changes worth knowing (notes) | During P1 |

---

## 6. Starting a wise-code session

Paste into the first message of a Claude Code session in this repo:

> Run `/wise-code:learn`. Context: `docs/DECISIONS.md` (decisions + roadmap), `docs/import.schema.json`,
> `docs/TRACKER_EXPORT.md`. We are at **P1, first unchecked item**. Follow §1 rules: don't write
> application code for me; question me, point me to the official docs section, review what I write.
> When a decision in this file looks wrong, challenge it, and record any change as a new dated entry.

Suggested repo layout:
```
job-tracker/
├── api/                    # Spring Boot (Maven)
├── docs/
│   ├── DECISIONS.md        # this file
│   ├── NOTES.md            # concept notes, Boot 3 ↔ 4 diffs, gotchas
│   ├── import.schema.json
│   ├── TRACKER_EXPORT.md
│   └── example.bundle.json # fake data
├── scripts/backup.sh
├── docker-compose.yml
└── README.md               # written at P4 from this file
```
