---
name: e2e-story-doc
description: Reads a backend codebase and writes an end-to-end "story" document that explains, in plain non-technical language, what happens inside each service or endpoint, which systems it touches, what can go wrong, and what to test. Output is a polished, readable HTML page. Use this skill whenever the user asks for an end-to-end document, E2E flow doc, service walkthrough, "what happens when X is called", a doc for QA or for third-party partners/consumers of an API, or wants to explain services to non-developers, even if they don't say "story" or "HTML".
---

# E2E Story Document

Turn the code of a service into a document that a QA tester or an outside partner can read like a story. They should finish it knowing what each service does, what happens step by step, and exactly what to test. No coding knowledge needed.

## Output

One self-contained HTML file built from `assets/template.html`.

1. Copy `assets/template.html` to the output location (default: `docs/e2e-story.html`).
2. Replace only the `DOC` object between `===== Fill this object` and `===== End of data`. Do not change the CSS or the rendering script.
3. Remove the `notice` field (it marks the example data), unless you need it for a real warning.

The `DOC` object shape:

- `project`, `lead` (1–2 sentences: what this doc is and who it's for), `updated`, `audience`
- `systems`: every system the services touch. `id`, `name` (a plain name, like "the old system", not "AS400 DB2"), `plain` (one sentence: what it is and why it matters)
- `services`: one entry per endpoint or flow, each with:
  - `id` (url-safe), `title` (an action in plain words: "Check policy details"), `summary` (why someone uses it), `callers` (who calls it)
  - `steps`: ordered `{ system, text }`. `system` is a system `id`, or `null` for logic inside the service itself
  - `problems`: `{ when, result }` — what goes wrong and what the caller sees
  - `checks`: `{ try, expect }` — a concrete thing to try and the expected result
  - `tech`: `{ method, path, files }` — the only place for technical names
- `openQuestions`: things you could not confirm from the code

## Process

### 1. Map the services
Find every entry point: controllers, routes, message handlers, scheduled jobs. If the user gave a list (for example "only the 4 endpoints in production"), stick to that list. Group related endpoints into one service when they form one flow from the caller's view.

### 2. Trace each one end to end
Follow the call from the entry point to the end. Read, don't guess:
- Guards, auth checks, and permissions (who is allowed, and what happens if not)
- Input validation (required fields, formats, limits)
- Every database read and write, and every call to another service or outside system
- Branches: if/else on business rules, feature flags, old vs. new data
- Errors: every exception thrown, every catch, retries, timeouts, fallbacks
- Side effects: messages sent, emails/SMS, logs or audit records, queues

Write down the file names as you go. They go in `tech.files`.

### 3. Write the story
Rules for language — this is the most important part:
- Write for someone who has never seen code. If a word would need explaining to a new tester, replace it.
- Translate terms: "returns 404" → "the caller gets a message that it wasn't found". "JWT guard" → "checks the partner is logged in". "upsert" → "saves the new value, or creates it if it isn't there".
- Each step is one or two short sentences: what happens, and *why* when the reason isn't obvious (security, legal, an old system that still needs the data).
- Use the same name for a system everywhere. Never switch between "the policy DB" and "the policies store".
- Write in the language the user asked for. If they wrote in Hebrew, write the document in Hebrew.
- Status codes, paths, class names and table names go only in `tech`.

### 4. Problems and checks
- One `problems` entry for every real error path you found in step 2. Say what the caller actually receives.
- `checks` must be specific enough to run in a meeting with a partner: real inputs ("a policy opened before 2015"), real expected results. Cover the happy path, each problem, each permission level, and each side effect (was the log written? was the SMS sent?).

### 5. Be honest about gaps
If the code does not make something clear (a config value, a race condition, behaviour of an outside system), do not invent it. Put it in `openQuestions`.

### 6. Safety
Never copy secrets, connection strings, real customer data, internal hostnames, or API keys into the document. It may be shared with outside partners.

### 7. Final check
Before finishing, open the file and confirm:
- Every service in scope is there, and every step is backed by code you read
- No technical jargon outside the `tech` blocks
- The page renders (valid JS object, no stray quotes; escape `"` inside strings)

## Example

`assets/template.html` already contains a complete example (a made-up partner policies API) showing the tone and level of detail to aim for.
