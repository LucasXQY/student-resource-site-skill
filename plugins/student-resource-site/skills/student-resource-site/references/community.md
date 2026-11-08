# Community features: reviews, lecturer directory, discussion, opportunities, blog, helper chatbot, moderation

Everything a visitor can write. The website stays static; every write path goes through one small serverless service. Each requirement: the rule, then why. "(gap)" marks a mistake seen in practice that the rule fixes.

## Contents
0. Core decisions
1. Shared platform rules
2. Course reviews
3. Lecturer pages: a directory only
4. Discussion boards
5. Opportunities board
6. Blog with images
7. Helper chatbot
8. Moderation
9. When the service is off
10. Deploy and secrets
11. Things not to build
12. Gotchas

## 0. Core decisions

- **One serverless edge service** for every write path, with a strongly consistent store (for example one SQLite-backed Durable Object per feature).
  → A static host can't hold API keys or count anything. One service means one deploy, one secret store and one admin page. A free tier is enough at student scale, and going over it makes requests fail instead of billing.
- **No login for any community feature.**
  → Students mostly don't have code-host accounts. An earlier version built on the code host's discussions needed one and got little use. Anonymous posting is safe only with the controls below.
- **Moderate before publishing.**
  → Showing comments at once and folding them later by keyword leaves abuse public until the bot runs.
- **Fail closed.** A model error, timeout, pause, spent budget or bad parse means "hold for a person", never "publish".
- **Hard dollar caps on every model use, checked before the call** (reserve, then settle; see `operations.md`), with count caps as a second guard.
  → On a public anonymous service, the model bill is the attack surface.
- **IP is only a loose hard cap.** It never de-duplicates, and never makes a vote or post suspicious.
  → A whole class shares one campus WiFi address. Pin this rule with a test.
- **One kill switch** (a deploy-time variable): the service answers "paused", calls no model, and the site hides the chat button.

## 1. Shared platform rules

- **Human check (for example Turnstile, managed mode):**
  - register it for the production host and localhost;
  - load its script only when a form opens, so read-only pages load no third-party script;
  - tokens are single use, so reset the widget after every submit attempt, failed ones included, or every retry fails.
- **How often to check, per feature:** once per item for reviews and opportunity posts. For discussion and the chatbot, check once, then issue an HMAC-signed session for an hour.
  → Votes are cheap to fake; conversations need continuity.
- **Slide the session:** renew it on every use. The client checks expiry (with a 60 s margin) before sending; on a 401 it clears the session and shows the check again.
  → A fixed hour once ran out while someone was writing a long blog post.
- **Browser identity:** a random id in localStorage, shared by all features, with every storage access in try/catch.
  → Private windows throw. Without storage, each vote counts as a new browser, and the caps still hold.
- **Hashing:**
  - Store IPs and browser ids only as truncated, salted SHA-256 hashes, with a different salt per feature.
  - The admin page shows short prefixes only.
  - The chatbot's visitor key changes daily: hash(IP, day, secret).
  - Use a dedicated salt secret. (gap) Deriving the salt from another secret means rotating that secret silently resets every browser id, giving each browser a second vote and losing its right to withdraw.
- **Origin allow-list** (production plus a localhost dev port) on POST routes. Public GETs may send `Access-Control-Allow-Origin: *`. The human check remains the real gate, because Origin can be forged.
- **Input hygiene:**
  - normalise (NFC/NFKC) and strip control characters;
  - set explicit length limits and a body cap (about 20 kB, or 30 kB for blog posts);
  - require an "I accept the rules" box, checked on the server.

  Render user text only with `textContent`/`createElement`, never `innerHTML`. User links get `rel="noopener nofollow ugc"`.
