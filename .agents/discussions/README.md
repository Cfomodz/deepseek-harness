# Upstream discussion research

Working notes on individual `deepseek-ai/deepseek-harness` discussion threads, one file per thread, named by discussion number.

These are **fork-local research notes, not project documentation.** This repository is a personal fork; its owner does not maintain the upstream project and speaks for no one there. Nothing here is an official position, and a draft reply in one of these files is a proposal for the fork owner to review — not something to post. See [`CLAUDE.local.md`](../../CLAUDE.local.md) for the full stance.

## What a file contains

Each report follows one structure: verdict, what was reported, research findings with `path:line` or URL citations, root cause, ordered next steps, a post-ready draft reply, and an explicit list of what could not be verified. Every claim is labelled CONFIRMED (checked against source in this checkout), PLAUSIBLE, REFUTED, or INSUFFICIENT-DATA.

Findings cite the commit they were read at. This fork can lag upstream, and a line number that was right at one commit is wrong at the next — re-check before relying on one.

## Index

Researched at `47f943859` (0.1.0-rc.5 tree; most reporters were on rc.5 or rc.6).

| Thread | Subject | Verdict |
|---|---|---|
| [2470](2470.md) | Subagent inherits the creation-seed route, billing the wrong provider | Confirmed; fixed on this fork in `fef958af1` |
| [2402](2402.md) | `pnpm install` loops on macOS exFAT via the lefthook lock's inode check | Confirmed defect |
| [2379](2379.md) | Fixed `maxTokens` overflows the context window by 91 tokens | Documented decision + a real unreachable-compaction-threshold defect |
| [2422](2422.md) | Read-only sandbox writes files; mode cannot be switched mid-session | Confirmed, but a preset composition defect rather than a sandbox escape |
| [2359](2359.md) | `corrupt session log: seq gap in committed region` | Recoverable; overlap from concurrent writers, not a hole |
| [2286](2286.md) | Mise/Aube cannot resolve the vendored Cordis peer cycle | Resolver limitation meeting a legal-but-hostile graph |
| [2385](2385.md) | `duplicate loader entry id: mcp-argo` persists across restarts | Answerable; config layers merge without id de-duplication |
| [2463](2463.md) | Web boot hangs on a plugin pending `remote.scriptLibrary` | Not a bug; fail-loud by design, plus a discoverability gap |
| [2392](2392.md) | `single slot 'details' already has a registration at priority 0` | Insufficient data on the trigger; invariant and fix class confirmed |
| [2363](2363.md) | A submitted API key is silently dropped | Reporter's mechanism refuted; launch-environment shadowing is the likely cause |

## Method

Three habits earned their keep and are worth repeating.

**Verify replies rather than repeating them.** Four commands recommended in these threads do not exist: `dsh-doctor`, `npx dsh-plugin-doctor`, `npx dsh-shelf rescue`, and `dsh plugin ls` as written. One was a third-party package, published a day earlier by the replier, that fetched its checks over the network at runtime.

**Treat rendered-page text as a paraphrase.** The GitHub Discussions API is not reachable from this environment, so these reports come from fetched pages summarized by a model. That mis-mapped a title to a number once already. Re-read a thread directly before quoting it.

**Check for duplicates first.** Four defects account for roughly half the threads reviewed: subagent route inheritance (2470, 1581, 1472, 2006, 2053), session-log seq gaps (2359, 2257, 1497, 2473), the read-only sandbox composition defect (2422, 523, 2226), and the context-window overflow (2379, 1930). Linking beats re-answering.
