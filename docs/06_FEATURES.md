# Every feature, and where it lives

This is the complete capability list for the system, grouped by stage of
the job search. Each row says whether the feature is code in this
repository, an operating rule the agent follows from `CLAUDE.md`, or a
component of the author's full private build that is documented here so
you can add it yourself.

| Status | Meaning |
|---|---|
| **ships** | Code in this repo. Run it as shown. |
| **rule** | Enforced by the agent reading `CLAUDE.md` or `ONBOARDING.md`. No script. |
| **described** | Built and measured in the author's run, not yet ported to the template. The design is in [`05_DESIGN_DECISIONS.md`](05_DESIGN_DECISIONS.md) and below. |

The template deliberately ships the smallest set that works with zero API
keys. Everything marked *described* was added to the author's build after a
specific failure, and each one is a few hundred lines of plain Python over
the same SQLite file, so porting one is a contained job.

---

## 1. Onboarding

| Feature | Status | Where |
|---|---|---|
| One-time interview that turns an existing resume (or a from-scratch conversation) into a bullet library, role headers, skills blocks and a filled profile | ships | `ONBOARDING.md`, `scripts/build_resume.py` |
| Salary, notice period, target roles and preferred locations captured once and reused | ships | `config/profile.json` |
| Verified-facts file: anything the user confirms in conversation becomes source of truth alongside the resume PDF | ships | `candidate_profile/verified_facts.md` |
| Base resume PDF is never overwritten | rule | `CLAUDE.md` |

## 2. Resume engine

| Feature | Status | Where |
|---|---|---|
| Locked one-page HTML template. Font, size, margins and line-height never change; only the content between markers does | ships | `templates/resume_base.html` |
| Bullet library plus one spec per application. Tailoring is selecting, ordering and re-labelling real bullets toward the JD's own words | ships | `scripts/build_resume.py`, `scripts/resume_specs.py` |
| Chrome-based PDF compile with `--fit`: shrinks the type scale until the page count is exactly one, then stops | ships | `scripts/compile_pdf.py` |
| Replaced PDFs are archived, never deleted | ships | `compile_pdf.py` writes to `resumes/_archive/` |
| Builder refuses a spec with fewer than 12 bullets | ships | `build_resume.py` |
| Matching cover-letter template, used only when a posting asks for one | ships | `templates/cover_letter_base.html` |
| Header shows LinkedIn, Portfolio and GitHub as embedded link labels, never raw URLs | ships | `templates/resume_base.html` |
| Tagline under the contact block is a positioning line, never a title claim | rule | `CLAUDE.md` |
| Bullet shape: result first, number second, method last, one line, two max | rule | `CLAUDE.md`, `ONBOARDING.md` |
| Fit-check before tailoring: skip a JD when more than half the must-haves have no grounding, never counting years of experience | rule | `CLAUDE.md` |
| Only the top two roles per company get a tailored resume | rule | `CLAUDE.md` |

## 3. Resume QA gate

`scripts/verify_resume.py` is pass/fail. Not PASS means not finished, and
the builder runs it automatically after every compile.

| Check | Status |
|---|---|
| Exactly one page | ships |
| Usable text layer (an image-only PDF cannot be parsed by an ATS) | ships |
| No em dashes or en dashes | ships |
| No ligature glyphs (`fi`, `fl`, `ffi`) in the text layer, which break literal keyword matching | ships |
| No leaked template or build text, and the page starts with the candidate's name | ships |
| Bullet floor and word floor (thin page) plus a word ceiling warning (cramped page) | ships |
| Required facts present (for example the correct issuer of a certification) | ships |
| Banned facts absent | ships |
| Summary does not open by claiming a title the candidate has never held | ships |
| No `[estimate - verify before sending]` marker left in | ships |
| Filler-phrase warnings | ships |
| `--all --unsent` audits every resume not yet sent, and never touches a sent one | ships |
| Retired-claim list: phrases the candidate has withdrawn fail any unsent resume even though the base PDF still carries them | described |
| **Provenance**: every bullet key carries an anchor phrase that must literally appear in the source files; the builder refuses to compile otherwise and a separate audit walks the whole corpus | described |
| Role-order check: the builder refuses any spec whose roles are not in strict reverse chronological order | described |
| Duplicate-fact check: the builder refuses a spec that selects two bullets stating the same fact | described |
| Sent resumes are frozen: no rebuild for a row in any post-application status | rule (template), enforced in code (described) |