- **Caching:** cache public lists at the edge for about 60 s, and purge after any visible change (a vote, a live post, a hide, an admin action). A purge may be local to one data centre, so other regions lag up to a minute.
- **Rate counters keep only today's rows,** deleting older rows on each bump, so no cleanup job is needed. Roll the day forward only.
- **The admin page is served by the service itself.**
  - Gate it with one long random token, kept as a secret and compared in constant time (hash both values, then compare).
  - Send `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `noindex` and `no-store`.
  - Keep the token in sessionStorage.
- **A weekly job** opens or updates an issue while flagged items wait, and closes it when the queue is empty.
- **Keep an action log** of every admin action: keep, drop, hide, restore, approve, reject. Record when, which item, and which maintainer (when there is more than one), without copying the content.
  → When someone asks why their post vanished, or a maintainer misclicks, the answer is in the log, and the action can be reversed.

## 2. Course reviews

- **Dimensions:** eight 1–5 scales, both ends labelled ("1 easy / 5 hard").
  - Required: course quality, teaching quality, difficulty, workload.
  - Optional, with a "skip" choice: amount learned, communication, instructor availability, would recommend.
- **Scores only:** no free-text field, and the form says "anonymous, scores only".
  → Free text about named lecturers is the riskiest content a student site can host.
- **A vote is** (course code, lecturer, optional session as YYYY-MM, scores).
- **Normalise lecturer names:** NFKC, collapse spaces, strip honorifics (Dr, Prof, local titles), and key them by a slug. → "Dr X" and "X" count as one person.
- **One vote per browser per (course, lecturer).** A re-vote replaces the earlier one, and the page says so.
- **Show averages only from 3 voters,** per (course, lecturer) and per dimension. Below that, show the count only ("2 rated; averages appear at 3").
  → Fewer voters would expose individual votes and let one person set the score.
- **The course-wide summary counts each voter once** (their latest vote).
- **Hard caps:** 100 votes per day per network and 20 per browser. A vote over a cap gets a polite "come back tomorrow".
- **Flag a vote, but still count it, when:**
  - more than about 8 other voters rated the same (course, lecturer) in the past hour; or
  - the course or lecturer is not on the site's list.

  → The maintainer chose "count first, review regularly" over "hold until reviewed". Make that choice explicitly with your maintainer.
- **Unlisted names:**
  - an unlisted course stays hidden;
  - an unlisted lecturer shows as "unconfirmed lecturer";
  - both stay that way until the maintainer keeps one of its votes. Scores still count from the first vote.

  → An insult typed into the name field never reaches the page.
- **Admin queue:** flagged votes grouped by (course, lecturer), with keep or drop per vote, and keep all or drop all per group. Each row shows time, session, scores, flags and hash prefixes. Keep marks a vote reviewed; drop removes it from the averages.
- **Kept lecturer names feed a bot** that opens a PR adding the lecturer to the course record, so the directory grows only through merged PRs.
- **The rate page:** a table of course × lecturer rows with a Rate button each, plus a search. Course pages show the score cards and a link to rate.
- **Migrating from an older store:** import idempotently, give old voter ids their own namespace, and let the newer vote win.
- **When ratings for a session appear: decide with the maintainer.** One option: accept votes at any time, but show the averages for a course run (course, lecturer, session) only after that session's exams and grades are out.
  → Students who are still being graded don't fear retaliation, a lecturer can't react to scores mid-course, and the school has one less reason to object.
  The cost is a delay and fewer early votes. If you choose it, say on the form when the scores will appear.

## 3. Lecturer pages: a directory only

- **One page per lecturer,** listing the courses and sessions they taught, linked to the course pages.
- **No comment thread, no written review and no score on the page.** Scores appear only per (course, lecturer), inside course reviews.
- **No wording anywhere invites students to write about lecturers.**
- **Give lecturers a contact route** to correct or remove their record.
- **The same policy holds in every feature:** the chatbot doesn't judge lecturers as people, and moderation holds complaints about a named person for a human.

→ Why:
- named staff are the easiest target for defamation and harassment;
- moderating after publication can't keep up;
- a student site that depends on faculty goodwill can't host quotable attacks.

Averages from at least 3 voters give the useful signal with nothing quotable. A first design with open comment threads was reversed the same day.

## 4. Discussion boards

- **Boards come from a config file:** key, name, icon, and a hint for the model.
  - Default boards: freshers, courses and study, research and internships, campus life, material requests, other.
  - The service holds the same keys, and the site build fails when the two lists differ.
- **One question per thread,** with search and sorting by latest reply, newest or unanswered.
- **Where threads appear:**
  - course pages list their own threads and offer "ask about this course";
  - the home page shows the latest 5;
  - material requests appear on the wanted page, grouped by course.
- **Posting:**
  - no login; the nickname is optional (blank shows as "anonymous student");
  - limits: title 4–80 characters, body 10–4,000, reply 2–2,000, nickname up to 30;
  - an unknown course code is dropped, not treated as an error.
- **Daily caps:** 60 items per network and 20 per browser (threads, replies and blog posts together).
- **Order of work:**
  1. check the caps;
  2. reserve the model's dollars;
  3. run the keyword lists (section 8);
  4. ask a small model for structured {publish | hold | reject, board, course, a reason in each site language}.

  Publish goes live; hold waits for the admin; reject goes back to the poster with a reason saying what to change.
- **A "hide"-class keyword hit is never published at once,** even if the model says publish.
- **The poster's own board and course choice win;** the model fills only blanks. A reply inherits its thread's board.
- **Reports:** reports from 3 different browsers move a live post to pending (hidden). A browser counts once per post and files at most 10 reports a day. Approving a post resets its report count.
- **Withdrawing:** only the browser that wrote a post can withdraw it. Withdrawing is final and deletes the post's images.
- **The thread author's replies** carry an "OP" mark.
- **Academic integrity.** Posts must not carry answers or worked solutions to assessments that are still running: current assignments, take-home exams, lab reports being marked.
  - Say so in the posting rules.
  - Tell the moderation model to hold anything that looks like a solution to a current task, or a request for one ("anyone done Q3 of assignment 2?" with answers).
  - Give the report button a reason "answers to a current assessment".
  - Discussing concepts, past papers and study approaches is fine.

  → This is the content most likely to make a school act against the site, and it breaks the same rule the archive keeps by refusing answer keys.
- **Appeals:** every reject reason says what to change. A held or rejected post shows a route to contact a maintainer (the same public contact as takedowns), so a wrong decision can be undone.
- **When replacing an older platform,** don't import old posts. Link them as a read-only archive.

## 5. Opportunities board

- **A card wall filtered by category:** research, internship, full-time job, referral code, competitions and events, study and scholarships.
- **Fields:**
  - organisation, location;
  - deadline (YYYY-MM-DD);
  - link (http/https only);
  - referral code, with a copy button;
  - contact, pay and mode.

  A referral needs a code or a link. A post whose deadline has passed moves to "closed".
- **Limits:** title 4–80 characters, body 20–4,000. A human check on every post. Daily caps: 20 per network and 5 per browser.
- **Model rules:**
  - **Reject:** scams (a fee or deposit to apply, order brushing, pyramid schemes, pay too good to be true), homework selling, personal data and spam.
  - **Hold:** the model is unsure, a post names a private person negatively, makes claims about a lecturer's conduct, carries an odd link, or may be a duplicate.
  - **Fine:** HR contact details and the poster's own contact.
- **Reports:** three reports from different browsers hide a post. A poster asks a maintainer to remove their own post.
- **Maintainer-curated entries** live in a repository markdown file. They render as pinned notes in plain HTML even without the service, with a getting-started guide under the wall.
- **Check caps and reserve dollars before the model call,** like the discussion boards. (gap) A board that called the model first, with only count caps, could be made to spend.

## 6. Blog with images

- **Blog posts are a third kind of item in the discussion store,** so they share one moderation path and one admin queue.
  - Categories: research, internship, job hunting and interviews, graduate school, competitions, student life.
  - Title up to 100 characters, body 50–20,000.
- **Format:** a small Markdown subset: subheadings, bold, italic, code, lists, quotes, http(s) links, and standalone image lines (`![alt](img:<id>)`).
  - Draw it element by element; never accept raw HTML.
  - The editor has a toolbar, a preview, counters, and a draft saved in localStorage.
- **Shrink images in the browser** to a long side of 1,600 px, WebP (JPEG where WebP is missing), about 400 KB.
  → This saves storage and strips EXIF, including GPS.
- **Server-side image rules:**
  - accept raw bytes up to 1 MB, typed by their magic bytes, never the declared type;
  - JPG, PNG and WebP only;
  - daily caps of 20 per browser and 100 per network;
  - cap the whole image store (for example 2 GB) and stop uploads when full.
- **Image lifecycle:**
  - unattached images are deleted after a day;
  - only the uploading browser may use an image, once, with at most 6 per post;
  - public images are served only while their post is live, cached a day, with `nosniff` and `Content-Security-Policy: default-src 'none'`;
  - removing or withdrawing a post deletes its images.
- **The model sees the images before the text.** It holds or rejects readable IDs, transcripts, offer letters, exam answers, chat screenshots with names or numbers, and sexual or violent content. Each image adds a small dollar reservation.

## 7. Helper chatbot

- **A floating button,** rendered only when the service URL and site key are configured and the kill switch is off.
- **Two stages:**
  1. **A small model is the scope gate:** structured {in_scope, reply} over the last 3 turns. It fails closed with a fixed off-topic reply, and the big model is never called, so off-topic questions cost almost nothing.
     - **In scope:** materials, courses, reviews, course choice, study methods, explaining a concept, using the site, campus life.
     - **Out of scope:** general chat, unrelated coding, news, politics, other schools, writing essays or homework.
  2. **The answering model has read-only tools only:** search the site index (courses, files, AI topics, lecturers) and course info (sessions, file types, review averages with 3+ voters). No web access.
- **Rules:**
  - never solve assignment, tutorial, lab or exam questions: explain the concept, point to material, and leave the solving to the student;
  - never judge lecturers;
  - never invent links, files or numbers: call a tool first;
  - about 150 words, in the student's language.
- **Bounds:**
  - 500 characters per question;
  - the last 6 turns;
  - 4 tool rounds, the last with tools off so it must answer;
  - 1,500 output tokens, low effort, and a cached system prompt.

  A refusal or running out of rounds gets a fixed reply.
- **Filter links twice:** the server keeps only the site's own links, and the client escapes everything and links only those hosts.
- **Caps:** for example 15 questions per visitor and 150 for the whole site per day, plus $2/day and $60/month.
  - Each question reserves $0.25 and settles at the real cost, priced by the model that answered.
  - Key the per-visitor cap on the browser or session, with the IP only as a looser cap. (gap) Keying it on IP makes a whole campus share 15 questions.
- **Privacy:** the conversation lives only in the browser's sessionStorage. The server stores counts only.
- **Placement:** run the service near the model API (for example smart placement) for latency, and to avoid regions the provider doesn't serve.

## 8. Moderation

- **Keyword lists are plain text files in the repository:** one entry per line, `#` for comments, edited by PR with no code change.
  - The service fetches them at run time with a 10-minute cache and keeps the last good copy if a fetch fails.
  - On a cold start with no copy, the model still checks, and a failure still holds.
