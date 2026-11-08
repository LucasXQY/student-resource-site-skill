# Operations: AI budgets, free hosting, secrets, testing and launch

The only thing such a site should pay for is AI API credit. Everything else runs on free tiers, and every paid call is capped in dollars. Each item: the requirement, then why.

## Contents
1. Budget pools and sizing
2. Dollar caps, not call counts
3. Reserve, call, settle
4. The ledger
5. Model choice per task
6. When a call fails or the budget is out
7. Kill switch
8. Timeouts
9. Free tiers
10. One price table
11. Secrets and variables
12. Analytics
13. Testing and launch
14. Gotchas
15. Incident runbook: personal data published, takedown requests

## 1. Budget pools and sizing

- Give every AI feature its own named pool with a monthly and a daily dollar cap. Features that cost about the same and run in the same place may share one (for example "cheap-model batch jobs" = moderation + topic indexing).
  Why: one feature must never spend another's money. A flood of forum posts or one huge upload must not leave the chat helper dead for the month.
- Keep the sum of pool caps below the total budget, with about 10% unallocated for new features and mistakes. A new pool is carved out of that reserve.
  Worked split for $150/month: uploads 60, chat helper 60, cheap-model batch jobs 10, forum checks 5, unallocated 15.
- Size the daily cap by traffic shape:
  - **Public-facing, steady pools: daily = monthly / 30.** Even an abuser can drain only 1/30 of the month in a day.
  - **Bursty pools fed by maintainers: daily ≈ monthly / 4.** A big batch finishes within days. Example: uploads 15/day.
- Add a per-run cap to batch pools where jobs queue (example: $8 per upload run). A large job then finishes over several runs without manual approval, and one 60-file upload cannot block three others.
- Set the provider-side organisation spending limit in the provider console to the total budget.
  Why: if your own ledger has a bug, the API refuses and the code treats that as "no answer".
- Make every cap overridable by a repository variable, with defaults in code. An unparsable, negative or infinite override falls back to the default; it never means "no cap".

## 2. Dollar caps, not call counts

- Cap in dollars computed from the usage the API returns, not in number of calls.
  Why: one call can cost 100× another (a 3-page vs a 100-page PDF, a small vs a top model, a fallback model, long-prompt price tiers). Count caps either block legitimate large jobs or let expensive ones through. On a bursty upload pool, "N files per day" caps hit first and stalled big uploads, so they were removed.
- Keep count caps only as a second guard where calls are uniform. Examples: the helper at 150/day site-wide and 15/day per visitor, moderation at 100/day.
  Why: a pricing bug that prices calls at $0 must not leave spend unbounded.

## 3. Reserve, call, settle

- Before each call or batch, atomically reserve an estimate. Refuse if spent + reserved would pass the daily or monthly cap. After the call, settle the difference, which may be negative.
  Why: concurrency. Without a reservation, N parallel requests each see "under cap" and all go through.
- Set each reservation at or above a realistic worst case, and bound the inputs so that worst case holds.
  - Chat helper: $0.25 per question, made true by at most 6 turns, 500 characters of user text, 4 tool rounds and 1,500 output tokens, with the last round forced to answer without tools.
  - Cheap classifications: $0.002 per item, with text capped at about 12k characters.
  - Full privacy check: $0.04 + $0.025 per page.
- A call that failed with unknown cost keeps its whole reservation, because it may still be billed. A response that came back unusable (refusal, truncated, bad JSON) is charged at its real cost.
- Where the ledger is a slow remote store, reserve per batch into an in-memory per-run meter and settle once per run in a `finally` block.
  Why: this avoids hundreds of writes, and a crash cannot leave reservations unsettled.

## 4. The ledger

- One small ledger holds today's calls per task and each pool's dollars for today and this month. Days and months are UTC.
- Use optimistic concurrency: write against the last-seen version; on conflict, re-read, re-apply and retry a few times; then raise.
- **Fail closed.** If the ledger cannot be read, holds a non-number, or retries run out, treat the pool as over budget and call nothing.
  Why: a broken ledger must never mean unlimited spending.
- If a settle fails, mark the ledger broken for the rest of the run: no more reservations. Settle itself never raises, so cleanup still runs.
- **Roll forward only.** If the stored day or month is later than the run's own (a run that started before midnight writes after it), keep the stored period and add on top. Clear a period only when it is older.
  Why: otherwise a late writer resets today's totals and the cap gets spent twice.
