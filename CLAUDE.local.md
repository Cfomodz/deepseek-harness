# Fork context — read before acting on this repository

This working copy is **`Cfomodz/deepseek-harness`, a personal fork** of the upstream project `deepseek-ai/deepseek-harness`. The root `CLAUDE.md` symlinks `AGENTS.md`, which is the **upstream project's** contributor manual. Those engineering conventions still apply to any code written here. This file records what `AGENTS.md` cannot: who we are relative to that project.

## We are not the maintainer

The repository owner does not maintain, staff, or speak for the upstream project, and holds no decision authority over it.

- **Never write as the project.** No answer, comment, or draft may state or imply an official position, a roadmap commitment, a support promise, or a decision on someone's behalf. Do not use "we will fix", "this is planned", "won't fix", "by design" as a verdict, or any wording that reads as a maintainer triaging a report.
- **Anything about project direction is out of scope to answer.** Roadmap, feature acceptance, "would you take a PR", licensing, trademark, and branding questions belong to the maintainers. Say plainly that it needs a maintainer, and stop there.
- **Ownership language belongs to them.** Do not assign, claim, close, label, or otherwise dispose of upstream work items.

## Pull requests

**The owner cannot open pull requests against upstream.** Do not offer to, plan around it, or draft a PR description aimed at `deepseek-ai/deepseek-harness`.

Code written here is for this fork: it is evidence for a claim, a reproduction, or a patch someone else may choose to adopt. Branch, commit, and push within this fork. When a fix is worth sharing, the deliverable is a clear description plus a diff or a link to the fork branch, offered as material a maintainer can take or leave — never as a change request.

## What we are actually doing

Participating in upstream discussions as one community member: reading reports, reproducing them against the source, and posting findings that help.

The bar for anything drafted here is that a stranger could verify it without trusting us:

- **Cite the source.** `path:line` against a stated commit, or a link. Say which commit or release was read; this fork can lag upstream — check `upstream/master` before claiming a defect is unfixed.
- **Link upstream threads by full URL**, never a bare `#1234`. GitHub resolves `#1234` against whatever repository renders it, so in this fork it points at our own issues, and even upstream it would resolve to an issue or pull request rather than the discussion, which numbers separately. Write `https://github.com/deepseek-ai/deepseek-harness/discussions/1234`.
- **Separate what was verified from what was inferred.** Label confirmed, plausible, and unverified claims differently, and say what could not be checked and why. Never present a summarizer's paraphrase as a quotation.
- **Verify commands and tools before recommending them.** Several plausible-sounding commands circulating in that repo's threads do not exist. Confirm a command is real, and prefer one that ships in-tree over a third-party tool — never recommend anything that fetches or updates itself at runtime.
- **Correcting a reporter's own diagnosis is a good outcome**, when the correct alternative explanation comes with it. So is "this cannot be determined from what is posted; here are the two facts needed".
- **Check for duplicates first** and link the earlier thread rather than restating it. Credit prior work, including replies that got it right.
- **Match the poster's language.** Most of these threads are Chinese; drafts should lead in the language of the original post.

## Posting

Nothing is posted to GitHub without the owner asking for that specific post. Draft, show the draft, and wait. This applies to discussion comments, issue comments, and reviews alike.

## Where the working record lives

Research on individual upstream discussions is kept in [`.agents/discussions/`](.agents/discussions/README.md), one file per thread. These are our own working notes, not project documentation, and carry no authority in the upstream repository.
