# Agent Note: Subagents inherit the parent's live route

Status: implemented

English | [中文](2026-08-16-subagents-inherit-the-parents-live-route.zh.md)

## Problem

The harness carries two representations of "the model this agent uses". `Agent.options` is the seed declared at `agents.create()` / `agents.resume()`. The live route is the `agent/request` waterfall result, which is what dispatches and what the loop appends as `request/header`, readable at any point through `session.requestHeader()`.

Everything that moves a route at runtime operates on the second. `installModelSelection()` replaces provider, model, and effort on the resolved request config and never writes back to the options; the api-proxy resolves its selection tiers on every read as process-local pick, then the session's own logged header, then the deployment default. Under the Web host `agentOptions()` seeds every create, resume, and fork from `defaults.defaultModelSelection()`, so a session's options name the deployment default of the moment it was created and never move again.

`resolveChildAgentOptions()` read the options. The two agree only for the first request of a freshly created agent under an unchanged deployment default, so a parent that switched model in the picker delegated children onto the route it was created under, while its own turns and its own log correctly showed the switched one.

A provider id is not a display preference: it selects the registered adapter, which owns its own credential reference and base URL. A child inheriting a stale provider authenticates with a different key against a different endpoint, and the account charged is the stale one — the consequence `dsh-llm-pi-ai` already names in its refusal to fall back to ambient keys, "billing another tenant for a request the deployment meant to authenticate differently". Nothing surfaced the divergence: the parent's own header stayed correct throughout, and no error, warning, or UI signal marked the child.

The durable descriptor made it outlive the process. `startContinuable()` snapshotted `agentProvider` / `agentModel` from the same stale options, cold resume rebuilt a child's `agentOptions` from that descriptor, and grandchildren inherited from the child's now-wrong options. One wrong parent turn contaminated a whole delegation subtree across restarts.

The repo had already decided this exact question twice in this file, both times for the parent's live state. `childSessionMeta()` reads the preset from the parent's live scope chain rather than its header, "because a parent that switched preset while blank runs on the newer composition while its header still names the older one". `captureDelegatedPolicyOverrides()` reads the parent session's explicit live sandbox override and pointedly never the deployment default. Only the model route read the creation snapshot, with no note and no stated rule — most likely written from the `Agent.options` JSDoc, which claimed to be "the provider route and model this agent's requests use" and had not been true since request-level selection shipped.

## Decision

A child inherits the route its parent's most recent request actually used: `session.requestHeader()?.config` for provider and model, falling back to `parent.options` only for a parent that has never issued a request, with an explicit per-call `agentOptions` still outermost. `parentRouteOf()` in `dsh-subagent`'s `child-agent.ts` is that single resolution, so both in-process drivers get the rule from the package that owns the delegation contract rather than each Consumer restating it.

Adapter-owned `maxTokens` is not inherited. `LlmCallConfigAdapterDefaults.maxTokens` marks a ceiling the exact-model adapter resolved because the caller supplied none; promoting it into a child's explicit options would pin that child to one adapter's default for every later model it runs. An explicit parent ceiling is never marked and therefore does carry. This mirrors `requestProposal()`, which strips the same marked fields before plugins propose the next config.

`startContinuable()` now resolves the child's options once, before its first await, and derives both the child's `AgentOptions` and the durable descriptor's `agentProvider` / `agentModel` from that one value. Resolving before the await matches the delegated-policy capture immediately below it: the route belongs to the parent as it stands at delegation, not as it stands whenever materialization acquires a lock. Deriving both from one value removes the duplicated precedence rule the descriptor previously restated, and fixes cold resume without touching the resume path — a descriptor holding the right route rebuilds the right options.

`Agent.options` is now documented as the creation seed, naming `session.requestHeader()?.config` as the authority for the route in force. The stale JSDoc is the most likely proximate cause of the defect and would have re-created it in the next consumer.

## Alternatives considered

**Write the selected route back into `Agent.options` when the picker changes it.** Rejected because it inverts the established direction: request-level selection is deliberately a waterfall over the request, so that a concurrent switch takes effect at the next step rather than mutating agent state mid-turn, and the api-proxy's tiering already treats the log as the authority. Making options mutable would give two writers to a value read as a creation invariant.

**Seed `agent.options` from the session's own logged header on resume, rather than from the deployment default.** Not rejected — deliberately deferred, and orthogonal once this landed. It would make `agent.options` honest for a resumed session and would match the preset resolution three lines above it in the same api-proxy path, which already resolves from the session's log. It does not fix Case A, where no resume occurs, and delegation no longer depends on it. Left as a separate decision so the money bug lands on its own.

**Fix it in the api-proxy, where the wrong seed originates.** Rejected as the wrong role. `resolveChildAgentOptions()` is the `resolve(request): Spec` step the delegation seam owns; patching a Consumer would leave the other in-process driver wrong and would put a routing rule in a host.

**Take provider and model independently, each falling back on its own.** Rejected because a provider and a model are a pair — a live provider with a snapshot model names a model the new adapter may not serve. A logged header always carries both, so the pair is inherited or neither is.

## Testing

`packages/subagent/subagent-in-process-driver/tests/subagent-in-process-driver.spec.ts` drives one parent turn under an `agent/request` listener that switches the route to a second registered adapter, then asserts the child's options and the child's OWN logged header name the switched route, that both requests reached the switched adapter, and that the creation-seed adapter was never called — the audit path a deployment would use to answer which account paid. Two further cases guard the new resolution rather than the old defect: an explicit request route still overrides the parent's live one, and an adapter-resolved `maxTokens` is not promoted while an explicit parent ceiling is. The existing routeless-parent case still pins that nothing is fabricated for a parent with no route at all.

`packages/subagent/subagent/tests/continuation.spec.ts` asserts the persisted `subagent/descriptor` records the parent's live route, which is what cold resume rebuilds from. The existing cold-resume case still pins that an explicitly requested child route survives a restart.

Both regression assertions were confirmed to fail against the previous resolver and pass after it, so they pin the fix rather than the surrounding machinery.

The assembled-transcript layer is the shipped Web composition's e2e rather than a keyless snapshot, for the reason recorded when child preset inheritance landed: no runnable example this repo ships composes a model picker AND delegates, so the defect is not observable in the snapshot harness at all. A snapshot scenario would first need such an example. The Web e2e boots the real composition where the picker and delegation coexist, which is the assembled evidence available today.

## Consequences

A delegation now reads the parent's folded request header, an O(new events) incremental read the session already maintains for the loop. No new event, no new service method, no persistence change.

A child's route follows its parent's picker, which is what a user switching models expects and what makes the delegation decision reconstructable from the log. A deployment that relied on children pinning the startup default must now say so explicitly through the delegation tool's `agentOptions`, which is the same knob it always was and is now the only way to express that intent.

Existing continuable children keep the route frozen in their descriptors. The fix corrects what new delegations record; it does not rewrite descriptors already written, so a child created before this change still cold-resumes onto the route it was created with. That is the same immutability the descriptor has always had, and re-parenting an existing child's route would be a migration rather than a fix.

`agent.options` still misreports the route for a resumed session under the Web host. Nothing in delegation reads it now, but a future consumer reading it as "the current route" would repeat this defect, which is what the corrected JSDoc exists to prevent and what the deferred alternative above would remove entirely.
