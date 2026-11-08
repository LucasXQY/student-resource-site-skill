# The website: generator, UI rules and every page

A static site generated from the repository on every push to main. It is built for a student on a phone looking for one course's files in exam week. Each requirement: the rule, then why. Community widgets (reviews, discussion, opportunities, blog, chatbot) are specified in `community.md`; this file covers how pages mount them.

## Contents
1. Generator architecture
2. Rules for every page
3. Header and navigation
4. Theme and motion
5. Home page
6. Programme pages
7. Course page
8. Upload helper page
9. Search
10. Wanted page
11. Recently added and feed
12. Changelog
13. Lecturer directory
14. Community mount points
15. Other pages, redirects, SEO
16. Per-session downloads
17. Deployment and repository rules
18. Gotchas

## 1. Generator architecture

- **Keep it plain:** one script (standard library plus a Markdown library with tables, fenced code and TOC), one HTML template filled by placeholder substitution, one stylesheet, and small vanilla JS files. No framework, bundler or Node.
  → A volunteer maintainer can read and fix it; there is nothing to upgrade.
- **Options:** it runs the same way locally and in CI, with options for the output folder, the URL prefix, and the root to build from. The upload bot builds a temporary tree with it to check a PR.
- **Data flow:**
  1. scan folders into course and file records;
  2. parse the READMEs: lecturers, contributors, timelines;
  3. take first-added dates from git;
  4. read the optional AI topic index;
  5. render the pages, `search.json`, `feed.xml`, `sitemap.xml`, `robots.txt`, redirect stubs and the 404 page;
  6. copy static files;
  7. run the self-check.
- **Rewrite links in repository markdown.** Without this, every relative link dies on the site.
  - A `.md` with its own page → that page. A `.md` folded into another page → that page's section.
  - A course folder → the course page. A session or type folder → the course page's `#<session>` anchor, or a repository link plus a WARNING when that session has no indexed files.
  - Any other repository file → the host's file URL. A missing target → the host URL plus a WARNING.
  - Leave `http(s)`, `mailto` and `#` links alone.
  - Decode percent-encoded hrefs first, so they end up encoded only once.
  - Use host-style heading slugs, so table-of-contents anchors survive.
- **Wide tables get a horizontally scrolling wrapper;** never truncate them.
- **Clean READMEs up at render time, leaving the repository untouched.** Drop:
  - the template's instruction line;
  - placeholder rows (`[Name]`, `YYYY`);
  - empty "Tips:" bullets;
  - folder links inside notes, and links in session cells;
  - columns that are empty in every row.

  Programme pages are generated rather than rendering their README, so its contributor instructions stay on the code host.
- **The self-check fails the build when:**
  - course pages ≠ course folders;
  - the index file count ≠ the scan count;
  - any indexed path is missing;
  - any internal href does not resolve to an output file;
  - keys shared with the backend (board and blog categories) don't match;
  - forbidden words appear in the generator's own sources. Content is not scanned, since papers may carry any name.
- **WARNING only:** a missing course README (the page is still built, without a lecturer), unindexed files, broken content links, no git history.
- **Refuse to wipe an output folder that lacks a previous build's marker file.** → A wrong output path cannot delete real data.

## 2. Rules for every page

- `<title>` as "Page · Site", plus a meta description, viewport, `lang`, a canonical URL (built from one site-URL constant) and an Atom alternate link.
- **Footer:**
  - a one-line disclaimer with the takedown contact;
  - links to the disclaimer page, recently added, the feed, the repository and contact;
  - the analytics note.

  The disclaimer appears in three places: its own page, every footer, and the posting rules of any discussion form.
- **Third-party scripts:**
  - analytics only if cookieless, free of personal data and disclosed in the footer;
  - otherwise only the bot-check widget on posting forms.
- **Static asset URLs carry a content hash** (`?v=<8 hex>`). → Hosts let browsers cache static files for minutes, and a new widget once rendered unstyled against the old CSS.
- **If students read two languages, put both in the same element:** primary, then secondary (`A | B`, or a line break with a smaller second line). No language switcher. Both languages are always edited together.
- **Empty states always say what to do next.** "⬜ wanted, upload it" beats a blank table, and "No results" beats silence.
- **Draw all user-generated text with `textContent` / `createElement`, never `innerHTML`.** A test asserts this.

## 3. Header and navigation

- **One block per destination:** an icon plus a two-line label (primary language over secondary).
  - Order the blocks by importance, and fit them in one row at the main content width (about 960 px); tighten the blocks rather than wrap.
  - Example order: Courses, Discussion, Reviews, Opportunities, About, Repository, Contribute, Upload.
  - Upload goes last, with an accent outline, because it is the call to action.
- **Group pages to save blocks.**
  - "Courses" opens a sub-row (programmes, shared courses, overview). Esc or an outside click closes it, and without JS it is a link to the overview page.
  - Lecturers live under Reviews, contributors under About, and the blog under Discussion.
