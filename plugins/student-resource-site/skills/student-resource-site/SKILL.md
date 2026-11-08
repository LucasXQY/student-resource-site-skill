---
name: student-resource-site
description: Design and build a student-run course-resource website with a community layer - past papers, syllabi, assignment briefs, lab manuals and notes organised by course and session; an upload channel where students just drop files and a bot sorts, renames, redacts personal data and checks for duplicates; course reviews, anonymous discussion boards, an opportunities board, a blog and a helper chatbot; hosted at zero fixed cost with hard caps on AI spend. Use this skill whenever someone wants to build, plan, extend or audit a site for sharing course materials or past papers for a university, faculty, school or student club (a "survival guide", study-resource hub, past-paper archive, course wiki or course-rating page), or asks how such a site should handle its UI, upload page, contributor credit, privacy and redaction of student data, answer keys, moderation, spam, reviews or LLM costs - even if they only ask about one piece, such as "how should my upload page work" or "how do I stop people posting student IDs".
---

# Student resource site

A site where students share the materials of the courses they took (blank past papers, syllabi, assignment briefs, lab manuals, notes, tips) and talk about them. It has three audiences, and every design choice should be judged against all three:

- **Readers** look for one course's material, usually on a phone, the week before an exam. They should reach any file in two or three clicks from the home page, or one search.
- **Uploaders** have files and very little patience. Anything beyond "pick your programme, drop the files" loses uploads. They also need to trust that their name and other people's data will not leak.
- **Maintainers** are one or a few volunteer students with little time. Each upload must be reviewable on one screen, and nothing may happen behind their back that is hard to undo.

The detailed requirements for each part live in `references/`. Read the file for the part you are working on before designing or building it; each one holds checklists with the reason behind every item, because the reasons are what let you adapt a rule to a new school.

| Part | Read |
|---|---|
| Folder layout, naming, sessions, course pages, what content is allowed | `references/content-model.md` |
| The website: navigation, every page type, search, mobile, theme, copy | `references/site-ui.md` |
| The upload form, the upload helper page, the bot's steps, the review PR, maintainer commands, importing existing material | `references/upload-flow.md` |
| AI sorting and checking, redaction of personal data, the re-check, duplicates | `references/ai-checks-redaction.md` |
| Course reviews, discussion boards, opportunities board, blog, helper chatbot, moderation | `references/community.md` |
| Budgets and model choice, free hosting, secrets, kill switch, testing, launch, the incident runbook | `references/operations.md` |

## Principles

These come from building such a site end to end. When a new situation is not covered by the references, decide it with these.

1. **Privacy fails closed.** Anything the system is unsure about (a check that could not run, a model that did not answer, a redaction that could not be verified) goes to a human instead of being published. A leaked student ID cannot be taken back; a held file costs a maintainer one click.
2. **Never silently drop a contributor's file.** A file the bot cannot judge stays visible to the maintainer (an inbox folder in the review PR, a "needs you" list) with the reason. Only things that can never be accepted are dropped automatically: disallowed file types, unsafe archive paths, unreadable files, oversize submissions. Silent drops lose material and the uploader's trust.
3. **The repository is the source of truth.** Files, course tables and contributor credit live in git; the website, indexes, search and zips are generated from it. Review happens in pull requests, so every change has a diff, an author and an undo.
4. **Uploaders do the minimum; machines and maintainers do the rest.** Every form field except the one that routes the upload is optional, and hints are read leniently (any date format, a course code anywhere in the text). Never send a form back for a format error that a machine could fix or ignore.
5. **One screen per review.** The bot's report says what it did with each file, why, what it is unsure about and what the maintainer must decide, with thumbnails. Commands in comments resolve the open points without editing files by hand.
6. **Zero fixed cost, capped variable cost.** Static hosting, CI and storage on free tiers; every paid AI call is booked against a dollar budget per feature, with a reserve, daily and monthly caps and a kill switch. When the budget is gone the feature degrades (holds for a human, answers "try tomorrow") instead of failing or overspending.
7. **Judge courses, not people.** Ratings are per course (and course run), scores only, published only above a minimum count. No written reviews of lecturers or students, no lecturer score pages: that is where defamation, harassment and takedown requests come from.
8. **Label time by the session a course was taken**, such as 2024-04, never by "Year 2" or intake year. Students take the same course at different points, and course codes get renamed; sessions stay true.
9. **Ask before anything hard to undo.** Repository or organisation settings, deleting data, publishing on someone's behalf, setting secrets, merging. The maintainer owns those decisions; prepare them and let the maintainer press the button.

