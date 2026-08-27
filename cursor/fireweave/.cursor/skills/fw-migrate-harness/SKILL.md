---
name: "fw-migrate-harness"
description: "Move an app's feature-flag reads onto the FireWeave control-points v1 SDK — the code half of migrating to FireWeave-managed flags. Handles two source shapes — a direct PostHog SDK, and an OpenFeature-era FireWeave harness. Rewrites provider wiring to `initFireweave` from `@fireweaveai/server-sdk` / `@fireweaveai/web-sdk`, maps identity calls to target registration, and reports the result to fw-server so the project page can pair it with the flag half. Use when the user asks to \"migrate to FireWeave flags\", \"move off PostHog\", \"switch to the FireWeave SDK\", \"upgrade to control points\", or invokes `/fireweave:migrate-harness`. `--check` reports what would change without writing."
---
> Interaction: there is no skill-callable prompt tool — gate each decision by STOP and ask the user to pick a labelled option (or move the gate into the bundled MCP server via Elicitation).

# Migrate harness (move flag reads onto FireWeave)

Migrating a project to FireWeave-managed flags is **two independent halves**:

| Half        | Where it happens                                                            | Who does it                                    |
| ----------- | --------------------------------------------------------------------------- | ---------------------------------------------- |
| **Flags**   | FireWeave server — flags copied into the managed project, binding repointed | the _Migrate flags_ button on the project page |
| **Harness** | The app's own repository — flag reads go through fw-server                  | this skill                                     |

Doing only the flags half is the dangerous outcome: FireWeave ramps a flag in
the managed project while the running app still reads the customer's old
PostHog project. The FireWeave UI shows a rollout progressing and **no real
user sees anything change**. Nothing server-side can detect that, which is why
step 5 reports back.

Run this AFTER the flags half, so the flags the rewritten code reads already
exist in the managed project.

## HARD ORDER

1. **Detect** — find every direct provider read.
2. **Confirm** — show the plan; never rewrite without the user seeing it.
3. **Rewrite** — provider wiring, then identity, then call sites, then anchors, then the tracker unification, then the upgrade notice.
4. **Verify** — typecheck/build; a migration that does not compile is not done.
5. **Report** — `POST /v1/projects/:projectId/harness-migration`.

Do not skip 5. An unreported migration reads as "never happened", and the
project page will keep warning that flags moved without the code.

## 1. Detect

**There are TWO source shapes, and they need different rewrites.** Detect which
one you have before planning anything — a repo can contain both, on different
surfaces.

### Source A — a direct PostHog SDK

- imports of `posthog-js`, `posthog-node`, `PostHog`
- `posthog.isFeatureEnabled` / `getFeatureFlag` / `getFeatureFlagPayload`
- `posthog.identify` / `posthog.reset`
- OpenFeature providers bound to a PostHog adapter
- `POSTHOG_*` / `NEXT_PUBLIC_POSTHOG_*` / `VITE_POSTHOG_*` env reads

### Source B — an OpenFeature-era FireWeave harness

A repo initialised before the control-points v1 cutover. It is already on
FireWeave, so nothing is _wrong_ with it today — but it is on a surface the
SDKs no longer ship, so it cannot take SDK updates.

- imports of `@fireweaveai/sdk` (the RETIRED npm name — v1 is
  `@fireweaveai/server-sdk`), or `fireweave[openfeature]` /
  `ai.fireweave:fireweave-openfeature`
- any `@openfeature/server-sdk` / `@openfeature/web-sdk` import in
  `fireweave/fw-providers.*` or `fireweave/fw-harness.*`
- `FireweaveProvider`, `FireweaveWebProvider`, `makeFireweaveLocalProvider`,
  `FireweaveLocalProvider.create`, `make_fireweave_local_provider`
- `OpenFeature.setProviderAndWait(...)` in the harness
- hand-constructed `new FireweaveRuntime(new FireweaveRemoteAdapter(...))`
- a manifest whose `harness.flags.api` is `"openfeature"`

