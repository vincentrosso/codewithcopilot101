# Copilot instructions

This file provides guidance to GitHub Copilot (chat and agent mode) when working in this repository.

## What This Repo Is

A curriculum repo for a 12-week self-paced programming course — the **GitHub Copilot edition** of codewithai101. Content is pure markdown — there is no build system, no test suite, and no runnable code in this repo. The student's actual project code lives in a separate local directory (`~/dev/brand-lens/`), not here.

## Structure

Each week's content is broken into daily lesson files at the root, named `WxxDy.md` — the **week number is zero-padded to two digits** (W01–W12) so files sort correctly in a plain file listing; the day is single-digit (D1–D5). Cross-references between lessons use the same padded token. **All twelve weeks are written** (W01D1 through W12D5; Week 1 is four days, Weeks 2+ are five — 59 days total). Weeks 9–12: W9 static site (render briefs to HTML, view locally), W10 branches/PRs/CI, W11 deploy to S3, W12 CI/CD auto-deploy + finale. The course **closes at Week 12**; Brand Lens has no runtime backend, so the deploy is a static site on S3 (not a FastAPI server — that, plus Docker/Fargate, is deferred to a Course 2; see the README's "Course 2" section).

```
W01D1.md … W12D5.md — daily lessons (the day count restarts at "day N:" in commit msgs; W8 = days 35–39, W12 = days 55–59)
README.md          — course overview, format, full week-by-week table, sidelines index, future ideas
CHANGELOG.md       — per-file version history (semver; default bump = patch)
.github/prompts/   — reusable Copilot prompt files (e.g. /done: stage, commit, push)
sidelines/         — reference material outside the day-by-day flow (read when curious, not paced)
qa/                — post-course QA track: QA01–QA06 + its own README (see below)
field/             — post-course field notes: F01… + its own README (see below)
```

The weekly rhythm from Week 2 on is consistent and should be preserved when drafting new weeks: **D1/D2/D4 are hands-on building, D3 is a no-AI foundations day (conceptual + mini-quiz), D5 is the Friday mentor review** (demo → break-and-catch → judge a Copilot-written artifact → retrospective journal). The capstone's `brand-lens` code state at the end of each week is cumulative — new weeks build on the files prior weeks created (`fetch.py`, `parse.py`, `runner.py`, `brands.yaml`, then `db.py`, `brief.py`, `render.py`, `templates/`, `tests/`).

Sidelines are kebab-case `.md` files in `sidelines/`, each linked from the README sidelines table and from the lesson day where they're most relevant.

## QA Track

`qa/QA01.md`–`qa/QA06.md` is a six-lesson post-course module on quality assurance, assuming all twelve weeks (pytest, git, PRs, CI, diff review). Same lesson format as a course day. Two constraints to preserve when editing:

- **Lessons 1–4 are AI-free**, on the Week 1 principle — the learner can't judge Copilot's test cases in Lesson 5 without having written his own. Lesson 5 is browser Copilot (thinking partner: gap analysis, pre-mortems, critique); Lesson 6 is Copilot (writing Playwright tests, CI).
- **One pinned practice app**: the Toolshop demo at `practicesoftwaretesting.com` (UI + an API at `api.practicesoftwaretesting.com`, seeded `customer@`/`admin@` accounts, password `welcome01`). Don't swap it for a generic "app of your choice" — concreteness is the point. Deliverables land in a separate repo (`~/dev/qa-lab`), never in `brand-lens`.

## Field Notes

`field/F01.md`… are case studies of real NIT America working days, for a learner who has finished the course and the QA track. Not paced, not Brand Lens. Each keeps the same lesson skeleton (goal + time budget → numbered sections → optional hands-on in a free vendor sandbox → journal with the required "A line I accepted but still don't fully grasp" section → "what done looks like"). Constraints: **anonymise the client** (no business names, store domains, phone numbers, account ids, secret names, internal hostnames); keep the vendor's real mechanics and the real decisions; the honest "where the vendor said no" section is mandatory. Add each note to `field/README.md` and the README's field-notes table.

## The Copilot toolset (keep lessons consistent with this)

