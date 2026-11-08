# Content model: folders, naming, sessions, course tables, content policy

The repository is the archive, and the website is generated from it, so these rules decide everything downstream: where the upload bot puts files, what the site can index, and what readers can find. Each rule: the requirement, then why.

## Contents
1. Principles
2. Folder layout
3. File names
4. Sessions
5. Course code changes, duplicates and variants
6. Course README
7. Programme README: index and timelines
8. Contributors, disclaimer, contributing guide
9. Content policy
10. Gotchas

## 1. Principles

- **The repository is the only content source.** The website holds no content of its own. → Drift between the two (content only on the site, or the site lagging the repo) is a failure.
- **Files are not copied to the website.** Pages link to the code host's copy (its file viewer for previews, raw for downloads). → The archive grows to hundreds of MB. Raw URLs are usually served as a download, so "Preview" should open the host's viewer page.
- **Bad content never stops the site.** The build fails only when the generator is wrong. Content problems degrade gracefully and print a WARNING. → One imperfect upload must not freeze the site.
- **One rule, one definition.** Folder names, the type list, the session pattern and the timeline pattern live in one shared module, used by every script (index generator, site builder, upload bot). → Two copies of a rule drift apart.

## 2. Folder layout

`<Programme>/<CODE>_<Course_Name>/<YYYY-MM | unknown>/<Type>/<file>`

- **One top-level folder per degree programme,** each with a README holding a generated course index and hand-written timelines.
- **A shared top-level folder** for courses every programme takes (university electives, national compulsory subjects).
  - → These used to sit under one programme, where students in the others never looked.
  - A shared course has no timeline of its own. Whoever took it lists it in their own programme's timeline with a link to the shared folder, and the index merges "taken in" across all timelines.
  - Recognise shared courses by code prefix (for example elective and national-subject prefixes), so the upload bot can route them automatically.
- **One folder per course, holding every year's material.** → Per-cohort trees (`<cohort>/Year-N/Semester-N/<course>`) scatter one course across many cohorts. Flattening them was the first big restructure.
- **Course folder name:** `<CODE>_<Course_Name>`.
  - The first underscore splits code from name; other underscores show as spaces.
  - Use the English name as it appears on the syllabus.
  - Codes may contain dots and names may contain `&`, so percent-encode every URL.
  - A course with an unknown code gets a placeholder prefix. Make each placeholder unique, because several maps are keyed by code (see Gotchas).
- **A fixed list of type folders.** Example list:
  - Past-Papers
  - Syllabus
  - Assignments
  - Labs
  - Tutorials
  - Handbooks (guides, templates, rubrics)
  - Notes
  - Exam-Review
  - Tips

  Create a type folder only when it has files. Anything off-shape (`2024-9`, an extra level, an unlisted type, a loose file) stays out of the index and is reported as a WARNING by diffing tracked files against the scan.

## 3. File names

- **Past papers:** `<CODE>_<YYYYMM>_<Assessment>[_SetA]_Question.<ext>`. → `_Question` says "blank paper, no answers" in the name itself.
- **Everything else:** `<CODE>_<YYYYMM>_<Label>.<ext>`.
- **Labels come from a short vocabulary:** Syllabus, Course_Intro, Assignment1, Lab1_Manual, Tutorial1, Rubric, Report_Template, and so on. Labels are ASCII `[A-Za-z0-9._-]`, at most about 40 characters.
- **Never derive a name from the uploader's file name.** → Students put their own names and IDs in file names.

## 4. Sessions

- **The session folder `YYYY-MM` is the material's own session:** the one printed on the paper, or the term a syllabus or brief was issued in. `unknown/` is the last resort.
  → Without a session layer, uploads from different years collided.
- **Never label by intake year or "Year N".** Electives, retakes and curriculum changes put the same course in different years for different people. Sessions stay true.
- **Sort newest first, with `unknown` last.**
- **Learn the school's session pattern** (for example February / April / September terms) and use it everywhere: the helper page's term dropdown, the bot's prompt ("never guess a year"), the session pattern. If terms are not months, change the pattern, not the folder shape.
- **Contributors' timelines record the session each person took the course,** which may differ from the material's session. Both are shown: "Taken in" and "Sessions covered".