**What Source B must NOT touch.** This is a provider-wiring migration only:
leave `// @fireweave-controlpoint` anchors, `FW_STAMPS` in the tracker, and every
rollout-ready manifest exactly as they are. Anchors and stamps are the change
identity that `reconcile` and the dev-checklist gates read out of the committed
tree; rewriting them here would look like feature work that never happened. The
manifest's `flags.api` may stay `"openfeature"` — the schema accepts both values
forever, and a manifest rewrite is a server round-trip
(`upsert_rollout_manifest`) that this skill has no reason to make. Set
`"control-points"` only if you are already re-authoring that manifest for
another reason.

Group by **surface** (server, web, worker, mobile). Surfaces migrate
independently and the report records which ones actually moved.

If nothing is found, stop and say so. Do not scaffold a harness — that is
`/fireweave:initialise`.

## 2. Confirm

Show: the files to change, the surfaces covered, and anything you are NOT
migrating and why. Get explicit agreement before writing.

## 3. Rewrite

### 3a. Provider wiring

**Both source shapes converge on the same target.** From Source A you are
replacing a PostHog provider; from Source B you are replacing an OpenFeature
provider **and deleting the OpenFeature dependency** — v1 bans an OpenFeature
provider outright, so there is nothing left to wrap. Either way the result is
one `initFireweave` call whose `mode` selects the tier.

`mode` is REQUIRED and never inferred. That is the point of the shape: with
inference, a missing credential in production silently becomes local
evaluation, every control point serves its default, and the boot log stays
green — a feature that never ramps, indistinguishable from a rollout nobody
started. Do NOT hand-construct the runtime/adapter/client; that compiles and
runs while bypassing the validation which makes a bad credential fail loudly.

Server:

```ts
import { initFireweave } from '@fireweaveai/server-sdk';

// FW_ATTEST_URL is the legacy spelling, still read and never written. Saying
// so once is what makes the fallback temporary: a silent one is kept forever,
// because nothing ever tells the deploy environment it is out of date.
if (!process.env.FW_API_URL && process.env.FW_ATTEST_URL) {
  console.warn(
    '[fireweave] FW_ATTEST_URL is a legacy name for the fw-server base URL. ' +
      'Rename it to FW_API_URL — the value does not change. See fireweave.md.'
  );
}
const apiUrl = (
  process.env.FW_API_URL ??
  process.env.FW_ATTEST_URL ??
  ''
).replace(/\/+$/, '');
const fireweave = await initFireweave({
  mode: 'remote',
  apiUrl,
  apiKey: process.env.FW_PROJECT_API_KEY!,
  // REQUIRED for a self-hosted fw-server: omitted, the SDK validates apiUrl
  // against a canonical *.fireweave.ai + loopback allowlist and init fails with
  // a bare `Configuration` error that names nothing.
  allowedHosts: [new URL(apiUrl).hostname, 'localhost', '127.0.0.1'],
});

const on = await fireweave.controlPoints.getBooleanValue('key', false, {
  targetingKey: user.id,
});
```

Web (browser) — control points come from `@fireweaveai/web-sdk`, and reads stay
SYNCHRONOUS. `PUBLIC_FW_*` is a build convention (the bundler inlines it), not a
wire concern; the SDK reads no environment, so the harness passes it in:

```ts
import { initFireweave } from '@fireweaveai/web-sdk';

const apiUrl = import.meta.env.PUBLIC_FW_API_URL;
const fireweave = await initFireweave({
  mode: 'remote',
  apiUrl,
  apiKey: import.meta.env.PUBLIC_FW_PROJECT_API_KEY,
  allowedHosts: [new URL(apiUrl).hostname, 'localhost', '127.0.0.1'],
  context: { targetingKey: anonymousDeviceId },
});

// No await — safe inside render.
const on = fireweave.controlPoints.getBooleanValue('key', false);
```

**Remove the OpenFeature dependency after the rewrite** (Source B): drop
`@openfeature/server-sdk` / `@openfeature/web-sdk` from `package.json`, the
`[openfeature]` extra from the python install, and
`ai.fireweave:fireweave-openfeature` from the java build file. Leaving them
installed leaves a second, now-unwired flag path in the graph.

**Local/dev tier.** The same `initFireweave` call with `mode: 'local'` and
`local: { controlPoints: {} }` — no credentials, no network. Seed a key there to
dogfood a feature ON locally; never by passing `true` as a call-site default.

