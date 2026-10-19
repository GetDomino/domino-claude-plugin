# Authoring and publication

This reference covers writing and publishing quests: snippets, upgrades, resource keys, settings, offers, leaderboards, external points, referral programs, rewards, announcements, hosted server applications, the project calendar, presentation and dashboards. The references listed under [More authoring references](#more-authoring-references) cover trackers, verification, quest rules and operating published resources.

Read the installed `@domino-sdk/relay/authoring` declarations and the project's existing quest modules. Define the requested behavior with the supported SDK helpers rather than inventing capabilities. Authored entries in relay.json resolve relative to that manifest and can select a named export with `exportName`.

The `discover.quests` and `discover.questTypes` arrays in relay.json select default-exported resource files. New projects use `quests/**/*.quest.ts` and `quests/**/*.quest-type.ts`. Add matching files without individual registration. Existing manifests without `discover` keep explicit-entry behavior. Omitting either discovery array or setting it to `[]` disables discovery for that resource kind while preserving explicit entries. An explicit entry overrides the matching discovered default export; omitted providers inherit `discover.provider` for those files. Discovery does not execute authored modules on the build host, publish them, or archive omitted resources.

## More authoring references

- [Trackers](trackers.md#declare-trackers): read before declaring a tracker (X, Discord, Telegram, YouTube Shorts, TikTok or wallet balances), triggering a quest from one, gating content on performance, paying engagement points, or checking a member's Telegram account and chat membership (`ctx.telegram`).
- [Verify member actions](verification.md#evaluator-readers-null-errors-and-pinning): read before a quest's `evaluate` checks what a member did: Discord membership, X, linked wallets, wallet balances, NFT holdings, contract state, Cosmos delegations, on-chain actions, text and URL submissions, AI text review, image checks, staff confirmation, or entry after app-owned pass verification. It also has the rules every evaluator reader follows: null results, errors, the call limit and pinning.
- [Quest rules and evaluation](quest-rules.md#campaigns-windows-and-prerequisites): read before shaping how a quest decides and repeats: campaigns, availability windows and prerequisites, automatic and repeatable quests, reject codes, deferred evaluation and progress reports, distinct-action counters, retry and completion cooldowns, evaluation time and randomness, calendar-day streaks, test members, and local quest tests.
- [Operate published resources](management.md#management-client): read before operating a project from a script or server: the management client and its error contract, project identity and settings, resource lifecycle, canonical lifecycle commands, recovering a quest lifecycle change, operator undo commands, and scheduled hosted jobs.

## Find a snippet

Before writing a new quest, fetch the public catalog. It requires no authentication:

```sh
curl -fsS https://snippets.domino.run/r/registry.json
```

1. Match the requested behavior against each item's `title`, `description`, and `categories`. Use the catalog's `name` to fetch `https://snippets.domino.run/r/<name>.json`; inspect its `docs` and `files` contents, targets, and prerequisites before installing. The live catalog is the source of available recipes, so newly published snippets need no skill update.
2. If a recipe fits, install it from the project root containing `relay.json` and `package.json`. For example, when the catalog lists `photo-quest`:

   ```sh
   pnpm dlx shadcn@4.21.0 add https://snippets.domino.run/r/photo-quest.json
   ```

   Preserve existing customized files; adapt those in place when the recipe is already installed. Use the project's package manager equivalent when it is not pnpm.

3. Read the installed `docs/snippets/<name>.md`. Adapt the source to the requested behavior, choose unique authoring keys for new quests, and configure settings, rewards, and verification prerequisites. Keep existing authoring keys stable and verification source out of the browser bundle.
4. Confirm the source matches `discover` in `relay.json`. Move it to a matching directory for custom layouts, or add an explicit entry if discovery is disabled or unsupported by the installed CLI. Run the project's checks and `domino build --json`, then exercise the requested participant behavior using the verification workflow in the Domino skill. Installation only copies source; publication is a separate step below.

If no recipe fits or the catalog is unavailable, author with the installed SDK declarations and existing project examples. Report an unavailable catalog as unchecked, rather than claiming no matching snippet exists. Snippets are editable starting points; check compatibility with the project's installed SDK before changing package versions.

## Upgrade existing authoring code

When an upgrade fails on removed APIs, follow the [SDK v1 upgrade guide](https://console.domino.run/docs/reference/upgrade-to-v1). Start with [typed handles and settings references](https://console.domino.run/docs/reference/upgrade-to-v1#primitive-handles), then [typed prerequisites](https://console.domino.run/docs/reference/upgrade-to-v1#prerequisites-are-typed-eligibility-rules). Apply the same changes to code generators, not only their generated files.

Authoring declarations use `key`; the old `id` alias is removed. Keep the old value when renaming `id` to `key`: it names the source declaration, not the server resource ID. Nested budget, tier, milestone, and bonus declarations use `key` too.

Run these commands from the project directory after upgrading the SDK and CLI:

```sh
domino migrate
domino migrate --write
domino check --json
```

`migrate` previews a diff without writing files or contacting Domino. `--write` applies the changes. It renames `id` to `key` on authoring constructors, referral bonuses and milestones, tiers, `budget()` calls, and collection declarations, renames `balance` to `ledger` on referral bonuses, leaderboards, engagement rewards, and `budget()` costs, and proposes `pointLedgers` entries for statically known ledger keys. Existing server IDs are not renamed. Review proposed ledger names before deploying. The codemod does not finish typed-handle conversion: migrate raw references to imported handles, or use `settings.pointLedger()` for a Console-selected ledger that already exists remotely. A remote ID remains a string for management methods; it is not an authoring handle.

The migration follows configured and discovered entry modules and their local imports. It recognizes SDK import aliases and namespace imports, keeps every renamed value, and does not execute source. Indirect objects, spreads, and dynamic ledger references receive manual instructions. A string where v1 takes a handle (a referral bonus or leaderboard `quests` entry, a referral `qualification.quest`, an engagement `qualifiedBy`, a `ledger`, a `requires` entry, or `completed()`) gets a `HANDLE_REFERENCE_REQUIRED` error naming the field; convert it to a handle import by hand. A budget cost defined in another module gets `MANUAL_MIGRATION_REQUIRED`; rename its `balance` to `ledger` there. It does not rewrite dependencies, unconfigured source, or files outside the project directory. Resolve conflicts before applying; a conflicting identity or ledger prevents all writes. `HANDLE_REFERENCE_REQUIRED` and `MANUAL_MIGRATION_REQUIRED` errors do not prevent the write, so apply the renames, then convert the references. Repeating the command after a completed migration produces no further diff.

`domino check --json` includes a `diagnostics` array with `code`, `severity`, `file`, `line`, `column`, `message`, and `suggestion` for source findings. `AUTHORING_ID_DEPRECATED` and `AUTHORING_BALANCE_RENAMED` are warnings for renames `migrate --write` applies. `AUTHORING_IDENTITY_CONFLICT`, `AUTHORING_LEDGER_CONFLICT`, `AUTHORING_KEY_MISSING`, `HANDLE_REFERENCE_REQUIRED`, `MANUAL_MIGRATION_REQUIRED`, `BUDGET_DECLARATION_CONFLICT`, and `AUTHOR_BUDGET_TRACKER_REQUIRED` fail the check. `LEDGER_REFERENCE_DYNAMIC` (a ledger reference that cannot be determined without running code) and `LEDGER_DECLARATION_MISSING` (a ledger with no local declaration) are warnings because the selected ledger may already exist remotely. `AUTHORING_IDENTITY_DYNAMIC` warns about a declaration built from an indirect object or spread, which `migrate --write` leaves unchanged. `LEADERBOARD_TIES_DIFFER` warns about a leaderboard whose live `ties` differs from its `settle.ties`: its standings show one order and its prizes pay another. `TRIGGER_REFIRES_ON_REFRESH` warns about a quest on a YouTube or TikTok tracker that runs on every observation, as `automatic()` does by default there: each refresh evaluates it again for every video ([trackers](trackers.md#youtube-shorts)). A `defineLedger` handle, defined in the same module or imported, is a static reference to its key, and a ledger published in `pointLedgers` as `{ "entry", "exportName" }` counts as a local declaration. A ledger read from a setting is dynamic. Manifest validation errors appear on stderr with `code: "MANIFEST_INVALID"` and field paths in `details`; a relay.json that is not valid JSON reports `MANIFEST_INVALID` without them. A `--config` file that does not exist reports `PROJECT_NOT_FOUND`, and a quest, type, engagement rule or image that fails to bundle reports `BUNDLE_FAILED`. Each exits 1. Agents should follow the reported suggestion and rerun the check; never invent a replacement key for an existing resource.

## Resource IDs and authoring keys

Use the IDs returned by Domino when referencing projects, quests, point ledgers, rewards, and inventory items. Creation requests do not accept a new resource's ID. The server assigns an opaque, prefixed ID. Existing stored IDs remain valid after upgrade.

In source, give every authoring constructor a stable `key`. Constructors return typed handles that carry that key. Pass those handles between authoring helpers; for example, `completed(welcome)`, `points(xp, 100)`, and `defineLeaderboard({ ledger: xp, ... })`. Domino maps each key to an opaque resource ID within the project and environment. Redeploying the same key updates the same resource; changing the key creates another resource. Display titles are independent of identity. Participant, management, Console, and stored wire values continue to use strings where their APIs accept IDs or keys.

## Offers

An offer exchanges points from one ledger for a tiered reward. Define the reward once, then default-export the offer from its own module:

```ts
import {
  automaticFulfillment,
  defineLedger,
  defineOffer,
  defineTieredReward,
  points,
} from "@domino-sdk/relay/authoring"

const community = defineLedger({ key: "community", name: "Community" })

const badge = defineTieredReward({
  key: "supporter-badge",
  title: "Supporter badge",
  tiers: [{ key: "standard", title: "Standard" }],
  inventory: { kind: "untracked" },
  fulfillment: automaticFulfillment(),
})

export default defineOffer({
  key: "supporter-badge-offer",
  price: points(community, 500),
  reward: badge.tier("standard"),
  stock: 100,
  limit: { perMember: 1 },
})
```

Add `{ "entry": "src/supporter-badge.offer.ts" }` to `offers` in `relay.json`. `title` defaults to the reward's title. `stock` (required, 0 to 1,000,000, never unlimited) is the total supply: increasing it adds units and decreasing it succeeds only while enough unredeemed units remain. `limit.perMember` (1 to 10,000) defaults to 1. Use `staffHandover()` instead of `automaticFulfillment()` when staff must confirm physical delivery. A staff-handover reward with pickup variants (sizes, configured on the reward's pickup policy) is not split by variant at redemption: participants see neither the variants nor their stock. Staff choose an in-stock variant when they hand it over, and from then on the participant's entitlement carries it as `allocation` (`item` and `label`). The reward must be a tier from `defineTieredReward`: `badge()` and `groupMembership()` type-check but `defineOffer` throws (`Badge rewards can only be granted by quests or manually, not by offers`; `Group membership rewards can only be granted by quests, not by offers`), and a deployment refuses them with 422 `validation-failed` and the same message.

To limit who can redeem, give the offer `requires`: the prerequisites quests take, `completed(quest)`, `balanceAtLeast(ledger, minimum)` and `inGroup(group)`, at most 30 (see [prerequisites](quest-rules.md#campaigns-windows-and-prerequisites)). For example, `requires: [inGroup(members), balanceAtLeast(community, 1000)]` sells only to group members holding at least 1,000. A balance prerequisite is a threshold, not a cost: the price is spent separately, and both read the balance when the member redeems. Publication refuses a `completed` quest or a ledger the deployment doesn't know. A required group the environment doesn't have is created, empty, with the handle's title: deploying creates it in test and approving the promotion creates it in live (see [groups](quest-rules.md#campaigns-windows-and-prerequisites)). Relay checks the prerequisites when the member redeems and refuses a member who misses one with 409 `prerequisites-incomplete` and `missing`, the same shape quests use, spending no points or stock. Meeting them later reserves nothing, and leaving a group later leaves earlier redemptions alone. Offer reads for a signed-in member list what they lack in `missing`, so an app can show the offer locked and why; otherwise `missing` is empty.

Omitting an offer entry from a later deployment does not delete or pause the published offer. Pause it in Console, or call `management.offers.pause` with its current revision and a reason.

Participants list offers with `relay.offers.list()` or `useOffers()` and redeem with `relay.offers.redeem(offer.id)` or `useRedeemOffer()`. A redemption spends points, consumes one unit, and grants the reward atomically. Show the returned availability and `missing` instead of calculating limits, stock or eligibility in the app.

Create point ledgers in Console **Rewards → Point ledgers → New ledger**, with `management.pointLedgers.create({ name, description })`, or define a module such as `ledgers.ts`:

```ts
import { defineLedger } from "@domino-sdk/relay/authoring"

export const community = defineLedger({
  key: "community",
  name: "XP",
  description: "Lifetime progress",
})
```

Publish that export from `relay.json`:

```json
{
  "pointLedgers": [
    { "entry": "ledgers.ts", "exportName": "community" }
  ]
}
```

Merge this property into the existing manifest, import `community`, and use `points(community, 100)`. A `points()` amount is a whole number from 0 to 2,147,483,647, in a quest reward, an evaluator decision, a leaderboard prize, or a referral reward. An offer price takes the same range but starts at 1. A deployment previews ledger declarations with the other resources and publishes them together. An unknown or archived ledger reference fails validation instead of creating a balance. `management.pointLedgers.list()` lists every ledger (`management.points.ledgers()` is its deprecated alias); management entries include nullable `archivedAt`. Creating, editing or archiving a ledger directly (`management.pointLedgers.create`, `management.pointLedgers.update`, `management.pointLedgers.archive`) needs `publish`. Live refuses these for every credential, Console included, with 422 `promotion-required`; a live ledger comes from a `defineLedger` declaration in a reviewed deployment. Archive hides a ledger from new reward choices and new published definitions; existing published rewards, balances, history and operator corrections keep working.

For a selectable ledger, use `settings.pointLedger({ label: "Point ledger", default: community })`. The optional default is a ledger handle. Without a default, the setting is empty until an operator chooses a Console-created ledger, and the quest stays paused. Guard rewards from the empty value: `settings.balance ? [points(settings.balance, 100)] : []`.

The management SDK generates an `actionId` when a keyed mutation omits it, including point changes, offer controls, reward corrections, enrollments, and resource creation. Typed methods whose input has an `actionId` send the same request again after a transient failure (see [Management client](management.md#management-client)). If a write still fails, the `ManagementError`'s `operationId` is that action ID, to retry the same input explicitly; see [Management client](management.md#management-client). For retries across process restarts, generate and persist an action ID before calling the helper. Direct HTTP creation requests still require `actionId`.

Use `actionId` to make a creation request retryable. Reuse the same action ID and payload after an uncertain response; a different payload with the same action ID fails. The action ID is not the resource ID. Provider subjects, tracker record IDs, completion keys, and nested quiz, tier, budget, and referral-rule keys keep their own scopes.

For pickup inventory, supply a label for each new variant and omit `item`; saving returns its assigned item ID. Reuse that ID to share existing stock. For standalone stock, type a name in the Console Stock item picker and choose **Create item “NAME”**, or use `management.inventory.createItem({ name })`. Both return a server-assigned item ID. Stock adjustments accept existing item IDs only. Pickup configuration and item creation are definitions, so live refuses them with `promotion-required`.

## Resource and date/time settings

Build every quest setting with a `settings.*` builder (`integer`, `text`, `boolean`, `choice`, `datetime`, `resource`, `pointLedger`, `quiz`, `telegramChat`, `amount`, `chain`) inside `settings.object({ ... })`. A plain zod schema, such as `z.number().default(1)`, is a type error, and `defineQuest` throws `Setting "<key>" is not a quest setting` for untyped callers: Console edits only what the builders describe.

### Secret settings

Use `settings.text({ label, secret: true })` for a value members must not see and that should not sit in source, such as a shared scan code. The operator sets it in Console (or with `management.quests.updateSettings`), and the quest's code reads it in `settings` as usual. It takes no `default`: it reads `""` until set, so guard the empty value. Its `minLength` defaults to 0.

- **Never in source.** A secret setting with a non-empty `default`, or a value from a deploy entry's `settings`, fails: `defineQuest` throws, and publication refuses with `Secret setting "<key>" must default to ""`. Reusable quest types refuse secret settings; use a quest written in code.
- **Never in responses.** Every management response leaves the key out of `values` and `configuration` (`domino deploy --json` and `--preview --json`, `quests list`/`get`, releases, deployments, attempts, members). A release's `withheldSettings` lists the secret settings that hold a value; its `fields` entry still says `secret: true`. An attempt capture leaves it out of `context.settings`, so `domino quests replay` runs it empty. Withholding follows each release's own `fields`: releases, deployments and captures from before a setting became secret still show its old value, so set a new value after marking an existing setting secret.
- **Write-only in Console.** Console shows **Set** or **Not set**, takes a new value in a password field, and **Clear** resets it. It never shows the current value.

The quest's title and presentation come from describing its code with the settings, so don't put a secret in them. A capability call records its request in the attempt's capture, so a secret passed into one, such as an AI review prompt, shows in `domino quests replay` captures; compare it in the evaluator instead.

Use `settings.resource({ label, source: discord.channels() })` for a Discord channel selector. Import `discord` alongside `settings` from `@domino-sdk/relay/authoring`. The connected project integration supplies the server; never add a guild ID setting. Filter with `discord.channels({ types: ["announcement"] })` when only announcement channels qualify. The default includes text and announcement channels. Resource values are stable IDs, not labels.

Omit the resource default until the operator chooses a value in Console. Its empty string represents missing configuration. Code-authored quests publish paused when any resource setting is empty. Publishing a configured value activates the code-authored quest; clearing it pauses the quest again. Console-created quests must have every required resource configured before activating. Existing operation-level pauses remain independent. Live refuses settings changes with `promotion-required`, so in live an empty resource setting cannot be filled: the quest stays paused unless the deployed source supplies a default.

Use `settings.amount({ label, default, decimals? })` for a token amount: a non-negative base-unit integer string without leading zeros, such as `"1000000000000000000"`, exact at any scale. With `decimals`, Console shows and edits it in whole tokens (`formatUnits`/`parseUnits`). A tracker filter may compare with it (`gte: { setting }`). Use `settings.chain({ label, default })` for a chain `ctx.wallet.balance()` reads: `"solana"`, `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp"`, or an EVM CAIP-2 ID such as `"eip155:8453"`; its value is typed as the balance query's `chain`. Use `settings.nft({ label, default })` for an NFT collection `ctx.wallet.nfts()` reads, such as `"eip155:1/erc721:0x…"` or `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/nft:<collection mint>"` ([NFT holdings](verification.md#nft-holdings)). Use `settings.contractCall({ label, default })` for a contract function `ctx.wallet.call()` reads, such as `"eip155:8453:0x… function stakedOf(address) view returns (uint256)"` ([Contract state](verification.md#contract-state)). Use `settings.contractEvent({ label, default })` for an EVM contract event `ctx.wallet.events()` reads ([On-chain actions](onchain.md#evm-events)); read a busy wallet, or a long window on a fast chain, with `newest` so the read returns the newest events instead of throwing at its caps. Use `settings.solanaProgram({ label, default })` for a Solana program `ctx.wallet.transactions()` reads ([Solana transactions](onchain.md#solana-transactions)). All six are text settings (`picker: "amount"`, `"chain"`, `"nft"`, `"contract-call"`, `"contract-event"` or `"solana-program"`, and `decimals`), and publication and Console edits refuse other values; an invalid `default` throws when the quest module loads.

Use `settings.datetime({ label, default: "2026-09-21T00:00:00+02:00" })` for an exact cutoff instant. Values normalize to UTC ISO timestamps. Console displays date and time with the operator's local timezone explicitly labeled. Offset-free timestamps and invalid calendar dates are rejected. This is not a date-only setting.

Resource selections are checked on the backend for Console and CLI publication, including final deployment publication after preview. The selected resource must belong to the current connected integration and be usable by its bot. Lookup caches do not replace publication validation. A temporary provider outage rejects the new publication without changing the existing release. Deleted selections remain visible for the operator to replace.

Replacing a text field with a resource or datetime field preserves Console overrides when their values pass the new validation. Keep the field key stable. Invalid overrides must be updated or reset explicitly; they are not silently discarded.

## Publish the authored resources

Source owns reusable quest definitions, collection slots, and deployment defaults. Console owns operator-created quest instances, their placement, and custom setting values. Keep identifiers stable. Registering a new reusable type version does not upgrade existing instances.

Entry `settings` supplies deployment defaults. Remote custom values take precedence on subsequent deployments. A removed or incompatible setting can block the batch. Use `settingRenames` for a deliberate rename, or have the operator reset the custom value in Console under **Quests**. Changing source defaults does not clear overrides.

Use `domino build --json` or `domino deploy --dry-run --json` for offline bundling. Quests and quest types import the installed SDK's [shared runtime](https://console.domino.run/docs/concepts/quests#shared-sdk-runtime) instead of bundling it, so each quest's code is a few kilobytes plus whatever else it imports. A `BAD_REQUEST` saying the Relay does not accept quests built against an SDK runtime means the Relay predates runtimes: update it, or restart an outdated local stack. One project holds up to 1,000 quests and 200 quest types; above that `domino build` fails with `BUNDLE_FAILED` ([limits](https://console.domino.run/docs/build/publishing#publish-behavior-to-test)). To save the bundle, use `domino build --out .domino-build/deployment.json --json`; the CLI creates missing parent directories. These checks do not validate effective remote settings. Use `domino deploy --preview --json` to resolve and validate the remote batch without publishing. Check the selected scope before publication.

A hosted activation can save its receipt and copy reference images before a later description fails. Such a failure is 502, even if the dependency reported 503. Treat these outcomes as uncertain and inspect the build before starting another. A 503 promises that the entire request made no committed write or external effect. See [write recovery](https://console.domino.run/docs/reference/upgrade-to-v1#treat-failed-writes-as-uncertain-when-relay-cannot-prove-refusal).

Use `domino deploy --environment test --json` when test publication is part of the task. It publishes configured quests, types, collections, and supported interactions together. After a lost test publication response, use the returned `domino deploy --resume <id>` recovery command. Starting another deployment may create another immutable release.

To undo the latest test deployment, run `domino deploy rollback --environment test --json` (`management.deployments.rollback(id)` with a test client; publish permission). It publishes the previous published test deployment again at once, as a new deployment with `source: { kind: "rollback", deployment }`; the result names it in `rolledBackTo`. Only that deployment, the second entry of `management.deployments.history()`'s `published`, can be rolled back to, and the API refuses any other ID with `validation-failed`; running it again undoes the rollback. Only what the earlier deployment contains goes back: quests and other definitions only the later deployment added stay, and the build `domino stage` made keeps serving staging. Once a hosted app build has published to test since that deployment, the API refuses with 409 `conflict`; stage the earlier commit instead. The CLI saves it before sending; after an uncertain response, run the error's `operations resume` command. The CLI refuses live with `ENVIRONMENT_UNSUPPORTED` and `apiCode: "promotion-required"`: roll back test, then promote.

Deleting or no longer discovering a quest file does not archive the published quest. Run `domino quests archive <key> --reason "…" --json` (publish permission, test only; `management.quests.archive(key, { reason, expectedRelease? })`) to stop new attempts and hide it from collections; history stays, and it no longer counts toward `supportedInteractions`, so archive retired quests before deploying fewer interactions. Deploying its code again keeps it archived. `domino quests unarchive <key> --reason "…"` reopens it, paused while a resource setting is empty, and fails when the app no longer supports its interaction or the prerequisite graph rejects it. Quests created in Console change participation in their Console editor instead.

To take test's definitions to live, run `domino promote --environment test --json` (`management.promotions.prepare()` with a test client; publish permission in test). It compares every definition in test with live by authoring key, code-defined and Console-authored alike, and saves a pending promotion: `changes` lists each `kind`/`key` live would change as `added`, `changed`, `archived` or `unarchived`; `warnings` lists `binding-missing` integrations, webhook endpoints, sign-in methods and app secrets live lacks, which a promotion never copies, `live-only` definitions it leaves alone, and `group-created` groups approval creates; `build` is the hosted app build serving staging, which it carries. Live already matching fails with `promotion-empty`. Nothing changes in live until a person signed in to Console with publish permission chooses **Approve and apply** in **Live → Deployments**; an agent cannot approve. Console's **Promote to live** prepares the same promotion. Approval applies it in one transaction and fails with `promotion-changed` when test or live changed since it was prepared; then prepare again. Preparing again supersedes an older pending promotion. Approval creates, empty, each group a quest, offer, leaderboard or referral program names by key and live lacks (members are never copied); a group archived in live is never unarchived, and a `groupMembership` reward or a leaderboard exclusion-group binding naming it fails approval with 409. `domino promotions list|get` read them; `management.promotions.rollback(id)` with a live client prepares a promotion of an earlier applied promotion's definitions and build, archiving every live definition that promotion did not carry; a pickup policy it did not carry goes back to untracked. The CLI saves `domino promote` before sending it; after an uncertain response, run the error's `operations resume` command. Existing attempts retain their pinned releases and promised rewards. When the promotion carries a hosted app build, approval deploys it as the live app; `management.liveApp.get()` (`read`, either environment) answers its address and `state` (`none`, `deploying`, `live` or `failed` with `failure`). An organization admin can serve the live app at their own domain with `management.domains.add(hostname)` and `list`, `get(id)`, `recheck(id)` and `remove(id)` (`read` lists them); see [custom domains](hosted.md#custom-domains).

Code reaches live only through a promotion. Live refuses `domino deploy --environment live`, with or without `--preview`, and `management.deployments.preview` with a live client: 422 `promotion-required` from every credential, before the bundle is checked, which the CLI reports as `ENVIRONMENT_UNSUPPORTED`. Deploy to test, then promote. Do not use `--resume` for live approval. Use `domino promote --environment test` (`management.promotions.prepare()`). The deprecated `domino deploy --promote TEST_DEPLOYMENT_ID --environment live` and `management.deployments.promote(id)`, removed in 2.0.0, check that the test deployment was published, then prepare the same promotion of all of test, answered as a pending deployment record whose first warning, code `environment-promotion`, names the promotion; its `id` is the promotion's, which `promotions.get` reads and `deployments.get` and `publish` reject. `deployments.submit` and `deployments.reject` are deprecated as well: they act only on live deployments made before the refusal.

Fixture providers simulate verification. `workers-ai` requires a configured service binding. A fixture success demonstrates the flow but does not establish that a real external activity was verified.

## Leaderboard definitions

Default-export `defineLeaderboard({ key, title, ledger, score, access, group, includeUnscored, ties, settle })` from `@domino-sdk/relay/authoring`. Pass handles from `defineLedger`, `defineQuest`, and `defineGroup`; add `{ "entry": "src/leaderboard.ts" }` to `leaderboards` in `relay.json`. Use `domino deploy --preview` to inspect definitions and standings before publishing to test. Rows show generated names such as `Participant 1a2b3c4d` while the project's [profile settings](participant-api.md#profiles) keep profiles private; with profiles visible, a member shows the name they chose, else their first X or Discord name the settings list as defaults. `domino stage` and `domino doctor --remote` warn while profiles are private or list no default. Ask the owner whether to show members' X or Discord names, then set it in test (`domino profiles configure`) and run `domino promote`; the owner's steps are in [show member profiles](https://console.domino.run/docs/build/profiles#turn-profiles-on).

```text
import { defineLeaderboard, points } from "@domino-sdk/relay/authoring"
import { community } from "./ledgers"

export default defineLeaderboard({
  key: "weekly",
  title: "Weekly contributors",
  ledger: community,
  access: "participants",
  score: {
    kind: "earned",
    window: { kind: "calendar", period: "week" },
  },
  settle: {
    at: "window-end",
    ties: "shared",
    prizes: [{ ranks: [1, 3], rewards: [points(community, 100)] }],
  },
})
```

Score defaults to `{ kind: "balance" }`. Use `earned({ window, quests })` for an earned score, or `{ kind: "net", window }` for net changes. `earned` takes quest handles. Windows are `all-time`, `fixed` with epoch-millisecond `start`/`end`, `rolling` with `days`, or `calendar` with `period: "day" | "week" | "month"`: the current period of the [project calendar](#project-calendar), so it matches the project's budgets. A calendar window takes no `timeZone`. Access defaults to `participants`; `public` allows anonymous reads through the configured project endpoint. A group handle narrows eligibility. `includeUnscored: true` also lists eligible members with no qualifying points, at score 0 and one shared rank after every member who scored, ordered by member ID; settlements never include them. `ties: "earliest"` orders live equal scores as settlement `earliest` does.

`settle` is available only on fixed or calendar earned/net boards. Prize ranges must be ordered and cannot overlap. Rewards are points or tiers from `defineTieredReward`; a prize cannot repeat the same point ledger or tiered reward. A `badge()` or `groupMembership()` prize makes `defineLeaderboard` throw and preview refuse it, naming the reward kind. `shared` awards every member sharing a winning rank. `earliest` orders equal scores by the last qualifying entry that produced the score, then member ID. At cutoff Relay freezes the board version, eligible rows, exclusions (the exclusion group bound at the cutoff, with its members then), ranks and assigned prizes. Review and deliver the frozen result in Console **Leaderboards**, or read it with `management.leaderboards.settlements(board)` and approve it with `management.leaderboards.approveSettlement(settlement, { reason })` (`review` permission). A board that counts an engagement rule's points (the rule pays into its balance and, with `quests`, qualifies through one) first takes a closing measurement: its settlement reads as `pending` with no rows and a `closing` field (`startedAt`, `deadline`) while Relay asks scheduled trackers to resample the content, a sample observed within 30 minutes after the cutoff counts as measured at it, and the settlement freezes when everything is measured or the wait ends, listing content frozen at its last good sample in `unmeasured` (`content`, `member`, `observedAt`). Approval refuses a closing settlement (`settlement-closing`); `retrySettlementMeasurement(settlement)` or `domino leaderboard-settlements retry-measurement` closes it again for another 30 minutes. Each winner receives a completion; its prize points earn no referral bonus. `settle.pool` (optional; `prizes` may then be empty, or left out in `defineLeaderboard`) splits a fixed amount over the frozen eligible rows in proportion to score: `{ amount: "<base units>", unit: "points" | { code, decimals }, ledger?, minScore? }`. `ledger` (a ledger handle or key) is required for and only allowed on a points pool, which holds at most 2,147,483,647 points. Members scoring 0 or less, or at or below `minScore`, get nothing; each pool share is `floor(amount × score / total)` in bigint, and leftover base units go one each to the largest remainders, earlier `position` first, so shares sum exactly to the pool. With no eligible score the whole pool is `unallocated`. Settlement reads carry `pool` (`amount`, `unit`, `allocated`, `unallocated`, …) and each row's `share` in base units. Approval pays a points pool's shares as points into its ledger in each member's prize completion (merged into a rank prize's points for that ledger; revocation reverses them). Any other unit moves nothing: Relay never holds or moves tokens, and `management.leaderboards.settlementExport(id)` or `domino leaderboard-settlements export SETTLEMENT` (CSV; `--json` for rows with `share`, formatted `amount` and verified `wallets` `{ chain, network, address }`) is the operator's payout list; a rejected settlement's export lists every share as 0. Correct one with `management.completions.revoke({ completion, reason, freeOccurrence: false })`; leaderboard prize occurrences cannot be freed and paid again.

Leaderboards and points remain project/environment-scoped. Custom scoring runs before points ingestion. Management keys with `points` permission can award, spend, adjust, reverse, or set totals through `management.points.record`. Every command requires an action ID and reason; exact retries return the original result. `set` requires the current `expectedTotal` and explicit `earned`/`spend` classification. Reversals reference an original award and correct its original period. Console member detail shows every ledger balance and offers history, award, adjustment and reversal controls to operators with `points`. Do not implement negative clamping, direct database writes, browser awards, or shared organization balances.

## Leaderboard settlement cutoff

Settlement points must take effect inside the window and be received by its inclusive cutoff. New entries use database time for receipt, independently of app clocks and test-member clock advances. Relay waits for in-flight point writers to commit or roll back before freezing, so an entry received before the cutoff still counts if its transaction commits just afterward. A backdated entry received after the cutoff does not count. Existing frozen settlements remain unchanged.

## External points

Run this on a trusted backend with a management key scoped to the project and environment. In a Domino-hosted app's Worker, use `hostedManagementClient(env)` instead of pasting a key; it reads the platform-bound `DOMINO_MANAGEMENT_KEY` and scope. The member must already be a project participant. Quest awards to the board's balance count automatically; use this API for externally verified activity or corrections.

```ts
import { createManagementClient } from "@domino-sdk/relay/server"

const management = createManagementClient({
  baseUrl: "https://relay.domino.run",
  organization: "my-organization",
  project: process.env.DOMINO_PROJECT_ID!,
  environment: "test",
  getToken: async () => process.env.DOMINO_MANAGEMENT_KEY!,
})

await management.points.record({
  kind: "award",
  actionId: "external-activity-42",
  member: "member_existing",
  balance: "community",
  amount: 100,
  reason: "Verified external activity 42",
})
```

Persist the action ID with the source event and reuse the same payload after an uncertain response. Amounts are safe whole-number integers. Backdating uses `effectiveAt` in epoch milliseconds; future dates fail. Deductions cannot make the balance negative or consume reserved points. Read operations such as `management.points.balance(member, balance)` and `history(member, balance)` require `read` permission.

For externally owned totals, dedicate a balance to that source. Read its current total before a `set` command; on an `expectedTotal` conflict, read again and decide whether to issue a new command with a new action ID. A set replaces the whole balance, including any quest awards written there.

## Verify a leaderboard

Publish to test and read through the participant endpoint described in [participant API](https://console.domino.run/docs/reference/participant-api#leaderboard-reads). Award known points to existing test participants and verify the expected ordering, shared ties, signed-in `me`, and access after sign-out. Confirm that retrying one external event does not award twice. For a period board, check an entry inside and outside the window. Report the tested project, environment, board, and observed scores.

Treat top-rank prizes as a separate reward workflow with an explicit tie policy and finalization time. Board exclusion changes eligibility only. A board leaves out the members of its exclusion group: its own generated group (key `domino--exclusions--<board ID>`), created by its first exclusion, or a group several boards share once an operator binds it with `management.leaderboards.bindExclusionGroup(board, { group, reason })`. `management.leaderboards.exclude` and `exclusions` change and read that group, as `management.groups` does; `exclusionGroup(board)` names it. Console or management calls with `publish` permission handle exclusions and [group](https://console.domino.run/docs/concepts/groups) membership; binding an exclusion group is a definition, made in test and carried to live by a promotion. A group bound as an exclusion group cannot be archived (409 `group-in-use`), and keys starting with `domino--exclusions--` are reserved. Exclusions and group membership are live operator controls: a personal API key with `publish` can change them in live, and Console **Audit history** records each change under the key's owner. A `domino login` session cannot change them in live. Rows show board aliases until the project enables participant profiles (`POST /management/v1/profile-settings`, `publish`); moderating a member's profile (`POST /management/v1/members/<id>/profile`) needs `review`. See [profiles](participant-api.md#profiles).

## Referral programs

For `defineReferral`, invitation qualification, fixed rewards, and selective inviter bonuses, read [referrals](referrals.md). Register modules in the manifest's `referrals` array; they publish atomically with quests and leaderboards. Referral bonuses use the existing project points ledger. Inviter, invitee, and milestone reward lists accept points and tiers. A Discord role there fails preview with `Referral programs can grant points and tiered rewards; Discord roles need a quest`; put the role on the qualifying quest. A `badge()` or `groupMembership()` there makes `defineReferral` throw and preview refuse it, naming the reward kind; put it on the qualifying quest too.

## Discord role rewards

Import `discord` from `@domino-sdk/relay/authoring`. Declare `rewards: [discord.grantRole(roleId)]`. Install `discord-role-reward` for a complete prerequisite-gated example. Its resource setting uses `discord.roles({ assignable: true })`; ordinary eligibility roles use `discord.roles()`.

The ledger commits one durable delivery per completion and role. Delivery retries separately from quest evaluation using an idempotent role assignment. Never write to Discord inside evaluate. Pending delivery is bound to the original account/server, so reconnecting a different identity cannot redirect it. Restoring the original connection and permissions permits recovery. The bot needs Manage Roles and a role above the target. Managed/everyone roles cannot be reward roles.

Use `relay.me.rewardDeliveries()` for the current member or `management.rewardDeliveries({ member })` for operator inspection. Both answer a page of up to 500 deliveries, newest first (`limit` sets fewer); pass the answer's `nextCursor` back as `cursor` for older ones until it is `null`. Each record contains quest, completion, action, role, status, attempts and retry details. The pages also hold quest announcements: `action` is `grant-role` for a role and `post-message` for an announcement (its role is empty), so show only `grant-role` rows as roles. `management.discordRoles.list()` (`GET /reward-deliveries/roles`, `read` permission) answers every role the project's deliveries have granted in the environment, however old, as `{ roles: [{ roleId, quests, deliveries: { pending, delivered, failed } }] }`. Status is `pending`, `delivered` or `failed`. Relay retries transient Discord errors for about a day. A delivery fails at once when Discord reports a permanent problem: an unknown role, a member who left the server, or missing bot permissions. After fixing the cause, call `management.retryRewardDelivery(id)` with `review` permission. For revoked grants this reconciles removal without restoring the grant. `management.discordRoles.preserve(id, { reason })` (deprecated name: `management.preserveRewardRole`) marks an independently owned assignment as external, preventing automatic removal across shared projects/environments. Console exposes both actions. See [role recovery](https://console.domino.run/docs/build/integrations#recover-or-preserve-a-role) for CLI routes and ownership limits. Participant UI should show pending/granted/failed status without operator error details. Referral reward definitions still accept points and tier rewards; qualify a role-granting quest if composing them.

### Discord role delivery

People's guide: [Grant a Discord role](https://console.domino.run/docs/build/integrations#grant-a-discord-role).

- **Recorded with completion.** Completion records a delivery request atomically with the other rewards. A Discord outage does not erase the completion or duplicate points.
- **Retried with backoff.** Console's member details and Rewards page show whether a role is granting, retrying or granted.
- **Bound to the original account.** Delivery stays bound to the original Discord account and server. A different connection cannot receive an old award. Restoring the original account/server and permissions lets pending delivery retry.
- **One membership across projects.** Relay serializes role changes by Discord server, account and role. All accepted, unrevoked completions across projects and Test/Live environments contribute to the desired membership. Revoking one completion cannot remove a role still granted by another.
- **Reconciled after revocation.** When the last grant is revoked, Relay removes assignments it owns. It preserves assignments already present before its first grant, including older grants without ownership evidence. The original `status`, `deliveredAt`, `dispatchedAt` and `revokedAt` remain delivery history; `reconciliation` reports the current provider state.
- **Verified after changes.** Relay reads membership before and after each change, checks again after 30 seconds, and checks settled memberships hourly. Discord calls have a 10-second timeout. A late provider effect can temporarily disagree with the desired state; later checks repair it. Provider failures retry up to 33 times, with permanent failures stopping earlier.
- **Recovery.** A Discord permission rejection while reading membership stops that retry round, as a rejected role write does. Reconciliation after revocation never restores that completion's grant.
- **Preserving an external assignment.** Preservation transfers removal ownership across every project and environment sharing that Discord membership. Discord does not identify an independent assignment to an already-present role, so Relay cannot discover that ownership change itself. Preservation prevents automatic removal until the role is absent and there are no accepted grants.

## Announcements

`announcements` (beside `rewards`; an array or a function of `{ settings }`) lists `telegram.postMessage({ chat, text, replyToTriggeringMessage })` or `discord.postMessage({ channel, text, replyToTriggeringMessage })` (declare `discord`). Relay posts after the rewards commit, as a `post-message` reward delivery. Placeholders and rules: [Telegram guide](https://console.domino.run/docs/build/telegram#announce-tips-in-the-group).

## Reward delivery internals

- **Role reconciliation.** A Discord role reward commits its delivery request with the completion and retries delivery independently, bound to the original account and server. One writer reconciles each server, account and role against accepted grants across all projects and environments. It removes only Relay-owned assignments after the last justification disappears; preexisting or explicitly preserved assignments stay.
- **Redemption ordering.** An offer redemption reserves one declared unit and the member's points before granting the tier entitlement; a failed redemption spends nothing and takes no stock. Redemption and cancellation use the same member payout order as referral reconciliation, so a concurrent referral payout or correction waits instead of partially applying the offer's point spend or refund.

## Hosted server applications

Quest discovery and private resource artifacts work the same way for static and Worker apps. Keep quest definitions outside the public asset directory. For server-rendered applications, configure the Worker module output and browser asset output separately in `relay.json`; see [hosted apps](hosted.md). With `app.routing="transparent"`, staging reserves only its own `/__domino/` paths (see [hosted apps](hosted.md)). Omitting the setting preserves legacy routing. The starter owns its `/relay/*` proxy as ordinary application code; custom handlers can take precedence.

Ledger names can repeat. Edit names and descriptions in Console, or call `management.pointLedgers.update(ledger, { name, description })` to update an existing ledger in test; in live it fails with 422 `promotion-required` for every credential, Console included, so rename a ledger in test and promote. `settings.pointLedger` also renders a searchable picker; existing text settings need a new type version and instance upgrade.

## Project calendar

Each project environment has one timezone that sets where days, weeks and months begin for budgets, engagement budgets, streaks and calendar leaderboard windows. It is UTC until set. Declare it in `relay.json` as `"calendar": { "timeZone": "Europe/Budapest" }`, or change it in Console **Settings → General → Project calendar**, or with `management.calendar.configure({ timeZone, reason })` (`domino api POST /calendar body.json` with a body of `actionId`, `timeZone` and `reason`, `publish` permission; live refuses it with `promotion-required`, so change the live calendar through a reviewed deployment of relay.json). `management.calendar.read()` (`GET /calendar`) returns `timeZone`, `today`, `nextBoundary`, `pending` and `declared`. Use a named IANA zone; fixed offsets such as `+02:00` or `Etc/GMT-2` are rejected. A deployment applies relay.json's zone only when it differs from the last deployed declaration, so a later Console change stays until relay.json is edited; the deploy preview's `calendar` shows the scheduled change or `null`. No quest, budget, streak or board names its own zone.

A change takes effect at the next midnight of the current zone and never rewrites history. The open day, week or month finishes in the old zone; the next one ends at the new zone's next boundary, so it can be shorter than usual. Streak days stay consecutive across the change. Saving another zone replaces a pending change; saving the current zone cancels it.

## Shared SDK runtime

A quest or quest type bundle imports the SDK instead of carrying its own copy. When the installed `@domino-sdk/relay` ships a runtime, `domino build`, `deploy`, `dev` and hosted builds leave `@domino-sdk/relay/authoring`, `@domino-sdk/relay/schema` and `zod` out of each bundle. `zod` stays out only when your app resolves the version the SDK runtime carries and the quest imports it as `zod`; subpaths such as `zod/v4` or `zod/mini` are bundled into the quest. The bundle records the runtime it imports, and the deployment carries that runtime once. Relay stores it by content and runs every release with the exact runtime it was built against, so upgrading the SDK never changes a published quest. A quest bundle shrinks from about 170 KB to a few kilobytes. Other imports, such as a chain library, are still bundled into each quest that uses them.

An older SDK release that ships no runtime builds self-contained bundles, as before. A Relay that predates SDK runtimes refuses a deployment that carries one; `domino deploy` then says to update the Relay, such as restarting an outdated local stack. Trackers, leaderboards, referrals, engagement rules, offers, badges and point ledgers always bundle their dependencies.

## Quest and collection presentation

A quest's `presentation` callback returns optional `steps`, `image`, `imageAlt` and `requirements` alongside instructions, video and quiz. Each ordered step is `{ title, description?, action?: { kind: "link", url, label } }`; link actions require HTTPS. Images use HTTPS or a same-site `/` path. Steps and account/wallet hints never replace evaluator verification.

Use `requirements: [{ kind: "account", provider: "google" }, { kind: "wallet" }]` for hints not derivable from `connections`. Account providers use the existing sign-in providers; wallet hints may include `chain` with an existing wallet family. Participants read these through `view.presentation.requirements`.

`relay.json` collection declarations accept `{ key, title, description?, image?, imageAlt? }`. Console's published placement controls each collection's quest order. Deploying explicitly authored optional metadata replaces its current value; omitting it retains the current value.

Console edits presentation on both type instances and code-authored quests. `management.quests.updateSettings({ quest, expectedRelease, set: {}, reset: [], presentation: { steps: [], image: "/image.png", imageAlt: "Entrance" } })` replaces steps, image and account hints together while retaining instructions, video and quiz. Omitted fields clear the corresponding override; `presentation: null` restores the code defaults. These overrides persist across deployments. Type instances carry the same optional `presentation` object in a catalog draft's quest changes; omitting it restores the type defaults. Collection draft changes carry their optional description/image fields with title and ordered `quests` IDs. Use the existing preview and publish workflow.

## Badges

Define a project achievement, then attach it to quest rewards alongside points or Discord roles:

```ts
import { defineBadge, badge, defineQuest, manual, decision, settings, once } from "@domino-sdk/relay/authoring"

export const firstStep = defineBadge({
  key: "first-step",
  name: "First step",
  description: "Complete your first activity",
  image: "https://example.com/badges/first-step.png",
})
export const welcome = defineQuest({
  key: "welcome",
  title: "Join an activity",
  settings: settings.object({}),
  completion: once(),
  trigger: manual(),
  rewards: [badge(firstStep)],
  evaluate: async () => decision.accept(),
})
```

Include the definition in `relay.json` through a module entry or inline data. Module entries accept `exportName` when the file exports several resources:

```json
{
  "badges": [{ "entry": "./quests/welcome.ts", "exportName": "firstStep" }],
  "quests": [{ "entry": "./quests/welcome.ts", "exportName": "welcome" }]
}
```

Run `domino deploy --dry-run` to check the bundle, then `domino deploy --yes` to publish it to test. Then run `domino promote` and have a person approve the promotion in Console for live. Promotion copies definitions and rewards, not member awards. Redeploying a key updates its name, description and image without changing identity or reopening an archived badge.

Each member holds a badge once. A completion retry cannot duplicate it. Completion revocation removes only that completion's claim; other quest or manual claims keep the badge. Manual revoke removes all current claims. A new legitimate completion can award the badge again. Archiving prevents new reward choices and manual grants while preserving existing awards and published quests.

Use Console **Rewards** to create, edit, archive or restore definitions, or these CLI commands:

```sh
domino badges create first-step --name "First step" --image https://example.com/badge.png
domino badges list --json
domino badges update first-step --name "First activity" --image https://example.com/badge.png --reason "Clarify the achievement"
domino badges archive first-step --reason "Retire this achievement"
domino badges unarchive first-step --reason "Resume this achievement"
domino badges member MEMBER --json
domino badges award MEMBER first-step --reason "Verified by a reviewer"
domino badges revoke MEMBER first-step --reason "Correct an award"
```

Definitions require `publish` permission; live refuses direct definition changes from every credential, Console included, with `promotion-required`, so live badges come from a promotion from test. Manual awards and revokes require `review` permission and a reason, and work in test and live. The management client exposes `management.badges.list/create/update/archive/unarchive/member/award/revoke`. CLI writes retain their operation ID for uncertain-answer recovery.

Badges are independent achievements. Apps compose tiers and levels from badges, points or ledgers. In SDK 1.x, the serialized reward uses the existing `kind: "tier"` envelope with `reward.badge`; use `badge(...)` to author it rather than constructing that envelope. Quest participant reads include the badge record at `reward.badge`.

## Group membership rewards

`groupMembership(group)` is a quest reward that adds the member to a [group](https://console.domino.run/docs/concepts/groups). Use it so approving an application is the one step that admits someone to a curated program, such as approved creators that an `inGroup(...)` prerequisite gates:

```ts
import {
  defineGroup,
  groupMembership,
  defineQuest,
  settings,
  manual,
  evidence,
  once,
  decision,
} from "@domino-sdk/relay/authoring"

export const creators = defineGroup({ key: "approved-creators", title: "Approved creators" })

export default defineQuest({
  key: "creator-application",
  title: "Apply to the creator program",
  settings: settings.object({}),
  trigger: manual({ input: evidence.url() }),
  completion: once(),
  rewards: [groupMembership(creators)],
  review: () => ({ enabled: true }),
  async evaluate() {
    return decision.review({ reason: "Check the applicant's portfolio" })
  },
})
```

- **The group.** Publishing creates the group if the environment lacks it, empty and titled from the handle: a deployment in test, an approved promotion in live. A release whose reward names an archived group fails with 422 `validation-failed` (409 `conflict` on promotion approval); unarchive it first. Publishing never renames an existing group or changes its members.
- **Delivery.** When the completion's rewards are delivered (on acceptance, or when a reviewer approves), Relay adds the member as an operator's add does: the audit history records `group-member-added` with the quest and completion as the reason, prerequisite-only automatic quests the member can now enter are checked, and a member already in the group stays as they were.
- **Revocation.** Revoking the completion (`management.completions.revoke`) removes the membership only when nothing else backs it: no operator added the member and no other unrevoked completion's group reward did. The result's `groups` lists `{ group, stillMember }`. An archived group keeps the member. Operators can always remove members in Console **Groups** or with `domino groups remove`.
- **Archiving.** A group that the current release of a quest that is not archived rewards cannot be archived: 409 `group-in-use`, with `quests` naming them. Remove the reward or archive the quest first.
- **Reads.** The reward uses the stable `kind: "tier"` envelope with `reward.group` (the key) in SDK 1.x; author it with `groupMembership(...)`. Participant quest reads show it with `label` set to the group's title and `reward.group` as `{ key, title }`, using the title the handle gives. It never becomes a tiered-reward entitlement. Only quest rewards can add members: offers, referral programs and leaderboard prizes refuse it.

## Dashboards

Default-export `defineDashboard({ key, title, description?, defaultRange?, position?, sections })` from `@domino-sdk/relay/authoring` and add `{ "entry": "dashboards/growth.ts" }` to `dashboards` in `relay.json`, with optional `exportName`, at most 20 entries. Each section has an optional `title` and its `panels`; each panel has `key`, `title`, optional `description`, `metric` (an ID from `domino analytics metrics`), optional `breakdown`, optional `filter` (`{ quest: ["a", "b"], group: "vip" }`), `ledger` for points metrics, `chart` (`number`, `line`, `bar` or `table`) and optional `width` (1 to 4 on a 4-column grid). `defaultRange` (`7d`, `30d`, `90d` or `all`) defaults to `30d`; `width` defaults to 1 for `number`, 2 for `line` and `bar`, and 4 for `table`. `position` orders dashboards in lists, those without one last, then by title. The types accept only the breakdowns, filters and charts the panel's metric allows, and a `ledger` or filter key can be a key or a handle from a `define…` function.

```ts
import { defineDashboard } from "@domino-sdk/relay/authoring"

export default defineDashboard({
  key: "growth",
  title: "Growth",
  sections: [
    {
      panels: [
        { key: "sign-ups", title: "Sign-ups", metric: "members.signUps", chart: "number" },
        { key: "xp", title: "XP awarded", metric: "points.awarded", ledger: "xp", chart: "line" },
        { key: "by-quest", title: "Completions by quest", metric: "quests.completions", breakdown: "quest", chart: "table" },
      ],
    },
  ],
})
```

`domino deploy` refuses a dashboard with a 422 and its own code (see [errors](errors.md)): an unknown metric (`unknown-metric`); a breakdown or filter the metric doesn't take, or a `number` panel with a breakdown (`breakdown-not-accepted`, `filter-not-accepted`); a `table` panel without one (`breakdown-required`); a chart the metric can't draw (`chart-not-accepted`); a points metric without `ledger`, or `ledger` on another metric (`ledger-required`, `ledger-not-accepted`); a quest, tracker, ledger, offer, referral program or leaderboard key the environment doesn't have (`unknown-reference`); a referral metric in a project with no referral program (`referral-program-required`); two dashboards, or two panels of one dashboard, with one key (`duplicate-key`); more than 24 panels (`too-many-panels`); or more than 20 dashboards that aren't archived (`too-many-dashboards`). Archived resources are valid references, and their history keeps counting. Groups are not checked at deploy: a panel filtered on a missing group reads `group-not-found` while the other panels read.

The preview's `dashboards` lists each one with `change`: `created`, `updated`, or `unchanged`, which publishes no new version. Omitting a dashboard never deletes it; archive it with `domino dashboards archive KEY --reason …` in test. Deploying an archived dashboard's code leaves it archived. A promotion carries dashboards to live by key, with no environment bindings. Read one back with `domino dashboards get KEY` to check its numbers; see [dashboards](automation.md#dashboards).
