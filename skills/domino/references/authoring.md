# Authoring and publication

Read the installed `@domino-sdk/relay/authoring` declarations and the project's existing quest modules. Define the requested behavior with the supported SDK helpers rather than inventing capabilities. Authored entries in relay.json resolve relative to that manifest and can select a named export with `exportName`.

The `discover.quests` and `discover.questTypes` arrays in relay.json select default-exported resource files. New projects use `quests/**/*.quest.ts` and `quests/**/*.quest-type.ts`. Add matching files without individual registration. Existing manifests without `discover` keep explicit-entry behavior. Omitting either discovery array or setting it to `[]` disables discovery for that resource kind while preserving explicit entries. An explicit entry overrides the matching discovered default export; omitted providers inherit `discover.provider` for those files. Discovery does not execute authored modules on the build host, publish them, or archive omitted resources.

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

`domino check --json` includes a `diagnostics` array with `code`, `severity`, `file`, `line`, `column`, `message`, and `suggestion` for source findings. `AUTHORING_ID_DEPRECATED` and `AUTHORING_BALANCE_RENAMED` are warnings for renames `migrate --write` applies. `AUTHORING_IDENTITY_CONFLICT`, `AUTHORING_LEDGER_CONFLICT`, `AUTHORING_KEY_MISSING`, `HANDLE_REFERENCE_REQUIRED`, `MANUAL_MIGRATION_REQUIRED`, `BUDGET_DECLARATION_CONFLICT`, and `AUTHOR_BUDGET_TRACKER_REQUIRED` fail the check. Dynamic references and missing local ledger declarations are warnings because the selected ledger may already exist remotely. A `defineLedger` handle, defined in the same module or imported, is a static reference to its key, and a ledger published in `pointLedgers` as `{ "entry", "exportName" }` counts as a local declaration. A ledger read from a setting is dynamic. Manifest validation errors appear on stderr with `code: "MANIFEST_INVALID"` and field paths in `details`; a relay.json that is not valid JSON reports `MANIFEST_INVALID` without them. A `--config` file that does not exist reports `PROJECT_NOT_FOUND`, and a quest, type, engagement rule or image that fails to bundle reports `BUNDLE_FAILED`. Each exits 1. Agents should follow the reported suggestion and rerun the check; never invent a replacement key for an existing resource.

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

Add `{ "entry": "src/supporter-badge.offer.ts" }` to `offers` in `relay.json`. `title` defaults to the reward's title. `stock` (required, 0 to 1,000,000, never unlimited) is the total supply: increasing it adds units and decreasing it succeeds only while enough unredeemed units remain. `limit.perMember` (1 to 10,000) defaults to 1. Use `staffHandover()` instead of `automaticFulfillment()` when staff must confirm physical delivery.