## 5. Course code changes, duplicates and variants

- **Code change:** create a new folder under the new code and never rename the old one.
  - Both READMEs get a "Code change:" bullet linking to each other.
  - The site shows that note at the top of the course page. → It tells readers at once whether they are on the right page.
- **Duplicates:** open and compare first. Identical files are skipped. Different ones get a suffix (`_SetB`, `_v2`), and the difference is noted in the course README. The upload bot automates this (see `ai-checks-redaction.md`).

## 6. Course README

- **One table, one row per session:** Session | Lecturer | Assessment | Contributed by | Notes. Then "Code change" and "Tips" bullets.
- **Add rows; never overwrite someone else's.** Lecturer and weighting change between sessions.
- **Bots fill only empty cells,** and AI-extracted values are filtered for personal data first.
- **The site reads columns by position** (lecturer, contributor) and keys them by session. The session cell may be a link.
- **Keep the template's instruction line and placeholder rows in the repo** for people editing on the code host; the site hides them at render time (see `site-ui.md`).

## 7. Programme README: index and timelines

- A short intro, then the course index between marker comments such as `<!-- COURSE-INDEX -->`, regenerated by a script.
  - **Columns:** Code | Course | Taken in | Sessions covered | Material counts. An empty course shows "wanted".
  - → Markers let the script rewrite its block without touching hand-written prose.
- **Then the timelines:** one table per contributor (`### nickname`, rows `| YYYY-MM | [CODE](folder/) | [Name](folder/) |`), plus an "Other contributors" table with a contributor column for material from people without their own table.
- **Regenerate the index after every content change.** The upload bot does this itself, and a site self-check compares the index file count with the scan.

## 8. Contributors, disclaimer, contributing guide

- **CONTRIBUTORS:**
  - grouped by intake year;
  - nickname, real name or anonymous, as each person chooses;
  - contact optional;
  - template rows kept for hand editing.

  **Anyone who asked not to be named never appears, in any file.** Anonymous uploads add no row.
- **DISCLAIMER:**
  - the site is unofficial;
  - copyright stays with the school and the lecturers;
  - non-commercial study use only;
  - a no-questions-asked takedown by email or issue;
  - blank papers only;
  - no guarantee of accuracy;
  - contributors must redact personal data;
  - no resale;
  - use at your own risk.

  Adapt it to local law.
- **CONTRIBUTING:**
  - do and don't upload lists, and a redaction how-to;
  - the naming and placement rules;
  - a "what you did → which files to edit too" table for git users;
  - three routes in: the upload form, fork + PR, and email;
  - a checklist and the credit options;
  - an "easiest: use the upload page" banner at the top.
- **Languages:** if students read two languages, write every doc in both, primary paragraph first, then secondary, and edit both together. Keep one language per sentence; no language switcher.

## 9. Content policy

- **Accepted:**
  - blank past papers;
  - syllabi and course introductions;
  - assignment briefs;
  - lab manuals;
  - tutorial sheets;
  - handbooks, rubrics and templates;
  - students' own notes and exam-review sheets (your own working on your own revision sheet is fine);
  - tips.
- **Refused:**
  - answers, marking schemes and model solutions;
  - other students' submissions;
  - lecture slides and textbooks (copyright and size; link to the official place instead);
  - anything with personal data that cannot be redacted;
  - private contacts;
  - LMS enrolment keys;
  - QR codes.
- **Redact before upload where possible:** student names and IDs, signatures, photos, personal phone numbers and emails, enrolment keys. The bot is a backstop, not the only line.
- **Takedown:** honour any request quickly and without argument.

## 10. Gotchas

- Several maps are keyed by course code alone (zip names, the helper's course list, "taken in"). Two placeholder courses with the same code, or one code in two programmes, collide. Give placeholders unique codes, and key by folder where you can.
- When the scan rules exist in two scripts, they drift, and one script's output gets frozen. Use one shared module.
- Moving folders (for example into the shared folder) needs follow-ups: redirect stubs for old URLs, rename-aware "recently added" dates, and updated path triggers in workflows that watch folders.