## How to run a project like this

Work in phases, each shipped as its own small pull request and checked live before the next. A phase that touches privacy gets an independent review before it ships (see the redaction lessons in `references/ai-checks-redaction.md`).

### Phase 0: Interview

Before designing, find out (ask only what you cannot infer):

- The institution and its structure: faculties or programmes, course-code format, the session pattern (for example February/April/September intakes), languages the students read.
- Who maintains it and how much time they have. Who else may get write access.
- What material exists and who owns it; what the school allows to be shared. Blank papers and briefs are usually fine; answers, marking schemes, model solutions and other students' work are not.
- Which accounts exist or can be created (a code host with free CI and static hosting, a serverless platform with a free tier, an AI API key with a monthly credit) and the monthly AI budget.
- Contributor credit: real names, nicknames or anonymous by default, and anyone who must never be named.
- Which features are wanted now versus later. Propose the phase order below as the default.

### Phase 1: Content model and rules

Decide the folder layout, file-naming convention, session labels, course table columns and the content policy, and write them into the contributing guide. Done when a new file's path can be derived mechanically from (programme, course code, session, type, label). Details: `references/content-model.md`.

### Phase 2: Static website

Generate the site from the repository on every push to the main branch. Done when every course, session and file is reachable from navigation and search on a phone, and the build fails on broken internal structure. Details: `references/site-ui.md`.

### Phase 3: Upload channel

An issue form (or equivalent) where uploaders pick a programme and drop files; a bot that turns each submission into one review pull request on its own branch. Seed the archive with existing material through the same pipeline before launch. Done when a real upload goes from form to merged files with no hand editing. Details: `references/upload-flow.md`.

### Phase 4: AI checks, redaction and duplicates

A cheap model sorts and names files; a strong model checks every kept page for answers and personal data; PDFs are redacted and re-checked; near-duplicates are compared. Done when a live test with synthetic personal data shows every leak path blacked out or held. Details: `references/ai-checks-redaction.md`.

### Phase 5: Community features

Course reviews, discussion boards, an opportunities board, a blog and a helper chatbot, on a serverless backend with bot checks and rate limits; the static site keeps working when the backend is off. Details: `references/community.md`.

### Phase 6: Operations and launch

Budgets, secrets, the kill switch, job timeouts, analytics, a changelog, a live end-to-end test with fake data, and a written runbook for leaked personal data and takedown requests. Details: `references/operations.md`.

## Working with the maintainer

- Protect the main branch: every change, including yours, goes branch → PR → merge. Merge with merge commits, not squash: squashing replaces an uploader's authorship with the merger's.
- The bot's PR branch is a pure function of (submission, current main). Nobody edits it by hand; to change the result, edit the submission or use a command, and the PR rebuilds.
- Keep a changelog page and add an entry for each release that users would notice.
- Before calling something done, run the test suites and, for anything users touch, check it in a browser at phone width.
- Test live with synthetic files (fake names, fake IDs) on a throwaway submission, then close it. Never test with a real student's data.
- When you show maintainers what is in an upload, describe personal data by kind and page ("a student ID on page 2"), never by value, in every public place: PR text, comments, logs, commit messages, tests and docs. Use made-up values in tests.