The app must end up holding **no PostHog key**. Flags are evaluated by
fw-server, which holds the managed credentials. Remove the now-unused
`POSTHOG_*` env vars from the app's config and deployment.

Reference implementation — read the **shipped templates**, not another customer's
repo. They live in the initialise skill's own directory, beside its `SKILL.md`:
`harness/ts-server/fw-providers.ts.tpl` (server) and
`harness/web/fw-providers.ts.tpl` (browser). Resolve that path relative to the
initialise skill's directory in this same bundle — `<dir of this SKILL.md>/../`
plus whatever this host names it (`initialise`, `fw-initialise` on Cursor,
`fw_initialise` on Codex). `apps/api/src/fireweave/fw-providers.ts` in the
pulse-folio repo is one instance of that template, not the source of truth, and
is not readable from here.

### 3b. Identity — the part that is easy to get wrong

`posthog.identify(id, props)` does **two** jobs:

1. declares who the current user is, and
2. **stores** those properties as a person profile.

The FireWeave equivalent is also two calls, and emitting only the first is the
common mistake:

```ts
// durable: stored server-side; rules keep matching without the app resending.
// Resolves rather than throwing — this runs in sign-in paths, where a targeting
// concern must not break authentication. LOG `ok: false`: a silently
// unregistered target is exactly how a rule ends up matching nobody.
const res = await fireweave.registerTarget(user.id, {
  kind: 'user',
  properties: { plan: user.plan, beta: user.inBeta },
});

// per-request: overrides the stored properties for this evaluation only
const on = await fireweave.controlPoints.getBooleanValue('key', false, {
  targetingKey: user.id,
  ...perRequestState,
});
```

On the **web** surface these collapse into one call — `fireweave.identify(id,
{ kind: 'user', properties })` registers the durable facts AND re-prefetches the
decision cache under the new id, so a percentage ramp buckets on the real user
rather than the anonymous placeholder every signed-out visitor shares.

Emit **both**. Context alone makes every rule only as good as what the app
remembers to send on each request — a rule targeting `plan` matches nobody if
one surface forgets it, silently and with no error.

Split the properties deliberately:

- **durable** (register): plan, beta membership, region, device model
- **per-request** (context): page, session, experiment context

`posthog.reset()` on logout becomes a re-prefetch under the anonymous id (web:
`identify(anonymousId, …)`; server: simply pass the anonymous key as
`targetingKey` on subsequent reads). Do NOT "unregister" the target — the stored
profile is still correct, the user simply stopped being the current one.

### 3c. Call sites

Flag reads become **control-point reads**: `controlPoints.getBooleanValue(key,
false, ctx)` — async on the server SDK, synchronous on the web SDK.

**Rewrite the retired `flag` alias too.** A repo scaffolded before ADR-022 reads
through a harness alias — `fw.flag(key, false, ctx)` on TS/web, `fw_flag(...)` in
python, `fwFlag(...)` in java/swift. Convert every one of them to the
control-points call. The alias existed partly so the `eject` codemod could
recognise and rewrite it; eject is retired and FireWeave is not eject-able, so
the alias now buys nothing and costs a name to learn.

Per surface, the target spelling:

| surface   | from                           | to                                                               |
| --------- | ------------------------------ | ---------------------------------------------------------------- |
| ts-server | `await fw.flag(k, false, ctx)` | `await fw.controlPoints.getBooleanValue(k, false, ctx)`          |
| web       | `fw.flag(k, false)`            | `fw.controlPoints.getBooleanValue(k, false)` — still no `await`  |
| python    | `fw_flag(k, False, ctx)`       | `fw_control_points().get_boolean_value(k, False, ctx)`           |
| java      | `fwFlag(k, false, id)`         | `FwProviders.getFwClient().controlPoints().getBooleanValue(...)` |
| swift     | `fwFlag(k, default: false)`    | `FwProviders.getFwClient().controlPoints.getBooleanValue(...)`   |

Do NOT invent a wrapper of your own. The RAMP-1 eval-site check keys on the
CALLEE name (`flag` / `getBooleanValue` / `getBooleanDetails`, plus python's
snake_case `get_boolean_value` / `get_boolean_details`); a bespoke name makes
every call site invisible to it, and an invisible call site is worse than a
wrong one — the file scans as having no evaluations at all, which is
indistinguishable from one that is genuinely safe.