- Clamp totals at ≥ 0 and round them; refunds never make a pool negative.
- Where it lives:
  - **Bots in CI:** a JSON file on an orphan state branch, which never touches the main branch.
  - **Serverless features:** a strongly consistent per-key store, such as one Durable Object per counter.
  - **Each pool has exactly one owner,** and the two runtimes share only the price table. They never have to sync totals.
- The state branch of a public repo is public and permanent: no personal data in ledgers or AI caches. Cache rectangles and verdicts, never redacted text.

## 5. Model choice per task

- **Small, cheap model** for high-volume classification: sorting uploads, near-duplicate comparison, comment moderation, forum checks, topic indexing, and the helper's on-topic gate.
- **Top model** only where a miss is irreversible (the privacy check of files about to enter permanent public history, and verifying redactions) or where quality is the product (helper answers). At the time of writing the small model was about 40× cheaper per token; check current prices.
- **Gate before you pay:** a small-model scope check rejects off-topic helper questions before the top model runs.
- Call a model only when deterministic rules cannot decide:
  - exact duplicates: content hash;
  - near-duplicates: text similarity first, and only the uncertain band goes to the model;
  - moderation: only text that hits a keyword list;
  - the lecturer list and the link checker: no model at all.
- Keep defaults in code and allow per-task overrides by repository variables, so a model can be swapped without a commit.
- Use low thinking effort for classifications and medium for the privacy check. Use a JSON-schema output format so every answer parses, and cache long system prompts.
- Bound input per call. Examples: 3 pages to sort, 100 pages for a full check sent in 25-page parts, images ≤ 4.5 MB, ≤ 8 page images per verify call. Watch long-prompt price tiers.
- Price by the model that answered (the response's model field), not the one requested: a fallback can route to another model. Price an unknown model at the dearest known rate.
- Cache results by content hash, so reruns, renames and moves cost nothing.

## 6. When a call fails or the budget is out

- One wrapper returns the parsed answer or None. None covers a missing key, an exception, a rate limit, a refusal, truncation, invalid JSON, or an answer outside the schema. Every caller has a non-AI fallback.
- Choose each fallback by asking which mistake cannot be undone:
  - **Publishing user content:** hold for a human on error, timeout, pause or an empty budget. A keyword hit is never auto-published, even if the model says publish.
  - **Accepting user content:** fail open. Take the post or file and queue it, so nothing is silently lost.
  - **Privacy check before a public commit:** do not commit. List the file under "needs a human" with the reason; a later run with budget retries it.
  - **Moderation second opinion:** fall back to the keyword rule.
  - **Indexing:** skip the item and retry next run.
  - **Chat helper:** a polite "today's budget is used up, try tomorrow".
- After 2 runs with no usable answer for the same file, stop asking until a maintainer sends a retry command, so a file that always fails doesn't cost money on every rebuild.
- A maintainer override (approve) is recorded with the approver's name. Retry never raises a cap.
- Show the money: each bot report lists this run's AI spend per step and what is left in the pool this month.

## 7. Kill switch

- One repository variable stops every bot (each job has a condition like `if: vars.BOTS_PAUSED != '1'`).
- Never gate the site deploy: the site stays up and only automation stops.
- Bake the same flag into the serverless backend at deploy time:
  - the helper answers "paused" and calls no model;
  - the forum still accepts posts but holds them all;
  - the site hides the chat button.

  Setting the variable alone does not pause the backend: re-run its deploy and the site build.

## 8. Timeouts

- Set `timeout-minutes` on every CI job. Examples:
  - 5 minutes: moderation;
  - 10 minutes: deploys and comment commands;
  - 15 minutes: the site build and the link check;
  - 20 minutes: zip bundles;
  - 60 minutes: upload processing with full AI checks.

  Why: a hung job holds its concurrency group and burns minutes.
- Set client-side timeouts too. Examples: the API client 180 s with 2 retries, 600 s for a long privacy check, 30–60 s on the serverless side.
- Use a concurrency group per item with `cancel-in-progress: false`, so two rebuilds of the same upload never race.

## 9. Free tiers

- **Public repository:** CI minutes, artifact storage, static hosting and release assets are free. Publish big download bundles (per-course or per-session zips) as release assets, not commits.
- Put no payment method on the organisation. Running out of quota then fails jobs instead of billing.
- Serverless backend on its free plan (requests per day, key-value or SQLite storage, a free human-check widget). Past a limit, requests error and nothing is billed.
- Deploy workflows do nothing until their secrets exist, so code can merge before accounts are made.

## 10. One price table

- Keep model prices in one plain JSON file: input, output, cache-write and cache-read rates, long-prompt tiers, and the model to assume for unknown names.
- Both runtimes read it: the bots at runtime, the serverless backend at bundle time.
- A change to it triggers a backend redeploy, and a test checks that both read the same file, so the tables never drift.

## 11. Secrets and variables

- The model API key is a CI secret, pushed into the backend's own secret store by the deploy job. Never put it in the static site; this is why chat goes through a backend at all.
- The deploy token and account ID are CI secrets only.
- The human-check secret goes to CI and the backend. Its site key is public and goes in a plain variable.
- The admin-page token for the maintainer's moderation page is a CI secret, then a backend secret.
- Bots use the built-in CI token with least privilege per workflow. PRs opened with it do not trigger other workflows; dispatch them explicitly where needed.
- Every non-secret knob is a repository variable: models, caps, pause flag, backend URL, site key.
- Only the maintainer sets secrets and organisation settings; prepare the exact names and let them do it.

## 12. Analytics

- Use a free, privacy-first, cookieless counter (for example GoatCounter): no cookies and no personal data, so no consent banner.
- Say so in the footer, and add a build test that the snippet is on every page.
- When a policy like this changes, update the design docs in the same PR.

## 13. Testing and launch

- Unit tests for every bot and the site build, run locally and ideally in CI. Derive expected counts from the repository instead of pinning numbers, so adding content never breaks tests. When real content makes a test stale (for example a programme that "has no courses" gets one), fix the test to check the rule, not the old fact.
- An independent review (a fresh reviewer, not the author) for any code that decides what becomes public: redaction, moderation, publishing gates. Ask it for concrete failing inputs. In practice this caught two leaks in a redaction rule that passed all of its author's tests.
- A live end-to-end test before announcing each pipeline:
  - one throwaway submission with synthetic files (fake names, fake IDs) covering text PDFs, scans, Office files, a zip, a duplicate and a file needing a decision;
  - exercise every maintainer command, then close the test submission unmerged.

  Record what broke and fix it before real users arrive.
- Live tests show platform quirks no unit test would. For example, a code host may write the literal "None" for an unselected optional dropdown.
- Ship a changelog page and add an entry for every user-visible release.
- When real uploads start, read the first few bot reports closely and tune rules from what actually goes wrong. False positives that hold good files cost maintainer time; false negatives that leak data cost trust.

## 14. Gotchas

- Code-host content APIs may omit files above about 1 MB; fetch raw. A stale version returns a conflict: re-read and retry.
- Creating an orphan branch through an API takes three calls: a tree, a commit with no parents, and a ref.
- Some key-value stores delete at most about 128 keys per call; chunk cleanups.
- You pay for refused and truncated responses; record cost whenever a response arrives.
- Validate every number read from variables or the ledger (NaN, infinity, booleans, negatives).
- Design docs drift from code; when they disagree, the code is the truth, and fix the doc.
- Some edge networks reject HTTP clients with a default library User-Agent (for example Python's `urllib`); send a named one from every script that calls the backend.

## 15. Incident runbook: personal data published, takedown requests

Write this runbook into the maintainer docs before launch, so nobody improvises under stress.

**Personal data reached a public place** (a commit, a PR branch, the site, a release zip, the state branch):
1. **Stop the spread first.** Set the pause variable so no bot rebuilds or re-publishes. If the file is on main, open a PR that deletes it and merge it; the site redeploys without it.
2. **Remove it from history.** Rewrite the branch history to drop the file, force-push, and delete any release asset or zip that contains it. Remember that code hosts keep pull-request refs and cached views after a force-push: file the host's support request for removing sensitive data so those are purged too.
3. **Remove other copies:** the workflow artifacts (thumbnails), the bot's state branch, and any issue attachment (ask the uploader to delete the comment or attachment, or delete the issue if you can).
4. **Tell the affected person** plainly what was exposed, where, for how long, and what was removed. Do it before they hear it elsewhere.
5. **Find the cause and add a test** that would have caught it (a new leak case in the redaction tests, a new pattern in the personal-data list). Write a short note in the changelog or maintainer log without repeating the data.

**A takedown request** (a lecturer, the school, a copyright holder, a student):
- Acknowledge it within a day and remove first, discuss later. Removal is cheap, and a dispute is not.
- Use one public contact route (an email address and an issue template), named in the disclaimer and every page footer.
- Keep a private log of requests and actions taken, without copying personal data into public issues.
- If a whole class of material is the problem (for example a lecturer asks that their papers not be shared), record the rule in the contributing guide and the bot's skip rules, so it doesn't come back with the next upload.

**A secret leaked** (API key, deploy token, admin token): revoke and rotate it at the provider first, then update the CI secrets and redeploy. Check the provider's usage page for spend since the leak.