- **Mark the current page** with the longest matching href prefix (`aria-current`). Course and programme pages light up Courses; the blog lights up Discussion.
- **Top right:** a mini search box (submitting to `/?q=`) and the theme toggle.
- **The chat button floats in a corner,** outside the nav. → The nav stays on one row.
- **Below 600 px:** blocks in a 2-column grid, with search at full width.
- **Re-declare `[hidden]{display:none}`.** → A mobile `display:grid` rule once overrode the `hidden` attribute on the sub-row.

## 4. Theme and motion

- **Light/dark:** follow the system by default.
  - The toggle stores `light` or `dark` in localStorage.
  - An inline `<head>` script sets `data-theme` before first paint, so there is no flash.
  - Write the CSS for both `:root:not([data-theme=light])` inside the dark media query and `:root[data-theme=dark]`.
  - Wrap every storage access in try/catch, because private browsing can throw.
- **Hover lift and a staggered fade-in** on the nav blocks, both turned off under `prefers-reduced-motion`.

## 5. Home page

Keep it short; a test asserts it has no long "more links" list. In order:

1. **The search box.**
2. **A stats line:** courses with material, files, "all free".
3. **A news strip:** the first 3 items of the newest release in the changelog.
4. **Programme cards:** "N of M courses with material · K files".
5. **Chips for links the header lacks:** the wanted list, and rate a course.
6. **The latest opportunities next to the latest discussion posts.**

## 6. Programme pages

Generated; they do not render the README.

- **A one-line intro** pointing to the shared courses, or "nothing yet, uploads welcome".
- **"Recently added":** the 5 newest groups for this programme.
- **A 4-column table:** code, then the course name (the only link), then sessions, then file count. An empty course shows "⬜ wanted".
- **The contributors' timelines,** folded in `<details>`, with one link per row.
- **Every course folder gets a page,** even when empty.
- **The shared-courses page** has two tables (electives, compulsory) and no timeline.

## 7. Course page

Order: title → chips (programme, repository folder) → code-change notice → reviews → files → details → discussion.

- **Code-change notice at the top** when the course was renamed, linking the other code.
- **Files:**
  - one heading per session, newest first, `unknown` last, each with an `id` for anchors, lecturer links, and a "Download all" button for the session zip;
  - under it, per type, "Type (n)", then each file's name, its size, and Preview (PDF only) and Download buttons;
  - an empty course shows "⬜ wanted" with an upload link.
- **Details:** "Taken in" (from the timelines), then the cleaned README.
- **Discussion:** the posting notice (no answers, no personal data, no private contacts), then this course's threads.
- **Reviews:** the course's score card(s), and a link to rate it (see `community.md`).

## 8. Upload helper page

→ Why it exists: issue forms cannot suggest values, so the site asks first, then opens the form prefilled.

- **Three steps, stated at the top:**
  1. Press the button (hints optional).
  2. On the code host's form, pick the programme, drop the files (many files, zips and mixed courses are all fine), tick the checklist and submit. An account is needed.
  3. A bot PR comes back within minutes, and the site updates once a maintainer merges it.
- **Nothing is required.** Hints sit in a collapsed "optional" section:
  - **Course code,** from a datalist of known codes.
    - On a match: "✓ existing course: name · programme", the name field fills in and hides, and this course's lecturers move to the top of the lecturer list labelled "taught this course".
    - A new but valid-looking code: "a folder will be made, add its name", with a note when it belongs in the shared folder.
    - A malformed code: example shapes, without blocking.
  - **Course name.**
  - **Lecturer** (blank if unsure).
  - **Session:** a year select, a term select with the school's fixed term months, and "Other" as free text.
  - **Type,** defaulting to "Not sure", in plain language.
  - **Assessment name,** shown only for past papers.
- **Do not ask for the programme here.** The code host's form asks for it, and asking twice was removed.
- **Submit:**
  - send only non-empty parameters, with a title of `[Upload] CODE session type`;
  - encode spaces as `%20`, not `+`;
  - a year without a term sends no session.
- **Odd shapes pass through as typed,** and the bot tidies them (`2024/04`, `Apr 2024`, `swe 404`). → Strict validation once bounced a real upload.
- **Show "what the bot does":** it sorts, renames, checks every page for answers and personal data, blacks out PDFs, skips duplicates, and leaves unsure files to a maintainer.
- **Then a warning:** attachments on the code host are public and the bot cannot change that, so black out IDs and names first, or email the file instead. Email is also the route for people without an account.
- **Embed course and lecturer data** as `<script type="application/json">` with `</` escaped. → No extra request, and no value can close the tag; a test injects `</script>`.
- **Link the helper** from the header (Upload), the home page, programme pages, the contributing guide and the wanted page.

## 9. Search

- **Fetch `search.json` on first input.** It holds:
  - courses: code, name, URL, lecturers, sessions, file count;
  - lecturers;
  - files: code, course, session, type, name, path, preview and download links, and optional AI topics and summary.
- **Matching:** case-insensitive substring, with a 150 ms debounce. Show up to 20 courses, then 50 files with their buttons and the matched topics. No library.
- **Deep links:** keep `?q=` in sync with `replaceState`. The empty state says "No results".
- **Full-text PDF search is a non-goal.** An AI topic index per file (cheap model, cached by hash) fills the gap, so a query like "Fourier transform" finds the right files.