If the surface still scaffolds a harness accessor (`fw.controlPoints` on TS/web,
`fw_control_points()` in python), route through it. java and swift have none by
design — their SDKs expose no importable `ControlPoints` type to name — so call
`getFwClient().controlPoints…` directly there.

**Reads never throw.** Every failure resolves to the default you passed, with
the reason on the `Decision` — so drop any try/catch that existed only to guard
a flag read, and reach for `getBooleanDetails` (same arguments) when you need to
see why a value came back.

Keep the same flag keys — the flags half copied them under their existing names,
so renaming here would break the mapping.

### 3d. Rewrite anchors to the control-points marker

Anchors written before the rename read `// @fireweave-flag <key>`. New ones read
`// @fireweave-controlpoint <key>`. Rewrite them.

**This is safe to do, and safe NOT to do.** Every scanner accepts both markers
permanently — the old spelling is never removed from the accepted set, because
these anchors live in the customer's own source files and dropping it would take
detection to zero repo-wide rather than degrading it. So this step converges a
repo on one vocabulary; nothing breaks if it is skipped or partially applied.

Rewrite the **marker only**:

```diff
-// @fireweave-flag checkout-v2
+// @fireweave-controlpoint checkout-v2
```

**Do not touch the key.** The key is the join between the anchor, the manifest
and the rollout — renaming it orphans all three.

Scope it to real anchors:

- Match `@fireweave-flag` followed by whitespace and a key. A bare mention in
  prose or a changelog entry is not an anchor and must stay as written.
- Every comment leader is in scope — `//`, `#`, `<!-- -->` — because the marker
  is matched independently of the leader.
- Skip `node_modules`, build output, and anything outside the repo's
  `sourceRoots`.

**Verify by scan, not by diff.** After rewriting, run
`mcp_rollout-server_detect_rollout_ready` and confirm the anchor count is
**unchanged**. A drop means a rewrite ate a key or broke a line; the count is
the only check that catches that mechanically.

Mixed markers in one repo are fine and expected mid-migration.

### 3e. Unify the tracker and drop the legacy pair

Two changes to how a repo records its `FW_STAMPS` trackers. Both apply to
**every** initialised repo, including one that needed no call-site rewrite.

**(a) Move a TypeScript tracker beside its harness.** Repos initialised before
the unification keep a `fw-tracker/` DIRECTORY somewhere in the tree
(`src/fw-tracker/index.ts`, `apps/api/src/fw-tracker/index.ts`, …) while every
other language keeps a single module next to the harness. Collapse it:

```
src/fw-tracker/index.ts   →   fireweave/fw-tracker.ts
```

Move the file with its contents intact — **the `FW_STAMPS` entries are the
record; never regenerate them empty.** Delete the now-empty `fw-tracker/`
directory. If the directory holds anything other than `index.ts`, stop and say
so rather than guessing: nothing in FireWeave writes a second file there, so its
presence means something else owns it.

**(b) Record per-surface, then delete the legacy pair.** Write each tracker onto
its own surface entry, then remove `rolloutReady.trackerPath` and
`rolloutReady.webTrackerPath` outright:

```jsonc
{
  "surfaces": [
    {
      "surfaceId": "sfc_…",
      "path": "fireweave/fw-harness.ts",
      "trackerPath": "fireweave/fw-tracker.ts",
    }, // ts-server
    {
      "surfaceId": "sfc_…",
      "path": "api/fireweave/fw_harness.py",
      "trackerPath": "api/fireweave/fw_tracker.py",
    }, // python
    {
      "surfaceId": "sfc_…",
      "path": "svc/fireweave/fw_harness.go",
      "trackerPath": "svc/fireweave/fw_tracker.go",
    }, // go
  ],
  // rolloutReady.trackerPath / webTrackerPath: REMOVED
}
```

The pair modelled exactly two surfaces and is now a second source of truth that
drifts — move a tracker, update one place, and the other keeps pointing at the
old path while still looking authoritative. Nothing server-side needs it:
fw-server stores tracker paths per surface and reconstructs the pair from its
own rows on the way out.

**Order matters.** Record every per-surface path and confirm each file exists
BEFORE deleting the pair. Deleting first leaves a window where a surface has no
tracker recorded at all.