- **Editor:** VS Code, from W01D1. Week 1 ticks **Chat: Disable AI Features**; it stays ticked through Week 3.
- **Weeks 2–3:** Copilot Chat in the browser at `github.com/copilot` (Copilot Free). Explain-only; the learner types every line.
- **Week 4 on:** Copilot in VS Code. **Ask** mode = read-only; **Agent** mode = edits (reviewed with Keep / Undo) and terminal commands (Allow / Skip). Inline ghost-text completions stay **off** for the whole course (`"github.copilot.enable": { "*": false }`) — they bypass diff review. Never suggest "always allow" / auto-approve.
- **Mentor-review "judgment" exercises** use Ask mode, so Copilot proposes and the learner judges, without files changing.
- **Week 7 LLM calls:** GitHub Models through the `openai` SDK — `base_url="https://models.github.ai/inference"`, model `openai/gpt-4.1-mini`, token in `.env` as `GITHUB_MODELS_TOKEN` (a fine-grained PAT with Models: read-only). Free tier = rate limits, not a bill. `brief.py` reads the token with `os.environ.get(..., "not-set")` so tests and CI can import it with no `.env`. Mocks target `client.chat.completions.create` and fake `response.choices[0].message.content`.
- Keep the two AIs distinct in prose: **Copilot** is the learner's assistant; **the model** (GitHub Models) is what Brand Lens's code calls.
- **Field notes** were worked with Claude Code in real life. Keep that history accurate and map each habit onto Copilot rather than rewriting what happened.

## Pedagogical Constraints


**Week 1 is AI-free by design.** The learner is prohibited from using Copilot, ChatGPT, Claude, Cursor, or any other AI assistant — and VS Code's built-in AI is switched off. All lesson files for Week 1 must reinforce this and never suggest using AI as a shortcut. Google and Stack Overflow are explicitly allowed.

Starting Week 2, AI tools come online. Lessons from that point can include AI-assisted workflows.

## Capstone

The capstone project is **Brand Lens** — a tool that helps Korean brands evaluate their US market presence using web scraping, structured data, and LLM-generated briefs. Every day's lesson is designed to advance this project. W01D3 introduces fetching and saving a single URL; W01D4 extends it to batch processing; the pipeline then grows a YAML config + modules (W3), a test suite (W5), a SQLite store (W6), LLM summarization (W7), and a rendered per-brand brief — the **MVP milestone reached at the end of Week 8**. The capstone is the throughline — lessons should not introduce concepts without connecting them to it.

## Learner Profile

Business background, no prior coding experience. Explanations should avoid assumed knowledge of CS terminology. The "Friday review" is a 30-minute mentor session (with Vincent) where the learner demos their work and debugs a deliberately-broken version of their own code live — no AI allowed during this session.

## Lesson Format

Each `WxDy.md` file (e.g. `W01D1.md`, `W03D5.md`) follows a consistent structure: goal + time budget → numbered sections with inline terminal commands → commit checkpoint → journal prompt → "what done looks like" checklist. Maintain this structure when adding new days. Journal prompts should ask for genuine reflection — name specific lines or concepts the learner can't yet explain — not just factual recall. Avoid vague framings like "what felt magical."

### Required journal sections

Every lesson day's journal prompt must include a section titled **"What I don't fully understand"** (or the equivalent **"A line I accepted but still don't fully grasp"** when the day involves accepting code from Copilot). The prompt must ask for a *specific named line, function, or concept* — not a general "anything confusing?" question. This is the section that drives the next mentor session, the next lesson's recap, and any sidelines the learner needs.

### The "Edit:" pattern — encourage, don't penalize

When the learner predicts an outcome ("I think this will print an error"), runs the code, finds they were wrong, and edits the journal with the correction (`### Edit` or similar), that is the most valuable habit in the journal. Mini-quiz prompts and "predict the output" exercises should explicitly invite this: ask for a prediction *before* running, and tell the learner that updating the entry afterward — saying what they got wrong and why — is the point of the exercise, not a sign of failure. Friday review notes and lesson recaps should call out specific instances of this pattern by name when they appear in the learner's journal. The behavior is fragile early on; praise it directly so it sticks.