- **List classes:**
  - **hide:** profanity in each site language, slurs, personal attacks, and the political topics the school wants kept off the site. A hit means hold.
  - **review:** complaints, accusations, legal words. These may be legitimate, so they are only logged, or passed to the model as hints.
  - **suspect:** broad terms and mild insults, acted on only with the model's opinion.
  - **allow:** innocent phrases removed from the text before matching.
- **Matching:**
  - normalise with NFKC, lowercase, and map full-width characters to half-width;
  - CJK words match as substrings after removing spaces and punctuation, which catches spaced-out spellings;
  - English words match on word boundaries ("assess" is not "ass");
  - scan the title, the body and the nickname;
  - share one rule set between the service and any repository bot, kept equal by a test.
- **The model prompt** holds the rules and what is allowed:
  - honest course criticism without abuse;
  - asking for or offering papers;
  - informal posts;
  - a flagged word used innocently.

  Keyword hits go in as "hints, not verdicts". The post, and any text inside its images, is wrapped in tags as untrusted data. The reason given must never repeat personal data or slurs.
- **Budget:** a small pool (for example $0.50/day, $5/month), reserving about $0.002 per check plus $0.001 per image, with a count cap (for example 400 checks/day). Past any cap, posts are still accepted and all of them wait for the maintainer.
- **A repository bot for issues and comments** folds or locks on a list hit and logs to a pinned issue with links instead of quotes. It skips maintainers and bots, asks the model only on a keyword hit, and never deletes anything.