All the rules in this table are stated in `config/resume_rules.json`, which
is the file you edit; the checks read it.

## 4. Scoring

| Feature | Status | Where |
|---|---|---|
| Six-dimension rubric, 0 to 4 each, weighted (core skills .35, role shape .20, domain .15, scope .15, tools .10, qualifications .05), times 25 | rule | `CLAUDE.md` |
| Score recorded on the row with a re-derivable breakdown, strengths and weaknesses | ships | `scripts/score.py set` |
| Literal JD keyword coverage check | ships | `scripts/score.py keywords` |
| Two scores, never blended: `fit` (employer-agnostic) and `odds` (after employer severity). A fit-minus-odds gap of 10+ means apply via referral, not skip | rule | `CLAUDE.md` |
| Thresholds: skip below 50, auto-approve at 70+, human review between. Auto-approve means build the resume, never send | rule | `CLAUDE.md` |
| Years of experience are never scored; location is a pre-filter, never a dimension | rule | `CLAUDE.md` |
| Calibration check: across any batch of 20+, a median in the 40s to 50s is expected; more than a third at 65+ means drift | rule | `CLAUDE.md` |
| Fit-to-odds module: tier multipliers (A x0.85, B x0.92, C x1.00), title-lineage penalty when a tier A/B JD demands years in a title never held, recruiter-email boost, knockout cap | described | `score_adjust.py` in the author's build |
| JD-quality cap: fit capped at 79 / 69 / 55 for a partial / stub / absent JD, uncapped value kept, cap lifts when the real posting is recovered | described | `score_quality.py` |
| Per-dimension columns backfilled from the breakdown string so the rubric is queryable | described | `score_quality_backfill.py` |
| Level gates: senior titles at tier A/B rejected before scoring, after reading the JD's stated years | described | `CLAUDE.md` in the author's build |
| Salary gate on published archetype bands, with an explicit "budget call, not fit" reason | described | `CLAUDE.md` in the author's build |
| Validation gate: a row cannot advance to APPROVED or APPLIED without fit, odds, location, strengths, weaknesses, a breakdown, and at least 200 chars of JD text | described | `db.update_status()` |
| Score-quality judge: a second agent grades a sample of live scores and flags disputes for a human; it never writes a score | described | `eval_judge.py` |
| Ghost-job signals (reposts, watch span, re-sighting count, posting age) accumulate on the row and never touch a score until validated | described | `ghost_jobs.py`, `posting_age.py` |

## 5. Discovery and capture

| Feature | Status | Where |
|---|---|---|
| Public ATS scraper for Greenhouse, Lever and Ashby boards, filtered by title and location keywords. No API key | ships | `scripts/scrape.py`, `config/job_sources.json` |
| Manual capture from a pasted JD or URL | ships | `scripts/track.py add` |
| Recruiter email captured on the row at log time | ships | `track.py add --recruiter-email` |
| Gmail job-alert scanning (Google Jobs, LinkedIn, Naukri and any sender you add), 7-day window, message-level and normalised-URL dedup | described | `gmail_alerts.py` |
| One door in: every source calls the same `add_job`, which applies company exclusions, an evidence gate (JD text or a URL that can yield one) and a per-company cap. Blocked rows are logged as rejected with the reason, never dropped | described | `db.add_job()` |
| Per-company cap: at most 5 live roles, and a 30-day freeze after 5 applications | described | `company_cap.py` |
| Location taxonomy in one module, with a priority-ordered accept list, an auto-reject list, and a remote rule that strips "remote" and judges the geography left | described | `location_rules.py` |
| Discovery gate: work-in-progress caps on the review queue so auto-discovery stops manufacturing supply nobody can drain | described | `discovery_gate.py` |
| NEED_JD clock: a row without a usable JD stays in review for 7 days, then expires with a reason and reopens if the JD arrives | described | `jd_deadline.py` |
| JD recovery: walk rows that have a URL and no JD text, fetch, reject auth walls, store the text itself (never a precis) | described | `refetch_jds.py` |
| ATS fingerprinting: resolve a company to its real board from its careers page and read titles back, instead of guessing tokens | described | `resolve_careers.py`, `verify_openings.py` |
| Email extraction: pull recruiter addresses out of JD text after every discovery pass | described | `extract_emails.py` |
| Off-profile filters: vetoed firms, public-sector consulting, commission field-sales programmes flagged at capture | described | `db.py` |