To limit who can redeem, give the offer `requires`: the prerequisites quests take, `completed(quest)`, `balanceAtLeast(ledger, minimum)` and `inGroup(group)`, at most 30 (see [prerequisites](#campaigns-windows-and-prerequisites)). For example, `requires: [inGroup(members), balanceAtLeast(community, 1000)]` sells only to group members holding at least 1,000. A balance prerequisite is a threshold, not a cost: the price is spent separately, and both read the balance when the member redeems. Publication refuses a `completed` quest or a ledger the deployment doesn't know. A required group the environment doesn't have is created, empty, with the handle's title: deploying creates it in test and approving the promotion creates it in live (see [groups](#campaigns-windows-and-prerequisites)). Relay checks the prerequisites when the member redeems and refuses a member who misses one with 409 `prerequisites-incomplete` and `missing`, the same shape quests use, spending no points or stock. Meeting them later reserves nothing, and leaving a group later leaves earlier redemptions alone. Offer reads for a signed-in member list what they lack in `missing`, so an app can show the offer locked and why; otherwise `missing` is empty.

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

The management SDK generates an `actionId` when a keyed mutation omits it, including point changes, offer controls, reward corrections, enrollments, and resource creation. Typed methods whose input has an `actionId` send the same request again after a transient failure (see [Management client](#management-client)). If a write still fails, the `ManagementError`'s `operationId` is that action ID, to retry the same input explicitly; see [Management client](#management-client). For retries across process restarts, generate and persist an action ID before calling the helper. Direct HTTP creation requests still require `actionId`.

Use `actionId` to make a creation request retryable. Reuse the same action ID and payload after an uncertain response; a different payload with the same action ID fails. The action ID is not the resource ID. Provider subjects, tracker record IDs, completion keys, and nested quiz, tier, budget, and referral-rule keys keep their own scopes.

For pickup inventory, supply a label for each new variant and omit `item`; saving returns its assigned item ID. Reuse that ID to share existing stock. For standalone stock, type a name in the Console Stock item picker and choose **Create item “NAME”**, or use `management.inventory.createItem({ name })`. Both return a server-assigned item ID. Stock adjustments accept existing item IDs only. Pickup configuration and item creation are definitions, so live refuses them with `promotion-required`.

## Resource and date/time settings

Build every quest setting with a `settings.*` builder (`integer`, `text`, `boolean`, `choice`, `datetime`, `resource`, `pointLedger`, `quiz`, `telegramChat`, `amount`, `chain`) inside `settings.object({ ... })`. A plain zod schema, such as `z.number().default(1)`, is a type error, and `defineQuest` throws `Setting "<key>" is not a quest setting` for untyped callers: Console edits only what the builders describe.

Use `settings.resource({ label, source: discord.channels() })` for a Discord channel selector. Import `discord` alongside `settings` from `@domino-sdk/relay/authoring`. The connected project integration supplies the server; never add a guild ID setting. Filter with `discord.channels({ types: ["announcement"] })` when only announcement channels qualify. The default includes text and announcement channels. Resource values are stable IDs, not labels.

Omit the resource default until the operator chooses a value in Console. Its empty string represents missing configuration. Code-authored quests publish paused when any resource setting is empty. Publishing a configured value activates the code-authored quest; clearing it pauses the quest again. Console-created quests must have every required resource configured before activating. Existing operation-level pauses remain independent. Live refuses settings changes with `promotion-required`, so in live an empty resource setting cannot be filled: the quest stays paused unless the deployed source supplies a default.

Use `settings.amount({ label, default, decimals? })` for a token amount: a non-negative base-unit integer string without leading zeros, such as `"1000000000000000000"`, exact at any scale. With `decimals`, Console shows and edits it in whole tokens (`formatUnits`/`parseUnits`). A tracker filter may compare with it (`gte: { setting }`). Use `settings.chain({ label, default })` for a chain `ctx.wallet.balance()` reads: `"solana"`, `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp"`, or an EVM CAIP-2 ID such as `"eip155:8453"`; its value is typed as the balance query's `chain`. Use `settings.nft({ label, default })` for an NFT collection `ctx.wallet.nfts()` reads, such as `"eip155:1/erc721:0x…"` or `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/nft:<collection mint>"` ([NFT holdings](#nft-holdings)). Use `settings.contractCall({ label, default })` for a contract function `ctx.wallet.call()` reads, such as `"eip155:8453:0x… function stakedOf(address) view returns (uint256)"` ([Contract state](#contract-state)). Use `settings.contractEvent({ label, default })` for an EVM contract event `ctx.wallet.events()` reads ([On-chain actions](onchain.md#evm-events)). Use `settings.solanaProgram({ label, default })` for a Solana program `ctx.wallet.transactions()` reads ([Solana transactions](onchain.md#solana-transactions)). All six are text settings (`picker: "amount"`, `"chain"`, `"nft"`, `"contract-call"`, `"contract-event"` or `"solana-program"`, and `decimals`), and publication and Console edits refuse other values; an invalid `default` throws when the quest module loads.

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

To take test's definitions to live, run `domino promote --environment test --json` (`management.promotions.prepare()` with a test client; publish permission in test). It compares every definition in test with live by authoring key, code-defined and Console-authored alike, and saves a pending promotion: `changes` lists each `kind`/`key` live would change as `added`, `changed`, `archived` or `unarchived`; `warnings` lists `binding-missing` integrations, webhook endpoints, sign-in methods and app secrets live lacks, which a promotion never copies, and `live-only` definitions it leaves alone; `build` is the hosted app build serving staging, which it carries. Live already matching fails with `promotion-empty`. Nothing changes in live until a person signed in to Console with publish permission chooses **Approve and apply** in **Live → Deployments**; an agent cannot approve. Console's **Promote to live** prepares the same promotion. Approval applies it in one transaction and fails with `promotion-changed` when test or live changed since it was prepared; then prepare again. Preparing again supersedes an older pending promotion. A leaderboard bound to a group needs a group with that key in live. `domino promotions list|get` read them; `management.promotions.rollback(id)` with a live client prepares a promotion of an earlier applied promotion's definitions and build, archiving every live definition that promotion did not carry except offers. The CLI saves `domino promote` before sending it; after an uncertain response, run the error's `operations resume` command. Existing attempts retain their pinned releases and promised rewards. When the promotion carries a hosted app build, approval deploys it as the live app; `management.liveApp.get()` (`read`, either environment) answers its address and `state` (`none`, `deploying`, `live` or `failed` with `failure`). An organization admin can serve the live app at their own domain with `management.domains.add(hostname)` and `list`, `get(id)`, `recheck(id)` and `remove(id)` (`read` lists them); see [custom domains](hosted.md#custom-domains).

Code reaches live only through a promotion. Live refuses `domino deploy --environment live`, with or without `--preview`, and `management.deployments.preview` with a live client: 422 `promotion-required` from every credential, before the bundle is checked, which the CLI reports as `ENVIRONMENT_UNSUPPORTED`. Deploy to test, then promote. Do not use `--resume` for live approval. `domino deploy --promote TEST_DEPLOYMENT_ID --environment live` checks that the test deployment was published, then prepares a promotion of all of test, answered as a pending deployment record whose first warning, code `environment-promotion`, names the promotion.

Fixture providers simulate verification. `workers-ai` requires a configured service binding. A fixture success demonstrates the flow but does not establish that a real external activity was verified.

## Leaderboard definitions

Default-export `defineLeaderboard({ key, title, ledger, score, access, group, includeUnscored, settle })` from `@domino-sdk/relay/authoring`. Pass handles from `defineLedger`, `defineQuest`, and `defineGroup`; add `{ "entry": "src/leaderboard.ts" }` to `leaderboards` in `relay.json`. Optional `exportName` selects a named export. Deploy through the existing project bundle workflow. Use `domino deploy --preview` to inspect definitions and standings before publishing to test.

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

Score defaults to `{ kind: "balance" }`. Use `earned({ window, quests })` for an earned score, or `{ kind: "net", window }` for net changes. `earned` takes quest handles. Windows are `all-time`, `fixed` with epoch-millisecond `start`/`end`, `rolling` with `days`, or `calendar` with `period: "day" | "week" | "month"`: the current period of the [project calendar](#project-calendar), so it matches the project's budgets. A calendar window takes no `timeZone`. Access defaults to `participants`; `public` allows anonymous reads through the configured project endpoint. A group handle narrows eligibility. `includeUnscored: true` also lists eligible members with no qualifying points, at score 0 and one shared rank after every member who scored, ordered by member ID; settlements never include them.

`settle` is available only on fixed or calendar earned/net boards. Prize ranges must be ordered and cannot overlap. Rewards are points or tiers from `defineTieredReward`; a prize cannot repeat the same point ledger or tiered reward. `shared` awards every member sharing a winning rank. `earliest` orders equal scores by the last qualifying entry that produced the score, then member ID. At cutoff Relay freezes the board version, eligible rows, exclusions (the exclusion group bound at the cutoff, with its members then), ranks and assigned prizes. Review and deliver the frozen result in Console **Leaderboards**, or read it with `management.leaderboards.settlements(board)` and approve it with `management.leaderboards.approveSettlement(settlement, { reason })` (`review` permission). A board that counts an engagement rule's points (the rule pays into its balance and, with `quests`, qualifies through one) first takes a closing measurement: its settlement reads as `pending` with no rows and a `closing` field (`startedAt`, `deadline`) while Relay asks scheduled trackers to resample the content, a sample observed within 30 minutes after the cutoff counts as measured at it, and the settlement freezes when everything is measured or the wait ends, listing content frozen at its last good sample in `unmeasured` (`content`, `member`, `observedAt`). Approval refuses a closing settlement (`settlement-closing`); `retrySettlementMeasurement(settlement)` or `domino leaderboard-settlements retry-measurement` closes it again for another 30 minutes. Each winner receives a completion; its prize points earn no referral bonus. `settle.pool` (optional; `prizes` may then be empty, or left out in `defineLeaderboard`) splits a fixed amount over the frozen eligible rows in proportion to score: `{ amount: "<base units>", unit: "points" | { code, decimals }, ledger?, minScore? }`. `ledger` (a ledger handle or key) is required for and only allowed on a points pool, which holds at most 2,147,483,647 points. Members scoring 0 or less, or at or below `minScore`, get nothing; each pool share is `floor(amount × score / total)` in bigint, and leftover base units go one each to the largest remainders, earlier `position` first, so shares sum exactly to the pool. With no eligible score the whole pool is `unallocated`. Settlement reads carry `pool` (`amount`, `unit`, `allocated`, `unallocated`, …) and each row's `share` in base units. Approval pays a points pool's shares as points into its ledger in each member's prize completion (merged into a rank prize's points for that ledger; revocation reverses them). Any other unit moves nothing: Relay never holds or moves tokens, and `management.leaderboards.settlementExport(id)` or `domino leaderboard-settlements export SETTLEMENT` (CSV; `--json` for rows with `share`, formatted `amount` and verified `wallets` `{ chain, network, address }`) is the operator's payout list; a rejected settlement's export lists every share as 0. Correct one with `management.completions.revoke({ completion, reason, freeOccurrence: false })`; leaderboard prize occurrences cannot be freed and paid again.

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

## Management client

Management SDK calls send `X-Domino-Client: @domino-sdk/relay/<version> (<runtime>)`. `createManagementClient` also takes `fetch`, which replaces the global `fetch` for every call (a test double, a proxy's or an edge runtime's own), and `headers`, added to every call; a `raw` call's own headers and the client's authorization, scope and content-type headers replace one of the same name, and a `getToken` that returns `null` sends no `Authorization` header at all. Response readers ignore added fields at every depth. Legacy methods retain closed 1.x return types.

Use `client.response(path, schema, input?, method?, call?)` for an extensible response. It accepts branded tolerant schemas, sends `X-Domino-Response: tolerant-v1`, and returns `{ status, operationId?, result }`. `managementResponses` from `@domino-sdk/relay/server` provides supported/unsupported envelopes for every named management response. `questCatalogResponseSchema` from `@domino-sdk/relay/schema` preserves unsupported catalog entities individually. `responseListSchema(entitySchema)` retains entity-list positions and reports `unsupportedCount`.

Unknown requirements or policies keep the entire entity as `{ kind: "unsupported", raw }`. Never act on an unsupported financial result or guess its outcome. Persist a write's `actionId` before sending; `response` writes are sent once and return that identity when the input supplies it. Older clients and legacy methods in the newest SDK still need their supported wire shapes.

`createManagementClient` from `@domino-sdk/relay/server` (or `hostedManagementClient(env)` in a hosted Worker) has a typed method for each management route an app or script needs. Use `client.request(path, schema, input?)` or `client.raw(path, init?)` only for routes without one, such as build event streams. From a terminal or an agent, the typed `domino <noun> list|get|…` commands call these same methods and print their results, such as `domino points award` for `points.record` or `domino trackers ingest` for `trackers.ingest`; each change is a saved operation whose `--operation-id` becomes its `actionId`. See [resource commands](automation.md#resource-commands); prefer them to `domino api`.

Definitions change only in test. In live, every call that writes one fails with a `ManagementError` of status 422 and code `promotion-required`, whatever the credential: quest publication, settings and archive state, the Quest Studio catalog and drafts, point ledgers, badges, the calendar, profile settings, pickup policy, a leaderboard's exclusion-group binding, stock item creation, renames and archive state, and every other archive or unarchive. Operations work in live under the caller's permissions: pauses, points, groups, moderation, reviews, reward corrections, settlements, stock adjustments and stopping or reverting a member import (`memberImports.stop(id)`, `memberImports.revert(id, { reason })`; applying a live import stays Console-only, see [member imports](imports.md)). Do not retry a `promotion-required` refusal; make the change in test, then run `domino promote`. See [test and live](https://console.domino.run/docs/concepts/environments).

`client.storage.list()`, `domino storage list --json` or `GET /management/v1/storage` reads hosted app storage in test with `read` permission. It returns declared and retained stores with their kind, binding name, declaration status, and creation time, never Cloudflare IDs. With `publish`, `client.storage.tables(name)`, `rows(name, table, { cursor })`, `cell(name, table, { row, column, cursor })`, `keys(name, { prefix, cursor })` and `value(name, key)` browse contents read-only (`cell` reads one SQL value in full where rows cut it), and `delete(kind, name)` removes a store the serving build no longer declares. `ManagementClient` types `storage` as optional, so code that implements the type keeps compiling; clients from `createManagementClient` always have it. Declare native KV and D1 bindings in `relay.json` `app.storage`; see [hosted storage](hosted.md#app-storage).

| Method                                                                                       | Route                                                        | Permission                         |
| -------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------- |
| `members.get(id)`, `members.people(ids)`                                                     | `GET /members/<id>`, `GET /people?ids=`                      | `read`                             |
| `members.lookup({ provider, subject })`                                                      | `GET /members/lookup`                                        | `read`                             |
| `x.lookup({ username })` | `GET /x/users/lookup?username=` | `read` |
| `members.progress(id)` | `GET /members/<id>/progress` | `read` |
| `members.identityStates.list(id)` | `GET /members/<id>/identity-states` | `read` |
| `members.identityStates.unlink(id, input)`, `relink(id, input)`, `setSignIn(id, input)` | `POST /members/<id>/identity-states/{unlink,relink,sign-in}` | `review` |
| Deprecated: `members.oauthAccounts`, `unlinkOAuthAccount`, `telegramAccount`, `unlinkTelegramAccount`, `unlinkWallet`, `setWalletSignIn`, `unlinkIdentity`, `relinkIdentity`, `identityLinks` | Their 1.x routes | As before |
| `attempts.confirm({ quest, member })`                                                        | `POST /attempts`                                             | `confirm`                          |
| `attempts.review(id, { generation, decision, reason })`, `attempts.retry(id)` | `POST /attempts/<id>/review`, `…/retry` | `review` |
| `attempts.acceptAnyway({ attempt, reason })` | `POST /attempts/<id>/accept-anyway` | `review` |
| `completions.revoke({ completion, reason, freeOccurrence })`, `completions.restore({ completion, reason })` | `POST /completions/<id>/revoke`, `…/restore` | `review` |
| `passes.verify({ provider, event, subject, member, quest })`                                 | `POST /passes/verify`                                        | `confirm`                          |
| `handover.begin(entitlement, { revision })`, `handover.confirm(entitlement, { handover })`   | `POST /entitlements/<id>/begin`, `…/confirm`                 | `handover`                         |
| `handover.cancel(entitlement, { revision, reason })`                                        | `POST /entitlements/<id>/cancel`                             | `handover`                         |
| `entitlements.void(entitlement, { revision, reason })`, `entitlements.restore(…)`, `entitlements.correctCollection(…)` | `POST /entitlements/<id>/void`, `…/restore`, `…/correct` | `review` |
| `quests.pause/resume(quest, { revision, reason })` and `redemptions.pause/resume({ revision, reason })` | `POST /quests/:id/pause` or `/resume`, and `/redemptions/pause` or `/resume`                                           | `publish`                          |
| `groups.list({ cursor, limit })`, `groups.get(group)`, `groups.members(group, { cursor, limit })`, `groups.member(group, member)` | `GET /groups`, `GET /groups/<key>`, `…/members`, `…/members/<id>` | `read`                             |
| `groups.create({ key, title })`, `groups.update(group, { title })`, `groups.archive(group, { reason? })`, `groups.unarchive(group, { reason? })` | `POST /groups`, `POST /groups/<key>/update`, `…/archive`, `…/unarchive` | `publish`                          |
| `groups.addMember(group, { member, reason })`, `groups.removeMember(group, member, { reason })` | `POST /groups/<key>/members`, `…/members/<id>/remove`        | `publish`                          |
| `inventory.addStock({ item, quantity })`                                                     | `POST /stock`                                                | `inventory`                        |
| `offers.list()`, `redemptions.history({ member, cursor, limit })`                           | `GET /offers`, `GET /redemptions/history`                    | `read`                             |
| `offers.control(id, { revision, enabled, reason })`                                         | `POST /offers/<id>/control`                                  | `inventory`                        |
| `offers.cancelRedemption(id, { revision, reason })`                                         | `POST /redemptions/<id>/cancel`                              | `inventory`                        |
| `quests.updateSettings({ quest, expectedRelease, set, reset, solanaIdls? })`                 | `POST /releases/settings`                                    | `publish`; test only               |
| `tokens.list()`, `tokens.create(input)`, `tokens.scope(id, input)`, `tokens.revoke(id)`                                 | `/tokens`                                                    | a person's session, never a key    |
| `grants.list()`, `grants.set({ subject, roles, environments })`, `team.list()`               | `/grants`, `/team`                                           | organization administrator         |
| `identityIntegrations.list()`, `create({ name, origin, expiresInDays? })`, `addSecret(id, { expiresInDays? })`, `revokeSecret(id, secret)`, `removeSecretExpiry(id, secret)`, `revoke(id)`; deprecated `rotate(id)` | `/identity-integrations`                                     | organization administrator, Console |
| `serviceKeys.list()`, `create({ name, permissions, expiresInDays? })`, `addSecret(id, { expiresInDays? })`, `revokeSecret(id, secret)`, `revoke(id)` | `/service-keys`                                              | organization administrator, Console |

For test scenarios, `management.testMembers.history(member, { since }?)` (`domino test-members history MEMBER [--since MS]`) reads attempts with their ledger-credited `eligibleAt` and skipped tracker outcomes with their reason `code` and `occurredAt`. It needs `publish` permission in the test environment, like the other [test-member methods](#test-members). See [Try quests with test members](https://console.domino.run/docs/use/quests#try-quests-with-test-members) for the window and limits.

Methods whose input has an `actionId` generate one when you leave it out. Every method takes an optional last argument `{ signal, timeout }`: `signal` cancels the call and `timeout` is its deadline in milliseconds, which also bounds your `getToken`. The client's own `timeout` option sets the default, 30 seconds unless you choose another. For typed methods the deadline covers every attempt, the waits between them and the answer's body; for `raw` it covers only the wait for the response, so read a stream with your own `signal`. Cancelling stops any retry at once and never undoes a write already sent: that write stays `uncertain`.

Typed methods send a call again, at most three times in all, when the server applies a repeat at most once: every read, and every write whose input has an `actionId`, which the server keeps a receipt for and answers with its first result. Every attempt sends the same payload. They retry when no answer arrived, when the answer broke off after its headers, on 500, 502, 503 and 504, and on a 429 whose `Retry-After` fits in the time left, waiting `Retry-After` when the server sent one and otherwise up to 0.25 seconds, then up to 1 second. A timeout, a cancellation, an unreadable answer, rejected input and any other refusal are never retried. Writes without an action ID (such as `secrets.set`, `grants.set` or `pointLedgers.update`), `request` with any method but GET, and `raw` are sent once. If an error body is cut off, its status still controls retries and its `Retry-After` is kept in `retryAfter`. When retries run out, the last failure is the rejection, with `retryAfter` set to the wait in milliseconds the server's `Retry-After` asked for; a write any attempt of which may have been applied stays `uncertain`.

`completions.revoke` takes back an accepted completion in one transaction and resolves with what it did. The completion stops counting for progress, prerequisites of new attempts, counts, profiles, and referral qualification. Completions it already unlocked and budgets it spent stay.

`points` lists each award with the amount `reversed` and the amount `unreversed` because the member already spent or reserved it. Relay persists that remainder for each point ledger on the revoked completion, includes it in the audit event, and never records member debt for it. The member keeps the value unless an operator records a separate `points.record({ kind: "adjust", ... })` correction. Console links the warning to that existing action.

`entitlements` lists each tiered reward it granted as `voided` (a handover in progress is cancelled first), `downgraded` to the best tier another completion grants, `kept`, or `flagged` when it was already collected. `deliveries` lists the completion's original Discord role grant outcomes. A delivered one is `flagged` for asynchronous membership reconciliation. A pending one is stopped as `cancelled`, or as `in-flight` when a call may already have reached Discord and can still land. A failed one is `not-delivered`. `stillGranted` says whether another accepted completion across any project or environment also grants the role to the same account and server. Read the delivery's `reconciliation` for current provider state. Relay removes only its own assignments after the last grant is revoked.

`referrals` lists programs that qualified on the completion as `unqualified` (the inviter's reward is reversed) or `requalified` on another completion of the quest. `streakMilestone` is true when the revoke removed a streak milestone. The days stay. `source` (`{ kind, id }`, as on `completion.created`) says whether it was a quest completion or a leaderboard prize, whose `quest` carries the board ID. `freeOccurrence: true` lets the member complete the quest again for the same completion key, which pays again. Unreversed points stay with the member. By default, later attempts for the occurrence are rejected.

Outcomes are open strings. The known values are `RevokedEntitlementOutcome`, `RevokedDeliveryOutcome`, and `RevokedReferralOutcome`. A second revoke fails with 409 `completion-revoked`. An unknown completion fails with 404. Find a completion's ID on its accepted attempt (`completion`) or in a member's `snapshot.completions` from `members.get(id)`.

`members.get(id)` also returns every active quest release with its evaluator code in `snapshot.releases`, megabytes in a project with many quests. Code that only needs a member's quest state reads `members.progress(id)` (`domino members progress MEMBER`): `member` and `quests` as `members.get(id).progress` has them, the 25 most recent `attempts`, newest first, and point `balances`. Its lists are bounded by the project's configuration: each quest's `streak` leaves out its `days`, and rewards, completion records and streak days stay in `members.get`. Read linked wallets with `members.identityStates.list(id)`.

`members.get(id).revocationShortfalls` returns up to 25 revoked completions with a non-empty point remainder, newest first. Each item contains `completion`, `quest`, `revokedAt`, and `unreversedPoints` keyed by point ledger ID. `revokedCompletions` lists up to 25 revoked ones; `completions.restore` re-applies a revoke's recorded result. `referral` is the member's accepted referral, `{ inviter, acceptedAt, suspended, import }` (`import` names the [member import](imports.md) that recorded it, else null), or null when nobody invited them; Relay versions before imported referrals leave it out.

Missing completion reward history returns `409 completion-grant-history-missing`, naming the completion whose history Domino support must restore before retrying. Only older completions whose original record could not be read lack it. Shared tier grants also require history for every other accepted completion of the member. The refusal precedes member-state changes. Collected entitlements retain `recovery: { completion, reason, flaggedAt, remainingTier }` in normal reads until an operator corrects collection. Correction voids the reward when `remainingTier` is null, or makes the remaining earned tier available, and records the resolved flag in history. A later grant updates eligibility without clearing the pending recovery.

Entitlement corrections take the `revision` you read and a `reason`, and Console **Audit history** records each one. `handover.cancel` returns a `handing-over` entitlement to `available`, `entitlements.correctCollection` does the same for one recorded `collected` by mistake, and both put a pickup size back into stock. `entitlements.void` turns an `available` entitlement `void`; a quest that grants the same reward to the member again makes it `available`, as a fresh grant at that quest's tier; a voided referral reward stays void until restored. `entitlements.restore` (CLI `domino entitlements restore`) undoes an operator's void, at the tier an accepted completion still earns and with the referral reviews the void settled. 409 `entitlement-status` refuses it when a revocation voided it or no accepted completion earns it (`completion` names what to restore), after a redemption cancel, and for a referral reward voided before voids kept reviews. An entitlement whose status this SDK does not know reads as `"unsupported"` with `reportedStatus`; offer no action on it. A stale `revision` fails with 409 and code `entitlement-changed`, and the wrong starting status with 409 `entitlement-status`. These are member-state corrections, so an API key with the permission can make them in live.

`members.lookup({ provider, subject })` finds the one member of the client's project holding exactly that account, with `provider` and `subject` as a member's `accounts` list them, such as `discord` and a Discord user ID. It never matches part of a value or another letter case. When nobody holds the account it fails with 404 and code `member-not-found`; when several members do, with 409 `account-ambiguous` and their IDs in `details.members`. Use it where a trusted server knows an external account and needs the member, instead of searching `members.list({ q })`.

An account Relay reads in the background with the member's own OAuth tokens also carries `status`: `connected` while the tokens work, `needs-reconnect` once they stopped, with a `reason` of `revoked`, `expired` or `missing-scope`. Other accounts carry neither field. Only the member clears `needs-reconnect`, by linking the same account again; see [participants](https://console.domino.run/docs/concepts/participants#accounts-domino-reads-in-the-background).

`members.list({ q?, needsReview?, limit?, cursor? })` (`GET /members/list`, `domino members list`) needs `read` and returns `{ items, total, nextCursor }`: the project's members, most recently joined first, each with `accounts`, `attempts`, `completions`, `rewards` and `lastActivity`, and `total`, how many match. `q` matches part of a member ID, a profile's display name, or an account's subject, name or email, ignoring letter case, or a profile's public ID exactly. `needsReview: true` keeps members with an attempt waiting for review. `limit` is 1 to 100, 25 by default. Pass `nextCursor` back as `cursor` for the next page until it is `null`; a cursor works only with the `q` and `needsReview` it came from, and any other answers 400 `invalid-cursor`.

Other reads of a project's state need `read` and page the same way, newest first, with `total` beside `items` (limit 1 to 100, 25 by default; a malformed cursor answers 400 `invalid-cursor`):

- `attempts.list({ q?, quest?, status?, limit?, cursor? })` (`GET /attempts`): `q` matches part of the member, quest or attempt ID; `quest` keeps one quest's attempts, by its ID. Beside `items`, `releases` are the releases the attempts were admitted under and `people` the members they name, with their accounts. `attempts.get(attempt)` (`GET /attempts/:attempt`) answers `{ attempt, release, events, person }`: its release (null if gone), up to 200 recorded events newest first, including a revocation of the completion it reached, and its member; an unknown attempt is 404.
- `entitlements.list({ q?, status?, limit?, cursor? })` (`GET /entitlements`): `q` matches part of the member ID or reward title; `people` sit beside `items`. `entitlements.get(entitlement)` answers `{ entitlement, person }`, or 404.
- `audit.list({ category?, q?, limit?, cursor? })` (`GET /audit/list`): reviews, reward changes, configuration and operators acting as test members (`category` keeps one: `reviews`, `rewards`, `configuration` or `test-members`) in the client's environment; `q` matches part of the actor, action, details or attempt ID. `audit.access({ q?, limit?, cursor? })` (`GET /audit/access`) pages the project's access and API key changes across test and live, for an organization administrator's Console session or CLI sign-in only.

Release history needs `publish` as well as `read`, because releases carry each quest's code. `releases.list({ quest?, limit?, cursor? })` (`GET /releases/list`) answers `{ items, nextCursor }`: every release the project's quests have had, current and earlier, newest first, without `total`. `quest` keeps one quest's releases, by its ID or key, and a cursor works only with the `quest` it came from. `releases.get(release)` (`GET /releases/:release`) reads one release by ID, current or historical, or answers 404.

Small collections come whole: `metrics.read()` (`GET /metrics`) answers the project's counts (`quests`, `members`, `attempts`, `completions`, `review`, `failed`, `entitlements`) and `completionsByQuest`, each quest's counting completions as `{ quest, completions }`; `quests.list()` (`GET /quests/list`) each quest's current release, most recently published first, archived quests included; `operations.list()` (`GET /operations`) every paused or resumed operation; and `inventory.list()` (`GET /stock-items`) every stock item with `available`, `reserved` and `allocated` counts and the `offers` that sell it.

`metrics.read()` counts all time. For the same records over time, `analytics.read({ range, ledger? })` (`GET /analytics?range=7d|30d|90d|all`, `read` permission, `domino analytics get`) answers each metric's total, its value per project-calendar day or week, and the previous range's total; see [project analytics](automation.md#project-analytics) for what each metric counts.

`usage.activeMembers()` (`GET /usage/active-members`, `read` permission, `domino usage active-members`) answers the project's live active members this UTC month against its plan: `{ month, activeMembers, plan, limit, history }`, where `limit` is null when the plan has none and `history` holds the six earlier months, newest first. The count is live only, whichever environment the client selects. An active member sent a signed-in participant request on a UTC day of the month after the day they joined; see [active members](https://console.domino.run/docs/concepts/participants#active-members).

`page(name, query)` reads Console's former screens and is deprecated: compose these reads instead (`releases.list` and `releases.get` replace its `releases` page). It is removed in 2.0.0.

Every failure is a `ManagementError`, including input the client rejects before sending. Check `kind` first:

| `kind`             | Meaning                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `http`             | The server refused the call. `message` and `code` are the server's own.                  |
| `network`          | The connection failed, before the response or while its body, an error body included, was read. |
| `timeout`          | The deadline passed, retries included.                                                   |
| `aborted`          | Your `signal` cancelled the call.                                                        |
| `invalid-response` | The server answered, but not with JSON this client version recognizes.                   |
| `client`           | Nothing was sent: the input or options were invalid, or your `getToken` failed.          |

`status` is the HTTP status whenever the response headers arrived, including a `network` or `timeout` failure while the body was read; otherwise it is absent. With kind `http`, `code` is the server's own code, including ones newer than your client, so compare it only against codes you handle; otherwise it is the client's (`network`, `timeout`, `aborted`, `invalid-response`, `invalid-input`, `credential`). `details` holds the rest of the error body when there is any, such as `moved` for `PROJECT_MOVED`, or `{ record, path, reason }` when `trackers.ingest` refuses a batch with `invalid-record-data`. `method` and `path` name the call once a request was formed. `ManagementError` extends the SDK's `RequestError`, the same core as the participant client's `RelayError`; `isManagementError(error)` and `isRequestError(error)` recognize them across copies of the SDK.

When `uncertain` is true, a write may have been applied: it was sent and then timed out, was cancelled or lost its connection, was answered 500, 502 or 504, or was answered 2xx with a body this client cannot read. A 503, a 429, another refusal, a failure while `getToken` runs, a connection that was refused or never resolved (so nothing was sent), and every read are never uncertain. `operationId` is the `actionId` the request carried, and `recovery` says what to do: `replay` means repeat the same call with the same payload and `operationId`, which the server applies at most once; `reconcile` means read the resource before deciding. Generic `request` and `raw` writes report `reconcile` even when the caller supplies an `actionId`, because the client cannot know whether that route keeps receipts. Typed writes without an action ID also require reconciliation.

## Verify a leaderboard

Publish to test and read through the participant endpoint described in [participant API](https://console.domino.run/docs/reference/participant-api#leaderboard-reads). Award known points to existing test participants and verify the expected ordering, shared ties, signed-in `me`, and access after sign-out. Confirm that retrying one external event does not award twice. For a period board, check an entry inside and outside the window. Report the tested project, environment, board, and observed scores.

Treat top-rank prizes as a separate reward workflow with an explicit tie policy and finalization time. Board exclusion changes eligibility only. A board leaves out the members of its exclusion group: its own generated group (key `domino--exclusions--<board ID>`), created by its first exclusion, or a group several boards share once an operator binds it with `management.leaderboards.bindExclusionGroup(board, { group, reason })`. `management.leaderboards.exclude` and `exclusions` change and read that group, as `management.groups` does; `exclusionGroup(board)` names it. Console or management calls with `publish` permission handle exclusions, bindings and [group](https://console.domino.run/docs/concepts/groups) membership. A group bound as an exclusion group cannot be archived (409 `group-in-use`), and keys starting with `domino--exclusions--` are reserved. These are live operator controls: a personal API key with `publish` can change them in live, and Console **Audit history** records each change under the key's owner. A CLI sign-in changes nothing in live. Rows show board aliases until the project enables participant profiles (`POST /management/v1/profile-settings`, `publish`); moderating a member's profile (`POST /management/v1/members/<id>/profile`) needs `review`. See [profiles](participant-api.md#profiles).

## Referral programs

For `defineReferral`, invitation qualification, fixed rewards, and selective inviter bonuses, read [referrals](referrals.md). Register modules in the manifest's `referrals` array; they publish atomically with quests and leaderboards. Referral bonuses use the existing project points ledger. Inviter, invitee, and milestone reward lists accept points and tiers. A Discord role there fails preview with `Referral programs can grant points and tiered rewards; Discord roles need a quest`; put the role on the qualifying quest.

## Verify Discord membership

Use the `discord-membership` snippet for a server membership check, with an optional role requirement. Connect the server first: Console **Integrations**, or `domino integrations discord connect`. Declare `integrations: [discord]` on the quest and call `await ctx.discord.member()` inside `evaluate`. There is no server ID setting. The result is `null` when the participant is not in the server or has not linked Discord, or `{ userId, roles, joinedAt, pending, nickname, displayName, boostingSince }`. Author code decides eligibility and the quest's declared rewards.

The platform supplies a read-only client bound to the project's configured server and the participant's Discord identity. Quest code cannot supply a user ID, server ID, access token, or arbitrary URL. For reads that need the project connection, a missing connection, unavailable bot configuration or removed bot access throws a non-retryable `CapabilityError` with code `integration-required`. Membership without a linked Discord identity still returns `null`. If the error escapes quest code, the attempt becomes `failed` immediately with that code and a participant-safe reason. Repair the connection before retrying; an explicit retry rechecks the failed read. Temporary Discord outages still use retryable `provider-unavailable`. These failures do not prove the participant is ineligible. Publication doesn't require the connection: a quest that needs Discord (declared, a role reward, or a Discord resource setting) publishes. A deployment's preview and result list it in `warnings` with code `integration-required`, and `domino deploy` prints the warning; other publication paths return none. When Discord becomes ready (first connection, or repair from needs attention), Relay admits a new attempt in the background for each member of an active automatic quest without a tracker that needs Discord whose latest attempt failed with `integration-required`; tracker-triggered attempts need an explicit retry. Role and channel selections are checked against the server only while it is connected. A quest that does not declare the integration gets `author-error`. Bot access problems appear in Console Integrations. Participants signing in with Discord need no extra permission flow; members using another login link their Discord identity once.

The evaluation run pins the participant's Discord account (or that they have none) and the server at its first Discord read and never rechecks them: linking, unlinking or reconnecting afterwards does not change that run, so only a new submission or scheduled deferred run uses a new connection. Reads, including absent membership, are captured and replayed on recovery. Membership screening (`pending`) and role requirements remain author decisions.

The campaign starter includes participant identity controls. Existing apps use the same APIs through `createParticipantBackend`; see [participant API](participant-api.md#connected-accounts). The managed bot is configured by the platform, never by a quest author. The injected Discord client supports `member()` and `message({ channelId, messageId })`. Discord messages and reactions reach quests as tracker records; see [Discord trackers](#discord-trackers). Discord role grants are declared rewards, never evaluator writes. X reads use the separate `ctx.x` capability; see [X verification](#x-verification).

The release's `provider` setting names the AI review backend, and only `workers-ai` publishes; it does not select the evaluator for Discord, quizzes, or other authored checks. `humanReview` allows quest code to return a review decision. It does not automatically route Discord failures to staff. The Discord membership snippet returns accept or reject decisions and leaves provider outages as errors.

## Campaigns, windows, and prerequisites

Use `defineCampaign({ key, title?, window?, enrollment? })` and pass the result as a quest's `campaign`. `enrollment` is `"none"` by default or `"required"`; `window` is `{ from, until }` in epoch milliseconds. A quest can also declare its own `availability: { from, until }`. Both windows are half-open (`from <= time < until`), and both must admit the activity. When both exist, publication requires them to overlap. Every active quest with one campaign key must carry the same campaign definition.

Required campaign enrollment keeps one durable first timestamp shared by every current and future quest with that campaign key. A participant uses `relay.campaigns.enroll({ actionId, campaign })`; a server uses `management.campaigns.enroll({ actionId, campaign, member })` with `enroll` permission. A participant can pause new admissions with `relay.campaigns.withdraw({ actionId, campaign })`; a server uses `management.campaigns.withdraw({ actionId, campaign, member, reason })`. Withdrawal retains the first enrollment time, attempts, completions, and rewards. Calling `enroll` again restores the enrollment. A campaign that does not require enrollment returns `enrollment-not-required`; one that has ended returns `outside-window` on enrollment or restoration.

`requires` accepts typed prerequisites:

```ts
import {
  balanceAtLeast,
  completed,
  defineGroup,
  defineLedger,
  defineQuest,
  inGroup,
  manual,
  once,
  settings,
} from "@domino-sdk/relay/authoring"

const community = defineLedger({ key: "community", name: "XP" })
const veterans = defineGroup({ key: "veterans", title: "Veterans" });
const welcome = defineQuest({
  key: "welcome",
  title: "Welcome",
  settings: settings.object({}),
  trigger: manual(),
  completion: once(),
  rewards: [],
})
const dailyCheckIn = defineQuest({
  key: "daily-check-in",
  title: "Daily check-in",
  settings: settings.object({}),
  trigger: manual(),
  completion: once(),
  rewards: [],
})

requires: [
  completed(welcome),
  completed(dailyCheckIn, { scope: "period", period: "day" }),
  balanceAtLeast(community, 100),
  inGroup(veterans),
]
```

An omitted completion scope means `"ever"`. A period completion uses the current `"day"`, `"week"`, or `"month"` of the project calendar; name the period explicitly. Balance and group rules are live eligibility observations. `defineGroup(...)` creates a typed project-group reference for `inGroup(...)`, `groupMembership(...)`, `defineLeaderboard({ group })`, and `defineReferral({ inviterGroup })`. Publishing code that names a group the environment doesn't have creates it, with no members: a deployment creates it in test, and approving a promotion creates it in live. It takes the handle's `title`, or its key when only a leaderboard or referral program names it, and the audit history records `group-created` with the deployment or promotion as its reason. The deployment preview and result, and the prepared promotion, list each such group as a `group-created` warning. Publishing never changes an existing group's title or members, never unarchives one (a `groupMembership` reward naming an archived group fails with 422 `validation-failed`, or 409 `conflict` on promotion approval), and never creates a `domino--exclusions--` key. Maintain members in Console **Groups**, with `domino groups add <key> <member> --reason <text>`, or with `management.groups.addMember(group, { member, reason })`; create a group ahead of code with `domino groups create <key> --title <title>` or `management.groups.create({ key, title })`. Reads need `read`; changes need `publish`. A group is archived, never deleted: its members keep counting, but cannot change until it is unarchived. `management.leaderboards.group` is deprecated. See [groups](https://console.domino.run/docs/concepts/groups).

Quest views expose each missing prerequisite as a discriminated value with `kind: "completion" | "balance" | "group"`. They expose a campaign, effective window, and budget capacity too. See [campaigns and eligibility](https://console.domino.run/docs/concepts/campaigns).

## Automatic and repeatable quests

`automatic()` without arguments runs once a member completes its prerequisites: it needs at least one `requires` entry and completes `once()`, and TypeScript rejects a missing or empty prerequisite list or a different completion policy. Untyped callers receive the same validation before publishing. A reusable `defineQuestType` may leave prerequisites for its Console instances; every published instance still needs at least one. For activity Domino collects or your backend reports, use `automatic({ tracker })`; see [tracker triggers](#tracker-triggers). Manual and automatic triggers are exclusive. Automatic quests never expose participant claim or retry actions. Set `visibility: "hidden"` to omit a quest from participant lists while it continues earning rewards. Set `profile: { visible: true }` to list its completions on participant profiles (`relay.profiles.completions`); they stay off by default because a completion can reveal holdings or activity. The active release decides, and archived quests never appear. A `visibility: "hidden"` quest with `profile: { visible: true }` exposes its title and completions on profiles.

Choose `once()` for one successful completion per participant, or `keyed(ctx => key)` for one per author-defined key. Write the policy as a function of the builders, `completion: ({ keyed }) => keyed(({ tracker, settings }) => tracker.record.id)`, to type the key's context from the quest: `tracker` is non-null and typed for a tracker trigger, and `settings` has the quest's settings. A policy built inline, `completion: keyed(...)`, still works, but its `tracker` is nullable and untyped and it has no `settings`. The builders also include `once`, `streak` and `progress`. The key is scoped to the project, quest, and participant. It must remain stable across deployments. For announcement reactions use the message ID; for a daily quest use `ctx.calendar.day().key`, the project-calendar key for `ctx.eligibleAt`. `completed(quest)` means ever completed unless it has an explicit period scope.

Only `keyed(...)` uses an evaluator's key: `decision.accept({ completionKey })` or `decision.review({ completionKey })` takes precedence over its key callback. The other policies ignore it: `once()` and `streak(...)` key their own completions, `progress(...)` keys each action with `distinct(...)`, and a content quest's key is its content item. `runQuest` reports such an ignored key in `run.diagnostics`, with code `completion-key-ignored`, and the run is otherwise unchanged.

`ctx.eligibleAt` is fixed when admitted: the record's `occurredAt` for a tracker trigger, the admission time otherwise. A tracker record's ID and `observedAt` deduplicate delivery; they are not the completion key. Only an accepted completion consumes a key. A rejected attempt stays rejected, but a later record can try an unconsumed key.

Declare budgets with `budget({ key, scope: "member" | "project" | "author", period: "day" | "week" | "month" | "lifetime", limit, cost })`. `"author"` counts per tracker record author's account (`kind:subject`), linked or not. Use `cost: 1` for completion counts or `cost: { ledger: community }` for the awarded points amount. Quests can share a named budget, with matching scope, period, limit, and cost: they share its usage, so declare it once and import it in each quest. `domino check`, `domino build` and `domino deploy` refuse quests that declare one budget key differently, naming both quests and the fields that differ; `check` reports it as `BUDGET_DECLARATION_CONFLICT`. The check reads literal values, so a budget built from a setting or computed value is compared at deploy, where Relay refuses the same conflict. All budget spends, the completion key, and rewards commit together. Day, week and month budgets use the [project calendar](#project-calendar) period containing the eligibility time; weeks are ISO weeks starting Monday (a bucket such as `2026-W40@Europe/Budapest`). An attempt that finds its budget exhausted is rejected permanently; it does not roll into tomorrow. Concurrent valid awards commit in whichever order wins the transaction.

`"author"` needs a tracker trigger (`AUTHOR_BUDGET_TRACKER_REQUIRED`) and caps what one giver gives, such as a reactor tipping a message's author. Descriptions list it in `authorBudgets`, not `budgets`; member budget views omit it; a record with no author is refused; imports don't count. A tip capped both ways, on `automatic({ tracker: reactions, beneficiary: "messageAuthor" })`:

```text
budgets: [
  budget({ key: "tips-received", scope: "member", period: "day", limit: 20, cost: 1 }),
  budget({ key: "tips-given", scope: "author", period: "day", limit: 10, cost: 1 }),
]
```

See [cap what each giver gives](https://console.domino.run/docs/build/trackers#cap-what-each-giver-gives).

Admission pins the release and prerequisite result. Pausing stops new admissions while accepted work finishes. Republication does not clear successful completion keys. Execution errors of automatic attempts retry with capped backoff and retain the original evidence and release. Points accumulate per completion. Tier rewards retain their existing entitlement upgrade behavior; repeating a quest does not create a new consumable handover.

Install `discord-announcements` for the first integration example. Its quest triggers from a `discord.reactions()` tracker and checks the configured channel, the emoji and `data.messagePublishedAt` in quest code. The gateway records the reaction addition; do not query current reactions to cancel it. Unseen replayed events with unknown timing are skipped and reported as coverage gaps. Manual Discord membership remains `ctx.discord.member()`. Use `discord.grantRole(roleId)` for a declared role reward; no arbitrary Discord writes are exposed. See [Discord trackers](#discord-trackers), and X verification below for X reads.

A trusted customer backend reports activity as records of a custom tracker: declare `customSource(namespace, { identities, data })` and send records with `management.trackers.ingest(tracker, { actionId, records })` using a key with `ingest` permission. Name the member through the account the activity came from (`wallet`, `x` or `discord`), or your app's own user ID as an `app:<name>` identity linked by your [app login](https://console.domino.run/docs/build/app-login) integration, so Relay attributes it through the member's verified link, or with a `member` identity when the backend vouches for the member ID. Tell kinds of activity apart with a `data` field that the trigger's `where` selects, or with separate trackers. Ingestion never joins the member to the project: a trigger skips an unjoined member with `not-joined`. Use stable record IDs on retries and server-side action times, never browser claims. Matching quests can grant real rewards from these records; keep the key on the server.

## Republishing and automatic quests

Publishing changed rules never rewrites work already admitted:

- **Admitted attempts keep their release.** Tracker-triggered attempts keep the release selected at admission, including its reward amount and budget rules.
- **Nothing is reset.** Publishing changed rules does not reset completion keys or reconsider skipped records. Rejected attempts stay final except when a prerequisite-only automatic attempt lost a claim-time budget race; that attempt can retry under the current release after another member event or reviewed operator recovery (see [prerequisite automatic outcomes](#prerequisite-automatic-outcomes)).
- **Pausing stops new admissions.** Already admitted work can finish.

## Reject with a code

`decision.reject({ reason })` rejects the attempt with a reason the member reads. Add `code` to name the cause with a stable key:

```ts
import { decision, defineQuest, manual, once, settings } from "@domino-sdk/relay/authoring"

export const launchQuiz = defineQuest({
  key: "launch-quiz",
  title: "Launch quiz",
  settings: settings.object({}),
  trigger: manual({ input: "quiz" }),
  completion: once(),
  presentation: () => ({
    quiz: [
      {
        id: "launch-year",
        prompt: "When did we launch?",
        choices: [
          { id: "2024", label: "2024" },
          { id: "2025", label: "2025" },
        ],
      },
    ],
  }),
  rewards: [],
  async evaluate({ input }) {
    return input.answers["launch-year"] === "2025"
      ? decision.accept()
      : decision.reject({
          reason: "Review your answers and try again.",
          code: "wrong-answers",
        })
  },
})
```

| Rule | Detail |
| --- | --- |
| When to use | One code per distinct rejection cause. Keep the code when you reword the reason, so rejections for that cause stay together. Rejections without a code behave as before. |
| Format | Lowercase letters, digits and hyphens, at most 64 characters, such as `wrong-answers`. |
| Reserved | Domino writes these codes itself, so a reject cannot use them: `budget-exhausted`, `defer-expired`, `defer-limit-exceeded`, `evaluator-deferred`, `evaluator-progress`, `imported`, `integration-required`, `not-linked`, `provider-unavailable`, `author-error`, `evaluator-error`, `capability-limit`. Relay checks the decision against its current list, so if it reserves another code later, a bundle built with an older SDK that rejects with it fails the attempt instead of storing a code that reads as Relay's. |
| Invalid code | The evaluation throws an authoring error naming the rule, such as `Invalid reject code "Wrong answers": use lowercase letters, digits and hyphens, at most 64 characters`, and fails like any uncaught error: the attempt fails once with code `evaluator-error`, and `runQuest` in the testing kit rejects with an `EvaluatorError` carrying the same message. |
| Where it shows | The attempt's `code` in `management.attempts.get` and `management.attempts.list`, `domino attempts get` and `domino attempts list`, and Console's attempt detail beside the reason. |
| Members | Never. In participant responses, such as quest progress, the quest experience and its collections, and submission replays, a rejected attempt carries a code only when Domino rejected it itself (`budget-exhausted`, `defer-expired`, `defer-limit-exceeded`), and `code: null` otherwise. Attempts in other states keep their codes, such as a failed attempt's `integration-required`. |

A content quest's `performance.evaluate` takes the same `code`. Staff rejections from review carry no code. `decision.review` takes none either, so a review request on a quest with human review off becomes a rejection without a code.

## Defer evaluation or report progress

Return `decision.defer({ reason, retryAfterMs?, until? })` when evidence may qualify later. Return `decision.progress({ current, target, reason })` to report an absolute value and check again in one minute. These are decisions from the quest's `evaluate`, not its content performance gate. Neither decision completes the quest, spends a budget, or grants a reward. Even `current === target` waits until a later `decision.accept()`.

| Setting | Rule |
| --- | --- |
| `retryAfterMs` | Integer milliseconds from 1,000 to 3,600,000. Default 60,000. Out-of-range values fail validation. |
| `until` | Absolute Unix milliseconds. Default and maximum are 24 hours from the first deferred commit. A later defer can shorten this deadline but cannot extend it. |
| Number of deferrals | At most 100 per attempt, counting both defer and progress. Another non-terminal decision rejects with `code: "defer-limit-exceeded"`. |
| Deadline | At or after the deadline, the next wake-up or evaluation commit rejects with `code: "defer-expired"`. The timer wakes at the earlier of the requested time and deadline. |
| Progress | Finite numbers with `0 <= current <= target` and `target > 0`. The value replaces the previous value; it is never an increment. |

Relay commits the result under the attempt's generation lock. Temporal waits on a durable timer, then starts a new evaluation run on the same attempt and pinned release. The new run gets a fresh frozen clock, random stream, capability captures, linked-account snapshots, and tracker watermark. Retries within that run reuse its captures and clock. The original submission, admitted tracker observation, beneficiary, and `ctx.eligibleAt` stay unchanged. For tracker progress that should include later activity, pass `asOf: ctx.now`; the default `ctx.eligibleAt` still selects the original activity time.

Derive progress from captured readers. Do not read previous progress and add to it. A rollback exposes no progress, a replay does not increment it, and a stale generation cannot overwrite it. The existing capability-error capture and escape rules apply separately within each run. Captures from earlier runs remain stored.

Participants receive `attempt.status: "deferred"`, a reason, `nextEvaluationAt`, and the open string `code`, normally `evaluator-deferred` or `evaluator-progress`. `QuestView.progress` contains the latest attempt's committed `{ current, target, reason }` while that attempt waits or evaluates again. A defer without new progress retains the last value. Resolution, review, or failure clears it; a newer attempt supersedes it. `questStatus()` reports a waiting tone and the corresponding evaluator code. Unknown codes need a fallback.

Review stops scheduled evaluations and uses the normal staff decision path without a defer deadline. An operator with `review` can stop a deferred attempt through `management.attempts.review(id, { generation, decision: "reject", reason })`; they cannot accept a deferred attempt before evaluation. Pause and archive stop new admissions, while admitted deferred attempts continue under their pinned release. To stop one, reject it. There is no completion or reward to revoke during defer. Later acceptance uses the ordinary budget and reward rules at the original activity time, and later reward reversal remains an explicit operator action. The same rules apply in test and live.

While an attempt is deferred, a new manual submission for the same member and quest returns `409` with code `attempt-pending`. Reusing the original action ID with the same request still returns that attempt. Wait for its scheduled evaluation or have an operator reject it before submitting again. Manual completion keys are computed during evaluation, so this guard also applies to keyed quests.

## Set a retry cooldown

Set `retryCooldownMs` on a quest members submit to make them wait after a rejection before trying again, for example so a quiz cannot be solved by guessing:

```ts
import { defineQuest, evaluators, manual, once, settings } from "@domino-sdk/relay/authoring"

export const launchQuiz = defineQuest({
  key: "launch-quiz",
  title: "Launch quiz",
  settings: settings.object({}),
  trigger: manual({ input: "quiz" }),
  completion: once(),
  retryCooldownMs: 300_000, // five minutes
  presentation: () => ({
    quiz: [
      {
        id: "launch-year",
        prompt: "When did we launch?",
        choices: [
          { id: "2024", label: "2024" },
          { id: "2025", label: "2025" },
        ],
      },
    ],
  }),
  rewards: [],
  evaluate: evaluators.quiz({ "launch-year": "2025" }),
})
```

| Rule | Detail |
| --- | --- |
| Value | Integer milliseconds from 1,000 (one second) to 604,800,000 (seven days). Out-of-range values fail when the quest is defined. Omit it for no cooldown. |
| Quests | Manual quests members submit. A staff-confirmed or automatic quest with a cooldown fails validation. |
| Starts | When the evaluator rejects the member's latest attempt (including a `decision.review` on a quest with review off) or a reviewer rejects it. Rejections for an exhausted budget, a revoked completion or an expired or over-limit deferral start none, and neither does a failed execution. |
| Refusal | A new attempt by that member on that quest returns `409` with `code: "retry-cooldown"` and `retryAt`, Unix milliseconds on the member's clock. This covers the participant's submissions and claims, and a test member's actions in Console. |
| Reads | `QuestView.retryAt` (and each quest in `/v1/me/progress`) is set while the cooldown runs; `availability` stays `available`. `questStatus(view)` reports `retry-cooldown` with `retryAt`, `canSubmit(view)` is false until then, and `useQuest` renders again when it ends. |
| Changes | The current release's value applies, including to members already waiting. An accepted attempt ends it. |

Console shows the cooldown on the quest. A quest made in Console has its own **Retry cooldown** field, which starts from its type's `retryCooldownMs`; a new quiz starts at five minutes. `retryCooldownMs` is not `decision.defer({ retryAfterMs })`, which schedules the next evaluation of an attempt that is still undecided.

## Set a completion cooldown

Set `completionCooldownMs` on a repeatable quest members submit to space out their completions, for example so a second story counts only 90 minutes after the first was accepted:

```ts
import { decision, defineQuest, keyed, manual, settings } from "@domino-sdk/relay/authoring"

export const eventStory = defineQuest({
  key: "event-story",
  title: "Post a story",
  settings: settings.object({}),
  trigger: manual(),
  completion: keyed(({ now }) => "story:" + now),
  completionCooldownMs: 5_400_000, // 90 minutes
  rewards: [],
  evaluate: async () => decision.accept(),
})
```

| Rule | Detail |
| --- | --- |
| Value | Integer milliseconds from 1,000 (one second) to 604,800,000 (seven days), as for `retryCooldownMs`. Out-of-range values fail when the quest is defined. Omit it for no cooldown. |
| Quests | Manual quests members submit whose completion is not `once()`. A quest that completes once, and a staff-confirmed or automatic quest, fail validation, and publishing one is refused with `422`. |
| Starts | When a completion of the quest by that member is accepted, of any key. A rejection does not start or restart it, and a revoked completion does not count. |
| Refusal | A new attempt by that member on that quest returns `409` with `code: "completion-cooldown"` and `retryAt`, Unix milliseconds on the member's clock. This covers the participant's submissions, their retries of a failed attempt, and a test member's actions in Console. When the retry cooldown also runs, the refusal names whichever ends later, with its code. |
| Reads | `QuestView.nextAvailableAt` (and each quest in `/v1/me/progress`) is set while it runs. A quest that is otherwise open keeps `availability: "available"`, and `questStatus(view)` reports `completion-cooldown` with `retryAt`; a counter that pays once per window reads `completed` until its next window. In both cases `canSubmit(view)` is false until then, and `useQuest` renders again when it ends. |
| Changes | The current release's value applies, including to members already waiting. A test member's clock advance moves past it. |

A quest made in Console has its own **Completion cooldown** field, which starts from its type's `completionCooldownMs`; empty means none.

`nextAvailableAt` is the field to use for countdowns: the latest of the retry cooldown's end, the completion cooldown's end, the next reset of a budget that can't cover another completion, and the next window of a counter that pays once per window. It is omitted when the member can submit now, or when nothing will reopen the quest (a used-up lifetime budget, a window that closes first).

## Evaluation time and randomness

An evaluation can run more than once for the same attempt: after a crash, a timeout, or an operator's retry. Each retry within one evaluation run replays captured capability results, so the evaluator must reach the same decision and completion key. A committed defer or progress schedules a new run with fresh reads. The sandbox makes time and randomness repeat:

| In the evaluator                                                   | Returns                                                                                                                                                                                                                                                       |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ctx.now`                                                          | The evaluation run's start time in epoch milliseconds. Every retry and replay of that run receives the same value. `keyed()` receives it too.                                                                             |
| `Date.now()`, `new Date()`, `Date()`                               | `ctx.now`. The clock does not advance, including across awaited capability calls and in module-scope code. `new Date(value)` and `Date.parse` are unchanged.                                                                                                  |
| `performance.now()`                                                | `0`.                                                                                                                                                                                                                                                          |
| `Math.random()`, `crypto.getRandomValues()`, `crypto.randomUUID()` | Values from generators seeded by the attempt ID and evaluation run: the same sequence on every retry, a different sequence for a scheduled rerun. Anyone who knows that identity can predict them, so they are not secret; never use them for credentials or tokens. |

Use `ctx.eligibleAt` for when the activity happened; day and week budgets count by it. `ctx.calendar.day()` returns the project-calendar day containing that instant as `{ key, date, timeZone, start, end }`, with an inclusive `start` and exclusive `end`. Its key includes the zone, such as `2026-10-01@Europe/Budapest`. Pass another epoch-millisecond instant to `day(at)` when the evaluator deliberately needs its day. Each evaluation run pins the calendar segments with its clock, so a later timezone change cannot alter a retry or replay of that run. A scheduled defer starts a fresh run with the current calendar. Use `ctx.now` for checks as of evaluation, such as whether a settings deadline has passed. A retried attempt reuses its original `ctx.now`, while capability reads that were not captured run fresh, so do not throw a "not yet" error on `ctx.now` hoping a retry will pass later: return `decision.defer` instead, or check `ctx.eligibleAt` when the condition is about when the activity happened. `Intl.DateTimeFormat().format()` without a date still reads the real clock; pass it `ctx.now`. Timers and Web Crypto key generation are not made repeatable; do not let a decision depend on them.

## Test quests locally

Unit-test an evaluator with `@domino-sdk/relay/testing` before publishing. It evaluates the quest the way the hosted runtime does: setting values, rewards and the review switch from the quest's `describe`, `Date`, `Math.random()` and `crypto.randomUUID()` frozen and seeded as in the sandbox, capability requests and results validated against Relay's capability contracts, and the decision's rewards checked against the published bundle. Starters keep these tests in `tests/quests.test.ts` and run them with `pnpm test`; the kit works in any runner.

```ts
import { defineQuest, decision, evidence, manual, once, settings } from "@domino-sdk/relay/authoring"
import { expectReview, runQuest } from "@domino-sdk/relay/testing"

const feedback = defineQuest({
  key: "feedback",
  title: "Suggest an improvement",
  settings: settings.object({}),
  trigger: manual({ input: evidence.text({ minLength: 20 }) }),
  completion: once(),
  rewards: [],
  async evaluate(ctx) {
    const result = await ctx.review.text({ text: ctx.input.value, criteria: "One concrete improvement" })

    return result.kind === "pass" ? decision.accept() : decision.review({ reason: result.reason })
  },
})

const run = await runQuest(feedback, {
  input: { value: "Add a dark mode to the dashboard" },
  facts: { review: { text: { kind: "unclear", reason: "Ambiguous" } } },
})
expectReview(run, "Ambiguous")
```

- **Options.** `settings` (Console values; defaults fill the rest), `input` (`{ value }`, `{ photo }` or `{ answers }`; `kind` is filled and text and URL limits are checked), `tracker` (build it with `trackerObservation(tracker, { data })`), `performance`, `now` (default 2026-01-01T00:00:00Z), `eligibleAt`, `timeZone` or `calendar`, `seed`, `member` and `facts`. `input` on a quest without evidence, and `eligibleAt` on a tracker-triggered quest (it takes the record's `occurredAt`), fail the run.
- **Facts.** Fake answers keyed like `ctx`: `member.identities`, `wallet.list`/`balance`, `x.*`, `discord.member`/`message`, `trackers[key].records`/`summary`/`content`, `images.*` and `review.text`/`photo`. Each is a value or a function of the call's arguments; throw `CapabilityError` to fail a call. `member.identities`, `wallet.list` and `x.connection` are pinned: answered once per run, `null` (nothing linked) when unset. Reads of the member's own accounts follow the pins: `x.profile`/`follows`/`reposted` and `wallet.balance` read `null` while the account is unlinked and need answers once it is linked (a balance on a chain with no linked wallet reads `null`); answering them without the link fails the run. A balance of an unlinked `address` throws `author-error`, and an image check of `{ x: "profile" }` without linked X throws `not-linked`. Every other read needs an answer, and a missing one fails the run with `TestSetupError`, even if quest code catches it. Answers must match the result shapes Relay delivers; `trackerPage(tracker, records)` builds a records answer.
- **Result.** `runQuest` resolves with `{ decision, calls, release }`; `calls` holds each capability request and outcome in order. It rejects with a delivered `CapabilityError` that escaped quest code, thrown or wrapped (Relay retries a retryable one); with an `EvaluatorError` (code `evaluator-error`) for anything else quest code throws, including an uncaught `capability-limit` from a ninth capability call; with `TestSetupError` for the test's own mistakes (a read the facts do not answer, a submission Relay refuses); and with Relay's message for a reward selection outside the published bundle.
- **Assertions.** `expectAccept`, `expectReject`, `expectReview`, `expectDefer` and `expectProgress` take the run and an optional exact or `RegExp` reason, and return the narrowed decision.
- **Scope.** One attempt's evaluation, within Relay's 15-second limit: prerequisites, availability, budgets, completion limits and reward delivery are not run, and nor are Relay's chain and token checks. Module-scope quest code runs at import with the real clock. `createTestContext(quest, options)` returns the host context for calling `quest.evaluate` directly, without the frozen clock or reward check.
- **Replay.** `replayAttempt(quest, capture, { facts? })` evaluates current quest code against one evaluation run of a real test-environment attempt: the release's settings and rewards, the attempt's input, clock, seed and calendar, and each captured capability answer by position. It resolves with `{ decision, recorded, changed, divergence, calls }`; a call the capture cannot answer (`new`, `changed` or `uncaptured`) stops the replay with `decision: null` unless `facts` answers it. Get a capture with `domino quests replay <attempt> --out <file> --json` (which also replays it against the discovered quest with that key, or `--quest <file>`) or `management.attempts.capture(attempt)` (`publish` permission: captures hold raw provider answers). `recorded` is what quest code decided in the run, before Relay amended, claimed or reviewed it. Captures are test-environment only: live is refused with 422 `test-environment-required`, and an unevaluated attempt with 409 `attempt-not-evaluated`.
- **Tests only.** `buildQuest`, and so `domino check` and deploy, refuse a quest bundle that imports `@domino-sdk/relay/testing`.

## Discord trackers

Declare `defineTracker({ key, title, source: discord.messages(), window })` or `source: discord.reactions()` (`discord` is exported by `@domino-sdk/relay/authoring`), list it under `trackers` in `relay.json`, and trigger quests with `automatic({ tracker })`. The project needs a ready Discord integration. The Discord gateway writes each new message or reaction into every tracker of that kind in each project and environment whose ready integration names the server and was connected before the event. `trackers.ingest` refuses a Discord tracker with 403 and a `refresh` control with 422. A paused tracker, or an event outside the window, stores nothing. Messages from bots and webhooks and reactions by bots are not recorded, and nothing is backfilled.

- **Members' activity only.** A record is stored only when at least one of its accounts is a Discord account a member of the tracker's project has linked: the message's poster, or the reactor or the message's author for a reaction.
- **`discord.messages()`.** Record ID: the message ID. `author`: the poster's Discord user ID. `occurredAt`: the publish time, or the receipt time when earlier. `data`: `{ guildId, channelId, messageId, replyTo, publishedAt }` (`DiscordMessageData`), with `replyTo` null for a message that replies to nothing. `discord.messages({ text: true })` adds `text` (`DiscordMessageTextData`), null when the platform bot lacks the privileged Message Content intent (`DISCORD_MESSAGE_CONTENT=true`). An edit updates stored text. `on: "new-observation"` and `on: "data-changed"` can select that edit and run `where` again; the default `new-record` cannot. A message deletion marks the record removed and produces `record-removed`. Deletion, or publishing the tracker without `text: true`, erases text from current records, retained observations, and pinned attempts.
- **`discord.reactions()`.** Record ID: `messageId:reactorId:emoji`, where `emoji` is a custom emoji's ID or the Unicode emoji. `author`: the reactor's Discord user ID. Named account `messageAuthor`: the message author's Discord user ID, absent when Discord's lookup failed. `occurredAt`: the receipt time, since Discord sends no reaction time. `data`: `{ guildId, channelId, messageId, emoji: { id, name }, reactorRoles, messagePublishedAt }` (`DiscordReactionData`), with `reactorRoles` the reactor's role IDs at reaction time. Adding a removed reaction again is a new observation of the same record.
- **Triggers.** The identity kind is `discord`, and the default `on` is `"new-record"`, so each message or reaction runs a quest once. A removal produces a `record-removed` outcome in every mode. Data types are fixed, so `where` paths such as `channelId` and `emoji.name` and `ctx.tracker.record.data` are typed. `where` values are literals; check a Console-chosen setting, such as a channel, in the evaluator.
- **Paying the message author.** `automatic({ tracker: reactions })` pays the reactor; `automatic({ tracker: reactions, beneficiary: "messageAuthor" })` pays the message's author ("reaction received"). A reaction from the message author's account, or from another account of the same member, is skipped with `beneficiary-is-author`, and a reaction without `messageAuthor` with `no-account`. The reactor need not be linked when the message's author is; check `data.reactorRoles` for moderator-only recognition and key by `data.messageId` to pay once per message.
- **Attribution.** Relay pays the member whose selected Discord account is the beneficiary, falling back to the member's only Discord connection when none is selected. An unlinked beneficiary is skipped with `unlinked-account`, a member who has not joined the project with `not-joined`.
- **Streaks.** `streak(...)` works on these triggers: each accepted record fills the calendar day of its `occurredAt`.

`ctx.discord.message({ channelId, messageId })` is server-scoped and returns null for deleted messages; `discordMessageSchema` describes its result. It reads with the bot in the server the attempt pinned, so it needs no linked Discord account. For manual authorship checks, compare message `userId` to a non-null `userId` from `ctx.discord.member()`. Older captured member facts can return null; fail closed. A tracker-triggered quest needs no such check: Relay attributes the record to the beneficiary's member. Captured reads survive evaluation retries.

Install `discord-contribution` for channel participation and `discord-moderator-recognition` for moderator heart reactions; `telegram-contribution` is the Telegram equivalent of `discord-contribution`. Use one snippet per capability; customize predicates in code. Daily budgets are caps, not streaks. Use the streak completion policy below for consecutive calendar days. General multi-action counters remain unavailable.

## Discord Gateway delivery

The API deployment includes a managed Discord Gateway receiver with the `GUILD_MESSAGE_REACTIONS` and `GUILD_MESSAGES` intents. It feeds the Discord trackers above. People's guide: [Where Discord events come from](https://console.domino.run/docs/build/integrations#where-discord-events-come-from).

- **Starting.** Installing the integration starts it in production. Existing installations can start it by reading `GET /management/v1/integrations/discord/events` with a scoped management credential. Local development does not start a real bot connection automatically.
- **Status.** That response contains the connection state, pending deliveries and recent coverage gaps. Console shows them on the Discord card in **Integrations** (**Events**, **Recent coverage gaps**).
- **Shared bot.** The bot is shared by every project, so a project cannot restart it. When the state is `blocked`, Domino has to fix the bot's configuration; it reconnects on its own afterwards.
- **Delivery.** New messages and reactions start delivery as soon as Relay records them. Delivery writes each one into the Discord trackers of every project connected to the server. The receiver drains an indexed queue in bounded batches and schedules failed deliveries with backoff.
- **No duplicates.** Successful delivery removes the captured event. Tracker writes repeat nothing on a retry, and durable session sequence numbers and quest completion keys prevent duplicate rewards.
- **Resuming.** Once started, the receiver resumes automatically when it wakes after a restart. A five-minute watchdog also wakes it when idle. Discord's required connection heartbeats run separately from delivery.
- **One shard.** A receiver supports one unsharded managed bot. If Discord requires multiple shards, event processing reports a configuration blocker. Membership verification remains a separate HTTP capability.
- **Coverage limits.** Discord does not include an action timestamp in reaction events. Relay processes durably received additions, but skips previously unseen replayed reactions whose timing is unknown. A new Gateway session can leave a coverage gap. Current reaction lists cannot reconstruct removed reactions or their dates. Gateway status reports these limitations; it is not a guarantee of complete Discord history.

## Telegram

Operators connect Telegram groups, supergroups and channels in Console **Integrations** (or `management.telegram.connect()`), through Domino's shared bot; a project can connect several. See the [Telegram guide](https://console.domino.run/docs/build/telegram).

- **Membership.** `await ctx.telegram.member({ chat })` reads the member's linked Telegram account in a connected chat, named by its numeric ID or `@username`. It needs no `integrations` entry: Relay supplies `ctx.telegram` to every quest, and the type is optional only because a test context may omit it. The result is `null` when the member is not in the chat or has not linked Telegram, or `{ userId, username, status, isMember, isAdmin, untilDate }`. An unconnected chat, an unconfigured bot or a chat whose bot lost administrator access throws `CapabilityError` `integration-required`; a Telegram outage `provider-unavailable`. The member's Telegram account (or that they have none) is pinned at the attempt's first Telegram read; chats are read live. Name the chat with `settings.telegramChat({ label })`, a text setting Console fills from the connected chats. The `telegram-membership` snippet is a complete join quest.
- **Reactions.** `defineTracker({ key, title, source: telegram.reactions(), window })` (`telegram` is exported by `@domino-sdk/relay/authoring`) records each reaction a member adds to a message in the project's connected groups and supergroups, from chats connected before the reaction. Record ID: `chatId:messageId:reactorId:reaction`. `author`: the reactor's Telegram user ID. Named account `messageAuthor`: the message author's user ID, absent for a message the bot did not see. `occurredAt`: Telegram's reaction time. `data` (`TelegramReactionData`): `{ chatId, chatType, messageId, reactionType, emoji, customEmojiId, reactorStatus, reactorIsAdmin }`, all scalars for `where`, such as `[{ path: "reactorIsAdmin", equals: true }, { path: "emoji", equals: "❤" }]`. A record is stored only when the reactor or the message's author is a member's linked Telegram account; a removed reaction marks the record removed. Channel reactions are anonymous and never recorded. Pay the message's author with `beneficiary: "messageAuthor"`; self-reactions skip with `beneficiary-is-author`. Attribution, link times and redecision follow the member's selected Telegram account, like Discord.
- **Messages.** `defineTracker({ key, title, source: telegram.messages(), window })` records each message a person posts in the project's connected groups and supergroups, from chats connected before the message. Record ID: `chatId:messageId`. `author`: the poster's Telegram user ID. `occurredAt`: Telegram's send time. `data` (`TelegramMessageData`): `{ chatId, chatType, messageId, replyToMessageId, threadId }`, all scalars for `where`; `replyToMessageId` is null when the message is not a reply (in a forum topic, the topic's first message is not a reply target), and `threadId` is the forum topic, or the reply thread outside forums, or null. No text is stored. A record is stored only when the poster is a member's linked Telegram account. Bots' messages, service messages (joins, pins, topic changes and other messages with no content), messages sent on behalf of a chat, and channel posts are never recorded. An edit is the same message and writes nothing; the Bot API sends no deletes, so a deleted message stays recorded. The `telegram-contribution` snippet pays once per member per project-calendar day and checks an optional `settings.telegramChat()` in the evaluator; for consecutive days replace its keyed completion with `streak({ days })` and remove its budget.
- **In SDK 1.x** both sources are custom sources on the reserved namespaces `telegram-reactions` and `telegram-messages`: `trackers.ingest` refuses them, and publishing either namespace with any other source is refused.

## YouTube Shorts

- **Source.** `defineTracker({ key, title, source: youtube.shorts(), window, refreshSeconds? })` (`youtube` is exported by `@domino-sdk/relay/authoring`) reads the Shorts members post on the YouTube channels they linked as creator accounts, with each member's own token, every `refreshSeconds`. No search, no API key. Inert until Domino configures YouTube.
- **Records.** One per Short, keyed by video ID. `author`: the channel ID, attributed through the member's selected YouTube channel, so link times, `historical` and redecision on link work as for X. `occurredAt`: publication time. Content body `YouTubeShortBody`: `{ title, description, tags, durationSeconds, publishedAt }`; metrics `views`, `likes`, `comments` (a hidden counter is unknown, never zero). Use it with `content({ tracker })`, `performance` and engagement rules like any content source.
- **Which videos.** Candidates from the channel's Shorts playlist (the uploads playlist when it is missing), kept only when at most 180 seconds long (60 for uploads before 2024-10-15) and vertical or square.
- **Removal and outages.** A deleted or private Short is removed. An unlinked channel is no longer read; its records stay. A token that needs reconnecting, an outage or a spent daily quota is `provider-unavailable` in the tracker's error and removes nothing; a spent quota defers collection to the next Pacific-time day.
- **View counts.** Shorts count every play from 2025-03-31, and every format counts at playback start from 2026-08-24; counts are not comparable across those dates.
- **In SDK 1.x** the source is a custom source on the reserved namespace `youtube-shorts`: `trackers.ingest` refuses it, and publishing that namespace with any other source is refused.

## TikTok videos

- **Source.** `defineTracker({ key, title, source: tiktok.videos(), window, refreshSeconds? })` (`tiktok` is exported by `@domino-sdk/relay/authoring`) reads the public videos members post on the TikTok accounts they linked as creator accounts, only with each member's own token, every `refreshSeconds`. No search; TikTok has no app-only read. Inert until Domino configures TikTok.
- **Records.** One per video, keyed by video ID. `author`: the account's `open_id`, attributed through the member's selected TikTok account, so link times, `historical` and redecision on link work as for X. `occurredAt`: creation time. Content body `TikTokVideoBody`: `{ title, description, durationSeconds, width, height, createdAt, shareUrl, isAigc }` (`description` is the caption, `isAigc` the creator's AI-generated label); metrics `views`, `likes`, `comments`, `shares` (a counter TikTok leaves out is unknown, never zero). Use it with `content({ tracker })`, `performance` and engagement rules like any content source.
- **Removal and outages.** A known video the account no longer returns is removed. An unlinked account is no longer read; its records stay. A token that needs reconnecting is not read at all, and a token TikTok refuses is refreshed once, so a revoked grant reads `needs-reconnect` at once. An account that can't be read, and TikTok's request limit (counted for the whole platform over any 60 seconds, deferring collection to the next minute), are `provider-unavailable` in the tracker's error; any other failure names the failure and retries from the saved cursor. Nothing of these removes a record.
- **In SDK 1.x** the source is a custom source on the reserved namespace `tiktok-videos`: `trackers.ingest` refuses it, and publishing that namespace with any other source is refused.

## Wallet balance samples

- **Source.** `defineTracker({ key, title, source: wallet.balances({ chain, token }), window })` (`wallet` is exported by `@domino-sdk/relay/authoring`) samples one token's balance in every wallet linked by a member who joined the project. `chain` and `token` are those of `ctx.wallet.balance()`: `"solana"` (stored as Solana mainnet's CAIP-2 ID) with an optional SPL or Token-2022 mint, an EVM chain ID with an optional ERC-20 contract, or a Cosmos network with a required bank denom. Omit `token` for a native coin. The builder checks the syntax; a chain Relay doesn't read, or a contract that isn't a token, shows on the tracker's error at the first sample and is retried hourly. Chain and token never change after publication (a new token needs a new key). `refreshSeconds` and `maxPages` don't apply.
- **When.** Once per project-calendar day, at an offset in the first three quarters of the day derived from a secret Relay holds, the tracker and the date: members can't predict it, and every retry of a day reuses it. A run whose cursor was cleared that day (a republish) resumes the day's sample at its pin from the stored samples, and once every wallet is read a run writes nothing more that day. After a day's sample, `nextSyncAt` is the next day boundary, where the run finds that day's time, so a calendar change never skips a day. Runs are identified by the day's start, not its date.
- **Records.** One per wallet, keyed by its CAIP-10 account on the sampled chain, which is also `author` (identity kind `wallet`); each day's sample is a new observation with `occurredAt` = the sample time. `data` (`WalletBalanceSample`): `{ amount, decimals, block }`, `amount` in base units as a decimal integer string, `decimals` null when the token doesn't report them. One run pins the chain's latest EVM block number or Cosmos height and reads every wallet at it, 100 wallets per step; Solana can't read a past slot, so every Solana read requires `minContextSlot` = the run's pinned slot and `block` is the slot that wallet's answer reflects. Every linked wallet of the chain's family is read: all EVM wallets for an EVM chain, all Solana wallets for Solana, Cosmos wallets linked on that exact network; unlinked wallets and members who never joined are not.
- **Failures.** A provider outage writes nothing for that step, sets the tracker's error, and retries 15 minutes later at the same pin. A run whose day ends before every wallet is read stops, so the remaining wallets have no sample that day. Relay never writes a balance it didn't read: a day whose reads all fail has no sample, and `streak({ days })` breaks on it; operators can grant the quest by hand. The test-member clock affects evaluation, never sampling.
- **Quests.** Use ordinary tracker triggers: `automatic({ tracker, where: { path: "amount", gte: { setting: "threshold" } } })` (an amount setting) with `streak({ days })` or `once()`. A `where` judges one wallet; total a member's wallets in the evaluator with `trackers.records(tracker, { member: member.id })`. The `token-hold-streak` snippet is a 7-day hold quest.
- **Limits.** At most 5 unarchived `wallet.balances()` trackers per project and environment: publishing or unarchiving a sixth fails with 409 `limit-reached`. EVM samples need the platform's dRPC key.
- **In SDK 1.x** the source is a custom source on the reserved namespace `wallet-balances`, with the chain and token in its `balance` field: `trackers.ingest` refuses it (403 `forbidden`), and publishing that namespace with any other source, or `balance` on another namespace, is refused.

## Discord role rewards

Import `discord` from `@domino-sdk/relay/authoring`. Declare `rewards: [discord.grantRole(roleId)]`. Install `discord-role-reward` for a complete prerequisite-gated example. Its resource setting uses `discord.roles({ assignable: true })`; ordinary eligibility roles use `discord.roles()`.

The ledger commits one durable delivery per completion and role. Delivery retries separately from quest evaluation using an idempotent role assignment. Never write to Discord inside evaluate. Pending delivery is bound to the original account/server, so reconnecting a different identity cannot redirect it. Restoring the original connection and permissions permits recovery. The bot needs Manage Roles and a role above the target. Managed/everyone roles cannot be reward roles.

Use `relay.me.rewardDeliveries()` for the current member or `management.rewardDeliveries({ member })` for operator inspection. Both answer a page of up to 500 deliveries, newest first (`limit` sets fewer); pass the answer's `nextCursor` back as `cursor` for older ones until it is `null`. Each record contains quest, completion, role, status, attempts and retry details. `management.discordRoles.list()` (`GET /reward-deliveries/roles`, `read` permission) answers every role the project's deliveries have granted in the environment, however old, as `{ roles: [{ roleId, quests, deliveries: { pending, delivered, failed } }] }`. Status is `pending`, `delivered` or `failed`. Relay retries transient Discord errors for about a day. A delivery fails at once when Discord reports a permanent problem: an unknown role, a member who left the server, or missing bot permissions. After fixing the cause, call `management.retryRewardDelivery(id)` with `review` permission. For revoked grants this reconciles removal without restoring the grant. `management.discordRoles.preserve(id, { reason })` (deprecated name: `management.preserveRewardRole`) marks an independently owned assignment as external, preventing automatic removal across shared projects/environments. Console exposes both actions. See [role recovery](https://console.domino.run/docs/build/integrations#recover-or-preserve-a-role) for CLI routes and ownership limits. Participant UI should show pending/granted/failed status without operator error details. Referral reward definitions still accept points and tier rewards; qualify a role-granting quest if composing them.

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

## Entry after app-owned pass verification

Author the entry quest with `manual({ actor: "staff" })` and no evidence input. The participant app verifies its event pass on its server, resolves the signed-in member, and calls `management.passes.verify(...)` (`POST /management/v1/passes/verify`) with `{ provider, event, subject, member, quest }`, where `quest` is the entry quest's ID or key. Use a server-only management key with `confirm` permission in the intended project and environment. Relay records unique pass ownership and submits the entry attempt; it does not decide whether the external ticket is valid. The member must already exist in that project and environment; an unknown member ID is refused with 404 `not-found`. Repeating the same pass for the same member reuses the durable attempt. A different member cannot claim that pass, and one member cannot link a second pass for the same provider and event.

## Staff confirmation

Staff confirm a `manual({ actor: "staff" })` quest for a member in Console with **Confirm activity** on the member's page. A server or script can do the same with `management.attempts.confirm({ quest, member })`, where `quest` is the quest's ID or key (`POST /management/v1/attempts` with `{ actionId, quest, member }`), using a management credential with `confirm` permission. Relay answers 202 with the queued attempt, and 200 with the same attempt when the action ID is repeated. Any other quest is refused with 403 "Choose a staff-confirmed quest". The member must already exist in that project and environment: an unknown member ID is refused with 404 "Member not found" (`not-found`) and creates nothing. A member who exists but has not joined the project joins it. Create members in test with test members; in live, members exist once they sign in. In live, only a Console session or a management key may confirm; a CLI sign-in changes nothing in live.

## Hosted server applications

Quest discovery and private resource artifacts work the same way for static and Worker apps. Keep quest definitions outside the public asset directory. For server-rendered applications, configure the Worker module output and browser asset output separately in `relay.json`; see [hosted apps](hosted.md). With `app.routing="transparent"`, hosting reserves no paths. Omitting the setting preserves legacy routing. The starter owns its `/relay/*` proxy as ordinary application code; custom handlers can take precedence.

Ledger names can repeat. Edit names and descriptions in Console, or call `management.pointLedgers.update(ledger, { name, description })` to update an existing ledger in test; live ledgers change only in Console. `settings.pointLedger` also renders a searchable picker; existing text settings need a new type version and instance upgrade.

## Project calendar

Each project environment has one timezone that sets where days, weeks and months begin for budgets, engagement budgets, streaks and calendar leaderboard windows. It is UTC until set. Declare it in `relay.json` as `"calendar": { "timeZone": "Europe/Budapest" }`, or change it in Console **Settings → General → Project calendar**, or with `management.calendar.configure({ timeZone, reason })` (`domino api POST /calendar body.json` with a body of `actionId`, `timeZone` and `reason`, `publish` permission; live refuses it with `promotion-required`, so change the live calendar through a reviewed deployment of relay.json). `management.calendar.read()` (`GET /calendar`) returns `timeZone`, `today`, `nextBoundary`, `pending` and `declared`. Use a named IANA zone; fixed offsets such as `+02:00` or `Etc/GMT-2` are rejected. A deployment applies relay.json's zone only when it differs from the last deployed declaration, so a later Console change stays until relay.json is edited; the deploy preview's `calendar` shows the scheduled change or `null`. No quest, budget, streak or board names its own zone.

A change takes effect at the next midnight of the current zone and never rewrites history. The open day, week or month finishes in the old zone; the next one ends at the new zone's next boundary, so it can be shorter than usual. Streak days stay consecutive across the change. Saving another zone replaces a pending change; saving the current zone cancels it.

## Calendar-day streaks

Use `streak({ days: 7 })` as the completion policy on a manual quest or a tracker-triggered quest (`automatic({ tracker })`). The evaluator accepts or rejects each attempt; the ledger records the calendar day of each accepted attempt and awards the declared rewards once when seven consecutive days qualify. It never mutates progress inside the evaluator. One quest defines one lifetime milestone; use separate quests for different milestones. A broken run can restart, but an earned milestone does not pay again.

Each accepted attempt counts the day of its trusted occurrence time, the same time as `ctx.eligibleAt`:

| Trigger                         | Day counted                                                                                                             |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `manual(...)` (member or staff) | When Relay admitted the submission. A reviewer who accepts it days later does not move it.                              |
| `automatic({ tracker })`        | The record observation's `occurredAt`: an X post's publication time, a Discord message's post time or reaction's receipt time, or the time your server sends for a custom record. |

A manual streak counts submission days, not activity days: a member can submit older evidence one day at a time, and each submission fills its own day. To count the days the activity happened, trigger the quest from a tracker. `content(...)` quests and prerequisite-only `automatic()` quests cannot be streaks; `defineQuest` throws and publication refuses them.

```text
import {
  defineQuest,
  manual,
  points,
  settings,
  streak,
} from "@domino-sdk/relay/authoring"
import { community } from "./ledgers"

export default defineQuest({
  key: "weekly-checkin",
  title: "Check in on seven consecutive days",
  settings: settings.object({}),
  // No evidence and no evaluator: each claim is accepted and marks today.
  trigger: manual(),
  completion: streak({ days: 7 }),
  review: () => ({ enabled: false }),
  rewards: [points(community, 100)],
})
```

For activity days, trigger from a tracker. `packages/sdk/examples/posting-streak-quest.ts` pays an approved creator who posts about the project on X on five consecutive days: an X tracker collects the posts, `automatic({ tracker })` runs once per new post for its author, `requires: [inGroup(approvedCreators)]` limits it to an operator-managed group (see [rewarding X posts](#rewarding-x-posts)), and each post marks the day it was published. A "hold N tokens for 7 days" streak uses a custom wallet tracker with the default `on: "new-observation"` and `where: { path: "amount", gte: { setting: "threshold" } }`, fed at least one sample per wallet per day; `"data-changed"` would miss a day whose balance did not change. Each sample must carry its own `occurredAt`, the time it was sampled; a sample that reuses the record's original time lands on that one day. Each record observation is one day candidate, and repeated observations on one day occupy it once. Once a member's milestone pays, the trigger stops admitting their observations for that quest, as for a completed `once()` quest. The ingesting server sets a custom record's `occurredAt`, so an `ingest` key holder can backdate records to fill days, back to the quest's publication and within the tracker window; keep those keys on trusted servers. Never use browser-supplied action times as evidence. Discord message quests use the same policy on a `discord.messages()` tracker, with the contribution snippet's channel check.

Days are those of the [project calendar](#project-calendar), not each participant's device timezone; `streak()` takes no `timeZone`. Days end at midnight in the project timezone and can last 23 or 25 hours at daylight-saving transitions. Multiple qualifying actions occupy one day. Today remains open: yesterday's run survives until today ends without qualifying evidence.

A project timezone change starts one new calendar day at the next midnight in the current zone, ending at the next midnight in the new zone; its date label may repeat or skip, but its day identity remains consecutive. Historical days retain their original zone and identity. Goal length and trigger kind (manual or tracker) cannot change on an existing streak quest; create a new quest for a new milestone or trigger. Presentation edits preserve progress. Qualification edits affect newly admitted evidence; already admitted attempts retain their release.

Eligible delayed tracker records count at their occurrence time, even when processed out of order. They can repair missing days and complete a milestone. Existing activation, identity, pause and prerequisite admission rules still apply; there is no historical Discord backfill or synthetic credit for capture outages. Removing a message or reaction does not erase a qualifying day. No grace days or streak freezes are implicit.

Qualifying days, completion, points and role-delivery intent commit atomically. Concurrent actions and retries cannot repeat a payout. A budget-exhausted milestone keeps its days and is permanently marked `budget-exhausted`; later retries or budget edits do not pay it. Budgets bucket by the project calendar and use the recorded milestone day action time. Discord role delivery still retries independently after completion.

Read `quest.streak` through `relay.me.progress()` or the quest experience. It includes `current`, `best`, `goal`, `todayQualified`, the current `timeZone`, `nextBoundary`, a pending timezone transition, qualifying-day history and milestone reward status. An accepted action below the goal has attempt status `progressed`, not `accepted`, and no completion. Upgrade the SDK and participant app before publishing streak quests so they understand this new completion policy and attempt status.

## Evaluator readers: null, errors and pinning

Every evaluator reader follows one rule: **missing is `null`, a failure throws**. A reader of the member's own account returns `null` when the member has not linked it; a reader of something else returns `null` when it does not exist. A reader that needs a linked account but whose result cannot be `null` throws `CapabilityError` with code `not-linked` instead.

| Reader | `null` when | Throws `CapabilityError` |
| --- | --- | --- |
| `ctx.x.connection()` | No linked X account | — |
| `ctx.x.profile()` | No linked X account, or X no longer has the account | `provider-unavailable` |
| `ctx.x.follows({ userId })`, `ctx.x.reposted({ postId })` | No linked X account | `provider-unavailable` |
| `ctx.x.post({ postId })` | The post does not exist; no linked account is needed | `provider-unavailable`, `author-error` (an ID that is not a decimal post ID) |
| `ctx.discord.member()` | Not in the server, or no linked Discord account | `provider-unavailable` (temporary outage), `integration-required` (bot or project connection needs configuration), `author-error` (`integrations: [discord]` not declared) |
| `ctx.discord.message({ channelId, messageId })` | The message does not exist; no linked account is needed | `provider-unavailable`, `integration-required`, `author-error` (`integrations: [discord]` not declared, a channel outside the connected server or one the bot cannot see) |
| `ctx.wallet.list()` | No linked wallet; a non-null list is never empty | — |
| `ctx.member.identities()` | No selected X or Discord account and no `app:<name>` identity in the project; a non-null list is never empty | — |
| `ctx.member.balance(ledger)` | Never: a ledger the member has no points in reads `0` | `author-error` (a ledger key or ID the project does not have, a malformed key) |
| `ctx.member.completions(query)` | Never: no completion is an empty list | `author-error` (a quest key or ID the project does not have, a malformed key, `since` or `limit`) |
| `ctx.member.groups()` | Never: no group is an empty list | — |
| `ctx.member.enrolled(campaign)` | Never: a withdrawn or missing enrollment reads `false` | `author-error` (a campaign no active quest declares and the member never enrolled in, a malformed key) |
| `ctx.member.referrals(program)` | Never: a member who invited no one reads zero counts | `author-error` (a referral program key or ID the project does not have, a malformed key) |
| `ctx.wallet.balance(query)` | No linked wallet of the chain's family | `provider-unavailable`, `author-error` (unsupported chain, malformed token, an `address` the member never linked) |
| `ctx.wallet.nfts(query)` | No linked wallet of the collection's family | `provider-unavailable`, `author-error` (malformed collection, unsupported chain, a contract that rejects the read, an `address` the member never linked) |
| `ctx.wallet.call(query)` | No linked EVM wallet | `provider-unavailable`, `author-error` (malformed call, unsupported chain, arguments that do not fit the function, a revert, return data that is not the declared type, an `address` the member never linked) |
| `ctx.wallet.delegations(query)` | No wallet linked on that Cosmos network | `provider-unavailable`, `author-error` (unsupported chain, malformed validator, an `address` the member never linked) |
| `ctx.wallet.events(query)` | No linked EVM wallet | `provider-unavailable`, `author-error` (malformed event, unsupported chain, an `account` that is not an indexed address argument, a bad `where`, `since` over 30 days back, more than 3,000,000 blocks or 1,000 logs, an `address` the member never linked) |
| `ctx.wallet.transactions(query)` | No linked Solana wallet | `provider-unavailable`, `author-error` (malformed program, an account that is not an executable program, a `mint` that is not a token mint, `since` over 30 days back, an `address` the member never linked) |
| `ctx.trackers.records()`, `ctx.trackers.summary()` | Never: no match is an empty page or zero totals | `author-error` (unknown tracker, a filter the tracker cannot answer) |
| `ctx.trackers.content(reference, { id, observation })` | The tracker has no such record or observation | `author-error` (not an X tracker) |
| `ctx.images.*` | Never | `not-linked` (`{ x: "profile" }` without a linked X account), `author-error` (missing, undecodable or another attempt's image), `provider-unavailable` |
| `ctx.review.text()`, `ctx.review.photo()` (deprecated) | Never | `provider-unavailable`, `author-error` (input over the limits) |

`CapabilityError` is exported from `@domino-sdk/relay/authoring` with `code` (`"not-linked"`, `"provider-unavailable"`, `"author-error"`, `"integration-required"` or `"capability-limit"`; treat an unknown code, which a newer Relay can add, as a failure its `message` explains; it is never retryable) and `retryable`. Check `error instanceof CapabilityError && error.code === "provider-unavailable"` to decide yourself, such as sending the attempt to review, and rethrow everything else. Provider messages never reach quest code; `message` is Relay's own.

- **A failure that is not retryable is captured like a result**, or follows from the release and attempt alone, so a replay throws the same error. An uncaught `integration-required` ends the attempt as `failed` immediately. Relay releases that failed read and later captures so an explicit retry after repair can recheck them; earlier captured results and account pins remain fixed.
- **A retryable failure (`provider-unavailable`) is captured only once quest code moves past it.** If quest code handles it and moves past it, it is final for the evaluation run: once quest code makes another capability call, Relay captures it, so a replay throws it again and takes the same branch. If it escapes the evaluation (thrown, rethrown, or wrapped as a `cause`, in an `AggregateError` or a `SuppressedError`), Relay releases it, along with every capability result captured after it, and retries the attempt, asking the provider and those later calls again. To let a failure you caught be retried, rethrow it, or wrap it with `new Error("…", { cause: error })`; `throw new Error(error.message)` loses the link, so the failure stays captured and every retry replays it. A decision you return after catching it stands.
- Any other error (Relay itself unavailable, an expired evaluation) is not a `CapabilityError`; never treat it as an answer.

**Quest code that fails on its own fails the attempt once.** A bundle that throws while loading, such as a top-level `ReferenceError`, an `evaluate` that throws anything without an escaped capability failure in it (a plain `Error`, a `TypeError`, an invalid reject code), and an `evaluate` that returns no decision end the attempt as `failed` with code `evaluator-error`. The reason is the thrown error's name and message, such as `ReferenceError: seed is not defined`, cut to 1,000 characters. Relay does not retry it, because every run would fail the same way. Members read the reason too, so throw messages you are content to show them. An attempt keeps the release it started on, so retrying it runs the same code and fails again: fix the code and publish, and new attempts use it. `runQuest` rejects with an `EvaluatorError` (code `evaluator-error`, the same message, and what quest code threw as `cause`).

**An evaluation run makes at most eight capability calls**, shared by every reader: `ctx.x`, `ctx.discord`, `ctx.wallet`, `ctx.member`, `ctx.trackers`, `ctx.images`, `ctx.review` and `ctx.telegram`. The ninth call throws `CapabilityError` with code `capability-limit`, never retryable, without reaching Relay; uncaught, it fails the attempt with `evaluator-error`. A retry or a replay makes the same calls, so it fails the same way. Each page of tracker records is one call, so read with filters and `summary()` instead of paging through every record, and ask `ctx.member.completions` once for several quests instead of once per quest. The testing kit applies the same limit and exports it as `maxCapabilityCalls` from `@domino-sdk/relay/testing`. A scheduled defer starts a fresh run with eight calls of its own.

**Linked accounts are pinned once per evaluation run.** The member's X account (read by `ctx.x`, `ctx.images` with `{ x: "profile" }` and `ctx.trackers.content` attribution), Discord account and server, wallets, and identities (`ctx.member.identities()`) are fixed at the run's first read of each, including "not linked", and never rechecked. Linking or unlinking afterwards does not change the run. A new submission or scheduled deferred run uses the new account.

**The member's balances, completions, groups, campaign enrollments and referral progress are pinned the same way**, each at the run's first call of its reader: `ctx.member.balance()`, `completions()`, `groups()`, `enrolled()` or `referrals()`. Points awarded or spent, completions recorded or revoked, group membership changes, enrollments, withdrawals and invitations afterwards, including this attempt's own reward, do not change the run.

**`ctx.member.balance(ledger)`** takes a ledger handle, key or ID and returns the member's balance in it as `relay.me.balances()` lists it: `{ ledger: { id, key, name }, balance, available }`, where `available` leaves out points that reservations hold.

**`ctx.member.completions({ quest?, since?, limit? })`** returns the member's accepted completions in the project that no operator revoked, newest first: `{ quest, key, completedAt }`, with the quest's key, the completion key, and when it was recorded in epoch milliseconds. `quest` takes a quest handle, key or ID. `since` keeps completions recorded at or after it. `limit` defaults to 100, and a larger one reads 100. Times are on the member's clock: an advanced test member's completions carry their advanced time, so compare them with `ctx.now` or `ctx.eligibleAt`. Older completions recorded without a time have `completedAt: null`; they come last, and `since` leaves them out. Each call counts toward the run's 8 capability calls. To check a few quests, ask for each with `limit: 1`; a list of every completion stops at the newest 100.

**`ctx.member.groups()`** returns the keys of the project's groups the member belongs to now, in byte order. Archived groups are left out. It returns keys only, not titles or how the member joined.

**`ctx.member.enrolled(campaign)`** takes a campaign definition or key and returns whether the member's enrollment in it is active: `false` after they withdraw, and for a campaign they never enrolled in, including one that does not take enrollment. Enrolling again reads `true`. It knows the campaigns an active quest declares and those the member enrolled in.

**`ctx.member.referrals(program)`** takes a referral program handle, key or ID and returns the member's progress as inviter, as `relay.profiles.referral()` returns it: `{ program, key?, title, invited, qualified, milestones }`. `invited` counts every member they invited, shared by all programs; `qualified` counts those who qualified in this program; each milestone has `reached`. It never returns codes or accounts, and not who invited the member. Archived programs still read.

Prerequisites (`requires`) are all required. Write a rule they cannot express in `evaluate` with these readers, such as a balance or a completed quest:

```ts
import {
  decision,
  defineLedger,
  defineQuest,
  manual,
  once,
  settings,
} from "@domino-sdk/relay/authoring"

const xp = defineLedger({ key: "xp", name: "XP" })

export default defineQuest({
  key: "veteran-badge",
  title: "Claim the veteran badge",
  settings: settings.object({}),
  trigger: manual(),
  completion: once(),
  rewards: [],
  async evaluate(ctx) {
    const { balance } = await ctx.member.balance(xp)
    const onboarded = await ctx.member.completions({
      quest: "onboarding",
      limit: 1,
    })

    return balance >= 100 || onboarded.length > 0
      ? decision.accept()
      : decision.reject({
          reason: "Earn 100 XP or finish onboarding first.",
        })
  },
})
```

Or a number of completed quests out of a set:

```ts
import {
  decision,
  defineQuest,
  manual,
  once,
  settings,
} from "@domino-sdk/relay/authoring"

const tour = ["profile", "wallet", "discord", "first-post", "referral"]

export default defineQuest({
  key: "tour-guide",
  title: "Finish two steps of the tour",
  settings: settings.object({}),
  trigger: manual(),
  completion: once(),
  rewards: [],
  async evaluate(ctx) {
    // One call per quest: a list of every completion stops at the newest 100.
    const done = await Promise.all(
      tour.map(
        async (quest) =>
          (await ctx.member.completions({ quest, limit: 1 })).length > 0
      )
    )

    return done.filter(Boolean).length >= 2
      ? decision.accept()
      : decision.reject({ reason: "Finish two steps of the tour first." })
  },
})
```

Or one of several ways in, through a group, a campaign and a referral program. Name them by key: the `vip` group, a `launch` campaign that other active quests declare with `enrollment: "required"`, and an `invite` referral program the project publishes. A campaign no active quest declares, and that the member never enrolled in, throws `author-error`:

```ts
import {
  decision,
  defineQuest,
  manual,
  once,
  settings,
} from "@domino-sdk/relay/authoring"

export default defineQuest({
  key: "insider-pass",
  title: "Claim the insider pass",
  settings: settings.object({}),
  trigger: manual(),
  completion: once(),
  rewards: [],
  async evaluate(ctx) {
    if ((await ctx.member.groups()).includes("vip")) {
      return decision.accept()
    }

    const enrolled = await ctx.member.enrolled("launch")
    const { qualified } = await ctx.member.referrals("invite")

    return enrolled && qualified >= 3
      ? decision.accept()
      : decision.reject({
          reason:
            "Join the VIP group, or enroll in the launch and bring three friends.",
        })
  },
})
```

**`ctx.member.identities()`** lists the identities trackers attribute records to the member through, besides wallets (`ctx.wallet.list()`) and the member ID (`ctx.member.id`): `{ kind, subject, revision, linkedAt }` for the member's selected X and Discord accounts (`subject` is the stable user ID) and each `app:<name>` identity an [app login](https://console.domino.run/docs/build/app-login) integration linked in the project and no operator unlinked (`subject` is your app's user ID), ordered by `kind`, then `subject`. `linkedAt` is null when unknown. Use it in quests that are not tracker-triggered; a tracker-triggered quest's beneficiary account is `ctx.tracker.attribution.account`. `kind` is open: compare it with the kinds you expect.

**Tracker reads are point in time.** `ctx.trackers.records()` and `summary()` take `asOf` in the query, in epoch milliseconds, defaulting to `ctx.eligibleAt`: only records whose observation occurred at or before `asOf` count, each as its latest such observation. Each evaluation run pins a watermark at its first tracker read, so every tracker read in that run, including its retries, sees exactly the observations Relay had stored then, and each tracker's `status` as first read. A scheduled rerun after defer or progress pins a fresh watermark. Pass `asOf: ctx.now` to count everything stored by the watermark. The `member` filter still matches the member's verified links when the read runs.

## Maintained tracker datasets

Declare `defineTracker({ key, title, source: x.search({ query }), window: { start, end? } })` from `@domino-sdk/relay/authoring` and add `trackers: [{ entry: "trackers/launch.ts" }]` to `relay.json`. `domino deploy` previews and publishes tracker definitions together with quests and ledgers. `source: discord.messages()` or `discord.reactions()` declares a [Discord tracker](#discord-trackers) instead. The X collector retains all returned matches and author verification. Apply `verified`, `member`, `author`, date, and engagement filters when reading, not during ingestion.

A tracker of a managed source can also be data instead of code, for operators who add campaigns without a deployment. `management.trackers.create({ key, title, source, window: { start, end? }, refreshSeconds?, maxPages? })` (`domino trackers create KEY --title T --source KIND --start MS`, or **New tracker** in Console **Trackers**) creates one in test with `publish` permission and the same validation as `defineTracker`. `source` is `{ kind: "x-search", query }`, `{ kind: "discord-messages", text? }`, `{ kind: "discord-reactions" }`, `{ kind: "telegram-messages" }`, `{ kind: "telegram-reactions" }`, `{ kind: "youtube-shorts" }`, `{ kind: "tiktok-videos" }` or `{ kind: "wallet-balances", chain, token }` (`chain` a CAIP-2 ID, `token` null for a native coin; `domino trackers create --source wallet-balances --chain C [--token T]`), stored as the source its builder makes; `query` alone, without `source`, is short for an X search. Custom sources are code only. An X tracker's first collection searches from `window.start`. `management.trackers.update(tracker, changes)` (`domino trackers update TRACKER [--query Q] [--text on|off] ...`, or **Edit** in Console) changes any of those fields but the key and the source kind (422 `validation-failed`); fields left out keep their value, a new query or window collects from now on, never searching the past, and turning Discord message text off erases the text already stored. Live refuses both with 422 `promotion-required`: the tracker reaches live, under the same key and still owned by data, through a promotion from test (`domino promote`). A key belongs either to code or to data, shown as `origin` (`code` or `data`) on tracker status: a deployment that declares a data tracker's key is refused with 409 `tracker-owned-by-data`, and creating or updating a key a deployment publishes with 409 `tracker-owned-by-code`. Deployments never change or remove data trackers. Pause, resume and refresh work on both kinds, in test and live; archive is a definition change, made in test.

X tracking is best effort and bounded per project. Each cycle searches the tracker's window; known posts the search didn't return are read again every 6 hours for 48 hours after they were posted, daily until day 7, then never. A tracker runs no more often than its project's plan allows: 6 hours on every plan but Enterprise. A shorter `refreshSeconds` still publishes, runs at that minimum (`effectiveRefreshSeconds` on tracker status), and a deployment warns `tracker-refresh-below-plan-minimum`. Tracker searches and refreshes, quest X checks, tracker previews and handle lookups count toward the project's monthly X usage. While plan limits are enforced, a live project that spends its plan's X budget (Launch 25,000, Growth 65,000, Scale 160,000 requests a month) has its live X trackers paused with `pauseReason: "x-budget-exhausted"` until `resumesAt`, the next UTC month, or a plan whose budget isn't spent; quest checks keep working. An organization's test X trackers share 5,000 requests a month and pause the same way when it is spent. A test X tracker collects for 14 days from its first sync, then pauses with `pauseReason: "test-window-ended"`; `trackers.resume` starts another 14 days. Resuming a budget-paused tracker is refused with 409 `x-budget-exhausted` while the budget is spent; pausing it yourself makes the pause `operator`, which only a resume ends. A resumed tracker searches its whole window again.

Management clients expose `trackers.create`, `update`, `list`, `get`, `records`, `content`, `summary`, `control`, `ingest`, `outcomes`, `retryOutcome`, and `preview`. `trackers.preview({ query, since })` returns a bounded X search sample (at most 100 posts from 3 pages, `since` up to 30 days back, `more` when X had further matches) with `publish` permission. It stores nothing and never creates or changes a tracker. Console offers the same preview and copies a definition or agent prompt from it. Hosted quest evaluators expose read-only `trackers.records(reference, query)` and `trackers.summary(reference, query)`, with results captured for replay; they read the tracker as of `query.asOf` (default `ctx.eligibleAt`) at the evaluation run's pinned watermark ([evaluator readers](#evaluator-readers-null-errors-and-pinning)). `member` filters the currently connected X identity, on a Discord tracker the records whose author is their selected Discord account, or on a custom tracker with `identities` the records whose author is one of the member's verified identities of those kinds (every wallet they currently link, their selected X or Discord account, their member ID). An unconnected member matches nothing; custom trackers without identities refuse it. Different quests can apply independent qualification and review policies to the same dataset.

Overlapping X trackers in the same project and environment share current post facts. Each tracker retains its own matches and first-seen times; another tracker's discovery never adds a match to yours. Known-post refreshes coordinate across collectors. Pausing a tracker stops its collection, but another active tracker can still update their shared posts. Captured quest reads keep their original results. Custom records remain isolated per tracker, including when namespaces match.

Pass `includeEvidence: true` to `trackers.records` for X `evidence: { contentRevision, observation }`. A content revision identifies the stable body; counters, author profile/verification, removal status, and sample time affect only the observation. Compare within the same post, project, and environment. Identical samples across trackers share references; reverting a body reuses its revision. Default reads omit the field; opted-in custom reads return `null`. Captured quest reads pin the references with their results. Use `trackers.content` below to read a retained observation. Use the content trigger below for automatic admission.

Hosted evaluators can call `trackers.content(reference, { id, observation })` using a record's `evidence.observation`. It returns the immutable X body, observation time, captured removal flag, and account attribution to the attempt's member. Evaluators cannot override that member. Check `attribution.kind`: `linked`, `unlinked`, or `different-author`. Matching uses the selected verified X account's stable ID, never a username or verification badge. The linked result records the account issuer, subject, revision, and known link time; legacy unknown link times are `null`. Metrics and mutable author profiles are excluded. Management clients use the same method with an additional required `member` and `read` permission. The post must belong to the tracker in the same project/environment. Attribution uses the evaluation run's pinned X account. In an evaluator a missing record or observation returns `null`. Captured quest results survive edits and disconnects for replay; they do not prove enrollment, historical eligibility, or current post availability and do not automatically admit or reward content.

Custom datasets use `customSource(namespace, { data })` and management ingestion with `ingest` permission. `data` is required: a zod object schema (`z` is exported by `@domino-sdk/relay/authoring`) for each record's `data`. Relay stores its JSON Schema form with the tracker and rejects the whole ingest batch with 422 `{ error, code: "invalid-record-data", record, path, reason }` when a record does not match; evaluators read `ctx.tracker.record.data` and `trackers.records(tracker)` with the schema's input type. `customSource` throws for what ingestion cannot check: string formats and patterns (`z.email()`, `.regex()`), `.multipleOf()`, tuples, `.refine()`, `.transform()`, `z.never()`, `z.json()` (use `z.unknown()`), dates, bigints, `.meta()` keys other than title, description, default and examples, a `__proto__` property, and schemas reused by ID; check those in the evaluator. Limits: 32 KB and 16 levels per schema, 256 properties, `required` keys, enum values or key names per node, 16 branches per union, and inside a union's branch a nested union may have only one branch with properties, items or unions. Validation has a step budget per record and per batch that charges every key, value, literal property and enum lookup (admission and `trackers.records(handle)` use it too); a record that spends it fails the batch with 422 `record-data-too-complex`. Records with `removed: true` are not checked, and a refused batch records nothing, so its action ID can carry the corrected batch. The schema can change under the same tracker key and applies to later records only; a deployment whose new schema breaks a live quest's `where` (Console quests included) is refused with 422. Stored records are not rechecked: a tracker trigger skips one that no longer matches with `invalid-record-data`, an evaluator rejects a pinned record that does not match the schema the quest was built with, and `trackers.records(handle)` drops such records from `items` and counts them in `excluded` (string-key reads stay untyped and unfiltered). A tracker published before record data takes no records (422 `tracker-schema-required`) until it is published with `data`. For `ingest` permission, a hosted app's own key (`hostedManagementClient`) has it, and an indexer's personal API key can hold only `ingest` (Console **Settings → API keys**; editors hold it). Your server or indexer owns collection and source-specific finality. Tracker controls (pause, resume, refresh) require `publish`, work in live with a personal API key, and appear in Console **Audit history**; reads require `read`. Custom collection does not trigger content quests; declare `automatic({ tracker })` instead.

### Rewarding X posts

X does not allow apps that reward users for posting, so Domino rewards X posts only for creators an operator approved. When asked to reward people for posting or tweeting, build a curated creator program: name a group with `defineGroup({ key: "approved-creators", title: "Approved creators" })` and add `requires: [inGroup(group)]` to every quest that pays for X posts, whether `automatic({ tracker })` paying the post's author, `content({ tracker })` on an X tracker, or a manual quest that verifies a submitted post. Deploying creates the group, empty, and promoting creates it in live; operators add approved creators in Console **Groups**, with `domino groups`, or with `management.groups`. Posts by anyone else are skipped with `prerequisites-incomplete`. A deployment that publishes an X tracker or content quest that pays (declared rewards or an engagement rule) without an `inGroup` prerequisite still publishes, with the warning `x-post-rewards-ungated`. Quests that check follows, bios or profile pictures, and `beneficiary: "inReplyTo"` quests that pay the account replied to, are not rewards for posting.

### Tracker triggers

A tracker handle determines the allowed `beneficiary` values in TypeScript: X has `author` and `inReplyTo`, Discord messages have `author`, Discord reactions add `messageAuthor`, and custom sources add exactly their declared `accounts`. Dynamic definitions still receive runtime validation.

`customSource(namespace, { identities, accounts? })` makes record accounts verified identities Relay attributes to members. `identities` lists kinds from `"wallet"` (canonical CAIP-10 account ID, the member who currently links it), `"x"` and `"discord"` (stable numeric user ID, the member whose selected, verified account has it), `"member"` (a member ID), and `"app:<name>"` (your app's own user ID, such as `"app:telegram"`: the member the project's identity integration created with that `identityKind` signed in with that subject; `app:` then 1–40 of `a-z 0-9 _ -`; an unlinked ID skips with `unlinked-account` "No member has signed in with this app:<name> identity"). At most 8 kinds. `accounts` names roles records can carry besides `author`, such as `["recipient"]` (not `author`; needs `identities`). Records then need `author: { id, kind? }` and may carry `accounts: { <role>: { kind?, id } }`; `kind` is required unless the source lists exactly one kind. The ingesting server names the accounts: Relay checks each is verified by a member, not that it took part in the event, so a record can pay any member (`member` kind) or linked account; hold `ingest` keys only in trusted servers such as your own indexer. Wallet IDs are canonical: `cosmos:<chain ID>:<lowercase bech32>` (any chain ID; attributed to the member who links that bech32 address, whichever chain ID it names), `eip155:<chainId>:<EIP-55 address>` (any chain ID matches a linked EVM wallet), or `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:<base58>`. Ingestion refuses a missing author, an undeclared kind or role, a non-canonical wallet (naming the expected ID), or a non-numeric X/Discord ID with 422; `author.verified` is stored as `null`. Record reads return `author.kind` and `accounts` as ingested (a kind the record left out stays absent). Without `identities`, authors are opaque, records carry no kinds or accounts, and no quest can trigger from them. Changing the source, except its `data` schema, needs a new tracker key.

`trigger: automatic({ tracker, on?, where?, beneficiary?, enrollment?, since?, historical? })` admits one attempt per quest for each record observation it selects, inside the ingest (custom), collector (X) or gateway write (Discord) transaction. `on`: `"new-record"` (a record's first observation in this tracker; default for X and Discord, so refreshes, metric changes, edits, and another tracker's discovery never admit again), `"new-observation"` (each observation that becomes the record's current one: new record or later `observedAt`, even with unchanged data; default for custom), or `"data-changed"` (new record, or `data` differing from the record's previous current observation). The authoring helper always materializes the selected `on` value in the published policy. `where`: one condition on a dot-separated path in `record.data`, `{ path, equals: string | number | boolean }` or `{ path, gt?, gte?, lt?, lte? }` comparing a JSON number or an integer string (such as a base-unit balance) exactly; a bound is a number or `{ setting: "<integer or amount setting of this quest>" }`, resolved from the release's settings; an amount setting's integer string compares exactly, so 18-decimal thresholds fit, and never matches a fractional value. A missing key, a key into an array or scalar, an `equals` of another type (`6` is not `"6"`), or a compared value that is not a number or `-?digits` string (`"1.5"`, `null`) never matches. Observations the mode or filter does not select record no outcome; `ingest` returns `admissions: { admitted, skipped, filtered? }`, where `filtered`, present only when nonzero, counts per quest the observations its mode selected but `where` did not match. `beneficiary`: the record account role whose member it pays, `"author"` (default), `"inReplyTo"` on X (`references.replyAuthorId`), `"messageAuthor"` on a Discord reactions tracker, or a role the custom source declares in `accounts`; when that account is the record's author account, or both resolve to the same member (the same wallet on another chain, another linked account), the observation is skipped (`beneficiary-is-author`, such as a self-reply). `enrollment`: `"none"` (default) or `"required"`: only members who enrolled in the quest (`relay.quests.enroll({ actionId, quest })`, or `management.quests.enroll({ actionId, quest, member })` with `enroll` permission; enrolling again keeps the first time; a quest without required enrollment refuses with `enrollment-not-required`). `since`: `"activation"` (default; records since the quest was published or last unarchived), `"window"` (every record in the tracker's window), or, only with `enrollment: "required"` (`automatic()` throws otherwise), `"enrollment"` (records since the member enrolled). `historical`: who releases activity from before the member linked the beneficiary account (a link time of zero or unknown counts as before; `member` accounts are never historical) or before they enrolled: `"staff"` (default; a reviewer's retry), `"claim"` (also the member, `relay.quests.claim`), or `"automatic"`. The other defaulted admission fields may remain absent. `where` may also be a list of conditions that must all hold. TypeScript checks `where` (at most 10 conditions) against the tracker's data (paths to string, number or boolean fields up to five levels, arrays end a path, a `z.record` field takes any key and a `z.unknown()` field any path; `equals` values of the field's type; comparisons on number or string fields) and a `{ setting }` bound against the quest's integer and amount settings; `settings.object` accepts only `settings.*` builders. `automatic()` throws for a role the source's records lack (X: `author`/`inReplyTo`; Discord messages: `author`; Discord reactions: `author`/`messageAuthor`), an undeclared custom role, a custom source without `identities`, a filter path the data schema does not declare or a value its field cannot hold, or (in `defineQuest`) a setting bound that is not an integer or amount setting; publication refuses the same with 422. Use `once()`, `keyed(...)`, or `streak(...)` (each accepted observation marks the calendar day of `record.occurredAt`; see [calendar-day streaks](#calendar-day-streaks)); content quests instead omit `completion`. The tracker must be published (same deployment or earlier); switching a quest into, out of, or between tracker triggers needs a new quest key. The evaluator receives `ctx.tracker`, non-null for a tracker trigger (null for other triggers), with `record.data` of the tracker's type: `{ tracker, key, beneficiary, record: { id, source, occurredAt, observedAt, firstSeenAt, author, accounts?, data, removed }, content, attribution: { member, capturedAt, account }, enrolledAt? }` (`enrolledAt`, the member's first enrollment time, only on quests that require enrollment). `content` is absent for sources without a content declaration; otherwise it holds `{ body, revision, observation }`, with the source’s typed body. X bodies include `authorId`, `text`, `isReply` and `references`; custom bodies match their source’s body schema. `account` is `{ provider: "wallet", id, revision, linkedAt }`, `{ provider: "x" | "discord", issuer, subject, revision, linkedAt }`, `{ provider: "member", id }`, or `{ provider: "app:<name>", subject, revision, linkedAt }`. The account can also be `{ provider: string }` for a kind a later Relay adds; branch on `provider` with a default branch. `beneficiary` is the stored canonical role. `ctx.eligibleAt` is `record.occurredAt` (an X post's publication time). Attribution is pinned at admission. Examples: a "Hold N" quest reads `ctx.trackers.records(tracker, { member: ctx.member.id })` to total every linked wallet (`packages/sdk/examples/hold-balance-quest.ts`); a quest that rewards members when an account replies to them uses `beneficiary: "inReplyTo"`, keys by `tracker.record.id` (one reward per reply) with a `period: "week"` budget, and takes the replying account IDs and required text as settings (`packages/sdk/examples/account-reply-quest.ts`); a posting streak for approved creators counts each new X post's publication day toward a five-day `streak(...)` (`packages/sdk/examples/posting-streak-quest.ts`); a kudos quest pays a custom record's `recipient` role with `on: "new-record"`, so a rejected kudos edited later never pays, and a giver no member has linked still pays the recipient, capped by a weekly budget (`packages/sdk/examples/kudos-quest.ts`).

Every selected observation is exactly one trigger outcome: `admitted` with its attempt (`code: null`), or `skipped` with the first failing rule as a stable `code` and a human `reason`, in this order: `quest-paused`, `before-activation` (with `since: "activation"`), `outside-window`, `future-record`, `record-removed`, `invalid-record-data` (data stored before the tracker's schema changed no longer matches it), `content-unavailable`, `no-account` (such as "Post is not a reply", "Discord did not name the message's author" or "Record has no recipient account"), `beneficiary-is-author` (such as a self-reply or a reaction to one's own message; a different account of the author's member is judged after it resolves, following `unlinked-account` and `ambiguous-account`), `unlinked-account` (such as "No member has linked this wallet"), `ambiguous-account`, `not-joined` (Relay never joins a member on their behalf), `already-completed` (a completed `once()` quest, or a streak whose milestone paid), `not-enrolled`, `before-enrollment` (with `since: "enrollment"`), `historical`, `prerequisites-incomplete` (a required quest was not complete when the record occurred, or the member was not in a required group when it was decided; completions or group membership recorded later never decide it again). Branch on `code`, never on `reason`; an `outside-window` outcome can include `window: "tracker" | "quest"` to name the tracker's collection window or the quest/campaign availability window, and older responses omit it. `trackerSkipCodes` exports the list, and later versions may add codes (`TrackerSkipCode` is an open string); a skip recorded before Relay stored codes, whose reason matches no rule, has `legacy`. A skip is decided again, under the quest's current release and every rule, only by the member event that clears its code, shortly after it commits (a durable workflow): linking the account (`unlinked-account`), removing a conflicting link (`ambiguous-account`), joining the project (`not-joined`), enrolling in the quest (`not-enrolled`); activity from before the link or enrollment then becomes a `historical` skip naming the member, or is admitted under `historical: "automatic"`. Every other skip is final unless retried. Read `management.trackers.triggerOutcomes(tracker, { quest?, record?, member?, state?, code?, limit?, cursor? })` with `read` (`{ items, nextCursor }`, newest first). `management.trackers.retryOutcome(tracker, { actionId, quest, record, observedAt })` with `review` re-runs the rules under the quest's current release, filter, links, membership and enrollment, releasing historical activity whatever the policy, and reuses the automatic attempt's action ID; it returns 409 with the current reason as `error` and its skip code as `code` (such as `already-completed`) when it still cannot admit, and 409 "The record no longer matches the quest's filter". Members read their own outcomes with `relay.quests.outcomes({ quest?, state?, code?, cursor? })` (`{ items, nextCursor }`, newest first, 50 a page; each item `{ quest, tracker, record, observedAt, state, code, reason, window?, attempt, claimable, createdAt, updatedAt }`; React `useQuestOutcomes`) and claim one with `relay.quests.claim({ actionId, quest, tracker, record, observedAt })` when it is `claimable` (a `historical` skip on a quest with `historical: "claim"`): it is decided again with historical activity released and returns the outcome, or rejects with the failing rule's skip code, or `not-claimable`. Console shows outcomes under the quest's **Trigger outcomes** and a record's **Quest outcomes**, with **Retry admission** for reviewers, **Enroll the member** on a `not-enrolled` skip for operators with `enroll`, and the quest's enrollment, `since` and `historical` under **How it works**. To rank the value itself: an external score gets its own ledger, updated with absolute values (`points.record({ kind: "set" })`); see [Rank an external score](https://console.domino.run/docs/build/leaderboards#rank-an-external-score). Trackers do not issue ongoing engagement rewards or reverse prior rewards. See [trackers](https://console.domino.run/docs/build/trackers) for full examples, and the [trackers reference](trackers.md) for collection options, record reads, trigger rules, outcome codes and engagement accounting.

### Automatic content admission

Author content quests with `trigger: content({ tracker, enrollment?, since?, historical?, where? })` and omit `completion`. The source must declare content: `x.search()` does so, or use `customSource(namespace, { identities, content: { body: z.object({ ... }), metrics: ["plays", ...] } })`. Custom content records carry `data: { body, metrics? }`; missing counters are unknown, zero is measured. The record ID identifies the content item, its author the creator and `occurredAt` the publication time. Plain custom sources and Discord sources cannot trigger content stages.

Content uses the shared tracker admission pipeline, fixed to the first observation and author beneficiary. Defaults are `enrollment: "none"`, `since: "activation"` and `historical: "claim"`. Choose `since: "window"` to count from the tracker window or `"enrollment"` with required enrollment. Each skip has a reason and is decided again when its blocking identity or membership condition changes. Admission pins the current release and beneficiary; later body edits qualify under that release. `ctx.tracker.content` holds `{ body, revision, observation }`. Attribution and enrollment remain on `ctx.tracker`; `ctx.content`, `ctx.tracker.x` and `perContent` are removed. Budgets use the record's occurrence time through `ctx.eligibleAt`.

Members enroll with `relay.quests.enroll`, read `relay.quests.outcomes`, and release claimable historical outcomes with `relay.quests.claim({ actionId, quest, tracker, record, observedAt })`. Management reads `trackers.triggerOutcomes(tracker, { quest, record? })` and retries with `trackers.retryOutcome(tracker, { actionId, quest, record, observedAt })` and `review` permission. Neither a retry nor a claim bypasses other admission rules. The old content admission/claim/approval APIs are removed. Content qualification history remains `content.qualifications({ quest, content?, member?, cursor? })`, scoped to the session member for participant reads. Console exposes content admission, qualification, performance and engagement through **Community → Trackers → Quest outcomes** for both X and custom content.

### Content performance gates

A content quest can declare `performance: { metrics: ["likes"], maxAgeMs: 3600000, evaluate: ({ metrics }) => metrics.likes >= 100 ? decision.accept() : decision.reject({ reason: "Reach 100 likes." }) }`. Only content triggers support this stage. Select a nonempty unique subset of the source’s declared metrics; X provides `likes`, `replies`, `reposts`, `quotes`, `views`, and `bookmarks`. Callback types expose exactly the selected counters. Context also includes settings, member, pinned `tracker.content` and sample `observedAt`, without provider capabilities.

Content approval is cached and does not award until performance passes. Each new provider observation can queue performance while retaining the content approval; body edits or changes to the verified account selection require new content approval first. Metrics collected during content review are used after approval. Missing counters and stale/future samples wait without executing code or fabricating zeros. `maxAgeMs` defaults to one hour and is bounded at 24 hours. New samples, body edits, removal, and ownership changes invalidate outstanding performance decisions and reviews.

Performance gates the content-approved reward selection; its callback cannot change those rewards. Its result is `decision.accept()`, `decision.review({ reason })` or `decision.reject({ reason })`; a result that carries `rewards` fails the evaluation, and the types reject it. The first acceptance resolves the ordinary completion and budget outcome. Later threshold drops preserve prior awards, and recoveries cannot repeat effects or retry budget exclusions. Add a separate engagement rule for daily settlement. Automatic reversals remain unsupported.

`content.performance({ quest, content, member?, cursor? })` returns `current` eligibility/reason/attempts/reward and up to 100 retained performance `items` with metrics and observation time, newest first. Continue with `cursor: page.nextCursor`. Participant identity is session-derived; management requires `read`. Content-only history remains in `content.qualifications`. See [performance thresholds](https://console.domino.run/docs/build/trackers#require-an-engagement-threshold-before-awarding).

### Daily engagement points

Use `defineEngagementReward` from `@domino-sdk/relay/authoring` with `key`, a content quest handle in `qualifiedBy`, a point-ledger handle in `ledger`, `baseline: "historical"`, nonempty `metrics`, and synchronous `target({ metrics })`. The target returns nonnegative safe whole-number cumulative points; use a cumulative floor for fractional rates. The isolated formula has no network/member capabilities and must not use time or randomness. Optional `maxAgeMs` defaults to one hour (maximum 24 hours). Optional `budgets` is a list of `{ scope: "project" | "member", period: "day" | "week" | "month" | "lifetime", limit }` with nonnegative limits, at most one per scope and period, independent per rule; periods are those of the [project calendar](#project-calendar), and each point entry is charged to the period of its `effectiveAt`. Accounts settle at most once per calendar day, plus a closing settlement when a leaderboard over their points closes.

Add `engagement: [{ entry: "engagement.ts" }]` to `relay.json`, with optional `exportName` for a named export and at most 50 entries. The module can live anywhere relative to the manifest. Engagement entries are explicit, independent of quest discovery globs. `domino deploy --preview` reports `create`, `unchanged`, or `change`, and `deploy --resume DEPLOYMENT_ID` publishes the saved preview. Tracker, quest, ledger, and rule can publish together. The initial discovery cutoff is publication time, not preview time. Omitting an entry never deletes a rule.

A normal `domino deploy` previews and applies formula changes atomically with the other resources. A stale version or accounting preview rejects the whole deployment with `409`; preview again. Retrying a committed deployment ID returns its saved receipt. `domino dev` synchronization and hosted catalog artifacts carry the same entries. CLI publication remains test-only, while live previews remain available.

For a standalone API request, bundle the default export with `buildQuest` from `@domino-sdk/relay/build`, then call `management.deployments.preview({ engagement: [{ code }] })` and `management.deployments.publish(preview.id)` with `publish` permission. Keep the deployment ID on retries. Formula versions are immutable: identical code replays, while changed code uses the preview/apply workflow below. Another key is a separate additive rule, not a replacement. Only discoveries after publication can accrue; there is no implicit backfill.

Current content qualification and, when configured, performance approval gate accounting. Complete fresh counters advance component-wise high-water values. Accounting processes the latest eligible sample; it does not replay earlier peaks from review or worker downtime. Settlements become due at the first midnight of the [project calendar](#project-calendar) after initial eligibility, then run at most once per project calendar day. Delayed settlements use their actual calendar day and available fresh observation, not reconstructed midnight samples. Entries are dated by measurement: each takes effect when the sample that measured its growth was observed, and growth between two samples that spans a leaderboard window or budget period boundary splits there in proportion to time (rounded down, remainder to the later part), one entry per side. Budgets pay partially; excluded growth advances the accounted boundary permanently. Credits go directly to the balance without another completion or referral payout. Counter drops and later rejection do not reverse credits.

Quest pause stops accounting. The first fresh eligible sample after resume becomes the new baseline; previously earned pending points survive and paused growth through that observation is skipped. Enrollment-only baselines, historical backfill, quality multipliers, reversals, restorations, and specialized Console controls are not supported. Read [daily settlement](https://console.domino.run/docs/build/trackers#settle-engagement-points-daily) before proposing them.

Management `engagement.list()` and `engagement.page({ rule, content, member?, cursor? })` require `read`. The page returns the rule, nullable last processed account checkpoint (`earned`, `accounted`, `paid`, `excluded`, `pending`, `counters`, `baseline`, `observation`, `nextSettlementAt`, `reason`), and up to 100 immutable settlement receipts, newest first. Continue with `cursor: page.nextCursor`; receipts reference their evidence, qualification, and point entry.

### Engagement formula versions

After changing only `target`, bundle the code and call `management.deployments.preview({ engagement: [{ code }] })`. Review the formula change engagement entries, then call `management.deployments.publish(preview.id)`. Both require `publish`. Management keys can publish in test; the change reaches live through a promotion. Retry publication with the same deployment ID. A changed requested version, admission set, or accounting checkpoint invalidates the preview (`409`). Quest, balance, metrics, baseline policy, freshness policy, and budgets must stay unchanged. The older engagement-only `previewChange` and `change` methods remain deprecated compatibility APIs.

The preview reports retained accounting, with project totals as decimal strings. It explicitly defers settlement; it does not promise a future payout. The supported boundary is `available-eligible-observation`: the fresh eligible sample available when a post transitions, possibly predating the request by up to `maxAgeMs`. Never describe it as an exact publication-time counter. Each post settles the current formula through its high-water counters and starts the latest requested formula at the same baseline in one transaction. The old earned/accounted totals, budget spend, and exclusions survive. Boundary credits can occur in addition to the daily receipt, using the same UTC-day budget.

Posts waiting on qualification, ownership, metrics, pause, or scoring retain their current formula. A later request supersedes unapplied versions for that post; no intermediate earnings are invented. New discoveries use discovery-time versions, and late admission preserves those original terms. Formula changes preserve the original discovery cutoff and do not backfill content. A failing requested formula can be superseded; an unscorable current formula cannot be force-skipped.

Read `management.engagement.ruleVersions(rule, { cursor? })` for the immutable request history. `engagement.boundaries({ rule, content, member?, cursor? })` returns `current`, `requested`, and up to 100 receipts with `from`, `to`, `baseline`, provider `observedAt`, and `settlement`; continue with `cursor: nextCursor`. Management requires `read`, and participant reads are session-scoped. `engagement.list()` shows requested versions; `engagement.page()` retains its shape and shows the account's governing version. See [formula changes](https://console.domino.run/docs/build/trackers#change-a-formula-without-repricing-past-engagement).

## Declared integrations

Import the `discord` and `x` modules from `@domino-sdk/relay/authoring` and list the ones the quest uses: `integrations: [discord, x]`. Only declared provider readers appear in the evaluator context and its inferred type. Discord role rewards and resource selectors require the Discord declaration too. An X image reference requires X. Wallet, member, trackers, images, review and Telegram (`ctx.telegram`) stay available without provider modules. A new publication changes declarations for new attempts; existing attempts retain their pinned release.

## X verification

Hosted evaluators expose `ctx.x.connection()`, `profile()`, `post({ postId })`, `follows({ userId })`, and `reposted({ postId })`. Use numeric X IDs as strings. Resolve a handle on the server with `management.x.lookup({ username })` or `domino x lookup HANDLE --json`: `{ id, username, name }` for the account that held the handle when X was last read, at most a day ago (a leading `@` is accepted), 404 `not-found` when none does; `read` permission, 30 a minute per user and 300 an hour per organization (429 `rate-limited`). It resolves the handle only; it does not show who owns the account, and handles can change hands, so store the `id`. `connection()` returns `{ userId }` or null and needs no provider credential. `profile()`, `follows()` and `reposted()` return `null` without a linked participant account, so check that before reading `false` as "does not follow"; `post()` needs no linked account and returns `null` for a post that does not exist. Provider reads need platform-configured `X_API_WRAPPER_API_KEY`. Declare `integrations: [x]` to enable these readers. X requires no project integration installation.

Profile, follow and repost checks bind to the participant's linked account. Profile fields are `id`, `username`, `name`, `bio`, `profileImageUrl`, `followersCount`, `verified`; unavailable optional fields are null. A post read returns content, author, metrics, nullable reply/quote/repost references (`inReplyToStatusId`, `inReplyToUserId`, `quotedTweetId`, `retweetedTweetId`), and `media` as `{ type, url }` in post order. Check profile pictures and post images with [image checks](#image-checks), not by fetching URLs. Post lookup does not assert authorship: compare `post.author.id` to `connection.userId`, check target references and the required time window. Missing metrics are null.

Reads are captured per evaluation run. The account is pinned at the run's first X read, so a change or disconnect afterwards does not affect that run. A new submission or scheduled deferred run can use another account and fresh evidence. Provider failures throw `CapabilityError` with code `provider-unavailable`, never negative evidence or acceptance. Quest code receives no X credential. These capabilities do not expose like checks or automatic X observation delivery. Combine them with URL/text submissions and text AI review below.

## Linked wallets

`ctx.wallet.list()` returns the member's wallets linked to the project: `{ chain, network, address, verifiedAt, revision, signIn }[]`, ordered by chain, newest first. `chain` is `"solana"`, `"eip155"` or `"cosmos"`; a Cosmos wallet's `network` is the CAIP-2 ID of the chain the app requested when linking it, such as `"cosmos:cosmoshub-4"`, and `null` for the others. `network` is not proof: Domino does not verify it, and any app can link an address under any `cosmos:<chain ID>`. Gate Cosmos wallets on the address: check its bech32 prefix against your own list of accepted chains, such as `linked.chain === "cosmos" && linked.address.startsWith("cosmos1")` for Cosmos Hub, and read the chain itself for balances or activity. The list is pinned at each evaluation run's first wallet read. Links and unlinks do not change that run, and its retries replay the same list. A scheduled run after defer or progress reads the current links. It returns `null` when the member linked no wallet, never an empty list; reject with a message that asks them to link one. Each call is one of the attempt's eight capability calls. The wallet a member signed in with is in the list with `signIn: true`; a member who signed up with a wallet may have no other account, so do not require one only to identify them.

`ctx.wallet.list()` returns all linked wallets or null; `ctx.wallet.balance({ chain: "solana" })` reads their total SOL balance. Pass a linked `address` to read one wallet. Wallet addresses belong to verified members only: an account is linked to at most one member.

## Wallet balances

`ctx.wallet.balance({ chain, token?, address? })` reads a fungible balance for the member's pinned linked wallets:

| `chain`                                                     | `token`                                                                    | Reads                                                      |
| ----------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `"solana"` (or `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp"`) | SPL or Token-2022 mint (base58); omit for SOL                              | Every linked Solana wallet, all token accounts of the mint |
| `eip155:<chain ID>` (CAIP-2), such as `"eip155:8453"`       | ERC-20 contract (`0x…`, checksummed or one case); omit for the native coin | Every linked EVM wallet, at the latest block               |
| `cosmos:<chain ID>` (CAIP-2), such as `"cosmos:cosmoshub-4"` | Bank denomination; required (such as `uatom` for the native coin)        | Wallets linked on that exact network, at one fixed block   |

The SDK types EVM and Cosmos chains as CAIP-2 IDs and validates their syntax; a Cosmos query always requires a denomination. Which chains the platform reads is a server list: `management.walletBalanceChains()` or `domino api GET /wallet-balance-chains` (`read` permission) returns `{ chains: [{ chain, name, testnet }] }`, with Solana, the Cosmos chains (Cosmos Hub, Osmosis and any other the platform operator configured), then EVM chains by chain ID. It returns `{ chain, token, decimals, total, wallets: [{ address, amount }], at, observedAt }`. `chain` is CAIP-2, and `token` is the canonical mint, EIP-55 address, or Cosmos bank denomination (null for a Solana or EVM native coin). `total` and each `amount` are integer strings in the token's smallest unit. `decimals` is null when token metadata does not provide it: send that to review or reject, never guess. `at` is `{ kind: "slot" | "block", number }`. Convert with `parseUnits(value, decimals)` (bigint; rejects excess precision) and `formatUnits(amount, decimals)`, both from `@domino-sdk/relay/authoring`: `BigInt(total) >= parseUnits("10", decimals)`. With a `settings.amount()` minimum, compare base units directly: `BigInt(total) >= BigInt(settings.minimum)`.

- `address` must be one of the member's pinned wallets of that family and, for Cosmos, that exact network (any letter case for EVM); otherwise the call throws `CapabilityError` with code `author-error` and message "Read only the member's linked wallets". Quest code cannot probe other addresses.
- No readable wallet returns `null`, never a zero total. Reject with a message asking the member to link one.
- Errors never mean ineligible. An unsupported EVM or Cosmos chain, a malformed token, a Solana account that is not a mint and no contract at the EVM address throw `author-error`; provider outages and network verification mismatches throw `provider-unavailable`.
- It reads the same pinned wallet list as `ctx.wallet.list()`. Each read is captured and replayed on retries, and uses one of the attempt's eight capability calls. EVM reads through dRPC are approximate `latest` reads. Cosmos reads verify the endpoint's chain ID and pin bank and metadata requests to one block.
- Domino chooses every endpoint: neither quest code nor project settings can supply an RPC or REST URL. For an unlisted chain or other on-chain state, feed a custom tracker source from the customer's own indexer, with `identities: ["wallet"]` records keyed by CAIP-10 account (see Tracker triggers), and read those records instead.
- Keep chain and token identifiers in settings, never in browser input. `settings.chain()` passes to `chain` as it is; an EVM `token` is typed `` `0x${string}` ``, so check a text setting with a type guard (`/^0x[0-9a-fA-F]{40}$/`) before passing it.
- An EVM `token` that is an NFT contract (ERC-165 reports ERC-721 or ERC-1155) throws `author-error` naming `ctx.wallet.nfts()`. No historical balances. When staking mints a receipt token (a liquid-staking token, ERC-4626 vault shares), read that token here; for stakes a contract keeps in a mapping use [`ctx.wallet.call()`](#contract-state), and for Cosmos native staking [`ctx.wallet.delegations()`](#cosmos-delegations).

Snippet `token-holder` installs a complete quest: chain (`settings.chain()`), token and minimum (`settings.amount()`, 6 decimals) settings, and the total across linked wallets compared in base units. The name discloses the evidence: a balance at verification time, not continued holding. The same quest is `packages/sdk/examples/token-holder-quest.ts`.

## NFT holdings

`ctx.wallet.nfts({ collection, address? })` counts the NFTs of one collection the member's pinned linked wallets of its family hold: EVM wallets for an EVM collection, Solana wallets for a Solana one. `collection` is a CAIP-19 asset type: `eip155:<chain ID>/erc721:<contract>` for an ERC-721 collection, `eip155:<chain ID>/erc1155:<contract>/<token ID>` for one ERC-1155 token ID (decimal, at most 2^256 − 1), or `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/nft:<collection mint>` for a Metaplex verified collection on Solana mainnet, named by its collection NFT's mint. An EVM chain is one `management.walletBalanceChains()` lists. Take it from a `settings.nft({ label, description?, default })` setting, typed `NftCollection`; never from member input. `isNftCollection()` and `parseNftCollection()` from `@domino-sdk/relay/authoring` check the syntax.

It returns `{ collection, chain, standard, total, wallets: [{ address, count }], at, observedAt }`: `collection` with an EVM contract EIP-55 checksummed, `standard` `"erc721"`, `"erc1155"` or `"metaplex"`, `total` and each `count` integer strings (ERC-721 `balanceOf(owner)`, ERC-1155 `balanceOf(owner, id)`, Solana the collection's NFTs held), and `at` the approximate latest EVM block (`{ kind: "block", number }`) or the lowest Solana slot read (`{ kind: "slot", number }`). Holding at least one is `BigInt(total) >= 1n`.

- On Solana an NFT counts when a linked wallet has a token account (SPL Token or Token-2022) holding exactly 1 of a mint with 0 decimals and supply 1, whose Metaplex metadata names the collection with `verified: true` and whose token standard is not fungible. An unverified collection field is ignored: anyone can write one. Compressed NFTs (they need a DAS-capable RPC) and Metaplex Core assets are not counted.
- Publication, Console settings edits and unarchiving check every `settings.nft()` value, archived quests included: the syntax, then on EVM a chain Relay reads and that the contract reports the named standard through ERC-165 `supportsInterface`, and on Solana that the mint is a Metaplex collection NFT (a 0-decimal, supply-1 mint with a unique master edition, `max_supply` 0, and token standard `NonFungible`, `ProgrammableNonFungible` or unset, whose metadata has collection details, or, for a legacy unsized collection, that is not itself a verified member of another collection). A token contract, an ERC-1155 contract written as `erc721` (and the reverse), a fungible Solana mint, a mint without Metaplex metadata, a print or limited edition, and an NFT that belongs to a collection (the message names that collection) are refused with `422 validation-failed` naming the setting's label and what is wrong. An RPC outage refuses with `503 unavailable`; retry. A platform without a dRPC key refuses every EVM collection with `503 not-configured`, since none can be checked.
- `address`, `null` and failures follow `ctx.wallet.balance()`: an address the member did not link, a malformed collection, a chain Relay does not read, and a contract that rejects the read throw `author-error`; no linked wallet of the collection's family returns `null`; outages throw `provider-unavailable`. Reads are captured, replayed on retries, and count toward the eight capability calls.
- `WalletReader.nfts` is optional in the type so that older test doubles compile; hosted evaluations always supply it. The testing kit answers it from `facts.wallet.nfts` (called with `{ collection, address }`), null while `facts.wallet.list` links no wallet of the collection's family; a run's `calls` does not list NFT reads.
- No token ID listing or Cosmos NFTs.

Snippet `nft-holder` installs a complete quest: a `settings.nft()` collection, accepted at a total of at least one across the linked wallets of its family.

## Contract state

`ctx.wallet.call({ call, args?, address? })` calls one `view` or `pure` function of an EVM contract once per pinned linked EVM wallet, every call at the same block, through dRPC on the chains `management.walletBalanceChains()` lists. Use it for state a contract keeps per address, such as `stakedOf(address)`, `getVotes(address)` or `userInfo(uint256 pid, address user)`. When staking mints a receipt token, read the token with `ctx.wallet.balance()` instead.

`call` is the contract's CAIP-10 ID, a space, and the function's human-readable signature: `"eip155:8453:0x… function stakedOf(address account) view returns (uint256)"`. Take it from a `settings.contractCall({ label, description?, default })` setting, typed `ContractCall`; never from member input. The function must be `view` or `pure`, take elementary parameters (`address`, `bool`, `string`, `bytes`, `bytes1`…`bytes32`, `uint8`…`uint256`, `int8`…`int256`; `uint` and `int` mean 256 bits), and return exactly one `uint`, `int`, `bool` or `address`. Arrays, tuples, structs and several return values are refused. `isContractCall()`, `parseContractCall()` and `parseContractFunction()` from `@domino-sdk/relay/authoring` check the syntax.

`args` lists the function's arguments in order (default `[]`). `{ wallet: true }` stands for each wallet's address and may fill at most one `address` parameter; the other values come from settings or code, never from member input. Integers are safe-integer numbers or decimal strings within the type's range, `bool` a boolean, `address` a checksummed or one-case hex address, `bytes` and `bytesN` `0x` hex of the right length.

It returns `{ call, chain, contract, returns, total, wallets: [{ address, value }], at, observedAt }`: `call` canonical (contract EIP-55 checksummed, parameter names dropped, `uint` as `uint256`), `returns` the return type, each `value` a decimal string for an integer, a boolean for `bool`, or an EIP-55 address, `total` the integer values' sum as a decimal string (null for `bool` and `address`), and `at` the block every call read (`{ kind: "block", number }`). Reaching a minimum is `BigInt(total) >= BigInt(settings.minimum)`.

- Publication, Console settings edits and unarchiving check every `settings.contractCall()` value, archived quests included: the syntax, and that Relay reads the chain. A wrong value is refused with `422 validation-failed` naming the setting's label.
- A revert throws `author-error` with the node's reason (such as `execution reverted: pool does not exist`); so do no contract at the address and return data that does not decode as the declared type. Arguments that do not fit the parameters, and an `address` the member did not link, throw `author-error` too. No linked EVM wallet returns `null`; an RPC outage, or a platform without its dRPC key, throws `provider-unavailable`. Reads are captured, replayed on retries, and count toward the eight capability calls.
- `WalletReader.call` is optional in the type so that older test doubles compile; hosted evaluations always supply it. The testing kit answers it from `facts.wallet.call` (called with `{ call, args, address }`), null while `facts.wallet.list` links no EVM wallet; a run's `calls` does not list contract calls.
- No state on Solana or Cosmos contracts, no history and no events.

Snippet `contract-state` installs a complete quest: a `settings.contractCall()` function called with the wallet as its only argument, accepted when the `uint` values reach a `settings.amount()` minimum.

## Cosmos delegations

`ctx.wallet.delegations({ chain, validator?, address? })` reads the native staking delegations of the member's wallets linked on one Cosmos SDK chain that `management.walletBalanceChains()` lists, through the same platform-owned REST endpoint as Cosmos balances, at one fixed block. `validator` is a validator's operator address (bech32 with a `…valoper` prefix, such as `cosmosvaloper1…`) to count only delegations to it; omit it to count every validator.

It returns `{ chain, denom, validator, total, wallets: [{ address, amount }], validators: [{ validator, amount }], at, observedAt }`: `denom` the chain's staking denomination (`bond_denom`, such as `uatom`), every amount an integer string in its base units, `total` the sum over `wallets`, `validators` what the wallets delegate to each validator (only `validator` when given; none at zero), and `at` the block (`{ kind: "block", number }`). Delegating at least a minimum is `BigInt(total) >= BigInt(settings.minimum)`.

- Every request after the latest block is pinned to its height with `x-cosmos-block-height`. An endpoint that cannot pin the staking queries answers at its own height: when every answer reports the same height, that block is recorded in `at`; answers at different heights are an outage.
- `address`, `null` and failures follow `ctx.wallet.balance()` for Cosmos: only wallets linked on that exact network are read; a chain Relay does not read, a malformed validator and an address the member did not link throw `author-error`; no wallet on the chain returns `null`; outages throw `provider-unavailable`. Reads are captured, replayed on retries, and count toward the eight capability calls.
- `WalletReader.delegations` is optional in the type so that older test doubles compile; hosted evaluations always supply it. The testing kit answers it from `facts.wallet.delegations` (called with `{ chain, validator, address }`), null while `facts.wallet.list` links no wallet on the chain.
- Bonded delegations only: no unbonding delegations, rewards, history or governance votes, and no Solana or EVM native staking.

Snippet `cosmos-staker` installs a complete quest: a Cosmos chain, an optional validator and a minimum delegation in base units.

## On-chain actions

EVM contract events (`ctx.wallet.events()`) and Solana transactions (`ctx.wallet.transactions()`) are in [on-chain actions](onchain.md). Snippet `onchain-action` installs a complete EVM event quest, and `solana-action` a quest for a transaction a linked Solana wallet signed that called a program, optionally receiving a minimum of a token.

## Text and URL submissions

Use `manual({ input: evidence.text(...) })` or `manual({ input: evidence.url(...) })`. Evaluators receive `ctx.input.kind` and the original string in `ctx.input.value`.

| Input             | Options                              | Defaults and limits                                                                                            |
| ----------------- | ------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| `evidence.text()` | `required`, `minLength`, `maxLength` | Required; 1–10,000 UTF-16 code units. Configure a maximum up to 10,000. Whitespace-only input counts as empty. |
| `evidence.url()`  | `required`, `allowedHosts`           | Required; at most 2,048 characters; HTTP or HTTPS without credentials.                                         |

`allowedHosts` contains exact hostnames, without schemes, ports, paths or wildcards. Matching is case-insensitive; listing `example.com` does not allow `sub.example.com`. Omit the list to allow any HTTP/HTTPS hostname. For an optional field, submit an empty string. Nonempty optional values still obey the constraints. Submitted values are stored unchanged, including text whitespace and URL query strings.

```ts
import {
  defineQuest,
  settings,
  manual,
  evidence,
  once,
  decision,
} from "@domino-sdk/relay/authoring"

export default defineQuest({
  key: "feedback",
  title: "Suggest an improvement",
  settings: settings.object({}),
  trigger: manual({ input: evidence.text({ minLength: 20, maxLength: 2000 }) }),
  completion: once(),
  rewards: [],
  review: () => ({ enabled: true }),
  async evaluate(ctx) {
    return decision.review({
      reason: `Review feedback (${ctx.input.value.length} characters)`,
    })
  },
})
```

Treat every submission as an untrusted claim. Validation checks the input shape, not truth, authorship, ownership, or safety of the destination. Relay never fetches a submitted URL automatically. Do not put a submitted string into an HTML renderer or a fetch request.

For X posts, use `xPostId` from `@domino-sdk/relay/schema` to parse the URL, then call `await ctx.x.post({ postId: id })`. The parser returns `null` for unsupported hosts and paths. The capability accepts only a decimal post ID, calls the configured X provider, and captures the result for the attempt; it returns `null` when the post does not exist. Captured reads remain stable across evaluator retries; provider failures throw `CapabilityError`. The URL's username is not proof of authorship. Check the returned post's content and author against your eligibility rules, or request human review. Participants must link X first. Compare the returned author ID with `userId` from the non-null result of `await ctx.x.connection()` to check authorship. The platform must configure `X_API_WRAPPER_API_KEY`; missing configuration throws `provider-unavailable` rather than passing the claim.
## Review text with AI

Call `ctx.review.text({ text, criteria })` from an evaluator. Supply the text to judge and author-defined criteria. The typed `TextResult` contains `kind: "pass" | "fail" | "unclear"` and a short `reason`. Map these to `decision.accept()`, `decision.reject({ reason })`, or `decision.review({ reason })`. Enable the quest's `review` option before returning a staff-review decision: with `review: () => ({ enabled: false })`, `decision.review({ reason })` becomes a rejection that keeps the reason and marks the attempt `requestedReview: true`, and Console shows "Would have gone to review; human review is off".

Use text from captured tracker records or a verified Discord message. For an X post, query an X tracker with `{ member: ctx.member.id, verified: true }`, parse the record with `xTweetSchema`, and pass `post.content`. Check the post type, campaign window, and required tags in quest code. AI review judges content; it does not fetch URLs, establish ownership, or prove whether a human wrote the text. A verified X tracker record means account ownership was verified, not that the content is trustworthy.

| Limit             | Value                                                             |
| ----------------- | ----------------------------------------------------------------- |
| Text              | 1–16,000 JavaScript string units; whitespace-only text is invalid |
| Criteria          | 1–2,000 units after trimming                                      |
| Result reason     | 1–2,000 units                                                     |
| Generated output  | 2,048 tokens                                                      |
| Provider response | 64 KiB                                                            |
| Provider timeout  | 12 seconds, including response reads                              |
| Capability budget | Shares the existing eight calls per evaluation                    |

Text review accepts any text the evaluator has obtained, including submitted content stored by a trusted application in a custom tracker. Use `evidence.text()` or `evidence.url()` to collect participant submissions. Do not place participant text in `criteria`.

The provider treats the text as untrusted evidence and instructs the model to ignore embedded instructions and flag suspected manipulation as `unclear`. AI can still misjudge adversarial content. Use staff review when the outcome requires it. Credentials stay on the backend, using the same `CLOUDFLARE_ACCOUNT_ID` and `CLOUDFLARE_AI_TOKEN` configuration as photo review. The release's photo `provider` selector does not select the text backend.

Network errors, timeouts, refusals, truncated output, and malformed results throw `CapabilityError` with code `provider-unavailable`. Never catch those errors and return acceptance. Successful results are captured before returning to quest code and reused on retries. Keep capability order, text, and criteria deterministic: changed inputs at a captured sequence are rejected. A crash before capture can cause another provider call. Captured text and criteria are stored with the attempt; Prometheus labels contain only provider, operation, HTTP status, and outcome.

## Image checks

Evaluators call `ctx.images` with image references; the backend fetches and decodes the image and returns only the result. Quest code never receives bytes or URLs.

| Reference | Image |
| --- | --- |
| `{ evidence: ctx.input.photo }` | This attempt's uploaded photo. Another attempt's evidence is refused. |
| `{ x: "profile" }` | The linked X account's current avatar, fetched at 400×400. Requires the X provider and a linked account: without one, the call throws `CapabilityError` with code `not-linked`. |
| `{ x: "post", postId, index? }` | The post's attached image at `index` 0–3 (default 0); videos and GIFs use their preview image. |
| `{ asset: "key" }` | A reference image deployed from `relay.json` `images: [{ key, path }]`: JPEG, PNG or WebP of at most 5 MiB and 25 megapixels, up to 10 per deployment, stored privately. `domino deploy` uploads new files directly to storage; the deployment carries only hashes. Hosted builds stage them from the repository. |

| Method                          | Result                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `inspect(image)`                | `width`, `height` (after EXIF rotation), `format`, `bytes`, `sha256`, 16-hex-character `phash` and `dhash`.                                                                                                                                                                                                                                                                                                                             |
| `compare(a, b)`                 | `identical` (same bytes), `phashDistance` and `dhashDistance` from 0 to 64. About 8 or less on `phashDistance` means the same picture resized or re-encoded; unrelated photos are usually 25 or more. Quest code chooses the threshold; expose it as a `settings.integer` so operators can tune it. Crops, frames and overlays change hashes substantially; use `review` for customized pictures.                                       |
| `review({ images, criteria })`  | `pass`, `fail` or `unclear` with `reason`, for 1–4 images numbered in order ("image 2 is the reference"). Criteria up to 2,000 characters.                                                                                                                                                                                                                                                                                              |
| `extract({ image, fields })`    | `{ values, reason }` for 1–12 fields declared as `{ type: "string" \| "number" \| "boolean", description }`. A value is `null` when the image does not show it clearly. Types are inferred from the declaration.                                                                                                                                                                                                                        |
| `seen(image, { maxDistance? })` | Registers the image for this attempt, then returns `matches` (other participants whose earlier images are within `maxDistance`, 0–8, default 8), `closestDistance` and `firstSeenAt`. The same participant never matches. Only images passed to `seen` are recorded, per project and environment. The image is recorded when `seen` runs, even if the attempt is later rejected, so concurrent submissions of one photo see each other. |

Each call uses one of the evaluation's eight capability calls and is captured for replay. AI inputs are re-encoded JPEGs without metadata; the model is told to ignore instructions inside images but can still misjudge them. A screenshot is a participant's claim: compare extracted values with verified data, such as `ctx.x.profile().username`, and send `null` or uncertain results to staff. Missing reference images, undecodable images and missing X pictures throw `CapabilityError` with code `author-error`; provider failures throw `provider-unavailable`. Deploying an existing key replaces its image for later checks; removing it from `relay.json` does not delete it. Checks describe the image at verification time and do not prove retention. `ctx.review.photo()` is deprecated and will be removed in 2.0.0: review an uploaded photo with `ctx.images.review({ images: [{ evidence: ctx.input.photo }], criteria })`. Snippets `x-profile-picture`, `unique-photo` and `community-screenshot` install complete quests with tunable thresholds; the same quests are SDK examples in `packages/sdk/examples`. See [check images](https://console.domino.run/docs/build/images).


### Evaluator protocol

The current `defineQuest()` bundle declares `protocol: 2`, and `describe()` emits `evaluatorProtocol: 2`. Relay verifies that they agree during description and chooses the matching execution adapter. Protocol 2 is the first stable contract and supplies the pinned `ctx.calendar`. Protocol-less, pre-release protocol-1 and unknown protocol versions are refused before evaluation. Rebuild and republish pre-v1 bundles with the current SDK; changing their version field alone is not an upgrade. Do not strip either field when transforming a bundle. SDK v1 upgrade details: [evaluator protocol](https://console.domino.run/docs/reference/upgrade-to-v1#evaluator-protocol).

Relay stores the declared protocol with each release and runs the matching host adapter, so a published bundle keeps its context and capability behavior as the host evolves. The SDK writes the declaration; normal publication needs no configuration. Future stable protocol versions keep their adapters for the supported lifetime of their published bundles.

## Shared SDK runtime

A quest or quest type bundle imports the SDK instead of carrying its own copy. When the installed `@domino-sdk/relay` ships a runtime, `domino build`, `deploy`, `dev` and hosted builds leave `@domino-sdk/relay/authoring`, `@domino-sdk/relay/schema` and `zod` out of each bundle. `zod` stays out only when your app resolves the version the SDK runtime carries and the quest imports it as `zod`; subpaths such as `zod/v4` or `zod/mini` are bundled into the quest. The bundle records the runtime it imports, and the deployment carries that runtime once. Relay stores it by content and runs every release with the exact runtime it was built against, so upgrading the SDK never changes a published quest. A quest bundle shrinks from about 170 KB to a few kilobytes. Other imports, such as a chain library, are still bundled into each quest that uses them.

An older SDK release that ships no runtime builds self-contained bundles, as before. A Relay that predates SDK runtimes refuses a deployment that carries one; `domino deploy` then says to update the Relay, such as restarting an outdated local stack. Trackers, leaderboards, referrals, engagement rules, offers, badges and point ledgers always bundle their dependencies.

## Recover a quest lifecycle change

`domino quests archive` and `domino quests unarchive` save the action ID, reason, expected release and scope before sending. After an uncertain response, run the error's `domino operations resume ID` command. It reuses that request and returns its committed result. Use `domino operations list --json` to find saved operations after restarting the CLI. A new archive/unarchive command without `--operation-id` starts a new operation. See [CLI automation](automation.md#recover-after-restarting-the-cli).

### Restrict an ingestion key

In Console **Settings → API keys**, choose **Ingest**, then **Selected trackers only**. The key stores immutable custom tracker IDs from that project and environment. A tracker outside the list returns 403 `TRACKER_SCOPE_DENIED` before ingestion. Omitting `ingestTrackers` or setting it to `null` keeps project-wide ingestion; an empty list is invalid. Hosted app keys remain project-wide. **Edit tracker scope** replaces the list without rotating the secret.

A Console session or an unscoped human CLI sign-in can create and edit personal keys; API keys and project-scoped playground sessions cannot. With a human CLI sign-in, use `domino api POST /tokens key.json`, `domino api GET /tokens`, `domino api PUT /tokens/<id>/scope scope.json`, and `domino api DELETE /tokens/<id>`. Creation takes `{ name, project, environment, permissions: ["ingest"], ingestTrackers: ["tracker-key-or-id"] }`; a scope edit takes `{ ingestTrackers: ["tracker-key-or-id"] }` or `{ ingestTrackers: null }`. Keep the one-time creation secret in the indexer's secret store. After an uncertain creation, list and revoke the unconfirmed key before issuing another; after an uncertain scope edit, list the key and reconcile its `ingestTrackers` before retrying.

### Service keys for a self-hosted backend

A backend that should keep working when people leave the organization uses a service key (`dsk_…`) instead of a personal key. An organization admin creates it in Console **Settings → API keys → Create a service key** for one project and environment, with chosen permissions (never `access`) and no expiry unless one is chosen. Its actor is `service:<key ID>`. It gets every key fence: it cannot approve live publication, manage test members, mint keys or use `/tokens`. Rotate without downtime with **Add secret**, deploy the new secret, then **Revoke** the old secret once its **Last used** stops changing. Agents cannot create service keys; ask the user to create one in Console and store it as `DOMINO_MANAGEMENT_KEY` in the backend's secret store.

## Distinct-action progress counters

Author `progress({ distinct, goal: count(3), window: calendar("week"), repeat: "once" })` as a quest completion policy. Omit the window for lifetime progress; use `rolling({ days: 7 })` for a trailing elapsed duration. These are native authoring builders; `distinct` reads trusted `KeyedContext`, including `calendar` and the pinned tracker observation; in the builder form, `completion: ({ progress }) => progress({ distinct: ({ tracker }) => … })`, it is typed from the quest and has its `settings`. Do not invent `ctx.progress.increment`.

Counters work on manual and tracker-triggered quests, not content or prerequisite-only automatic quests. Default settlement is once per calendar bucket, or once over the quest lifetime for lifetime/rolling windows. `repeat: "repeat"` consumes disjoint action batches. Calendar lateness defaults to seven days after bucket end, configurable with `latenessMs` from zero through 90 days. Expired rolling evidence is retained and cannot be reused. Budget-denied achievements are final, retain their exact evidence and budget ID, and do not qualify referrals.

Read `management.members.counters.history(member, quest, { cursor, limit })`. With review permission, use `management.members.counters.moderate(member, quest, contribution, { actionId, excluded, reason })` to exclude or restore unsettled qualifying evidence. CLI: `domino api GET /members/MEMBER/counters/QUEST` and `domino api POST /members/MEMBER/counters/QUEST/CONTRIBUTION/moderate correction.json`. Reuse the action ID for an identical retry. Settled evidence cannot be edited; revoke an awarded completion to correct its reward, without reusing its actions.

Compose qualified referrals with `questCompleted(qualificationQuest)`. Install the `qualified-referrals` snippet (see [Find a snippet](#find-a-snippet)): three verified tracker records complete a lifetime qualification quest. Its alternative continuous-membership quest checks Discord's current `joinedAt` for a seven-day interval and rejects early claims with a not-yet-eligible reason, allowing the member to submit again after the interval. Rejoining resets that interval. It does not promise continuous role possession or automatic polling beyond the platform's defer horizon.
## Resource lifecycle

Use `domino resources list --json` to inspect active and archived trackers, referral programs, leaderboards, engagement rules, collection slots, quest types, images and point ledgers. Archive or restore by key or ID:

```sh
domino trackers archive events --reason "Source retired" --environment test
domino trackers unarchive events --reason "Source returned" --environment test
```

Accepted kinds: `tracker`, `referral`, `leaderboard`, `engagement`, `collection`, `quest-type`, `image`, `point-ledger`. Read needs `read`; mutations need `publish`, an action UUID and a reason. Resource-specific commands accept `--action-id UUID`; retain that UUID and repeat the identical command after an interruption. The deprecated `resources archive/unarchive` commands save a journal and recover through `domino operations resume <uuid>`. Live refuses archive and restore from every credential, Console included, with `promotion-required`. A quest instance retains its separate `quests archive|unarchive` commands.

Management code uses `management.resources.list()` and `management.trackers.archive(reference, { actionId, reason, expectedArchivedAt })` and `management.trackers.unarchive(reference, { actionId, reason, expectedArchivedAt })`. Use `unarchive` to restore. Optional `expectedArchivedAt` fences the change against the timestamp or `null` the operator read. A different state returns a conflict; read again before deciding.

Archive retains IDs and historical data. Trackers stop new collection and ingestion; referral programs stop qualification and bonus accrual; leaderboards and collection slots leave participant reads; quest types refuse new instances while existing quests work; images refuse fresh named captures while stored evidence replays; engagement rules stop settlement but retain watermarks. Restoring engagement rules can pay growth since those watermarks. Archived point ledgers reject new definitions but keep existing rewards and balances working; restoring makes the ledger selectable again. Archived offers (`domino offers archive`) leave member reads; redeeming fails with `offer-paused`.

Archive state is a definition. Deleting declarations, redeploying or rolling back a code deployment never implicitly archives or restores a resource; a promotion carries test's archive state to live. Lifecycle changes invalidate earlier catalog previews. Tracker configuration and point ledger names/descriptions remain current operator state, while captured evaluator facts and accounting entries preserve history.

## Prerequisite automatic outcomes

An `automatic()` quest with prerequisites and no tracker records each eligibility transition as admitted or skipped. Completion, group inclusion, campaign enrollment, publication and freed occurrences check current eligibility; every committed point entry schedules an asynchronous current-balance check, including refunds and referral or leaderboard awards. Multiple events reuse a pending once occurrence's attempt. An attempt rejected with `budget-exhausted` leaves the occurrence available: the next member event checks current rules and creates one deterministic retry attempt. Evaluator, staff and revoked-occurrence rejections stay final. A missing prerequisite can qualify on a later transition; a paused event requires explicit operator recovery.

Read `management.quests.automaticOutcomes({ quest, member?, state?, code?, cursor? })`, or `domino api GET '/quests/automatic-outcomes?quest=QUEST'`. Admitted outcomes can include `linkedAttempt` with its ID, status and open code. Select up to 50 `quest-paused` IDs or admitted outcomes whose linked attempt has status `rejected` and code `budget-exhausted` or status `failed` and code `integration-required` (`isRetryableAutomaticOutcome` from `@domino-sdk/relay/schema` tells you), then call `management.quests.previewAutomaticCatchUp({ quest, outcomes })`, or `domino api POST /quests/automatic-outcomes/preview selection.json`. Review proposed evaluations, rewards, budget capacity and fixed or dynamic costs before confirming. Use `management.quests.catchUpAutomatic({ quest, outcomes, token, actionId, reason })`, or `domino api POST /quests/automatic-outcomes/catch-up confirmation.json`. Both writes require `review`; reuse the action ID for an identical retry. A changed preview returns `catch-up-changed`. Admission does not reserve budgets or guarantee rewards.

Read pending/retrying/failed point checks with `management.quests.automaticScheduling({ member?, cursor? })`, or `domino api GET /quests/automatic-scheduling`. After resolving a failure, reviewers use `management.quests.retryAutomaticScheduling({ id, actionId, reason })`, or `domino api POST /quests/automatic-scheduling/retry retry.json`. The durable point-entry event stays the same.

### Test members

The management client's `testMembers.create/list/history/act/pass/acceptAnyway/advance/acceptReferral/linkWallet/unlinkWallet/linkAccount/unlinkAccount/record/reset` methods, and the matching `domino test-members create|list|history|act|pass|accept-anyway|advance|accept-referral|link-wallet|unlink-wallet|link-account|unlink-account|record|reset` commands, need `publish` permission in the test environment. A Console session, a personal API key, a `domino login` CLI session and an organization automation key can all use them; hosted app keys and service keys cannot, and live refuses them with `test-environment-required`. The CLI commands always address the test environment. Audit records name the person behind the credential, or `automation:<id>` for an automation key. They do not issue participant sessions; only the staging toolbar signs a staging site in as a test member, for the operator whose staging link opened it (see [check participant flows on staging](campaign-testing.md#check-participant-flows-on-staging)). `domino test-members act MEMBER QUEST [--input JSON] [--release RELEASE]` submits as the member: omit `--input` for a claim, or pass `{"kind":"text","value":…}`, `{"kind":"url","value":…}` or `{"kind":"quiz","release":…,"answers":{…}}`. Photo submissions are unavailable to test members. `pass(member, { actionId, quest, completionKey? })` (`domino test-members pass MEMBER QUEST [--completion-key KEY]`) admits an attempt on a manual quest as `act` would, with no input, then accepts it as the operator without running the quest's check, and answers the decided attempt; the action ID replays it. Enrollment, availability, prerequisites, a waiting attempt (`attempt-pending`) and the retry cooldown (`retry-cooldown`) refuse it as they refuse a submission. The completion policy, budgets and rewards apply as for an evaluator's accept: one pass is one accepted action (a progress step, a streak day), a budget that can't pay leaves the attempt `rejected` with `code: "budget-exhausted"`, and `completionKey` keys a keyed quest or is a progress quest's distinct action (other quests refuse it with `validation-failed`); without one the pass uses a fresh key. Automatic quests are refused with `validation-failed`. `acceptAnyway(member, { actionId, attempt })` (`domino test-members accept-anyway MEMBER ATTEMPT`) accepts the member's `rejected` attempt as the operator, such as a tracker-triggered attempt its check rejected: the same attempt, decided again under its pinned release, keeping its rejection in its events; when it is the member's latest attempt on the quest, the retry cooldown ends. Like a staff review it doesn't re-check enrollment, the availability window or an archived quest. Any other status is refused with 409 `attempt-not-rejected` (retry a failed attempt, review one that needs review). Both record a `test-member-decision` event with the operator as actor, listed under the `test-members` audit category, and `history` attempts carry `operatorDecision: { kind: "pass" | "accept-anyway", actor, at }`. `advance(member, { actionId, days })` moves that member forward 1–30 project-calendar days, keeping the time of day; `{ actionId, hours }` (1–720), `{ actionId, minutes }` (1–43200) and `{ actionId, to }` (an ISO 8601 instant with an offset, later than the member's clock, else `test-member-clock-past`) move it by exact time (`domino test-members advance MEMBER --days|--hours|--minutes|--to`). The clock can run at most 90 days ahead of real time, whichever units moved it (`test-member-clock-limit`). Each advance captures a fixed duration shift for all activity and progress until the next advance. A later DST transition can cause one hour of wall-clock drift. Optional `today` reports the server’s current member date as `YYYY-MM-DD`. Evaluator tracker records and summaries filter and report projected activity using the evaluation’s frozen shift. Custom records still send and store real timestamps; Relay projects activity per resolved beneficiary when deciding and crediting a trigger. Content performance freshness uses that member’s clock too. History defaults to the fourteen days ending at the member’s current clock. Deferred evaluation and outbox timers keep running in real time. `acceptReferral(member, { actionId, code })` or `{ actionId, inviter }` (`domino test-members accept-referral MEMBER --code CODE` or `--inviter MEMBER`) accepts a referral invitation as the member, as `POST /referrals/accept` would for a signed-in member, and answers `{ ok: true }`: the program's qualification and rewards apply, and so do `self-referral`, `inviter-already-accepted`, `invitation-closed`, `referral-cycle` and `invitation-not-found`. `inviter` names another test member and uses its invitation code, allocating one if needed. The audit records a `test-member-action` with `operation: "referrals.accept"` and the operator as actor, and reset then refuses both members with `test-member-reset-blocked`. `linkWallet(member, { actionId, chain, address, network? })` gives the member a Solana, EVM or Cosmos address without a signature (`network`, such as `cosmos:cosmoshub-4`, is required for Cosmos); `unlinkWallet(member, { actionId, chain, address })` removes one. Both answer the test member, whose `wallets` lists `{ chain, network, address }` in canonical form. `ctx.wallet.list()` and `ctx.wallet.balance()` read these addresses as they read a member's linked wallets, so balance-gated quests can be tried; tracker records from the address attribute to the member. At most five per chain family (`limit-reached`); an address another member holds is refused with `account-linked-elsewhere`. Relinking a listed Cosmos address with another `network` moves it. The addresses never sign in; unlinking or reset frees them, and the owner signing in with or connecting the wallet in the project takes the address from the test member. `domino test-members link-wallet MEMBER --chain CHAIN --address ADDRESS [--network NETWORK]` and `unlink-wallet MEMBER --chain CHAIN --address ADDRESS` do the same from the CLI. `linkAccount(member, { actionId, provider, id })`, or `{ actionId, provider: "x", username }` resolved through X (`not-found` when no account holds the handle), links a real X or Discord account to the member without OAuth (`domino test-members link-account MEMBER --provider x|discord --id ID` or `--username HANDLE`); `unlinkAccount(member, { actionId, provider, id })` (`unlink-account`) unlinks it. Both answer the test member, whose `accounts` lists `{ provider, id, username, source }` with `source` `linked`, `synthetic` or `connected`. Quest code's `x.connection()`, `profile()`, `follows()`, `reposted()` and Discord roles then read the real account, tracker records it authors attribute to the member, and synthetic records post as it. Linking replaces a synthetic or previously linked account of the provider and is refused with `conflict` while the member holds one it connected itself; an account another member holds is refused with `account-linked-elsewhere`. The account never signs in; unlinking or reset frees it, and its owner signing in with or connecting it in the project and environment takes it from the test member. Member identities mark it `testLinked: true`. Relay never writes to it: a test-linked Discord account never receives role rewards, including on an attempt accepted anyway after the link. Such a reward creates no delivery; the completion stands, and its attempt in `history` carries `rewardsNotSent: [{ reason }]`, so resetting the member still works. `record(member, { actionId, tracker, data?, as? })` (`domino test-members record MEMBER --tracker TRACKER [--data JSON] [--as author|messageAuthor]`) writes one synthetic X post, Discord message or Discord reaction into a managed X or Discord tracker; attribution, `where` filters and quest triggers run as for collected records. `data` takes the stored record's field paths (X: `content`, `author.username`, `isReply`, the counts such as `likeCount`; messages: `channelId`, `replyTo`, `text` only with `discord.messages({ text: true })`; reactions: `channelId`, `messageId`, `emoji`, `reactorRoles`, `messagePublishedAt`); Relay sets IDs and real time. `as: "messageAuthor"` (reactions only) makes the member the reacted-to message's author with a synthetic reactor. The record names the member's real account of the provider when it has one (linked or connected); otherwise the first record per provider links the member a synthetic X or Discord account (a 20-digit ID no real account has), listed in member `identities` and returned by `x.connection()`; X and Discord don't know it. The answer is `{ tracker, record, account: { kind, id }, stored, admissions }`, and record reads mark the record `synthetic: true`. Custom trackers, fields the source doesn't take and windows that exclude now are refused with `validation-failed`; paused and archived trackers with `tracker-paused` and `tracker-archived`. Reset removes the synthetic accounts and records. Test members have no `accounts` except those synthetic ones, the linked ones and the ones connected in the app while a staging site acts as the member, which reset unlinks without touching the accounts themselves; member reads and the `people` or `person` of `members.people(ids)`, attempt and entitlement reads name them with `test: true` and their display name as `name`. See [test members](https://console.domino.run/docs/use/quests#try-quests-with-test-members) for supported submissions and reset restrictions.

### Test member clock

`advance(member, { actionId, days })` moves the member's current time by the requested dates in the project calendar; `hours`, `minutes` and `to` move it by exact time or to a later instant. The server then stores the duration between that new time and real time. Until the next advance, every activity timestamp of that member adds the same duration. `offsetDays` counts the days advanced with `days`; it is not a formula for the current date, so read `today` instead. If daylight saving changes between advances, local wall-clock time can drift by an hour, while timestamp order and elapsed ages stay unchanged.

A `days` advance includes pending timezone changes and preserves the local time where possible. A nonexistent time during the spring DST gap moves forward by the gap; a repeated time during the autumn overlap uses the earlier instant. If a timezone change starts a shortened day after that local time, the clock uses the day's opening boundary.

Claims, campaign and quest enrollment, evaluator time, availability, period prerequisites, streaks, counters and budget periods use the member's clock; `members.get` reports progress and reset boundaries on it.

- **Custom tracker records.** Keep sending real `occurredAt` and `observedAt` timestamps. Relay stores the real timestamps, resolves the beneficiary, and projects the activity for that member's trigger decision and attempt. With `identities: ["member"]`, `{ author: { id: member.id } }` resolves the same way as an explicit `kind: "member"`. An author's offset cannot change another beneficiary's clock. Tracker windows and account-link history still use the real evidence. Keep the ingest action ID on retries to reuse the original outcome.
- **Evaluator tracker reads.** `ctx.trackers.summary()` and `records()` filter `since`, `until` and `asOf` in projected time; returned occurrence and observation times use the evaluation's frozen shift. Attempt identity uses raw observation timestamps, so repeated local times stay distinct. Content performance admission and freshness checks use the same member clock.
- **Operator point mutations.** Omit `effectiveAt` to use the member's current clock, or send a real timestamp that Relay shifts by the offset. Point receipt timestamps stay on real time.
- **Existing work.** Advancing changes no existing attempt or record. An evaluation freezes its execution clock on first execution and reuses it on retries. Deferred evaluations and outbox retries still wait in real time; advancing does not wake them early.
- **Budgets and leaderboards.** Project budgets receive only this member's shifted charges. Leaderboard windows and settlements stay on real time, so future-dated points enter a calendar window when that window arrives. Lifetime accounting still includes those points.

Real members and live activity keep real time. There is no backward adjustment: reset the member and create another to start at zero.

## Project identity and settings

`management.access()` includes each project's `createdBy` user ID when recorded and `creator` details when available. Audit records include `user` details for resolved operators while preserving `actor`. Missing users or unavailable lookups keep the original actor ID.

Organization admins use Console **Settings → General** to rename a project (**Project name**) and change its hosted test address (**Project slug → Change slug**). The management client's `updateProject({ name })` or `updateProject({ slug, confirm })` calls `PATCH /project`; `confirm` is the current project name. A new slug needs 3 to 35 characters and cannot be a name Domino reserves (exact match, such as `console`, `api`, or an active legacy portal): a reserved slug returns 409 `Project slug is reserved`, a shorter one 400. New projects never receive a reserved or shorter slug; the derived slug gets a short suffix instead. These writes require a Console session, like deleting and transferring a project. CLI sessions and API keys cannot make them. Names change without changing the slug. Slug changes affect both environments and existing staging links; update address references and reopen previews. Changes appear in the admin-only **Access & API keys** audit history. See [project settings](https://console.domino.run/docs/use/projects).


## Canonical lifecycle commands

Use the resource's `archive`, `unarchive`, `pause` or `resume` SDK method. The CLI commands use the same verbs: `trackers`, `point-ledgers`, `referral-programs`, `leaderboards`, `engagement`, `collections`, `quest-types` and `images` support archive/unarchive; `quests` and `groups` keep their archive/unarchive commands. Trackers, quests, offers, webhooks and referral-relationships support pause/resume. `redemptions pause|resume` controls all collection. Quests, offers and redemptions require `--revision`; quests, offers, referrals and redemptions require `--reason`.

The older `trackers.control`, `operations.configure`, `offers.control`, `webhooks.disable`, `referrals.suspend`, `resources.lifecycle` and body-ID `/releases/lifecycle` remain supported through 1.x, deprecated for removal in 2.0.0. Use deployments for code publication. Catalog deployment wrappers and engagement-only publish/change/previewChange methods are deprecated. So are `offers.redemptions()`, which answers every redemption at once, and `deployments.list()`, which stops at the newest 100: use `redemptions.history` and `deployments.history`, which page by `cursor`. Console draft publication and quest settings overrides stay separate. The low-level `/releases` and `/releases/batch` routes stay supported without new SDK wrappers; use deployments.

## Operator undo commands

Read the current state before correcting it. Each new mutation accepts an optional action ID, an observed revision or settlement creation time, and an audited reason. Keep the exact action ID and input for an identical retry. Console exposes the same controls; CLI session writes remain Test-only. Renaming, archiving and restoring a stock item are definition changes that live refuses with `promotion-required`; stock corrections work in live.

| Action | SDK and path | Permission |
| --- | --- | --- |
| Read quest enrollment | `quests.enrollments(member)`, `GET /members/:member/quest-enrollments` | `read` |
| Withdraw quest enrollment | `quests.withdraw(quest, member, { expectedRevision, reason })`, `POST /quests/:quest/enrollments/:member/withdraw` | `enroll` |
| Read inventory state | `inventory.items()`, `GET /stock-items/states` | `read` |
| Rename, archive or restore an item | `inventory.rename/archive/unarchive(item, { expectedRevision, reason, name? })`, `POST /stock-items/:item/{rename,archive,unarchive}` | `inventory` |
| Correct free stock | `inventory.adjustStock(item, { expectedRevision, reason, delta })`, `POST /stock-items/:item/adjust-stock` | `inventory` |
| Read settlement decisions | `leaderboards.settlementReviews(board, { cursor, limit })`, `settlementReview(id)`; `GET /leaderboards/:board/settlement-reviews`, `GET /leaderboard-settlements/:id/review` | `read` |
| Export a settlement's payout list | `leaderboards.settlementExport(id)`, `GET /leaderboard-settlements/:id/export`; `settlementPayoutCsv(result)` formats it as CSV; `domino leaderboard-settlements export SETTLEMENT` prints CSV, `--json` the rows | `read` |
| Reject a pending or closing settlement | `leaderboards.rejectSettlement(id, { expectedCreatedAt, reason })`, `POST /leaderboard-settlements/:id/reject` | `review` |
| Measure a settlement's unmeasured content again | `leaderboards.retrySettlementMeasurement(id, { actionId? })`, `POST /leaderboard-settlements/:id/retry-measurement`; refuses a pending settlement with nothing unmeasured (`settlement-measured`) | `review` |
| Read a member's identities | `members.identityStates.list(member)`, `GET /members/:member/identity-states` | `read` |
| Unlink or relink an identity | `members.identityStates.unlink/relink(member, { identity, expectedRevision, reason })`, `POST /members/:member/identity-states/{unlink,relink}` | `review` |
| Turn wallet sign-in on or off | `members.identityStates.setSignIn(member, { identity, enabled, reason })`, `POST /members/:member/identity-states/sign-in` | `review` |

CLI equivalents:

- `domino quests enrollments MEMBER`, `domino quests enroll QUEST MEMBER`, `domino quests withdraw QUEST MEMBER --expected-revision N --reason TEXT`.
- `domino stock-items list`; `rename ITEM NAME`, `archive ITEM`, `unarchive ITEM`, or `adjust-stock ITEM DELTA`, each with `--expected-revision N --reason TEXT`.
- `domino leaderboard-settlements list BOARD --limit 25`, `get SETTLEMENT`, `export SETTLEMENT` (CSV payout list; `--json`), `reject SETTLEMENT --expected-created-at TIME --reason TEXT`.
- `domino members identities MEMBER`, then `domino members unlink-identity MEMBER change.json` or `relink-identity MEMBER change.json`. The file contains `identity`, `expectedRevision` and `reason`. Identity is `{ kind: "wallet", chain, address }`, `{ kind: "oauth", provider: "x" | "discord", issuer, subject }`, `{ kind: "telegram", subject }` or `{ kind: "app", appKind, subject }`, copied from the list. `domino members identity-sign-in MEMBER sign-in.json` takes `identity` (a wallet), `enabled` and `reason`. `members identity-links` and `members relink` are their deprecated wallet and OAuth forms.
- An X, Discord or Telegram account a [member import](imports.md) vouched for lists with `imported: true` until the member links an account of that kind themselves. Unlinking and relinking it work as for a proved account.

Quest withdrawal preserves admitted attempts, completions and rewards. Re-enrolling after withdrawal sets a new consent time; repeating an active enrollment keeps that enrollment's time. This differs from campaign enrollment, whose original time survives withdrawal.

Inventory corrections change free stock and cannot consume reserved or redeemed units. Rename preserves frozen allocation labels. Archive refuses with `inventory-in-use` while any offer or pickup variant references the item, naming them in the refusal. Paused and archived offers still block; archive the offer instead. Reconfigure pickup references away first. Archived items refuse new allocation, restocking and pickup configuration with `inventory-archived`; unarchive remains allowed.

Settlement rejection is final for that frozen window. It keeps winners and exclusions inspectable and grants no prizes. It cannot reject an approved settlement or re-freeze the window. New review reads include rejected settlements; legacy reads keep their original pending/approved contract.

Unlink and relink dispatch by identity kind; ownership never changes. Wallet links belong to the project, preserve their proof and first-link time, and keep sign-in unchanged. X, Discord and Telegram selections belong to the project and environment too; relink only an exact retained owned account, after unlinking any current selection of that provider. Account relink starts a fresh link time and preserves sign-in grants and existing sessions. App identities sign in only while linked: unlinking one ends the sessions its integration started for it. Every relink redecides `unlinked-account` history using the current quest policy: automatic can admit it; claim or staff leaves historical activity assigned to the member. Admitted attempts remain pinned. Wallet sign-in has its own command and never follows a link change. A stale `expectedRevision`, or unlinking an unlinked identity, is refused with 409 `account-changed`; an identity the member never proved is 404.

Inventory states, quest enrollments and retained identity choices are cursor pages: read `items` and pass `nextCursor` as `--cursor` until it is null. Each accepts `--limit` (1–100, default 50). Console's **Inventory items** includes stock owned by offers; **Load more inventory items**, **Load more enrollments**, **Load more owned identities** and **Load more settlement windows** reach later pages.

## Scheduled hosted jobs

Declare `app.schedules` beside secrets and storage in a Worker app's relay.json. Stage the app to activate signed POST jobs in test, and verify them with `verifyScheduledRequest(request, env.DOMINO_SCHEDULE_KEY)` before doing work. Use a scheduled route to refresh custom trackers through `hostedManagementClient(env).trackers.ingest`, with the verified run ID as the ingestion action ID, in one batch per tracker. Answer with `scheduleResult({ outcome, summary, metrics, diagnostics })` so `client.schedules.runs(name)` and `domino schedules runs NAME` show what each run did. See [hosted schedules](hosted.md#scheduled-jobs) for limits, results, visibility and uncertain-run recovery.

### Schedule dispatch guarantees

People's guide: [Timeouts and recovery](https://console.domino.run/docs/build/schedules#timeouts-and-recovery).

- **At most once.** Relay attempts each automatic tick at most once. It commits the tick's receipt before sending and never retries automatically. Missed ticks during downtime are skipped; dispatch resumes with one due run and computes the next future occurrence.
- **Failures permit the next tick.** A failed app HTTP response, or a result with a `partial` or `failed` outcome, records a failure and permits the next occurrence. An explicit bridge rejection before delivery, such as a credential mismatch, also fails that tick and permits future occurrences. Other bridge failures remain uncertain.
- **Serial calls.** Calls from one project run serially, so the same schedule cannot overlap itself. Activation waits for an in-flight dispatch before replacing the build. A replacement build resets the next occurrence from its activation time.
- **Uncertain runs block.** If delivery or completion is uncertain, Relay blocks that schedule, including across build replacement. A dispatcher crash after admission also leaves it blocked. After the 30-second timeout, Console shows **Uncertain** and offers **Recover uncertain run**; `domino schedules recover NAME --yes` and `management.schedules.recover(name, { stopped: true })` do the same. Recovery allows future ticks and never replays the uncertain tick. If a dispatcher disappeared, the run's duration is unknown.

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