## 9. When the service is off

- **The chatbot** isn't rendered unless it is configured and unpaused.
- **Reviews** fall back to the last static ratings file; if that fails too, only the ratings box shows an error. The rating form says "can't reach the service" and resets the check.
- **Discussion and the blog** show a no-script notice, or "posts not available right now" when a fetch fails.
- **The opportunities wall** keeps its curated notes in the HTML, so it is never empty.
- **Paused or over budget:** posts are still accepted and held. Votes keep working, since they need no model.
- **Never default the build to a production service URL.** (gap) A fork otherwise posts to the original site's service. Leave the URL unset until configured.

## 10. Deploy and secrets

- **When to deploy:** on a push to main that touches the service, the price table or the workflow, and by hand.
  - The workflow does nothing until the cloud secrets exist.
  - The service's tests must pass first.
  - Pin the deploy tool's version, run one deploy at a time, and time out after 10 minutes.
  - Pass the kill switch and dollar caps as deploy variables from repository variables.
- **Secrets:** the cloud API token and account id, the human-check secret, the model API key, the admin token, and a dedicated hash salt. Send them with the platform's secret command on stdin.
- **Variables:** the human-check site key, the service URL, the pause flag, and per-pool dollar caps.
- **Setup order:**
  1. the cloud account and subdomain;
  2. the human-check widget;
  3. an API token from the "edit Workers" template;
  4. secrets and variables;
  5. deploy;
  6. put the printed URL into a variable;
  7. rebuild the site.

  For local development, use the human-check provider's always-pass test keys.
