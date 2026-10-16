# Quest rules and evaluation

Read this when you shape how a quest decides and repeats. It covers campaigns, availability windows and prerequisites, automatic and repeatable quests and republishing them, prerequisite automatic outcomes and test members, reject codes, deferred evaluation and progress reports, distinct-action progress counters, retry and completion cooldowns, evaluation time and randomness, calendar-day streaks, and testing quests locally. [Verify member actions](verification.md) covers the readers `evaluate` checks a member with, and [Authoring](authoring.md) covers the rest of a quest.

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

`automatic()` without arguments runs once a member completes its prerequisites: it needs at least one `requires` entry and completes `once()`, and TypeScript rejects a missing or empty prerequisite list or a different completion policy. Untyped callers receive the same validation before publishing. A reusable `defineQuestType` may leave prerequisites for its Console instances; every published instance still needs at least one. For activity Domino collects or your backend reports, use `automatic({ tracker })`; see [tracker triggers](trackers.md#tracker-triggers). Manual and automatic triggers are exclusive. Automatic quests never expose participant claim or retry actions. Set `visibility: "hidden"` to omit a quest from participant lists while it continues earning rewards. Set `profile: { visible: true }` to list its completions on participant profiles (`relay.profiles.completions`); they stay off by default because a completion can reveal holdings or activity. The active release decides, and archived quests never appear. A `visibility: "hidden"` quest with `profile: { visible: true }` exposes its title and completions on profiles.

Choose `once()` for one successful completion per participant, or `keyed(ctx => key)` for one per author-defined key. Write it as a function of the builders, `completion: ({ keyed }) => keyed(({ tracker, settings }) => tracker.record.id)`, to type the key's context from the quest: `tracker` is non-null and typed for a tracker trigger, and `settings` has the quest's settings. An inline `completion: keyed(...)` still works, with a nullable, untyped `tracker` and no `settings`. The builders also include `once`, `streak` and `progress`. The key is scoped to the project, quest and participant, or with `keyed(fn, { across: "members" })` to the [project and quest](https://console.domino.run/docs/concepts/quests#keys-shared-by-members) (`key-used`), and must stay stable across deployments. For announcement reactions use the message ID; for a daily quest use `ctx.calendar.day().key`, the project-calendar key for `ctx.eligibleAt`. `completed(quest)` means ever completed unless it has an explicit period scope.

Only `keyed(...)` uses an evaluator's key: `decision.accept({ completionKey })` or `decision.review({ completionKey })` takes precedence over its key callback. The other policies ignore it: `once()` and `streak(...)` key their own completions, `progress(...)` keys each action with `distinct(...)`, and a content quest's key is its content item. `runQuest` reports such a key in `run.diagnostics` (code `completion-key-ignored`); the run is otherwise unchanged.

`ctx.eligibleAt` is fixed when admitted: the record's `occurredAt` for a tracker trigger, the admission time otherwise. A tracker record's ID and `observedAt` deduplicate delivery; they are not the completion key. Only an accepted completion consumes a key. A rejected attempt stays rejected, but a later record can try an unconsumed key.

Declare budgets with `budget({ key, scope: "member" | "project" | "author", period: "day" | "week" | "month" | "lifetime", limit, cost })`. `"author"` counts per tracker record author's account (`kind:subject`), linked or not. Use `cost: 1` for completion counts or `cost: { ledger: community }` for the awarded points amount. Quests can share a named budget, with matching scope, period, limit, and cost: they share its usage, so declare it once and import it in each quest. `domino check`, `domino build` and `domino deploy` refuse quests that declare one budget key differently, naming both quests and the fields that differ; `check` reports it as `BUDGET_DECLARATION_CONFLICT`. The check reads literal values, so a budget built from a setting or computed value is compared at deploy, where Relay refuses the same conflict. All budget spends, the completion key, and rewards commit together. Day, week and month budgets use the [project calendar](authoring.md#project-calendar) period containing the eligibility time; weeks are ISO weeks starting Monday (a bucket such as `2026-W40@Europe/Budapest`). An attempt that finds its budget exhausted is rejected permanently; it does not roll into tomorrow. Concurrent valid awards commit in whichever order wins the transaction.

`"author"` needs a tracker trigger (`AUTHOR_BUDGET_TRACKER_REQUIRED`) and caps what one giver gives, such as a reactor tipping a message's author. Descriptions list it in `authorBudgets`, not `budgets`; member budget views omit it; a record with no author is refused; imports don't count. A tip capped both ways, on `automatic({ tracker: reactions, beneficiary: "messageAuthor" })`:

```text
budgets: [
  budget({ key: "tips-received", scope: "member", period: "day", limit: 20, cost: 1 }),
  budget({ key: "tips-given", scope: "author", period: "day", limit: 10, cost: 1 }),
]
```

See [cap what each giver gives](https://console.domino.run/docs/build/trackers#cap-what-each-giver-gives).

Admission pins the release and prerequisite result. Pausing stops new admissions while accepted work finishes. Republication does not clear successful completion keys. Execution errors of automatic attempts retry with capped backoff and retain the original evidence and release. Points accumulate per completion. Tier rewards retain their existing entitlement upgrade behavior; repeating a quest does not create a new consumable handover.

Install `discord-announcements` for the first integration example. Its quest triggers from a `discord.reactions()` tracker and checks the configured channel, the emoji and `data.messagePublishedAt` in quest code. The gateway records the reaction addition; do not query current reactions to cancel it. Unseen replayed events with unknown timing are skipped and reported as coverage gaps. Manual Discord membership remains `ctx.discord.member()`. Use `discord.grantRole(roleId)` for a declared role reward; no arbitrary Discord writes are exposed. See [Discord trackers](trackers.md#discord-trackers), and X verification below for X reads.

A trusted customer backend reports activity as records of a custom tracker: declare `customSource(namespace, { identities, data })` and send records with `management.trackers.ingest(tracker, { actionId, records })` using a key with `ingest` permission. Name the member through the account the activity came from (`wallet`, `x` or `discord`), or your app's own user ID as an `app:<name>` identity linked by your [app login](https://console.domino.run/docs/build/app-login) integration, so Relay attributes it through the member's verified link, or with a `member` identity when the backend vouches for the member ID. Tell kinds of activity apart with a `data` field that the trigger's `where` selects, or with separate trackers. Ingestion never joins the member to the project: a trigger skips an unjoined member with `not-joined`. Use stable record IDs on retries and server-side action times, never browser claims. Matching quests can grant real rewards from these records; keep the key on the server.

## Republishing and automatic quests

Publishing changed rules never rewrites work already admitted:

- **Admitted attempts keep their release.** Tracker-triggered attempts keep the release selected at admission, including its reward amount and budget rules.
- **Nothing is reset.** Publishing changed rules does not reset completion keys or reconsider skipped records. Rejected attempts stay final except when a prerequisite-only automatic attempt lost a claim-time budget race; that attempt can retry under the current release after another member event or reviewed operator recovery (see [prerequisite automatic outcomes](#prerequisite-automatic-outcomes)).
- **Pausing stops new admissions.** Already admitted work can finish.

## Prerequisite automatic outcomes

An `automatic()` quest with prerequisites and no tracker records each eligibility transition as admitted or skipped. Completion, group inclusion, campaign enrollment, publication and freed occurrences check current eligibility; every committed point entry schedules an asynchronous current-balance check, including refunds and referral or leaderboard awards. Multiple events reuse a pending once occurrence's attempt. An attempt rejected with `budget-exhausted` leaves the occurrence available: the next member event checks current rules and creates one deterministic retry attempt. Evaluator, staff and revoked-occurrence rejections stay final. A missing prerequisite can qualify on a later transition; a paused event requires explicit operator recovery.

Read `management.quests.automaticOutcomes({ quest, member?, state?, code?, cursor? })`, or `domino api GET '/quests/automatic-outcomes?quest=QUEST'`. Admitted outcomes can include `linkedAttempt` with its ID, status and open code. Select up to 50 `quest-paused` IDs or admitted outcomes whose linked attempt has status `rejected` and code `budget-exhausted` or status `failed` and code `integration-required` (`isRetryableAutomaticOutcome` from `@domino-sdk/relay/schema` tells you), then call `management.quests.previewAutomaticCatchUp({ quest, outcomes })`, or `domino api POST /quests/automatic-outcomes/preview selection.json`. Review proposed evaluations, rewards, budget capacity and fixed or dynamic costs before confirming. The preview's `rewardEntries` lists the same rewards as `rewards`, as `{ type, data }`, including kinds the SDK does not know. Use `management.quests.catchUpAutomatic({ quest, outcomes, token, actionId, reason })`, or `domino api POST /quests/automatic-outcomes/catch-up confirmation.json`. Both writes require `review`; reuse the action ID for an identical retry. A changed preview returns `catch-up-changed`. Admission does not reserve budgets or guarantee rewards.

Read pending/retrying/failed point checks with `management.quests.automaticScheduling({ member?, cursor? })`, or `domino api GET /quests/automatic-scheduling`. After resolving a failure, reviewers use `management.quests.retryAutomaticScheduling({ id, actionId, reason })`, or `domino api POST /quests/automatic-scheduling/retry retry.json`. The durable point-entry event stays the same.

### Test members

The management client's `testMembers.create/list/history/act/pass/acceptAnyway/advance/acceptReferral/linkWallet/unlinkWallet/linkAccount/unlinkAccount/record/reset` methods, and the matching `domino test-members create|list|history|act|pass|accept-anyway|advance|accept-referral|link-wallet|unlink-wallet|link-account|unlink-account|record|reset` commands, need `publish` permission in the test environment. A Console session, a personal API key, a `domino login` CLI session and an organization automation key can all use them; hosted app keys and service keys cannot, and live refuses them with `test-environment-required`. The CLI commands always address the test environment. Audit records name the person behind the credential, or `automation:<id>` for an automation key. They do not issue participant sessions; only the staging toolbar signs a staging site in as a test member, for the operator whose staging link opened it (see [check participant flows on staging](campaign-testing.md#check-participant-flows-on-staging)). `domino test-members act MEMBER QUEST [--input JSON] [--release RELEASE]` submits as the member: omit `--input` for a claim, or pass `{"kind":"text","value":…}`, `{"kind":"url","value":…}` or `{"kind":"quiz","release":…,"answers":{…}}`. `--photo FILE` (`uploadEvidence`) sends a photo. `pass(member, { actionId, quest, completionKey? })` (`domino test-members pass MEMBER QUEST [--completion-key KEY]`) admits an attempt on a manual quest as `act` would, with no input, then accepts it as the operator without running the quest's check, and answers the decided attempt; the action ID replays it. Enrollment, availability, prerequisites, a waiting attempt (`attempt-pending`) and the retry cooldown (`retry-cooldown`) refuse it as they refuse a submission. The completion policy, budgets and rewards apply as for an evaluator's accept: one pass is one accepted action (a progress step, a streak day), a budget that can't pay leaves the attempt `rejected` with `code: "budget-exhausted"`, and `completionKey` keys a keyed quest or is a counter quest's distinct action (other quests refuse it with `validation-failed`); without one the pass uses a fresh key. Automatic quests are refused with `validation-failed`. `acceptAnyway(member, { actionId, attempt })` (`domino test-members accept-anyway MEMBER ATTEMPT`) accepts the member's `rejected` attempt as the operator, such as a tracker-triggered attempt its check rejected: the same attempt, decided again under its pinned release, keeping its rejection in its events; when it is the member's latest attempt on the quest, the retry cooldown ends. Like a staff review it doesn't re-check enrollment, the availability window or an archived quest. Any other status is refused with 409 `attempt-not-rejected` (retry a failed attempt, review one that needs review). Both record a `test-member-decision` event with the operator as actor, listed under the `test-members` audit category, and `history` attempts carry `operatorDecision: { kind: "pass" | "accept-anyway", actor, at }`. `advance(member, { actionId, days })` moves that member forward 1–30 project-calendar days, keeping the time of day; `{ actionId, hours }` (1–720), `{ actionId, minutes }` (1–43200) and `{ actionId, to }` (epoch milliseconds or an ISO 8601 instant with an offset, later than the member's clock, else `test-member-clock-past`) move it by exact time (`domino test-members advance MEMBER --days|--hours|--minutes|--to`). The clock can run at most 90 days ahead of real time, whichever units moved it (`test-member-clock-limit`). Each advance captures a fixed duration shift for all activity and progress until the next advance. A later DST transition can cause one hour of wall-clock drift. Optional `today` reports the server’s current member date as `YYYY-MM-DD`. Evaluator tracker records and summaries filter and report projected activity using the evaluation’s frozen shift. Custom records still send and store real timestamps; Relay projects activity per resolved beneficiary when deciding and crediting a trigger. Content performance freshness uses that member’s clock too. History defaults to the fourteen days ending at the member’s current clock. Deferred evaluation and outbox timers keep running in real time. `acceptReferral(member, { actionId, code })` or `{ actionId, inviter }` (`domino test-members accept-referral MEMBER --code CODE` or `--inviter MEMBER`) accepts a referral invitation as the member, as `POST /referrals/accept` would for a signed-in member, and answers `{ ok: true }`: the program's qualification and rewards apply, and so do `self-referral`, `inviter-already-accepted`, `invitation-closed`, `referral-cycle` and `invitation-not-found`. `inviter` names another test member and uses its invitation code, allocating one if needed. The audit records a `test-member-action` with `operation: "referrals.accept"` and the operator as actor, and reset then refuses both members with `test-member-reset-blocked`. `linkWallet(member, { actionId, chain, address, network? })` gives the member a Solana, EVM or Cosmos address without a signature (`network`, such as `cosmos:cosmoshub-4`, is required for Cosmos); `unlinkWallet(member, { actionId, chain, address })` removes one. Both answer the test member, whose `wallets` lists `{ chain, network, address }` in canonical form. `ctx.wallet.list()` and `ctx.wallet.balance()` read these addresses as they read a member's linked wallets, so balance-gated quests can be tried; tracker records from the address attribute to the member. At most five per chain family (`limit-reached`); an address another member holds is refused with `account-linked-elsewhere`. Relinking a listed Cosmos address with another `network` moves it. The addresses never sign in; unlinking or reset frees them, and the owner signing in with or connecting the wallet in the project takes the address from the test member. `domino test-members link-wallet MEMBER --chain CHAIN --address ADDRESS [--network NETWORK]` and `unlink-wallet MEMBER --chain CHAIN --address ADDRESS` do the same from the CLI. `linkAccount(member, { actionId, kind, subject })`, with `kind` `x` or `discord` and the numeric user ID as a member import names it, or `{ actionId, provider: "x", username }` resolved through X (`not-found` when no account holds the handle), links a real X or Discord account to the member without OAuth (`domino test-members link-account MEMBER --kind x|discord --subject ID` or `--username HANDLE`); `unlinkAccount(member, { actionId, kind, subject })` (`unlink-account`) unlinks it. `{ provider, id }` and `--provider`/`--id` are the deprecated forms, removed in 2.0.0. Both answer the test member, whose `accounts` lists `{ provider, id, username, source }` with `source` `linked`, `synthetic` or `connected`. Quest code's `x.connection()`, `profile()`, `follows()`, `reposted()` and Discord roles then read the real account, tracker records it authors attribute to the member, and synthetic records post as it. Linking replaces a synthetic or previously linked account of the provider and is refused with `conflict` while the member holds one it connected itself; an account another member holds is refused with `account-linked-elsewhere`. The account never signs in; unlinking or reset frees it, and its owner signing in with or connecting it in the project and environment takes it from the test member. Member identities mark it `testLinked: true` and `source: "test-link"`. Relay never writes to it: a test-linked Discord account never receives role rewards, including on an attempt accepted anyway after the link. Such a reward creates no delivery; the completion stands, and its attempt in `history` carries `rewardsNotSent: [{ reason }]`, so resetting the member still works. `record(member, { actionId, tracker, data?, as? })` (`domino test-members record MEMBER --tracker TRACKER [--data JSON] [--as author|messageAuthor]`) writes one synthetic X post, Discord message or Discord reaction into a managed X or Discord tracker; attribution, `where` filters and quest triggers run as for collected records. `data` takes the stored record's field paths (X: `content`, `author.username`, `isReply`, the counts such as `likeCount`; messages: `channelId`, `replyTo`, `text` only with `discord.messages({ text: true })`; reactions: `channelId`, `messageId`, `emoji`, `reactorRoles`, `messagePublishedAt`); Relay sets IDs and real time. `as: "messageAuthor"` (reactions only) makes the member the reacted-to message's author with a synthetic reactor. The record names the member's real account of the provider when it has one (linked or connected); otherwise the first record per provider links the member a synthetic X or Discord account (a 20-digit ID no real account has), listed in member `identities` and returned by `x.connection()`; X and Discord don't know it. The answer is `{ tracker, record, account: { kind, id }, stored, admissions }`, and record reads mark the record `synthetic: true`. Custom trackers, fields the source doesn't take and windows that exclude now are refused with `validation-failed`; paused and archived trackers with `tracker-paused` and `tracker-archived`. Reset removes the synthetic accounts and records. Test members have no `accounts` except those synthetic ones, the linked ones and the ones connected in the app while a staging site acts as the member, which reset unlinks without touching the accounts themselves; member reads and the `people` or `person` of `members.people(ids)`, attempt and entitlement reads name them with `test: true` and their display name as `name`. See [test members](https://console.domino.run/docs/use/quests#try-quests-with-test-members) for supported submissions and reset restrictions.

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
| Reserved | A reject cannot use `budget-exhausted`, `defer-expired`, `defer-limit-exceeded`, `evaluator-deferred`, `evaluator-progress`, `imported`, `integration-required`, `not-linked`, `provider-unavailable`, `author-error`, `evaluator-error` or `capability-limit` (`reservedAttemptCodes`). The list never grows. A code Relay added later, such as `key-used`, is also yours to use: the attempt's `codeSource` (`relay` or `evaluator`) tells who wrote it. |
| Invalid code | The evaluation throws an authoring error naming the rule, such as `Invalid reject code "Wrong answers": use lowercase letters, digits and hyphens, at most 64 characters`, and fails like any uncaught error: the attempt fails once with code `evaluator-error`, and `runQuest` rejects with an `EvaluatorError` carrying the same message. |
| Where it shows | The attempt's `code` in `management.attempts.get`/`list`, `domino attempts get`/`list`, and Console's attempt detail beside the reason. |
| Members | Never. Participant responses (quest progress, the quest experience and its collections, submission replays) give a rejected attempt a code only when Domino rejected it itself (`codeSource: "relay"`: `budget-exhausted`, `key-used`, `defer-expired`, `defer-limit-exceeded`), else `code: null`. Attempts in other states keep their codes, such as a failed attempt's `integration-required`. |

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

## Distinct-action progress counters

Author `progress({ distinct, goal: count(3), window: calendar("week"), repeat: "once" })` as a quest completion policy. Omit the window for lifetime progress; use `rolling({ days: 7 })` for a trailing elapsed duration. These are native authoring builders; `distinct` reads trusted `KeyedContext`, including `calendar` and the pinned tracker observation; in the builder form, `completion: ({ progress }) => progress({ distinct: ({ tracker }) => … })`, it is typed from the quest and has its `settings`. Do not invent `ctx.progress.increment`.

Counters work on manual and tracker-triggered quests, not content or prerequisite-only automatic quests. Default settlement is once per calendar bucket, or once over the quest lifetime for lifetime/rolling windows. `repeat: "repeat"` consumes disjoint action batches. Calendar lateness defaults to seven days after bucket end, configurable with `latenessMs` from zero through 90 days. Expired rolling evidence is retained and cannot be reused. Budget-denied achievements are final, retain their exact evidence and budget ID, and do not qualify referrals.

Read `management.members.counters.history(member, quest, { cursor, limit })`. With review permission, use `management.members.counters.moderate(member, quest, contribution, { actionId, excluded, reason })` to exclude or restore unsettled qualifying evidence. CLI: `domino api GET /members/MEMBER/counters/QUEST` and `domino api POST /members/MEMBER/counters/QUEST/CONTRIBUTION/moderate correction.json`. Reuse the action ID for an identical retry. Settled evidence cannot be edited; revoke an awarded completion to correct its reward, without reusing its actions.

Compose qualified referrals with `questCompleted(qualificationQuest)`. Install the `qualified-referrals` snippet (see [Find a snippet](authoring.md#find-a-snippet)): three verified tracker records complete a lifetime qualification quest. Its alternative continuous-membership quest checks Discord's current `joinedAt` for a seven-day interval and rejects early claims with a not-yet-eligible reason, allowing the member to submit again after the interval. Rejoining resets that interval. It does not promise continuous role possession or automatic polling beyond the platform's defer horizon.

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

For activity days, trigger from a tracker. `packages/sdk/examples/posting-streak-quest.ts` pays an approved creator who posts about the project on X on five consecutive days: an X tracker collects the posts, `automatic({ tracker })` runs once per new post for its author, `requires: [inGroup(approvedCreators)]` limits it to an operator-managed group (see [rewarding X posts](trackers.md#rewarding-x-posts)), and each post marks the day it was published. A "hold N tokens for 7 days" streak uses a custom wallet tracker with the default `on: "new-observation"` and `where: { path: "amount", gte: { setting: "threshold" } }`, fed at least one sample per wallet per day; `"data-changed"` would miss a day whose balance did not change. Each sample must carry its own `occurredAt`, the time it was sampled; a sample that reuses the record's original time lands on that one day. Each record observation is one day candidate, and repeated observations on one day occupy it once. Once a member's milestone pays, the trigger stops admitting their observations for that quest, as for a completed `once()` quest. The ingesting server sets a custom record's `occurredAt`, so an `ingest` key holder can backdate records to fill days, back to the quest's publication and within the tracker window; keep those keys on trusted servers. Never use browser-supplied action times as evidence. Discord message quests use the same policy on a `discord.messages()` tracker, with the contribution snippet's channel check.

Days are those of the [project calendar](authoring.md#project-calendar), not each participant's device timezone; `streak()` takes no `timeZone`. Days end at midnight in the project timezone and can last 23 or 25 hours at daylight-saving transitions. Multiple qualifying actions occupy one day. Today remains open: yesterday's run survives until today ends without qualifying evidence.

A project timezone change starts one new calendar day at the next midnight in the current zone, ending at the next midnight in the new zone; its date label may repeat or skip, but its day identity remains consecutive. Historical days retain their original zone and identity. Goal length and trigger kind (manual or tracker) cannot change on an existing streak quest; create a new quest for a new milestone or trigger. Presentation edits preserve progress. Qualification edits affect newly admitted evidence; already admitted attempts retain their release.

Eligible delayed tracker records count at their occurrence time, even when processed out of order. They can repair missing days and complete a milestone. Existing activation, identity, pause and prerequisite admission rules still apply; there is no historical Discord backfill or synthetic credit for capture outages. Removing a message or reaction does not erase a qualifying day. No grace days or streak freezes are implicit.

Qualifying days, completion, points and role-delivery intent commit atomically. Concurrent actions and retries cannot repeat a payout. A budget-exhausted milestone keeps its days and is permanently marked `budget-exhausted`; later retries or budget edits do not pay it. Budgets bucket by the project calendar and use the recorded milestone day action time. Discord role delivery still retries independently after completion.

Read `quest.streak` through `relay.me.progress()` or the quest experience. It includes `current`, `best`, `goal`, `todayQualified`, the current `timeZone`, `nextBoundary`, a pending timezone transition, qualifying-day history and milestone reward status. An accepted action below the goal has attempt status `progressed`, not `accepted`, and no completion. Upgrade the SDK and participant app before publishing streak quests so they understand this new completion policy and attempt status.

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
- **Scope.** One attempt's evaluation, within Relay's 15-second limit: prerequisites, availability, budgets, completion limits and reward delivery are not run, and nor are Relay's chain and token checks. Module-scope quest code runs at import with the real clock. `createTestContext(quest, options)` returns the host context for calling `quest.evaluate` directly, without the frozen clock or reward check. `fakeEvaluationMember(methods)` builds a `ctx.member` for a context a test assembles itself: the methods left out answer for a member with no accounts, completions, groups or enrollments, and `balance` and `referrals` fail with `author-error`. Never write an `EvaluationMember` by hand: a minor release can add methods.
- **Replay.** `replayAttempt(quest, capture, { facts? })` evaluates current quest code against one evaluation run of a real test-environment attempt: the release's settings and rewards, the attempt's input, clock, seed and calendar, and each captured capability answer by position. It resolves with `{ decision, recorded, changed, divergence, calls }`; a call the capture cannot answer (`new`, `changed` or `uncaptured`) stops the replay with `decision: null` unless `facts` answers it. Get a capture with `domino quests replay <attempt> --out <file> --json` (which also replays it against the discovered quest with that key, or `--quest <file>`) or `management.attempts.capture(attempt)` (`publish` permission: captures hold raw provider answers). `recorded` is what quest code decided in the run, before Relay amended, claimed or reviewed it. Captures are test-environment only: live is refused with 422 `test-environment-required`, and an unevaluated attempt with 409 `attempt-not-evaluated`.
- **Tests only.** `buildQuest`, and so `domino check` and deploy, refuse a quest bundle that imports `@domino-sdk/relay/testing`.