## 6. Tracking

| Feature | Status | Where |
|---|---|---|
| SQLite tracker, single source of truth on your machine | ships | `scripts/track.py` |
| Status flow `PENDING_REVIEW -> APPROVED | REJECTED -> EMAILED | APPLIED -> INTERVIEWING | OFFER | EMPLOYER_REJECTED` | ships | `track.py` |
| "Rejected" means the employer passed; "Auto Skip" means you passed. The two are never confused | ships | `track.py`, `dashboard_app/` |
| Notes are append-only everywhere, including the dashboard API | ships | `track.py`, `dashboard_app/api/action.js` |
| `applied_on` stamped on entry to APPLIED or EMAILED | ships | `dashboard_app/api/action.js` |
| Transition stamps in code: `applied_on` plus `apply_channel` on APPLIED/EMAILED, `interviewed_on` on INTERVIEWING/OFFER | described | `db.update_status()` |
| Guarded bulk update: dry-run by default, prints the matched rows grouped by kind, refuses to write without `--apply` | described | `bulk_update.py` |
| Consistency check: orphaned resume files, rows pointing at missing files, duplicate rows, status and score contradictions | described | `consistency_check.py` |
| Backfill audit: classifies every live row by what it would take to pass the validation gate, without inventing a field | described | `backfill_audit.py` |
| Pending actions: one row per obligation (assessment, take-home, slot to book, form to submit) with a calendar-day deadline quoted from the employer's message | described | `actions.py`, dashboard To Do tab |
| Referral match: cross-references a LinkedIn connections export against live rows | described | `referral_match.py` |
| Base-versus-tailored A/B test, restricted to high-fit small-company roles | described | `ab_assign.py` |
| Backups before any destructive work, kept to the last few | rule | `CLAUDE.md` |

## 7. Outreach

| Feature | Status | Where |
|---|---|---|
| **The agent drafts, the user sends.** Gmail draft creation only; the kit has no send path | ships | `scripts/gmail_auth.py` |
| Resume attached to the draft, optional Bcc | ships | `gmail_auth.py` |
| Telegram digest of the review queue, sent on demand or on a schedule | ships | `scripts/telegram_setup.py` |
| Outreach style rules: fixed greeting and sign-off, three paragraphs, no dashes, no apostrophes, subject format that the employer's stated format outranks | described | `config/profile.json -> outreach.style_rules` |
| Send function that raises: `send_email()` exists and refuses, so a future caller cannot send by accident | described | `gmail_send.py` |
| Always-on Telegram listener with inline Approve and Reject buttons, typed commands (DIGEST, STATUS, DASHBOARD), URL capture from any message, and link-less JD pastes turned into rows | described | `telegram_bot.py` |
| Every Telegram update archived raw to a local file before any handler runs, so nothing the user sends the bot can be lost | described | `telegram_bot.py` |
| No form autofill, ever. The agent summarises what a form needs; the user fills it | rule | `CLAUDE.md` |
| Never volunteer a gap in an email, resume or cover letter | rule | `CLAUDE.md` |
| A writing-style skill so anything in the user's first person sounds like them | described | `.claude/skills/` in the author's build |

## 8. Outcomes and measurement

