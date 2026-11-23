# Sahara: Survivor Pension Claims (Prototype)

> **Hackathon project.** Built by Kissa Zehra together with my teammate [Aiza Gazyani](https://github.com/Aiza166) at the Coder's Cup '26 AI Hackathon (ACM, FAST-NUCES Karachi, 20-22 Nov 2026). This is my copy with the full commit history; the original repository is [Aiza166/AI-Hackathon](https://github.com/Aiza166/AI-Hackathon).

> **PROTOTYPE ONLY. Not an official government service.**
> Every identity, family record, death record, pension and payment in this project is **simulated (mock)**. No real personal data is used anywhere.

Entry for **Coder's Cup 2026 AI Hackathon: "The Last Mile: AI on Public Rails"** (ACM, FAST-NUCES Karachi). Team: Aiza and Kissa.

[![CI](https://github.com/Aiza166/AI-Hackathon/actions/workflows/ci.yml/badge.svg)](https://github.com/Aiza166/AI-Hackathon/actions/workflows/ci.yml) Coverage: [Codecov](https://app.codecov.io/gh/Aiza166/AI-Hackathon) (backend about 94% of lines). Errors: Sentry, with personal data scrubbed ([docs/INFRA.md](docs/INFRA.md#monitoring)).

## The problem in one sentence

**A widow in Pakistan has to claim her late husband's pension even though she can't read the forms, her phone signal is weak, her records spell his name three different ways, and scammers and false heirs go after the money.**

## Try it (2 minutes)

- App: https://confident-growth-production-1cfebeef.up.railway.app (works best at phone width)
- API docs: https://ai-hackathon-production-31dd3038.up.railway.app/docs
- Demo video (about 5 minutes, English voice-over and captions, mock data only): https://drive.google.com/file/d/1lA4QQ0hvbmJRb_Ys79lxVwjFmOwDdixQ/view?usp=sharing

On the login screen tap **Demo accounts (mock)**. Every demo PIN is `1234` (public on purpose; all data is mock).

| Who | Phone | What to try |
|---|---|---|
| Zainab (widow) | 00000000001 | Start a new pension claim, "Fill demo data", send. Then "Is this message a scam?" and "Change payout account" |
| Rukhsana (second claimant) | 00000000002 | Claim the same pension: the app does not reject her, it sends both claims to an officer |
| Ayesha (community helper) | 00000000010 | 40 families on one phone; one has taken back her access and shows locked |
| Kamran (officer) | 00000000020 | Review queue, decision trace, Evaluation tab |
| Admin | 00000000030 | Judge console: break the mock government services, simulate AI outages, reset the demo |
| Auditor | 00000000040 | Audit log and "verify the chain" |

The 7-minute demo story is in [docs/05-DEMO-SCRIPT.md](docs/05-DEMO-SCRIPT.md). Before trying things, an admin can tap **Reset demo data** in the Judge console.

## What it does

1. **Claim by voice and pictures.** One question per screen, Urdu with a Listen button on every step (Sindhi and Punjabi text too, machine-assisted and labelled as such). Each step is saved on the phone; a claim sent with no signal waits and is sent exactly once when the signal returns.
2. **Decide, explain, or say "I am not sure".** The claim is checked against mock identity, family, death, pension and bank records. The result is one of three: approved, "one more paper is needed" (named exactly), or "an officer will look at your claim" with reason codes. **The system never rejects anyone by itself.** Only an officer can, with a written reason.
3. **Explain in plain Urdu.** Every reason is a fixed sentence (Urdu + English + audio + icon), never generated text. The officer and auditor see the full trace: rule version, model id, probabilities, every check.
4. **SafeSwitch** protects the money. A change of payout account is checked against the bank's account-holder name (mock Raast lookup) and held for an officer when it doesn't match. Her own PIN can't release a mismatched change.
5. **Scam Shield** checks a suspicious SMS, WhatsApp message, link, screenshot or voice note and answers "likely scam", "not sure" or "looks genuine", with reasons and where to report it (NCCIA 1799, PTA). It cross-checks the message against her real case, and a pension never charges a fee.
6. **Document checks.** Uploaded papers are read (OCR), compared with the registry, and checked for signs of editing (copy-move, error-level analysis). A doubt sends the case to an officer as "needs a second look", never "fake".
7. **Her data, her say.** She sees who can see her data and who looked at it, and can take back a helper's access. That blocks access at once and is logged.
8. **Tamper-evident audit.** Every action by a person, the AI or the system is written to an append-only, SHA-256 hash-chained log that anyone with the auditor role can verify.

## Where to see each challenge in the brief

| Brief challenge | Where to see it (demo logins above, PIN 1234) |
|---|---|
| 01 Cost of being wrong | Officer or Admin, **Evaluation** tab: cost curve with the 0.95 line, wrong approvals, human-review rate |
| 02 "I am not sure" | Rukhsana's claim goes to Kamran's **Review queue** with reason codes. The system never rejects by itself |
| 03 Bad data | Zainab's husband is spelled three ways and still matches on his CNIC. A blank or wrong date of birth asks for proof, it never rejects |
| 04 Rules that change | Admin, **Rules**, v2 draft, **Run Digital Twin**, then Publish. Every decision shows its rule version |
| 05 Explain and audit | Citizen: reason in Urdu plus Listen. Officer: decision trace. Auditor: **verify the chain** |
| 06 Consent and take-back | Zainab, **My data**, take back Ayesha's access: her next request is refused and logged |
| 07 Works for everyone | Listen on every step, icons, one question per screen, offline outbox (try airplane mode), Sindhi and Punjabi |
| 08 Scams and fake documents | **Is this message a scam?** (Judge console has sample messages) and the edited death certificate (Judge console, sample documents) |

## How the AI is used

- **Jev (`jev-1.14.0`, pinned) is a sensor, code decides.** Jev answers small factual yes/no questions (for example "does this message ask for a fee?"). Rules and thresholds in code turn the answers into a route. Above 0.7 counts as yes, below 0.3 as no, anything between as "not sure", which goes to a person.
- **Two-model cascade.** Deterministic checks run first. Jev runs on everything that's left (fast, cheap). Only when Jev is unsure does a second model (Azure OpenAI `gpt-5-mini`) look. On documents the second model can clear a doubt but can never add a flag by itself.
- **Why auto-approve at 0.95.** A wrong approval costs about 20 times a delay. On our synthetic set of 200 claims: 2 wrong approvals (both identity theft with full stolen credentials), total cost 110 against 130 for never auto-approving, and all 40 clean claims auto-approved.
- **Measured, on invented test sets (said on screen):** Scam Shield on 80 messages: 74% accuracy with Jev alone, 84% with the cascade, 0 false alarms and 0 missed scams. Documents: the cascade reaches 0.8 accuracy on 20 mock documents. The Evaluation tab shows the cost curve, error rates, latency and cost per check, with a table view under every chart.
- **When an AI service is down**, the app doesn't guess: the case goes to an officer with `SERVICE_UNAVAILABLE`. The Judge console can simulate each outage live.

## Real vs simulated

| Part | Real or simulated |
|---|---|
| Identity (NADRA-like), family records, death registry, pension office | Simulated: mock APIs with seeded fake data (mock CNICs start `90000`) |
| Payments and account-holder lookup (Raast-like) | Simulated (mock IBANs start `PK00MOCK`) |
| SMS / WhatsApp delivery | Simulated (messages are pasted or shared into the app) |
| Jev decision model | Real API calls (TypeSafe), pinned version |
| Second model in the cascade | Real: Azure OpenAI `gpt-5-mini`, called only when Jev is unsure |
| Urdu audio | Real: Azure neural voice (`ur-PK-UzmaNeural`); falls back to the phone's own voice |
| Sindhi and Punjabi audio | Real voice, unreviewed text: ElevenLabs v3 pre-generated clips from machine-assisted text, not yet reviewed by a native speaker (`docs/AUDIO-REVIEW.md`); free-plan audio credited to ElevenLabs. Keys without a clip stay Urdu |
| Document reading (OCR) | Real: Azure Document Intelligence `prebuilt-read`, Tesseract as fallback. The documents are mock specimens |
| Document edit checks (copy-move, ELA, registry cross-check, Jev image questions) | Real code; indicators for an officer, never a verdict |
| Scam Shield link and caller checks | Real code, deterministic; links are never opened. Official domains and short codes are real-world patterns, treated as evidence, never proof |
| Screenshot and voice-note checks | Real: OCR and Azure speech-to-text (ffmpeg conversion). Not stored |
| Share from WhatsApp / SMS into the app | Real PWA Web Share Target (installed app) |
| Login, roles, consent, audit chain, rules engine, offline outbox | Real, built in this project |
| Pension rules | v1 and v2 follow real federal Finance Division memos (sources linked in the demo script); the cases are mock |
| Sindhi and Punjabi text | Machine-assisted translation, to be reviewed by native speakers. Voice: ElevenLabs v3 clips for most lines, unreviewed; the rest stay Urdu |

## Honest limits

- The claim decision itself is rules plus fuzzy name matching, not AI. The AI is in Scam Shield, the document checks and the voice.
- Our test sets are small and invented (200 claims, 80 messages, 20 documents). The numbers show the method, not real-world accuracy.
- No real government system is connected. Urdu speech recognition is weak (Azure only offers Indian Urdu); the voice output is good.
- The demo PIN is public by design.
- Ticking a document box counts as having that paper, even without a photo (the ticks stand in for papers checked at the counter). Counting only uploaded papers is the first fix after judging.

## Architecture (short)

- **Frontend:** React + Vite + TypeScript PWA, offline-first (outbox in IndexedDB), citizen JavaScript under 100 KB gzipped, strict CSP.
- **Backend:** FastAPI (Python 3.12), SQLAlchemy 2, Alembic, Postgres. One response envelope `{success, data, error, meta}`. Roles, consent and tenant checks live on the server.
- **Rules:** versioned JSON; every decision stores its rule version. A Digital Twin replays past cases under a draft rule before it's published.
- **Hosting:** Railway (backend, frontend, Postgres), Azure (speech, document reading, `gpt-5-mini`), Sentry, Codecov. Details in [docs/INFRA.md](docs/INFRA.md).
- **Security:** audited against the GitHub Well-Architected and Invicti checklists (148 items, [docs/SECURITY-CHECKLIST.md](docs/SECURITY-CHECKLIST.md)); CI runs pip-audit, npm audit, Bandit and gitleaks.

## Docs

| Doc | What is in it |
|---|---|
| [docs/01-IDEA.md](docs/01-IDEA.md) | The idea, the story step by step, users and roles |
| [docs/02-COMPLIANCE.md](docs/02-COMPLIANCE.md) | Every requirement in the brief mapped to where we meet it |
| [docs/03-ARCHITECTURE.md](docs/03-ARCHITECTURE.md) | Stack, decision engine, data model, API, security |
| [docs/05-DEMO-SCRIPT.md](docs/05-DEMO-SCRIPT.md) | The 7-minute demo story, judge console, what to try |
| [docs/SECURITY-CHECKLIST.md](docs/SECURITY-CHECKLIST.md) | Security audit, STRIDE threat model, scanner results |
| [docs/TEST-REPORT.md](docs/TEST-REPORT.md) | Browser test runs and bugs found |
| [docs/research/](docs/research/) | Survivor pension rules and scam patterns, with sources |
| [docs/STATUS.md](docs/STATUS.md) | Build log and decisions |

## Run it locally

```bash
docker compose up -d                      # Postgres + Redis
cd backend && python -m venv .venv && .venv\Scripts\activate
pip install -r requirements.txt && copy .env.example .env
alembic upgrade head && python -m app.seed   # demo data, all PINs 1234
pytest && uvicorn app.main:app --reload   # API docs: http://localhost:8000/docs
cd ../frontend && npm install && npm run dev   # http://localhost:5173
```

Without the Azure keys the app still runs: Tesseract reads documents, the phone speaks, and the second model is off. Without the TypeSafe key, every question Jev would answer counts as "not sure" and goes to an officer.

## Team working rules

- All code in this repo was written **during the hackathon**. No code, models or data files were copied from any earlier project.
- Mock data only. Secrets live in `.env` (git-ignored) and on Railway, never in git.
- Branch per task, pull request into `main` with CI green, conventional commit messages.