- **Storage migrations are append-only,** one tag per new class. Add columns at start-up with `PRAGMA table_info` + `ALTER TABLE ADD COLUMN`. Use one named instance per feature: counters stay consistent and the code stays simple, which is plenty at student scale.

## 11. Things not to build

- **Comments embedded from the code host's discussions:** they need an account and are moderated only after publishing.
- **Reviews through an issue form:** it needs an account, stays public until deleted, and watchers' notification emails keep the text anyway.
- **Written reviews of lecturers,** or lecturer score pages.
- **One-vote-per-IP rules,** or "same IP" suspicion flags.

## 12. Gotchas

- **Some edge networks answer Python's default `urllib` User-Agent with a 403** (error 1010), so maintenance bots couldn't reach the service. Send a named User-Agent from every script through one shared helper. Run the bots right after the first deploy, because this showed up only after go-live.
- **Roll daily limiters forward only,** or a late request dated yesterday resets today's counts.
- **Deleting a day's keys can mean hundreds at once,** while some stores delete at most 128 keys per call. Delete in batches.
- **Key lists duplicated between the service and the site drift** unless a build check compares them.
- **Images can live as base64 in SQLite rows:** 1 MB raw is about 1.33 MB, under a 2 MB row limit. Free plans give a few GB and around 100k requests and row writes a day; past those, calls fail instead of billing. Keep a monthly limit in the model provider's console as the last guard.
