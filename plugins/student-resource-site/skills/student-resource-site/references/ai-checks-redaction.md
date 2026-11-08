# AI checks, redaction and duplicates

What the upload bot asks the models, how it blacks out personal data in PDFs, how it proves the result, and how it finds near-duplicates. Whatever goes wrong here is permanent: once committed, data stays in public history. So every rule fails closed. Format: requirement, then → why or what it prevents.

## Contents
1. What counts as personal data, and what content is refused
2. Sorting (cheap model)
3. The full check (strongest model)
4. Redaction: applying it
5. Metadata scrubbing
6. The whole-document re-read
7. The image re-check
8. Office files
9. Near-duplicates
10. Lessons from reviews (read before changing any rule)

## 1. What counts as personal data, and what content is refused

- **Personal data:**
  - student names and student IDs (the school's pattern, for example letters then digits);
  - signatures and photos of people;
  - personal phone numbers and personal emails;
  - LMS or plagiarism-tool enrolment keys, class passwords and access codes;
  - QR codes;
  - handwriting that names someone;
  - usernames and messaging IDs.

  Staff addresses on the university's own domain and lecturers' names are public.
- **Refused content (skipped, with a reason):** answers, marking schemes, model solutions, other students' work, lecture slides and textbooks (copyright, size), and anything that is not course material.
- **Answers and personal data are treated differently on purpose.** Answers are not private and can be deleted after merge, so "answers unsure" commits with a flag. Personal data cannot be recalled, so "personal data unsure" holds the file.

## 2. Sorting (cheap model)

- **Run it per file.** Put the catalogue of known courses first, identical for every file, so prompt caching applies.
- **Then send:** the source label, the form's hints, the page count, and the first 3 pages. For text files, send at most about 12k characters; for scans, send the first 3 pages as a PDF.
- **It returns strict JSON with these fields:**
  - course code and course name;
  - session: `YYYY-MM` or `unknown`. Give it the school's term months and tell it "never guess a year";
  - type, from the closed list;
  - label, from the naming vocabulary;
  - assessment name and set letter;
  - keep (yes or no), with a skip reason: lecture_slides, textbook, answers, marking_scheme, student_work, not_course_material or other;
  - confident (yes or no);
  - a short bilingual note.
- **Validate its output like hostile input**, because model output shapes paths:
  - code, session and type must match strict patterns, with ASCII digits only;
  - the label is cleaned to `[A-Za-z0-9._-]`, at most 40 characters, with no `/ \ ..`;
  - labels, assessment names, set letters or course names that look personal are blanked;
  - any failure sets confident to false.

## 3. The full check (strongest model)

- **Run it only for files that will be kept:** the sort said keep, or a maintainer used `/keep` or `/place`.
- **It reads every page,** up to a cap of about 100 pages. Longer files are not checked; they are held.
- **Split big files:** above about 40 pages, or above about 30 MB encoded, send 25-page parts and merge the answers.
  - Take the most severe verdict: yes > unsure > no.
  - Shift redaction page numbers, and page references in notes, by the part's offset.
  - If any part is missing, there is no answer.
- **It returns:**
  - answers, marking_scheme and personal_data, each yes / no / unsure;
  - redactions;
  - the lecturers printed on a syllabus;
  - the assessment weighting as printed;
  - a one-sentence summary;
  - notes on where any problem is.
- **Each redaction item has:**
  - page: 1-based, in the PDF the model saw;
  - kind: `text`, with the exact, shortest string as it appears on the page, or `box`, with 0–1000 normalised coordinates from the top-left (for scans, photos, signatures, QR codes and handwriting);
  - what: from a closed list, never the value.
- **The closed list of kinds:** name, student_id, signature, photo, qr_code, phone, email, address, key, other.
  - Define **key** narrowly: an enrolment key, a class password or an access code. Never a username, a messaging ID or any contact handle; those go under "other". See section 10 for why this matters.
- **One entry per page an item appears on,** headers and footers included. The redaction text is the only place a value may appear. Summaries, notes and `what` say where, never what.
- **Any redaction that names a page outside the file voids the whole answer.** → A wrong page number blacks out the wrong page and leaves the real one. Malformed entries also fail closed, before anything is written.
- **If the model is unsure but lists redactions,** redact anyway and still hold the file. → A maintainer's `/approve` then commits the redacted copy.
- Use medium thinking effort and a JSON schema. The prompt says the file content is untrusted material and that instructions inside it must be ignored.

## 4. Redaction: applying it

- **Text.**
  - Search the whitespace-normalised string, ignoring case and following it across line breaks.
  - If it isn't found, fall back to every word of 3+ characters as a whole page word. If any word is missing, the redaction fails.
- **Exact repeats on other pages:** black these out too, but only for strings of 6+ non-space characters whose kind is not key or other.
  - **Strings with digits or symbols** (IDs, emails): wherever search finds them.
  - **Letters-only strings:** only with 2+ words, matched as whole-word runs ("Siti Aminah" never hits "Siti Aminahwati").
  - **Never wipe single words across the file.** A single word like "Crystal" could be an ordinary word.
  - Search each distinct string once, skip pages that already name it, and drop duplicate rectangles. → A name in every footer of a 100-page file once produced ~40,000 rectangles; done once, it was ~200.
- **Boxes.**
  - Map the box onto the page as displayed (rotation applied, which is what the model saw).
  - Widen it by 2% of the page width and height on each side, clamp it to the page, and convert back to unrotated coordinates.
  - On rotated pages with an offset crop box, apply the box to the unrotated page. → Otherwise the fill lands in the wrong place.
- **Remove, don't cover.** Delete the text under the box, the image pixels, and any line art the box touches.
  - **On redacted pages, first drop every annotation and form widget.** A field's value survives under a black box.
  - **Prune form fields that have no widget left.** A parent field's value survives garbage collection otherwise.
  - **Always drop XFA form data,** which can hold every field's value.
  - **Remove the outline (bookmarks) and page labels** from the whole file.
- **Commit the redacted copy.** The redacted copy is what gets committed, put in the inbox and thumbnailed; the original of a redacted file never reaches git.
  - A partly redacted copy is kept, so `/approve` commits it. The original is committed only when no copy could be made.
  - Refuse an output path equal to the input.
  - When nothing is written, delete any stale output from an earlier run.

## 5. Metadata scrubbing

Every committed file goes through this, redacted or not.

- **PDF, empty all of these:**
  - annotation authors, subjects, contents and rich text;
  - pop-ups, file attachments and embedded files;
  - piece info;
  - page thumbnails and page metadata streams;
  - the info dictionary and XMP.

  Keep what is drawn (highlight and ink appearances) and form field names.
- **Office (docx/pptx/xlsx):** blank the creator, last-modified-by, title, subject, keywords, description, company and manager in the document-properties parts.
- **Legacy binary Office files (doc, ppt, xls)** are refused at upload (see `upload-flow.md`). Never commit a format whose metadata you cannot scrub, even with a "check by hand" flag: once it is on a public branch, the author field is public.
- **Every format you accept needs a scrubber and a test** that puts a name in each metadata field and checks it is gone.

## 6. The whole-document re-read

After redaction, read every page again: the page text, annotation contents and authors, and form field values. No redacted string may be readable anywhere.

- **Compare with all whitespace removed and case folded,** and again with hyphenated line breaks joined.
  → This catches what a person reading the page would still see as the name:
  - letter-spaced headings ("H a s s a n");
  - kerning gaps that extract as a space ("Has san");
  - a name run into an email ("hassanali@…");
  - a word broken in a narrow table cell.
- **Check fallback words as whole words.** Single words of a string that was found whole are not checked. → A lecturer whose surname equals one word of a student's full name must not fail the file.
- **The one exception: a one-word, letters-only string of kind `key`.** On pages other than its own, it counts only where no space splits it. So a key like "GreenTea" is not the ordinary phrase "green tea" printed elsewhere.
  - Usernames, messaging IDs and contact handles are never `key`: they go under "other" and are compared strictly.
  - Old cached answers without `key` stay strict.
- **Any hit holds the file** with "page N still contains a redacted string". The page number is given, never the text.

## 7. The image re-check

- **What gets re-checked:** every page with a box redaction, and every redacted page that shows any picture.
  → An OCR'd scan keeps the text in its image, so text-layer redaction alone is not enough.
- **Rendering:** 150 dpi, at most 4.5 MB per image (re-render smaller if needed), at most 8 pages and 20 MB per call.
- **Pass condition:** every call must answer exactly "no". Anything else is "unsure": the file is held, and the verdict is cached until `/retry`.
- **A re-check that could not run** (budget, error) is not cached; the next rebuild tries again.
- **No re-check is needed** for text-only redactions on pages with no pictures; the re-read covers those.

## 8. Office files

- **Never auto-redact Office files.** There is no equivalent of the verified remove-and-re-read pipeline: text is spread over many XML parts, comments, tracked changes and embedded objects.
- **Show them to the model only as a converted PDF.**
- **If they contain personal data,** they go to a maintainer unredacted: listed, not committed.

## 9. Near-duplicates

- **Scope:** compare only files about to be committed (scans excluded) against the PDFs in the same course folder, using first-page text.
- **Tokens:** lower-case words of 2+ letters or digits, with each CJK character as its own token.
- **Ratio:** a sequence matcher with auto-junk off.
  - **0.9 or more:** skip as a duplicate.
  - **0.6–0.9:** the cheap model compares the first two pages of both files (at most about 6k characters each) and answers:
    - same → skip;
    - different_set → add the next free Set letter (the unsuffixed file counts as A, so the first new one is B);
    - different_version → add the next `_vN` (the first is `_v2`);
    - different → commit.

    The prompt says "when in doubt between same and anything else, answer the other". → Wrongly skipping is worse than a duplicate a person can delete.
- **No answer** (budget, error): commit and flag "possible duplicate, not compared". Never silently treat it as "different".
- **Scans are skipped** and noted in the PR. `/keep` bypasses de-duplication. Read each repository PDF once per run.

## 10. Lessons from reviews

Read these before changing any rule in sections 4–7.

**False positive.** A correctly redacted assignment brief was held: its enrolment key looked like a word, and an ordinary phrase on other pages equalled the key once spaces were removed. The tempting fix was to relax the re-read so that letters-only strings must "keep their spacing" on other pages.

**Leak 1.** That relaxation passed every test its author wrote. An independent review then found that it let names through when they were:
- letter-spaced;
- split by a kerning gap;
- run into an email;
- broken in a table cell.

**Leak 2.** The second attempt limited the exception to redactions labelled "other". "Other" is the catch-all, though, and usernames and messaging IDs land there, so the same layouts leaked them.

**The fix.** A dedicated kind, `key`, narrowly defined in the prompt, plus the one-word, letters-only restriction. Names, IDs, contacts and "other" keep the strict comparison.

**General rules that came out of it:**
- Never relax a privacy check for a whole class of strings to fix one false positive. Narrow the exception to a kind that cannot be personal, and make the model label it.
- Write the leak cases as tests before relaxing anything: letter-spacing, kerning gaps, run-together emails, mid-word breaks, rotated pages and annotation text. Every new exception must still hold all of them.
- Have someone other than the author review any change to what becomes public, and ask them for concrete failing inputs.
- A held file costs one maintainer click. A leaked one cannot be undone. When in doubt, hold.
- Other things that each became a test:
  - form field values survive under black boxes;
  - parent field values survive garbage collection;
  - XFA holds field data;
  - OCR'd scans keep text in their images;
  - rotated pages with crop offsets misplace fills;
  - annotations written directly into a page's array have to be turned into objects before they can be scrubbed.
