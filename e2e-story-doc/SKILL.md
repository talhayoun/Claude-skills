---
name: e2e-story-doc
description: Deep-dives into ONE backend service or endpoint the user names (for example "createProcess") and writes an end-to-end document that explains, in plain non-technical language, everything it does — especially how it behaves differently for each value of a parameter (process type, action, status, etc.), which SMS/emails/notifications go out, to whom, when and under what conditions, which systems it touches, what can go wrong, and what to test. Output is a polished HTML page. Use this skill whenever the user names a service and asks to explain it, document it end to end, map its flows or branches, list what messages it sends, or prepare a document for QA or third-party partners, even if they don't say "story" or "HTML".
---

# E2E Story Document — one service, every path

The user names one service. Go deep into that service only, and produce a document a QA tester or an outside partner can read like a story. When they finish, they should know exactly what happens for every possible input value, every message that goes out, and what to test. No coding knowledge needed.

## Scope

- Document **only the service the user named**. If the name is ambiguous (several endpoints or classes match), list the matches and ask which one before starting.
- Go as deep as the code goes: follow calls into helper services, handlers, strategies, event listeners, queues and scheduled jobs that this service triggers. Stop at the boundary of an outside system, and describe what is sent to it.
- If the user asked for several services, make one document per service.

## Output

One self-contained HTML file built from `assets/template.html`.

1. Copy `assets/template.html` to the output location (default: `docs/e2e-<service-name>.html`).
2. Replace only the `DOC` object between `===== Fill this object` and `===== End of data`. Do not change the CSS or the rendering script.
3. Remove the `notice` field (it marks the example data).

The template already contains a full example (`createProcess` with 3 process types plus an unknown value). Match its tone and level of detail.

### The `DOC` object

- `serviceName` (the code name), `title` (plain name of what it does), `lead` (1–2 sentences), `updated`, `audience`, `callers` (who calls it)
- `inputs`: `{ name, code, required, isBranch, plain }` for every field the caller sends. Set `isBranch: true` on the parameter that decides the path.
- `systems`: `{ id, name, plain }` for every system touched. Plain names only ("the old system", not "AS400").
- `actions`: the columns of the overview table — one per kind of effect the service can have (SMS to customer, email to agent, task created, old system updated, scheduled job…). Build this list from what you actually find in the code.
- `shared`: steps `{ system, text, cond? }` that happen for every value, before the paths split.
- `scenarios`: one per branch value:
  - `id`, `value` (the real value, e.g. `CLAIM`), `title` (plain name), `when` (when someone would use this value)
  - `actions`: for each action id — `true`, `false`, or a short string for a condition ("only if the customer agreed to SMS")
  - `steps`: what happens after the shared steps, in order. Put conditions in `cond`.
  - `messages`: every message sent — `{ channel: "sms" | "email" | other, to, when, content, cond? }`. `content` describes what the message says in plain words (don't paste templates or personal data).
  - `problems`: `{ when, result }`, `checks`: `{ try, expect }`, `files`: code files for this path
  - Add `reject: true` on a scenario that only rejects the request.
- `tech`: `{ method, path, files }` for the service as a whole.
- `openQuestions`: anything you could not confirm.

## Process

### 1. Find the branch parameter and all its values
Find the parameter (or combination) that decides what the service does: an enum, a `switch`, an if/else chain, a strategy map, a handler registry, a factory. List **every** value:
- All enum members, even ones with no handler (say what happens to them)
- The `default` / fallback case
- Missing, empty or unknown values
- Values that share a handler (group them in one scenario and list all values in `value`)

If the behavior also depends on a second factor (who called, customer settings, feature flags), show it as conditions (`cond`) inside each scenario, not as extra scenarios — unless it changes the flow completely.

### 2. Trace each value end to end
For each value, follow the code all the way down. Read, don't guess:
- Permissions and validation specific to this value
- Every database read and write
- Every outgoing message: SMS, email, push, letter. For each one find: **who** receives it (customer, agent, internal team), **when** (immediately, after another step, delayed or scheduled), **what** it says (from the template name, subject, or the variables passed), and **the condition** for sending it (opt-in flags, missing phone/email, who opened the request)
- Calls to other services and outside systems, and what is sent to them
- Events published, queue messages, scheduled or delayed jobs, and what they do when they run
- Errors, retries, timeouts and fallbacks — and whether earlier steps are rolled back or stay done
- The order of steps, when it matters (is the SMS sent before or after the save? what if the save fails after the SMS went out?)

Write down file names as you go.

### 3. Write the story
Language rules — the most important part:
- Write for someone who has never seen code. If a word needs explaining to a new tester, replace it.
- Translate terms: "returns 409" → "the request is blocked because one already exists". "publishes an event" → "tells another system that a process was opened". "cron job" → "an automatic check that runs every X".
- Each step is one or two short sentences: what happens, and *why* when the reason isn't obvious.
- Use the same name for a system or a person (customer, agent) everywhere.
- Write in the user's language. If they wrote in Hebrew, write the document in Hebrew.
- Code names (enum values, field names, files, paths) appear only in `value`, `inputs[].code`, `files` and `tech`.

### 4. Problems and checks per scenario
- One `problems` entry for every real error path in that branch. Say what the caller sees and what state things are left in.
- `checks` must be runnable in a meeting with a partner: concrete inputs and concrete expected results. For each scenario cover: the main path, every condition that changes which messages go out (with and without), every problem, and side effects (was the message received? was the task created?). For delayed actions, note if the delay can be shortened in a test environment.

### 5. Be honest about gaps
Message text that lives in an outside template system, config values not in the repo, behavior of outside systems — don't invent them. Put them in `openQuestions`.

### 6. Safety
Never copy secrets, connection strings, real phone numbers, emails, customer data, internal hostnames or API keys into the document. It may be shared with outside partners.

### 7. Final check
Before finishing, confirm:
- Every branch value from step 1 has a scenario (including unknown/default)
- Every message sent in the code appears in `messages`, with recipient, timing and condition
- The overview table matches the scenarios (no ✓ without a step or message behind it)
- No jargon outside the code-name fields
- The page renders (valid JS object; escape `"` inside strings)
