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

## Plugin feedback

Peer reviews of plugins shared in the upstream `Show Your Plugins!` threads, chosen because each sits on a seam these notes already cover. Same structure and labelling, plus an explicit note on what was and was not read — plugin source is downloaded and read, never cloned, installed, or executed.

| Thread | Plugin | Headline |
|---|---|---|
| [2564](2564.md) | six-plugin suite (Wang-Lin-Chang) | Default `remind` schedules can never fire; `createdBy` is hardcoded undefined |
| [2532](2532.md) | dsh-session-sync | Each push deletes the other device's sessions; credentials in a remote URL reach the model |
| [2471](2471.md) | dsh-file-undo | The pre-write content they believe is discarded is available at `tools/post-execute` |
| [2472](2472.md) | dsh-permission-rules | Enforces at the real gate; `paths` rules silently drop out-of-workspace paths, so the baseline's `**/.ssh/**` misses `~/.ssh` |
| [2553](2553.md) | dsh-budget | One global provider/model pair means concurrent sessions cross-price each other |
| [2549](2549.md) | dsh-defend | No runtime fetching at all; `pwsh` missing from the delete guard's tool list while every rule is PowerShell |
| [2487](2487.md) | dsh-background-agents | Exposed to the subagent route defect in its restart-surviving form |
| [2492](2492.md) | dsh-checkpoint-rewind | Cleared of the corruption it looked exposed to; its own event gate is silently inactive |

### The custom-session-event trap

Five of these reviews land on one harness gap, and no single plugin author can see it from outside.

`Session.append` reads only `sourceEventSeqs` and `surfaceOp` from its options, so a plugin cannot mark its own event `ignorable`. `KNOWN_SESSION_EVENT_TYPES` is generated from in-repo sources, and `declare module` merging has no runtime effect, so an out-of-repo event type is unknown by construction. The persistence write path accepts unknown types deliberately; the read path throws on them. A plugin that appends a custom event can therefore leave a session that refuses to cold-resume.

Two authors defended against this with a probe that cannot succeed, one passes an `ignorable` flag that is discarded, and one appends unmarked events. The mechanism is confirmed line by line; the end-to-end trigger is not reproduced, so it stays PLAUSIBLE until someone does. It is worth a write-up of its own rather than a note in eight separate plugin threads.

## Method

Three habits earned their keep and are worth repeating.

**Verify replies rather than repeating them.** Four commands recommended in these threads do not exist: `dsh-doctor`, `npx dsh-plugin-doctor`, `npx dsh-shelf rescue`, and `dsh plugin ls` as written. One was a third-party package, published a day earlier by the replier, that fetched its checks over the network at runtime.

**Treat rendered-page text as a paraphrase.** The GitHub Discussions API is not reachable from this environment, so these reports come from fetched pages summarized by a model. That mis-mapped a title to a number once already. Re-read a thread directly before quoting it.

**Check for duplicates first.** Four defects account for roughly half the threads reviewed: subagent route inheritance (2470, 1581, 1472, 2006, 2053), session-log seq gaps (2359, 2257, 1497, 2473), the read-only sandbox composition defect (2422, 523, 2226), and the context-window overflow (2379, 1930). Linking beats re-answering.