## 10. Wanted page

- **Per programme:**
  - courses with no material, with the sessions in which someone took them;
  - courses with no past papers;
  - sessions someone took that have no material.
- **Material requests** from the discussion board.
- **An upload link.**

→ Turns gaps into a to-do list for contributors.

## 11. Recently added and feed

- **Take the first-added date per file from git history, following renames** (`git log --name-status -M --diff-filter=AR`). → A folder move must not make old files look new. This needs a full-history checkout in CI.
- **Group by (date, course, session, type),** with "from" taken from the README's contributor cell.
- **Show** 100 groups on the page, 50 in the Atom feed and 5 on programme pages.
- **No git history:** one WARNING and an empty list with a message.

## 12. Changelog

- **Format:** `## vN · YYYY-MM-DD` headings with one-line items, in both languages if bilingual.
- **Write from the student's side** ("you can now…") and as briefly as possible.
- **Its own page lists every release;** the home page shows the newest release's first 3 items.
- **Add an entry for every release users would notice.**

## 13. Lecturer directory

- **One page per lecturer** named in a course README, listing their courses and sessions and linking to each course's reviews.
- **No comments, written reviews or per-lecturer scores.** → A first version with comment threads was reversed; written reviews of people are where defamation and harassment come from.
- **Splitting name cells:** split on `, ; 、 &` and " and ", but never on `/`, because some honorifics contain a slash ("A/P").
- **Slugs drop honorifics.** Rating widgets link only names that have a page.

## 14. Community mount points

- **Server side:** render an empty container with data attributes (backend URL, mode, course) plus embedded JSON config. The JS fills it in.
- **Fallbacks:** curated content (for example maintainer-picked opportunities) renders without JS; the rest show a `<noscript>` line.
- **When the backend is not configured,** hide the widgets and the chat button rather than showing broken forms.

## 15. Other pages, redirects, SEO

- **About:** the root README, with its contributors section replaced by the rendered contributors list (headings demoted, template rows dropped).
- **The contributing page** opens with an "easiest: use the upload page" banner.
- **404:** "the folder may have been renamed, try search", with a search box.
- **Moved pages** get meta-refresh stubs with `noindex` and a canonical link to the new address. This covers old course URLs after a folder move too.
- **The sitemap** lists real pages only. **robots.txt** allows everything and names the sitemap.

## 16. Per-session downloads

- **A workflow zips each `<course>/<session>/`** as `<CODE>_<session>.zip` and uploads it to one fixed release, replacing old copies. It runs on content changes and on demand.
- **Course pages link to the release download URL.** Zips stay out of git and out of the site artifact, at no cost.
- **Delete zips for renamed or emptied sessions too.** → An add-only workflow leaves stale downloads.

## 17. Deployment and repository rules

- **Pages workflow triggers:** a push to main (deploy), PRs (build only, so WARNINGs show before merge), and manual runs.
  - **Steps:** full-history checkout; install the Markdown library; fetch the topic index on non-PR runs, skipping it on failure; pass optional widget config as repository variables; upload the artifact; deploy on main only.
- **Concurrency:** one group per PR. Main deploys share a group with cancel-in-progress, so a newer build supersedes the older.
  → One shared group once let PR builds cancel deploys.
- **CI runs the build's self-check.** Unit tests must not pin content state.
  → Tests that asserted things like "no stragglers" once blocked deploys over content. Derive expected counts from the repository.
- **A weekly link check:**
  - build to a temp folder, then check internal links, zips that pages link to but the release lacks, and external links (with a thread pool and a timeout);
  - skip links checked elsewhere, plus an allowlist of sites that refuse bots;
  - keep one labelled "Broken links" issue open and update it, and close it with a comment when everything passes.
- **Every job has a timeout.** A pause variable skips the bot workflows while deploys continue.
- **Branch protection on main:** PRs only (admins included), and no force-push or deletion.
- **Merge upload PRs with a merge commit.** The bot commits as the uploader; a squash would make the merger the author.
- **Commits and PRs made with the workflow's own token trigger no other workflows.** The upload bot builds the site itself, and any bot that changes something the site depends on must dispatch the Pages workflow itself.
- **Choose the final address before sharing the URL.** An organisation-named Pages repository serves at the root. Moving from a sub-path did not redirect the old Pages URL, only the old repository URL. Keep the repository slug, the site URL and the default prefix as single constants.

## 18. Gotchas

- A strict upload helper bounced a real upload over its session format. Accept loosely, and normalise on the server.
- A cached stylesheet left a new widget unstyled. Content-hash query strings on every asset fixed it.
- A test file's script runner sat above its last tests, so running the file as a script skipped them. Keep the runner at the very end, or use pytest.
- Splitting lecturer names on "/" broke names containing "A/P".
- Embedded JSON must escape `</`.
- Design docs written in dated phases go stale. Mark superseded decisions inline ("replaced by …").
- Test the upload helper's URL building with a small JS harness and a fake DOM; skip it when node is missing.
