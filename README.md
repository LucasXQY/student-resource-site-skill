# Student Resource Site — a Claude skill

**English** · [中文](README.zh-CN.md)

A [Claude](https://claude.com) skill for designing and building a **student-run course-resource website with a community layer**: past papers, syllabi, assignment briefs, lab manuals and notes organised by course and session, plus an upload channel, course ratings, anonymous discussion, an opportunities board, a blog and a helper chatbot. It runs at zero fixed cost, with hard caps on AI spend.

The skill is distilled from building and running one such site end to end. Most of what is in it is a requirement plus the reason behind it, and many of those reasons are mistakes that happened once and then became a rule.

## What it gives Claude

When you ask Claude to plan, build, extend or audit a site like this, even just one piece of it ("how should my upload page work?", "how do I stop people posting student IDs?"), the skill gives it:

- **Three audiences to design for**: readers on a phone in exam week, uploaders with no patience, and one or two volunteer maintainers with little time.
- **Nine principles**, among them:
  - privacy fails closed;
  - never silently drop a contributor's file;
  - the repository is the source of truth;
  - judge courses, not people;
  - label material by the session it was taken, never by "Year 2";
  - every paid AI call sits under a dollar cap.
- **A phased plan**, from the interview questions to launch, each phase with a "done when" condition.
- **Detailed requirement checklists**, each with its reason:

| Part | Reference file |
|---|---|
| Folder layout, file naming, sessions, course tables, what content is allowed | [`content-model.md`](plugins/student-resource-site/skills/student-resource-site/references/content-model.md) |
| The website: navigation, every page type, search, mobile, light/dark theme, copy | [`site-ui.md`](plugins/student-resource-site/skills/student-resource-site/references/site-ui.md) |
| The upload form, the upload helper page, the bot's steps, the review PR, maintainer commands, importing existing material | [`upload-flow.md`](plugins/student-resource-site/skills/student-resource-site/references/upload-flow.md) |
| AI sorting and checking, PDF redaction of personal data, the re-check, duplicates, lessons from security reviews | [`ai-checks-redaction.md`](plugins/student-resource-site/skills/student-resource-site/references/ai-checks-redaction.md) |
| Course ratings, the lecturer directory, discussion boards, opportunities, blog, helper chatbot, moderation | [`community.md`](plugins/student-resource-site/skills/student-resource-site/references/community.md) |
| AI budgets and model choice, free hosting, secrets, kill switch, testing, launch, incident runbook | [`operations.md`](plugins/student-resource-site/skills/student-resource-site/references/operations.md) |

The default stack is a GitHub repository with GitHub Actions and Pages, an issue form plus a bot for uploads, a Cloudflare Worker with Durable Objects for the community features, and the Claude API for sorting, privacy checks and moderation. The principles carry over to other hosts.

## Install

### As a Claude Code plugin (recommended)

In Claude Code:

```
/plugin marketplace add LucasXQY/student-resource-site-skill
/plugin install student-resource-site@lucasxqy-skills
```

Or from a terminal:

```bash
claude plugin marketplace add LucasXQY/student-resource-site-skill
claude plugin install student-resource-site@lucasxqy-skills
```

Claude uses the skill on its own when a request fits. To call it directly, type `/student-resource-site:student-resource-site`. Update later with `claude plugin marketplace update lucasxqy-skills`.

### By hand

Copy the skill folder into your personal skills folder:

```bash
git clone https://github.com/LucasXQY/student-resource-site-skill.git
cp -r student-resource-site-skill/plugins/student-resource-site/skills/student-resource-site ~/.claude/skills/
```

### On claude.ai

Download `student-resource-site.skill` from the [latest release](https://github.com/LucasXQY/student-resource-site-skill/releases/latest) and upload it under Settings → Capabilities → Skills (your plan and organisation must allow custom skills).

## Try it

After installing, ask Claude things like:

- "We're the student union of a computer science faculty. Design a website where students share past papers, syllabi and notes and can discuss courses anonymously. Two of us maintain it, and we have no budget."
- "People email me exam PDFs with their student ID on the cover. Design an upload page and a bot that processes uploads before anything is published."
- "We want to add course ratings and a discussion board to our static resource site without getting complaints from lecturers, vote stuffing or spam. How should they work?"

## Does it help?

The skill was tested on the three prompts above. Each prompt was run by Claude once with the skill and once without, and the resulting design documents were scored against 22 checks for things a site like this gets wrong. Examples:

- The upload form needs only the files.
- Uploaders are warned that attachments are public before review.
- Redaction removes content rather than drawing boxes over it.
- Ratings are scores only, with a minimum number of voters.
- Votes are not limited by IP, because campus networks share one address.
- AI spend is capped in money.

| | With the skill | Without |
|---|---|---|
| Checks passed | 20 / 22 (92%) | 12 / 22 (54%) |

Without the skill, the designs:
- required course code and session on the upload form;
- accepted answer keys;
- published discussion posts before moderating them;
- allowed written reviews of lecturers;
- limited votes by IP;
- required school-email login.

With it, they avoided all of these. The test cases live in [`evals/`](plugins/student-resource-site/evals). This is a small sample, so treat the numbers as a sanity check, not a benchmark.

## Repository layout

```
.claude-plugin/marketplace.json          the plugin marketplace (this repo)
plugins/student-resource-site/
  .claude-plugin/plugin.json             the plugin manifest
  skills/student-resource-site/
    SKILL.md                             principles, phases, which reference to read when
    references/                          the detailed requirement checklists
  evals/                                 test cases (prompts and checks)
```

## Contributing

Issues and pull requests are welcome, especially:
- requirements from other schools' sites that turned out to matter;
- mistakes you made that should become a rule;
- adaptations for other hosts.

Keep two rules:
- Every requirement carries its reason.
- Examples never contain real personal data. Use obviously fake names and IDs.

## License

[MIT](LICENSE)