**Do not invent a path.** Record only a tracker that actually resolves — a
recorded path that does not exist is worse than an absent one, because the gate
stops falling back to its conventional locations. If a surface has no tracker
module, scaffold one from the initialise skill's `harness/<surface>/fw*tracker*`
template first.

### 3f. Tell the team to upgrade

This migration moves the repo onto shapes older tooling does not read: the
unified tracker location, per-surface tracker paths, and the
`@fireweave-controlpoint` anchor. A teammate on an older `fw` CLI or an older
installed plugin bundle will not fail loudly — their gates will simply find
less, which is the failure mode this whole area exists to prevent.

FireWeave cannot detect their version from here, so **print this in the session
summary and put it in the PR description**:

> This repo has been migrated to FireWeave control points. Before your next
> feature, upgrade both:
>
> ```
> fw self-update
> ```
>
> …and update the FireWeave plugin bundle in your agent (re-run
> `/fireweave:adopt` after upgrading). Older versions read the previous tracker
> layout and will under-report rollout state rather than erroring.

Say it plainly — an upgrade notice buried in a commit body is one nobody reads.

## 4. Verify

Run the project's typecheck/build and its tests. Report failures; do not
proceed to step 5 with a broken build — a migration that does not compile is
not migrated, and reporting it as such makes the project page lie.

## 5. Report

Send the report **over the CLI profile**, not the project API key.

`{projectId}` comes from the repo's FireWeave pointer — read `projectId` from
`.fireweave/project.json`, or call `mcp_rollout-server_select_project` if the
pointer is absent. Do not invent it and do not use the project _name_.

```
fw api POST /v1/projects/{projectId}/harness-migration --body '{
  "status": "migrated",
  "sdkVersion": "<installed @fireweaveai/server-sdk version>",
  "surfaces": ["server", "web"],
  "notes": "go surface still reads PostHog directly"
}'
```

**Substitute every value before sending.** The fields above are a worked
example, not a template to paste: `sdkVersion` must be the version actually in
`package.json`, `surfaces` the surfaces you actually rewrote, `notes` real prose
or omitted entirely. The server accepts any string, so a pasted placeholder is
stored verbatim and rendered on the project page — `sdkVersion` appears in the
migration badge exactly as sent. Omit `notes` rather than sending filler.

`status` is one of `migrated` | `partial` | `failed`.

**Do not send a raw `Authorization: Bearer {FW_PROJECT_API_KEY}` POST.**
`FW_PROJECT_API_KEY` is the `fw_ingest_pub_*` runtime ingest key (the prod
flags credential); this
endpoint authenticates on the org plane (CLI access tokens and `fw_org_*` API
keys), so a key-bearing POST 401s. `fw api` carries the CLI profile, which is
the credential this route accepts — and the one plane a repo always has,
including a dev-only project that never gets an ingest key at all.

**Precondition — check before you send.** Unlike `initialise`, this skill has no
Step 0 that already gated on auth, so establish both here:

- `fw --version` resolves. If not, the report cannot be sent from this session.
- `fw api GET /v1/projects/{projectId}` returns 2xx. This settles the profile,
  the org binding, and `{projectId}` in one call.

If either fails, **do not guess a credential and do not fall back to the
ingest key** — the endpoint would reject it. Report the harness as migrated in the
session summary, name the project page as still-warning, and say which of the
two checks failed so the operator can run `fw login` / `fw profile use` and
re-send.

On a failure, say so in the session summary and name the project page as
still-warning. Do not PARK: the harness itself is migrated and committed — an
unreported success is a stale page, not a broken repo. Note that a missing or
ambiguous profile fails **before** any HTTP call, so `fw api` prints an error
envelope with no status code — treat "no 2xx observed" as the failure
condition, not "a non-2xx was returned".

Status honestly:

- `migrated` — every detected read now goes through FireWeave.
- `partial` — some surfaces still read the provider directly. Name them in
  `notes`. Partial does **not** count as done: the project page keeps warning,
  which is correct, because those surfaces still bypass FireWeave.
- `failed` — the rewrite did not land. Say why.

Never report `migrated` to make the page green. The page's whole purpose is to
show when the two halves disagree.

## `--check`

Do steps 1–2 and stop. Write nothing, report nothing.