| Feature | Status | Where |
|---|---|---|
| Outcome tracker: reads Gmail for employer replies, writes first-response date, outcome and interview date, with tier-specific wait windows. Never replies to a thread | described | `track_outcomes.py` |
| Outcome guardrail: no EMPLOYER_REJECTED, INTERVIEWING or OFFER without matching company, role and a possible timeline; a "names a different role" flag is a stop sign | rule | `CLAUDE.md`, incident 7 in `05_DESIGN_DECISIONS.md` |
| Scoreboard with one North Star: interviews per 10 applications sent | described | `metrics.py` |
| Score validation: does a higher odds band actually reply more? Refuses to conclude under 25 resolved applications | described | `metrics.py --validate` |
| "Reached interview" counted from dated milestone columns, not current status, so an interview that ended in rejection still counts | described | `metrics.reached_interview()` |
| Interview-prep report: company research, resume walkthrough, STAR stories, likely questions, every fact tagged VERIFIED / CARRIED OVER / COULD NOT VERIFY, compiled to PDF | ships as a guide | `docs/04_INTERVIEW_PREP_GUIDE.md` |

## 9. Dashboard

| Feature | Status | Where |
|---|---|---|
| Hosted approval dashboard on Supabase and Vercel, passcode-gated, light and dark theme | ships | `dashboard_app/` |
| Review tab (Approve / Pass), To Apply tab (Mark applied), Pipeline tab (status table) | ships | `dashboard_app/index.html` |
| Fit and odds shown side by side with strengths and weaknesses per card | ships | `dashboard_app/index.html` |
| Pull and push between local SQLite and Supabase, newest `updated_at` wins | ships | `scripts/sync_supabase.py` |
| Schema file to run once in Supabase | ships | `dashboard_app/schema.sql` |
| Self-contained demo dashboard rendered from a seeded fictional database, for screenshots with zero network calls | ships | `demo/` |
| To Do tab backed by the pending-actions table | described | author's build |
| Rate tile showing interviews reached, using the same function as the scoreboard | described | author's build |
| Three-way sync discipline: pull first, push last, purge rows that leave the live set, paginate past the 1000-row PostgREST cap | described | `supabase_sync.py`, `supabase_push.py` |
| Schema tool to add a column hosted-side so a new local column is not silently dropped on push | described | `supabase_schema.py` |
| One-command deploy | described | `deploy_dashboard.py` |

## 10. Operating the agent

| Feature | Status | Where |
|---|---|---|
| Operating rules the agent loads every session, with the incident behind each guardrail | ships | `CLAUDE.md`, `docs/05_DESIGN_DECISIONS.md` |
| Guardrails: never generalise from a spot check, preview before bulk mutation, verify identity before building, back up before destructive work, report misses plainly | rule | `CLAUDE.md` |
| Two scheduled tasks: a morning pipeline that owns all mutation (discovery, JD recovery, scoring) and ends at the review stage, and an evening audit that verifies only and ends with a chat summary | described | Claude Code scheduled tasks in the author's build |
| Listener kept alive by the OS scheduler and proven alive by its own heartbeat file, which is trusted over any process probe | described | `ensure_listener.py`, `run_listener.cmd` |
| Health check that answers "is anything silently broken" and can notify by Telegram | described | `healthcheck.py` |
| Per-run tracing: one line per pipeline run with a phase breakdown, so "it felt slow" becomes "the JD fetch hung" | described | `trace.py` |
| Large scoring batches delegated to subagents that return text; SQLite is never written concurrently | rule | `CLAUDE.md` in the author's build |
| Model policy: the stronger model owns rules, templates and thresholds; the faster model does per-JD tailoring against the locked template and the pass/fail verifier, and escalates on anything the template does not fit | described | `CLAUDE.md` in the author's build |

---

## Porting a *described* feature

Every one of them reads and writes the same `jobs` table that `track.py`
creates, through one helper module. The order that worked for the author:

1. `db.update_status()` with the validation gate and append-only notes. Everything else assumes it.
2. `score_adjust.py` and `score_quality.py`, so fit and odds stop being the same number.
3. `location_rules.py` as the single copy of the location filter.
4. `track_outcomes.py` and `metrics.py`, so the system can be measured before more discovery is added.
5. Only then discovery sources, each behind `discovery_gate.py`.

If you port one, add the check to code first and the sentence to `CLAUDE.md`
second. The project's own evidence is that the sentence alone does not hold.
