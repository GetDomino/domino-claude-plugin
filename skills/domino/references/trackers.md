# Trackers reference

The contract behind [Track posts and activity](https://console.domino.run/docs/build/trackers): declaring trackers of each source, collection options, record reads and attribution, the rules a tracker trigger applies, its outcome codes, and engagement accounting. The guide has the tasks and examples; this page has the detail they rely on.

## Declare trackers

### Maintained tracker datasets

Declare `defineTracker({ key, title, source: x.search({ query }), window: { start, end? } })` from `@domino-sdk/relay/authoring` and add `trackers: [{ entry: "trackers/launch.ts" }]` to `relay.json`. `domino deploy` previews and publishes tracker definitions together with quests and ledgers. `source: discord.messages()` or `discord.reactions()` declares a [Discord tracker](#discord-trackers) instead. The X collector retains all returned matches and author verification. Apply `verified`, `member`, `author`, date, and engagement filters when reading, not during ingestion.

A tracker of a managed source can also be data instead of code, for operators who add campaigns without a deployment. `management.trackers.create({ key, title, source, window: { start, end? }, refreshSeconds?, maxPages? })` (`domino trackers create KEY --title T --source KIND --start MS`, or **New tracker** in Console **Trackers**) creates one in test with `publish` permission and the same validation as `defineTracker`. `source` is `{ kind: "x-search", query }`, `{ kind: "discord-messages", text? }`, `{ kind: "discord-reactions" }`, `{ kind: "telegram-messages" }`, `{ kind: "telegram-reactions" }`, `{ kind: "youtube-shorts" }`, `{ kind: "tiktok-videos" }` or `{ kind: "wallet-balances", chain, token }` (`chain` a CAIP-2 ID, `token` null for a native coin; `domino trackers create KEY --title T --source wallet-balances --chain C [--token T] --start MS`), stored as the source its builder makes; `query` alone, without `source`, is short for an X search. Custom sources are code only. An X tracker's first collection searches from `window.start`. `management.trackers.update(tracker, changes)` (`domino trackers update TRACKER [--query Q] [--text on|off] ...`, or **Edit** in Console) changes any of those fields but the key and the source kind (422 `validation-failed`); fields left out keep their value, a new query or window collects from now on, never searching the past, and turning Discord message text off erases the text already stored. Live refuses both with 422 `promotion-required`: the tracker reaches live, under the same key and still owned by data, through a promotion from test (`domino promote`). A key belongs either to code or to data, shown as `origin` (`code` or `data`) on tracker status: a deployment that declares a data tracker's key is refused with 409 `tracker-owned-by-data`, and creating or updating a key a deployment publishes with 409 `tracker-owned-by-code`. Deployments never change or remove data trackers. Pause, resume and refresh work on both kinds, in test and live; archive is a definition change, made in test.

X tracking is best effort and bounded per project. Each cycle searches the tracker's window; known posts the search didn't return are read again every 6 hours for 48 hours after they were posted, daily until day 7, then never. A tracker runs no more often than its project's plan allows: 6 hours on every plan but Enterprise. A shorter `refreshSeconds` still publishes, runs at that minimum (`effectiveRefreshSeconds` on tracker status), and a deployment warns `tracker-refresh-below-plan-minimum`. Tracker searches and refreshes, quest X checks, tracker previews and handle lookups count toward the project's monthly X usage. While plan limits are enforced, a live project that spends its plan's X budget (Launch 25,000, Growth 65,000, Scale 160,000 requests a month) has its live X trackers paused with `pauseReason: "x-budget-exhausted"` until `resumesAt`, the next UTC month, or a plan whose budget isn't spent; quest checks keep working. An organization's test X trackers share 5,000 requests a month and pause the same way when it is spent. A test X tracker collects for 14 days from its first sync, then pauses with `pauseReason: "test-window-ended"`; `trackers.resume` starts another 14 days. Resuming a budget-paused tracker is refused with 409 `x-budget-exhausted` while the budget is spent; pausing it yourself makes the pause `operator`, which only a resume ends. A resumed tracker searches its whole window again.

Management clients expose `trackers.create`, `update`, `list`, `get`, `records`, `content`, `summary`, `pause`, `resume`, `refresh`, `ingest`, `triggerOutcomes`, `retryOutcome`, and `preview` (`control` and `outcomes` are deprecated). `trackers.preview({ query, since })` returns a bounded X search sample (at most 100 posts from 3 pages, `since` up to 30 days back, `more` when X had further matches) with `publish` permission. It stores nothing and never creates or changes a tracker. Console offers the same preview and copies a definition or agent prompt from it. Hosted quest evaluators expose read-only `trackers.records(reference, query)` and `trackers.summary(reference, query)`, with results captured for replay; they read the tracker as of `query.asOf` (default `ctx.eligibleAt`) at the evaluation run's pinned watermark ([evaluator readers](verification.md#evaluator-readers-null-errors-and-pinning)). `member` filters the currently connected X identity, on a Discord tracker the records whose author is their selected Discord account, or on a custom tracker with `identities` the records whose author is one of the member's verified identities of those kinds (every wallet they currently link, their selected X or Discord account, their member ID). An unconnected member matches nothing; custom trackers without identities refuse it. Different quests can apply independent qualification and review policies to the same dataset.

Overlapping X trackers in the same project and environment share current post facts. Each tracker retains its own matches and first-seen times; another tracker's discovery never adds a match to yours. Known-post refreshes coordinate across collectors. Pausing a tracker stops its collection, but another active tracker can still update their shared posts. Captured quest reads keep their original results. Custom records remain isolated per tracker, including when namespaces match.

Pass `includeEvidence: true` to `trackers.records` for X `evidence: { contentRevision, observation }`. A content revision identifies the stable body; counters, author profile/verification, removal status, and sample time affect only the observation. Compare within the same post, project, and environment. Identical samples across trackers share references; reverting a body reuses its revision. Default reads omit the field; opted-in custom reads return `null`. Captured quest reads pin the references with their results. Use `trackers.content` below to read a retained observation. Use the content trigger below for automatic admission.

Hosted evaluators can call `trackers.content(reference, { id, observation })` using a record's `evidence.observation`. It returns the immutable X body, observation time, captured removal flag, and account attribution to the attempt's member. Evaluators cannot override that member. Check `attribution.kind`: `linked`, `unlinked`, or `different-author`. Matching uses the selected verified X account's stable ID, never a username or verification badge. The linked result records the account issuer, subject, revision, and known link time; legacy unknown link times are `null`. Metrics and mutable author profiles are excluded. Management clients use the same method with an additional required `member` and `read` permission. The post must belong to the tracker in the same project/environment. Attribution uses the evaluation run's pinned X account. In an evaluator a missing record or observation returns `null`. Captured quest results survive edits and disconnects for replay; they do not prove enrollment, historical eligibility, or current post availability and do not automatically admit or reward content.

Custom datasets use `customSource(namespace, { data })` and management ingestion with `ingest` permission. `data` is required: a zod object schema (`z` is exported by `@domino-sdk/relay/authoring`) for each record's `data`. Relay stores its JSON Schema form with the tracker and rejects the whole ingest batch with 422 `{ error, code: "invalid-record-data", record, path, reason }` when a record does not match; evaluators read `ctx.tracker.record.data` and `trackers.records(tracker)` with the schema's input type. `customSource` throws for what ingestion cannot check: string formats and patterns (`z.email()`, `.regex()`), `.multipleOf()`, tuples, `.refine()`, `.transform()`, `z.never()`, `z.json()` (use `z.unknown()`), dates, bigints, `.meta()` keys other than title, description, default and examples, a `__proto__` property, and schemas reused by ID; check those in the evaluator. Limits: 32 KB and 16 levels per schema, 256 properties, `required` keys, enum values or key names per node, 16 branches per union, and inside a union's branch a nested union may have only one branch with properties, items or unions. Validation has a step budget per record and per batch that charges every key, value, literal property and enum lookup (admission and `trackers.records(handle)` use it too); a record that spends it fails the batch with 422 `record-data-too-complex`. Records with `removed: true` are not checked, and a refused batch records nothing, so its action ID can carry the corrected batch. An ingest batch is at most 64 MB and 2,000,000 JSON values; a larger one is refused with 413 `payload-too-large` ([large request bodies](errors.md#codes-by-status)). The schema can change under the same tracker key and applies to later records only; a deployment whose new schema breaks a live quest's `where` (Console quests included) is refused with 422. Stored records are not rechecked: a tracker trigger skips one that no longer matches with `invalid-record-data`, an evaluator rejects a pinned record that does not match the schema the quest was built with, and `trackers.records(handle)` drops such records from `items` and counts them in `excluded` (string-key reads stay untyped and unfiltered). A tracker published before record data takes no records (422 `tracker-schema-required`) until it is published with `data`. For `ingest` permission, a hosted app's own key (`hostedManagementClient`) has it, and an indexer's personal API key can hold only `ingest` (Console **Settings → API keys**; editors hold it). Your server or indexer owns collection and source-specific finality. Tracker controls (pause, resume, refresh) require `publish`, work in live with a personal API key, and appear in Console **Audit history**; reads require `read`. Custom collection does not trigger content quests; declare `automatic({ tracker })` instead.

#### Rewarding X posts

X does not allow apps that reward users for posting, so Domino rewards X posts only for creators an operator approved. When asked to reward people for posting or tweeting, build a curated creator program: name a group with `defineGroup({ key: "approved-creators", title: "Approved creators" })` and add `requires: [inGroup(group)]` to every quest that pays for X posts, whether `automatic({ tracker })` paying the post's author, `content({ tracker })` on an X tracker, or a manual quest that verifies a submitted post. Deploying creates the group, empty, and promoting creates it in live; operators add approved creators in Console **Groups**, with `domino groups`, or with `management.groups`. Posts by anyone else are skipped with `prerequisites-incomplete`. A deployment that publishes an X tracker or content quest that pays (declared rewards or an engagement rule) without an `inGroup` prerequisite still publishes, with the warning `x-post-rewards-ungated`. Quests that check follows, bios or profile pictures, and quests paying the account a post replies to, quotes or reposts (`inReplyTo`, `quoted`, `reposted`) are not rewards for posting.

#### Tracker triggers

A tracker handle determines the allowed `beneficiary` values in TypeScript: X has `author`, `inReplyTo`, `quoted` and `reposted`, Discord messages have `author`, Discord reactions add `messageAuthor`, and custom sources add exactly their declared `accounts`. Dynamic definitions still receive runtime validation.

`customSource(namespace, { identities, accounts? })` makes record accounts verified identities Relay attributes to members. `identities` lists kinds from `"wallet"` (canonical CAIP-10 account ID, the member who currently links it), `"x"` and `"discord"` (stable numeric user ID, the member whose selected, verified account has it), `"member"` (a member ID), and `"app:<name>"` (your app's own user ID, such as `"app:game"`: the member the project's identity integration created with that `identityKind` signed in with that subject; `app:` then 1–40 of `a-z 0-9 _ -`; an unlinked ID skips with `unlinked-account` "No member has signed in with this app:<name> identity"). At most 8 kinds. `accounts` names roles records can carry besides `author`, such as `["recipient"]` (not `author`; needs `identities`). Records then need `author: { id, kind? }` and may carry `accounts: { <role>: { kind?, id } }`; `kind` is required unless the source lists exactly one kind. The ingesting server names the accounts: Relay checks each is verified by a member, not that it took part in the event, so a record can pay any member (`member` kind) or linked account; hold `ingest` keys only in trusted servers such as your own indexer. Wallet IDs are canonical: `cosmos:<chain ID>:<lowercase bech32>` (any chain ID; attributed to the member who links that bech32 address, whichever chain ID it names), `eip155:<chainId>:<EIP-55 address>` (any chain ID matches a linked EVM wallet), or `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:<base58>` (`solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1:<base58>` on devnet; both attributed to the member who links that Solana wallet). Ingestion refuses a missing author, an undeclared kind or role, a non-canonical wallet (naming the expected ID), or a non-numeric X/Discord ID with 422; `author.verified` is stored as `null`. Record reads return `author.kind` and `accounts` as ingested (a kind the record left out stays absent). Without `identities`, authors are opaque, records carry no kinds or accounts, and no quest can trigger from them. Changing the source, except its `data` schema, needs a new tracker key.

`trigger: automatic({ tracker, on?, where?, beneficiary?, enrollment?, since?, historical? })` admits one attempt per quest for each record observation it selects, inside the ingest (custom), collector (X) or gateway write (Discord) transaction. `on`: `"new-record"` (a record's first observation in this tracker; default for X, Discord and Telegram, so refreshes, metric changes, edits, a re-added Telegram reaction, and another tracker's discovery never admit again), `"new-observation"` (each observation that becomes the record's current one: new record or later `observedAt`, even with unchanged data; default for other custom sources, which in SDK 1.x include YouTube Shorts, TikTok and wallet balance trackers, so each refresh or daily sample is selected again; `domino check` [warns](authoring.md#upgrade-existing-authoring-code) `TRIGGER_REFIRES_ON_REFRESH` for a non-content quest on a YouTube or TikTok tracker in this mode), or `"data-changed"` (new record, or `data` differing from the record's previous current observation). The authoring helper always materializes the selected `on` value in the published policy, so a published quest keeps its mode: Telegram quests built with SDK 1.62 or earlier keep `"new-observation"` until republished. `where`: one condition on a dot-separated path in `record.data`, `{ path, equals: string | number | boolean }` or `{ path, gt?, gte?, lt?, lte? }` comparing a JSON number or an integer string (such as a base-unit balance) exactly; a bound is a number or `{ setting: "<integer or amount setting of this quest>" }`, resolved from the release's settings; an amount setting's integer string compares exactly, so 18-decimal thresholds fit, and never matches a fractional value. A missing key, a key into an array or scalar, an `equals` of another type (`6` is not `"6"`), or a compared value that is not a number or `-?digits` string (`"1.5"`, `null`) never matches. Observations the mode or filter does not select record no outcome; `ingest` returns `admissions: { admitted, skipped, filtered? }`, where `filtered`, present only when nonzero, counts per quest the observations its mode selected but `where` did not match. `beneficiary`: the record account role whose member it pays, `"author"` (default), `"inReplyTo"`, `"quoted"` or `"reposted"` on X, `"messageAuthor"` on a Discord reactions tracker, or a role the custom source declares in `accounts`; when that account is the record's author account, or both resolve to the same member (the same wallet on another chain, another linked account), the observation is skipped (`beneficiary-is-author`, such as a self-reply). `enrollment`: `"none"` (default) or `"required"`: only members who enrolled in the quest (`relay.quests.enroll({ actionId, quest })`, or `management.quests.enroll({ actionId, quest, member })` with `enroll` permission; enrolling again keeps the first time; a quest without required enrollment refuses with `enrollment-not-required`). `since`: `"activation"` (default; records since the quest was published or last unarchived), `"window"` (every record in the tracker's window), or, only with `enrollment: "required"` (`automatic()` throws otherwise), `"enrollment"` (records since the member enrolled). `historical`: who releases activity from before the member linked the beneficiary account (a link time of zero or unknown counts as before; `member` accounts are never historical) or before they enrolled: `"staff"` (default; a reviewer's retry), `"claim"` (also the member, `relay.quests.claim`), or `"automatic"`. The other defaulted admission fields may remain absent. `where` may also be a list of conditions that must all hold. TypeScript checks `where` (at most 10 conditions) against the tracker's data (paths to string, number or boolean fields up to five levels, arrays end a path, a `z.record` field takes any key and a `z.unknown()` field any path; `equals` values of the field's type; comparisons on number or string fields) and a `{ setting }` bound against the quest's integer and amount settings; `settings.object` accepts only `settings.*` builders. `automatic()` throws for a role the source's records lack (see the roles above), an undeclared custom role, a custom source without `identities`, a filter path the data schema does not declare or a value its field cannot hold, or (in `defineQuest`) a setting bound that is not an integer or amount setting; publication refuses the same with 422. Use `once()`, `keyed(...)`, or `streak(...)` (each accepted observation marks the calendar day of `record.occurredAt`; see [calendar-day streaks](quest-rules.md#calendar-day-streaks)); content quests instead omit `completion`. The tracker must be published (same deployment or earlier); switching a quest into, out of, or between tracker triggers needs a new quest key. The evaluator receives `ctx.tracker`, non-null for a tracker trigger (null for other triggers), with `record.data` of the tracker's type: `{ tracker, key, beneficiary, record: { id, source, occurredAt, observedAt, firstSeenAt, author, accounts?, data, removed }, content, attribution: { member, capturedAt, account }, enrolledAt? }` (`enrolledAt`, the member's first enrollment time, only on quests that require enrollment). `content` is absent for sources without a content declaration; otherwise it holds `{ body, revision, observation }`, with the source’s typed body. X bodies include `authorId`, `text`, `isReply` and `references`; custom bodies match their source’s body schema. `account` is `{ provider: "wallet", id, revision, linkedAt }`, `{ provider: "x" | "discord", issuer, subject, revision, linkedAt }`, `{ provider: "member", id }`, or `{ provider: "app:<name>", subject, revision, linkedAt }`. The account can also be `{ provider: string }` for a kind a later Relay adds; branch on `provider` with a default branch. `beneficiary` is the stored canonical role. `ctx.eligibleAt` is `record.occurredAt` (an X post's publication time). Attribution is pinned at admission. Examples: a "Hold N" quest reads `ctx.trackers.records(tracker, { member: ctx.member.id })` to total every linked wallet (`packages/sdk/examples/hold-balance-quest.ts`); a quest that rewards members when an account replies to them uses `beneficiary: "inReplyTo"`, keys by `tracker.record.id` (one reward per reply) with a `period: "week"` budget, and takes the replying account IDs and required text as settings (`packages/sdk/examples/account-reply-quest.ts`); a posting streak for approved creators counts each new X post's publication day toward a five-day `streak(...)` (`packages/sdk/examples/posting-streak-quest.ts`); a kudos quest pays a custom record's `recipient` role with `on: "new-record"`, so a rejected kudos edited later never pays, and a giver no member has linked still pays the recipient, capped by a weekly budget (`packages/sdk/examples/kudos-quest.ts`).

Every selected observation is exactly one trigger outcome: `admitted` with its attempt (`code: null`), or `skipped` with the first failing rule as a stable `code` and a human `reason`, in this order: `quest-paused`, `before-activation` (with `since: "activation"`), `outside-window`, `future-record`, `record-removed`, `invalid-record-data` (data stored before the tracker's schema changed no longer matches it), `content-unavailable`, `no-account` (such as "Post is not a reply", "Discord did not name the message's author" or "Record has no recipient account"), `beneficiary-is-author` (such as a self-reply or a reaction to one's own message; a different account of the author's member is judged after it resolves, following `unlinked-account` and `ambiguous-account`), `unlinked-account` (such as "No member has linked this wallet"), `ambiguous-account`, `not-joined` (Relay never joins a member on their behalf), `already-completed` (a completed `once()` quest, or a streak whose milestone paid), `not-enrolled`, `before-enrollment` (with `since: "enrollment"`), `historical`, `prerequisites-incomplete` (a required quest was not complete when the record occurred, or the member was not in a required group when it was decided; completions or group membership recorded later never decide it again). Branch on `code`, never on `reason`; an `outside-window` outcome can include `window: "tracker" | "quest"` to name the tracker's collection window or the quest/campaign availability window, and a `not-enrolled` outcome `enrollment: "quest" | "campaign"` to name the enrollment the member lacks, the quest's own (`enrollment: "required"`) or its campaign's (an open string: a later Relay can add values); older responses omit both. `trackerSkipCodes` from `@domino-sdk/relay/schema` lists the codes this SDK knows, and later versions may add codes (`TrackerSkipCode` is an open string); a skip recorded before Relay stored codes, whose reason matches no rule, has `legacy`. A skip is decided again, under the quest's current release and every rule, only by the member event that clears its code, shortly after it commits (a durable workflow): linking the account (`unlinked-account`), removing a conflicting link (`ambiguous-account`), joining the project (`not-joined`), enrolling in the quest or its campaign (`not-enrolled`); activity from before the link or enrollment then becomes a `historical` skip naming the member, or is admitted under `historical: "automatic"`. Every other skip is final unless retried. A quest budget never skips an observation: it is admitted, and its attempt is rejected with `budget-exhausted` when the budget cannot cover another completion. Read `management.trackers.triggerOutcomes(tracker, { quest?, record?, member?, state?, code?, limit?, cursor? })` with `read` (`{ items, nextCursor }`, newest first). `management.trackers.retryOutcome(tracker, { actionId, quest, record, observedAt })` with `review` re-runs the rules under the quest's current release, filter, links, membership and enrollment, releasing historical activity whatever the policy, and reuses the automatic attempt's action ID; it returns 409 with the current reason as `error` and its skip code as `code` (such as `already-completed`, with `enrollment` on `not-enrolled`) when it still cannot admit, and 409 "The record no longer matches the quest's filter". Members read their own outcomes with `relay.quests.outcomes({ quest?, state?, code?, cursor? })` (`{ items, nextCursor }`, newest first, 50 a page; each item `{ quest, tracker, record, observedAt, state, code, reason, window?, enrollment?, attempt, claimable, createdAt, updatedAt }`; React `useQuestOutcomes`) and claim one with `relay.quests.claim({ actionId, quest, tracker, record, observedAt })` when it is `claimable` (a `historical` skip on a quest with `historical: "claim"`): it is decided again with historical activity released and returns the outcome, or rejects with the failing rule's skip code, or `not-claimable`. Console shows outcomes under the quest's **Trigger outcomes** and a record's **Quest outcomes**, with **Retry admission** for reviewers, **Enroll the member** on a `not-enrolled` skip for operators with `enroll`, and the quest's enrollment, `since` and `historical` under **How it works**. To rank the value itself: an external score gets its own ledger, updated with absolute values (`points.record({ kind: "set" })`); see [Rank an external score](https://console.domino.run/docs/build/leaderboards#rank-an-external-score). Trackers do not issue ongoing engagement rewards or reverse prior rewards. See [trackers](https://console.domino.run/docs/build/trackers) for full examples, and the [trackers reference](trackers.md) for collection options, record reads, trigger rules, outcome codes and engagement accounting.

#### Automatic content admission

Author content quests with `trigger: content({ tracker, enrollment?, since?, historical?, where? })` and omit `completion`. The source must declare content: `x.search()` does so, or use `customSource(namespace, { identities, content: { body: z.object({ ... }), metrics: ["plays", ...] } })`. Custom content records carry `data: { body, metrics? }`; missing counters are unknown, zero is measured. The record ID identifies the content item, its author the creator and `occurredAt` the publication time. Plain custom sources and Discord sources cannot trigger content stages.

Content uses the shared tracker admission pipeline, fixed to the first observation and author beneficiary. Defaults are `enrollment: "none"`, `since: "activation"` and `historical: "claim"`. Choose `since: "window"` to count from the tracker window or `"enrollment"` with required enrollment. Each skip has a reason and is decided again when its blocking identity or membership condition changes. Admission pins the current release and beneficiary; later body edits qualify under that release. `ctx.tracker.content` holds `{ body, revision, observation }`. Attribution and enrollment remain on `ctx.tracker`; `ctx.content`, `ctx.tracker.x` and `perContent` are removed. Budgets use the record's occurrence time through `ctx.eligibleAt`.

Members enroll with `relay.quests.enroll`, read `relay.quests.outcomes`, and release claimable historical outcomes with `relay.quests.claim({ actionId, quest, tracker, record, observedAt })`. Management reads `trackers.triggerOutcomes(tracker, { quest, record? })` and retries with `trackers.retryOutcome(tracker, { actionId, quest, record, observedAt })` and `review` permission. Neither a retry nor a claim bypasses other admission rules. The old content admission/claim/approval APIs are removed. Content qualification history remains `content.qualifications({ quest, content?, member?, cursor? })`, scoped to the session member for participant reads. Console exposes content admission, qualification, performance and engagement through **Community → Trackers → Quest outcomes** for both X and custom content.

#### Content performance gates

A content quest can declare `performance: { metrics: ["likes"], maxAgeMs: 3600000, evaluate: ({ metrics }) => metrics.likes >= 100 ? decision.accept() : decision.reject({ reason: "Reach 100 likes." }) }`. Only content triggers support this stage. Select a nonempty unique subset of the source’s declared metrics; X provides `likes`, `replies`, `reposts`, `quotes`, `views`, and `bookmarks`. Callback types expose exactly the selected counters. Context also includes settings, member, pinned `tracker.content` and sample `observedAt`, without provider capabilities.

Content approval is cached and does not award until performance passes. Each new provider observation can queue performance while retaining the content approval; body edits or changes to the verified account selection require new content approval first. Metrics collected during content review are used after approval. Missing counters and stale/future samples wait without executing code or fabricating zeros. `maxAgeMs` defaults to one hour and is bounded at 24 hours. New samples, body edits, removal, and ownership changes invalidate outstanding performance decisions and reviews.

Performance gates the content-approved reward selection; its callback cannot change those rewards. Its result is `decision.accept()`, `decision.review({ reason })` or `decision.reject({ reason })`; a result that carries `rewards` fails the evaluation, and the types reject it. The first acceptance resolves the ordinary completion and budget outcome. Later threshold drops preserve prior awards, and recoveries cannot repeat effects or retry budget exclusions. Add a separate engagement rule for daily settlement. Automatic reversals remain unsupported.

`content.performance({ quest, content, member?, cursor? })` returns `current` eligibility/reason/attempts/reward and up to 100 retained performance `items` with metrics and observation time, newest first. Continue with `cursor: page.nextCursor`. Participant identity is session-derived; management requires `read`. Content-only history remains in `content.qualifications`. See [performance thresholds](https://console.domino.run/docs/build/trackers#require-an-engagement-threshold-before-awarding).

#### Daily engagement points

Use `defineEngagementReward` from `@domino-sdk/relay/authoring` with `key`, a content quest handle in `qualifiedBy`, a point-ledger handle in `ledger`, `baseline: "historical"`, nonempty `metrics`, and synchronous `target({ metrics })`. The target returns nonnegative safe whole-number cumulative points; use a cumulative floor for fractional rates. The isolated formula has no network/member capabilities and must not use time or randomness. Optional `maxAgeMs` defaults to one hour (maximum 24 hours). Optional `budgets` is a list of `{ scope: "project" | "member", period: "day" | "week" | "month" | "lifetime", limit }` with nonnegative limits, at most one per scope and period, independent per rule; periods are those of the [project calendar](authoring.md#project-calendar), and each point entry is charged to the period of its `effectiveAt`. Accounts settle at most once per calendar day, plus a closing settlement when a leaderboard over their points closes.

Add `engagement: [{ entry: "engagement.ts" }]` to `relay.json`, with optional `exportName` for a named export and at most 50 entries. The module can live anywhere relative to the manifest. Engagement entries are explicit, independent of quest discovery globs. `domino deploy --preview` reports `create`, `unchanged`, or `change`, and `deploy --resume DEPLOYMENT_ID` publishes the saved preview. Tracker, quest, ledger, and rule can publish together. The initial discovery cutoff is publication time, not preview time. Omitting an entry never deletes a rule.

A normal `domino deploy` previews and applies formula changes atomically with the other resources. A stale version or accounting preview rejects the whole deployment with `409`; preview again. Retrying a committed deployment ID returns its saved receipt. `domino dev` synchronization and hosted catalog artifacts carry the same entries. Publication and deployment previews are test-only: in live they fail with 422 `promotion-required` for every credential, so deploy to test and promote.

For a standalone API request, bundle the default export with `buildQuest` from `@domino-sdk/relay/build`, then call `management.deployments.preview({ engagement: [{ code }] })` and `management.deployments.publish(preview.id)` with `publish` permission. Keep the deployment ID on retries. Formula versions are immutable: identical code replays, while changed code uses the preview/apply workflow below. Another key is a separate additive rule, not a replacement. Only discoveries after publication can accrue; there is no implicit backfill.

Current content qualification and, when configured, performance approval gate accounting. Complete fresh counters advance component-wise high-water values. Accounting processes the latest eligible sample; it does not replay earlier peaks from review or worker downtime. Settlements become due at the first midnight of the [project calendar](authoring.md#project-calendar) after initial eligibility, then run at most once per project calendar day. Delayed settlements use their actual calendar day and available fresh observation, not reconstructed midnight samples. Entries are dated by measurement: each takes effect when the sample that measured its growth was observed, and growth between two samples that spans a leaderboard window or budget period boundary splits there in proportion to time (rounded down, remainder to the later part), one entry per side. Budgets pay partially; excluded growth advances the accounted boundary permanently. Credits go directly to the balance without another completion or referral payout. Counter drops and later rejection do not reverse credits.

Quest pause stops accounting. The first fresh eligible sample after resume becomes the new baseline; previously earned pending points survive and paused growth through that observation is skipped. Enrollment-only baselines, historical backfill, quality multipliers, reversals, restorations, and specialized Console controls are not supported. Read [daily settlement](https://console.domino.run/docs/build/trackers#settle-engagement-points-daily) before proposing them.

Management `engagement.list()` and `engagement.page({ rule, content, member?, cursor? })` require `read`. The page returns the rule, nullable last processed account checkpoint (`earned`, `accounted`, `paid`, `excluded`, `pending`, `counters`, `baseline`, `observation`, `nextSettlementAt`, `reason`), and up to 100 immutable settlement receipts, newest first. Continue with `cursor: page.nextCursor`; receipts reference their evidence, qualification, and point entry.

#### Engagement formula versions

After changing only `target`, bundle the code and call `management.deployments.preview({ engagement: [{ code }] })`. Review the formula change engagement entries, then call `management.deployments.publish(preview.id)`. Both require `publish`. Management keys can publish in test; the change reaches live through a promotion. Retry publication with the same deployment ID. A changed requested version, admission set, or accounting checkpoint invalidates the preview (`409`). Quest, balance, metrics, baseline policy, freshness policy, and budgets must stay unchanged. The older engagement-only `previewChange` and `change` methods remain deprecated compatibility APIs.

The preview reports retained accounting, with project totals as decimal strings. It explicitly defers settlement; it does not promise a future payout. The supported boundary is `available-eligible-observation`: the fresh eligible sample available when a post transitions, possibly predating the request by up to `maxAgeMs`. Never describe it as an exact publication-time counter. Each post settles the current formula through its high-water counters and starts the latest requested formula at the same baseline in one transaction. The old earned/accounted totals, budget spend, and exclusions survive. Boundary credits can occur in addition to the daily receipt, using the same UTC-day budget.

Posts waiting on qualification, ownership, metrics, pause, or scoring retain their current formula. A later request supersedes unapplied versions for that post; no intermediate earnings are invented. New discoveries use discovery-time versions, and late admission preserves those original terms. Formula changes preserve the original discovery cutoff and do not backfill content. A failing requested formula can be superseded; an unscorable current formula cannot be force-skipped.

Read `management.engagement.ruleVersions(rule, { cursor? })` for the immutable request history. `engagement.boundaries({ rule, content, member?, cursor? })` returns `current`, `requested`, and up to 100 receipts with `from`, `to`, `baseline`, provider `observedAt`, and `settlement`; continue with `cursor: nextCursor`. Management requires `read`, and participant reads are session-scoped. `engagement.list()` shows requested versions; `engagement.page()` retains its shape and shows the account's governing version. See [formula changes](https://console.domino.run/docs/build/trackers#change-a-formula-without-repricing-past-engagement).

### Discord trackers

Declare `defineTracker({ key, title, source: discord.messages(), window })` or `source: discord.reactions()` (`discord` is exported by `@domino-sdk/relay/authoring`), list it under `trackers` in `relay.json`, and trigger quests with `automatic({ tracker })`. The project needs a ready Discord integration. The gateway writes each new message or reaction into every tracker of that kind in each project and environment whose ready integration names the server, connected before the event. `trackers.ingest` refuses a Discord tracker with 403 and a `refresh` control with 422. A paused tracker, or an event outside the window, stores nothing. Bot and webhook messages and bot reactions are not recorded; nothing is backfilled.

- **Members' activity only.** A record is stored only when at least one of its accounts is a Discord account a member of the tracker's project has linked: the message's poster, or the reactor or the message's author for a reaction.
- **`discord.messages()`.** Record ID: the message ID. `author`: the poster's Discord user ID. `occurredAt`: the publish time, or the receipt time when earlier. `data`: `{ guildId, channelId, parentChannelId, messageId, replyTo, publishedAt }` (`DiscordMessageData`), with `replyTo` null for a non-reply. `discord.messages({ text: true })` adds `text` (`DiscordMessageTextData`), null when the platform bot lacks the privileged Message Content intent (`DISCORD_MESSAGE_CONTENT=true`). An edit updates stored text. `on: "new-observation"` and `on: "data-changed"` can select that edit and run `where` again; the default `new-record` cannot. A message deletion marks the record removed and produces `record-removed`. Deletion, or publishing the tracker without `text: true`, erases text from current records, retained observations, and pinned attempts.
- **`discord.reactions()`.** Record ID: `messageId:reactorId:emoji`, where `emoji` is a custom emoji's ID or the Unicode emoji. `author`: the reactor's Discord user ID. Named account `messageAuthor`: the message author's Discord user ID, absent when the lookup failed. `occurredAt`: the receipt time, since Discord sends no reaction time. `data`: `{ guildId, channelId, parentChannelId, messageId, emoji: { id, name }, reactorRoles, messagePublishedAt }` (`DiscordReactionData`), with `reactorRoles` the reactor's role IDs at reaction time. Adding a removed reaction again is a new observation of the same record.
- **Triggers.** The identity kind is `discord`, and the default `on` is `"new-record"`, so each message or reaction runs a quest once. A removal produces a `record-removed` outcome in every mode. Data types are fixed, so `where` paths such as `channelId` and `emoji.name` and `ctx.tracker.record.data` are typed. `where` values are literals; check a Console-chosen setting, such as a channel, in the evaluator.
- **Paying the message author.** `automatic({ tracker: reactions })` pays the reactor; `automatic({ tracker: reactions, beneficiary: "messageAuthor" })` pays the message's author ("reaction received"). A reaction from the message author's account, or from another account of the same member, is skipped with `beneficiary-is-author`, and a reaction without `messageAuthor` with `no-account`. The reactor need not be linked when the message's author is; check `data.reactorRoles` for moderator-only recognition and key by `data.messageId` to pay once per message.
- **Attribution.** Relay pays the member whose selected Discord account is the beneficiary, falling back to the member's only Discord connection when none is selected. An unlinked beneficiary is skipped with `unlinked-account`, a member who has not joined the project with `not-joined`.
- **Streaks.** `streak(...)` works on these triggers: each accepted record fills the calendar day of its `occurredAt`.

`ctx.discord.message({ channelId, messageId })` is server-scoped and returns null for deleted messages; `discordMessageSchema` describes its result. It reads with the bot in the server the attempt pinned, so it needs no linked Discord account. For manual authorship checks, compare message `userId` to a non-null `userId` from `ctx.discord.member()`. Older captured member facts can return null; fail closed. A tracker-triggered quest needs no such check: Relay attributes the record to the beneficiary's member. Captured reads survive evaluation retries.

Install `discord-contribution` for channel participation and `discord-moderator-recognition` for moderator heart reactions; `telegram-contribution` is the Telegram equivalent of `discord-contribution`. Use one snippet per capability; customize predicates in code. Daily budgets are caps, not streaks. Use the streak completion policy below for consecutive calendar days. General multi-action counters remain unavailable.

### Discord Gateway delivery

The API deployment includes a managed Discord Gateway receiver with the `GUILD_MESSAGE_REACTIONS` and `GUILD_MESSAGES` intents. It feeds the Discord trackers above. People's guide: [Where Discord events come from](https://console.domino.run/docs/build/integrations#where-discord-events-come-from).

- **Starting.** Installing the integration starts it in production. Existing installations can start it by reading `GET /management/v1/integrations/discord/events` with a scoped management credential. Local development does not start a real bot connection automatically.
- **Status.** That response contains the connection state, pending deliveries and recent coverage gaps. Console shows them on the Discord card in **Integrations** (**Events**, **Recent coverage gaps**).
- **Shared bot.** The bot is shared by every project, so a project cannot restart it. When the state is `blocked`, Domino has to fix the bot's configuration; it reconnects on its own afterwards.
- **Delivery.** New messages and reactions start delivery as soon as Relay records them. Delivery writes each one into the Discord trackers of every project connected to the server. The receiver drains an indexed queue in bounded batches and schedules failed deliveries with backoff.
- **No duplicates.** Successful delivery removes the captured event. Tracker writes repeat nothing on a retry, and durable session sequence numbers and quest completion keys prevent duplicate rewards.
- **Resuming.** Once started, the receiver resumes automatically when it wakes after a restart. A five-minute watchdog also wakes it when idle. Discord's required connection heartbeats run separately from delivery.
- **One shard.** A receiver supports one unsharded managed bot. If Discord requires multiple shards, event processing reports a configuration blocker. Membership verification remains a separate HTTP capability.
- **Coverage limits.** Discord does not include an action timestamp in reaction events. Relay processes durably received additions, but skips previously unseen replayed reactions whose timing is unknown. A new Gateway session can leave a coverage gap. Current reaction lists cannot reconstruct removed reactions or their dates. Gateway status reports these limitations; it is not a guarantee of complete Discord history.

### Telegram

Operators connect Telegram groups, supergroups and channels in Console **Integrations** (or `management.telegram.connect()`), through Domino's shared bot; a project can connect several. See the [Telegram guide](https://console.domino.run/docs/build/telegram).

- **Account and membership.** `await ctx.telegram.connection()` reads the member's linked Telegram account, vouched or proved: `{ userId }`, or `null`, with no chat or bot. `await ctx.telegram.member({ chat })` reads that account in a connected chat, named by its numeric ID or `@username`. No `integrations` entry: Relay supplies `ctx.telegram` and `connection()` to every quest; the types are optional only so hand-written test contexts from before them compile, so call `ctx.telegram!.connection!()` and `ctx.telegram!.member(...)` under strict TypeScript, and build test contexts with `fakeTelegramReader(facts)` from `@domino-sdk/relay/testing`. `member()` is `null` when the member is not in the chat or has not linked Telegram, or `{ userId, username, status, isMember, isAdmin, untilDate }`. An unconnected chat, an unconfigured bot or a chat whose bot lost administrator access throws `CapabilityError` `integration-required`; a Telegram outage `provider-unavailable`. The member's Telegram account (or that they have none) is pinned at the attempt's first Telegram read; chats are read live. Name the chat with `settings.telegramChat({ label })`, a text setting Console fills from the connected chats. The `telegram-membership` snippet is a complete join quest.
- **Reactions.** `defineTracker({ key, title, source: telegram.reactions(), window })` (`telegram` is exported by `@domino-sdk/relay/authoring`) records each reaction a member adds to a message in the project's connected groups and supergroups, from chats connected before the reaction. Record ID: `chatId:messageId:reactorId:reaction`. `author`: the reactor's Telegram user ID. Named account `messageAuthor`: the message author's user ID, absent for a message the bot did not see. `occurredAt`: Telegram's reaction time. `data` (`TelegramReactionData`): `{ chatId, chatType, messageId, reactionType, emoji, customEmojiId, reactorStatus, reactorIsAdmin }`, all scalars for `where`, such as `[{ path: "reactorIsAdmin", equals: true }, { path: "emoji", equals: "❤" }]`. A record is stored only when the reactor or the message's author is a member's linked Telegram account; a removed reaction marks the record removed. Channel reactions are anonymous and never recorded. Pay the message's author with `beneficiary: "messageAuthor"`; self-reactions skip with `beneficiary-is-author`. Attribution, link times and redecision follow the member's selected Telegram account, like Discord.
- **Messages.** `defineTracker({ key, title, source: telegram.messages(), window })` records each message a person posts in the project's connected groups and supergroups, from chats connected before the message. Record ID: `chatId:messageId`. `author`: the poster's Telegram user ID. `occurredAt`: Telegram's send time. `data` (`TelegramMessageData`): `{ chatId, chatType, messageId, replyToMessageId, threadId }`, all scalars for `where`; `replyToMessageId` is null when the message is not a reply (in a forum topic, the topic's first message is not a reply target), and `threadId` is the forum topic, or the reply thread outside forums, or null. No text is stored. A record is stored only when the poster is a member's linked Telegram account. Bots' messages, service messages (joins, pins, topic changes and other messages with no content), messages sent on behalf of a chat, and channel posts are never recorded. An edit is the same message and writes nothing; the Bot API sends no deletes, so a deleted message stays recorded. The `telegram-contribution` snippet pays once per member per project-calendar day and checks an optional `settings.telegramChat()` in the evaluator; for consecutive days replace its keyed completion with `streak({ days })` and remove its budget.
- **In SDK 1.x** both sources are custom sources on the reserved namespaces `telegram-reactions` and `telegram-messages`: `trackers.ingest` refuses them, and publishing either namespace with any other source is refused.

### YouTube Shorts

- **Source.** `defineTracker({ key, title, source: youtube.shorts(), window, refreshSeconds? })` (`youtube` is exported by `@domino-sdk/relay/authoring`) reads the Shorts members post on the YouTube channels they linked as creator accounts, with each member's own token, every `refreshSeconds`, held to the plan's minimum like an X tracker's (6 hours on every plan but Enterprise, `effectiveRefreshSeconds` on the status). Unlike X search, a shorter `refreshSeconds` gets no deployment warning. No search, no API key. A non-content quest on it in the default `new-observation` mode is evaluated again for every Short on each refresh, and `domino check` [warns](authoring.md#upgrade-existing-authoring-code) `TRIGGER_REFIRES_ON_REFRESH`; use `content({ tracker })` with `performance` for a metric threshold. Inert until Domino configures YouTube.
- **Records.** One per Short, keyed by video ID. `author`: the channel ID, attributed through the member's selected YouTube channel, so link times, `historical` and redecision on link work as for X. `occurredAt`: publication time. Content body `YouTubeShortBody`: `{ title, description, tags, durationSeconds, publishedAt }`; metrics `views`, `likes`, `comments` (a hidden counter is unknown, never zero). Use it with `content({ tracker })`, `performance` and engagement rules like any content source.
- **Which videos.** Candidates from the channel's Shorts playlist (the uploads playlist when it is missing), kept only when at most 180 seconds long (60 for uploads before 2024-10-15) and vertical or square.
- **Removal and outages.** A deleted or private Short is removed. An unlinked channel is no longer read; its records stay. A token that needs reconnecting, an outage or a spent daily quota is `provider-unavailable` in the tracker's error and removes nothing; a spent quota defers collection to the next Pacific-time day.
- **View counts.** Shorts count every play from 2025-03-31, and every format counts at playback start from 2026-08-24; counts are not comparable across those dates.
- **In SDK 1.x** the source is a custom source on the reserved namespace `youtube-shorts`: `trackers.ingest` refuses it, and publishing that namespace with any other source is refused.

### TikTok videos

- **Source.** `defineTracker({ key, title, source: tiktok.videos(), window, refreshSeconds? })` (`tiktok` is exported by `@domino-sdk/relay/authoring`) reads the public videos members post on the TikTok accounts they linked as creator accounts, only with each member's own token, every `refreshSeconds`, held to the plan's minimum like an X tracker's (6 hours on every plan but Enterprise, `effectiveRefreshSeconds` on the status). Unlike X search, a shorter `refreshSeconds` gets no deployment warning. No search; TikTok has no app-only read. `domino check` [warns](authoring.md#upgrade-existing-authoring-code) `TRIGGER_REFIRES_ON_REFRESH` as for YouTube. Inert until Domino configures TikTok.
- **Records.** One per video, keyed by video ID. `author`: the account's `open_id`, attributed through the member's selected TikTok account, so link times, `historical` and redecision on link work as for X. `occurredAt`: creation time. Content body `TikTokVideoBody`: `{ title, description, durationSeconds, width, height, createdAt, shareUrl, isAigc }` (`description` is the caption, `isAigc` the creator's AI-generated label); metrics `views`, `likes`, `comments`, `shares` (a counter TikTok leaves out is unknown, never zero). Use it with `content({ tracker })`, `performance` and engagement rules like any content source.
- **Removal and outages.** A known video the account no longer returns is removed. An unlinked account is no longer read; its records stay. A token that needs reconnecting is not read at all, and a token TikTok refuses is refreshed once, so a revoked grant reads `needs-reconnect` at once. An account that can't be read, and TikTok's request limit (counted for the whole platform over any 60 seconds, deferring collection to the next minute), are `provider-unavailable` in the tracker's error; any other failure names the failure and retries from the saved cursor. Nothing of these removes a record.
- **In SDK 1.x** the source is a custom source on the reserved namespace `tiktok-videos`: `trackers.ingest` refuses it, and publishing that namespace with any other source is refused.

### Wallet balance samples

- **Source.** `defineTracker({ key, title, source: wallet.balances({ chain, token }), window })` (`wallet` is exported by `@domino-sdk/relay/authoring`) samples one token's balance in every wallet linked by a member who joined the project. `chain` and `token` are those of `ctx.wallet.balance()`: `"solana"` (stored as Solana mainnet's CAIP-2 ID) or Solana devnet's `"solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1"` with an optional SPL or Token-2022 mint (both sample every linked Solana wallet), an EVM chain ID with an optional ERC-20 contract, or a Cosmos network with a required bank denom. Omit `token` for a native coin. The builder checks the syntax; a chain Relay doesn't read, or a contract that isn't a token, shows on the tracker's error at the first sample and is retried hourly. Chain and token never change after publication (a new token needs a new key). `refreshSeconds` and `maxPages` don't apply.
- **When.** Once per project-calendar day, at an offset in the first three quarters of the day derived from a secret Relay holds, the tracker and the date: members can't predict it, and every retry of a day reuses it. A run whose cursor was cleared that day (a republish) resumes the day's sample at its pin from the stored samples, and once every wallet is read a run writes nothing more that day. After a day's sample, `nextSyncAt` is the next day boundary, where the run finds that day's time, so a calendar change never skips a day. Runs are identified by the day's start, not its date.
- **Records.** One per wallet, keyed by its CAIP-10 account on the sampled chain, which is also `author` (identity kind `wallet`); each day's sample is a new observation with `occurredAt` = the sample time. `data` (`WalletBalanceSample`): `{ amount, decimals, block }`, `amount` in base units as a decimal integer string, `decimals` null when the token doesn't report them. One run pins the chain's latest EVM block number or Cosmos height and reads every wallet at it, 100 wallets per step; Solana can't read a past slot, so every Solana read requires `minContextSlot` = the run's pinned slot and `block` is the slot that wallet's answer reflects. Every linked wallet of the chain's family is read: all EVM wallets for an EVM chain, all Solana wallets for Solana, Cosmos wallets linked on that exact network; unlinked wallets and members who never joined are not.
- **Failures.** A provider outage writes nothing for that step, sets the tracker's error, and retries 15 minutes later at the same pin. A run whose day ends before every wallet is read stops, so the remaining wallets have no sample that day. Relay never writes a balance it didn't read: a day whose reads all fail has no sample, and `streak({ days })` breaks on it; operators can grant the quest by hand. The test-member clock affects evaluation, never sampling.
- **Quests.** Use ordinary tracker triggers: `automatic({ tracker, where: { path: "amount", gte: { setting: "threshold" } } })` (an amount setting) with `streak({ days })` or `once()`. A `where` judges one wallet; total a member's wallets in the evaluator with `trackers.records(tracker, { member: member.id })`. The `token-hold-streak` snippet is a 7-day hold quest.
- **Limits.** At most 5 unarchived `wallet.balances()` trackers per project and environment: publishing or unarchiving a sixth fails with 409 `limit-reached`. EVM samples need the platform's dRPC key.
- **In SDK 1.x** the source is a custom source on the reserved namespace `wallet-balances`, with the chain and token in its `balance` field: `trackers.ingest` refuses it (403 `forbidden`), and publishing that namespace with any other source, or `balance` on another namespace, is refused.

## Collection options

`defineTracker` accepts these options for an X search source:

| Option           | Default                | Accepts                                                                                                                                                 |
| ---------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `refreshSeconds` | Poll every six hours   | 300–86400. The project's plan sets the shortest interval a tracker runs at: 6 hours on every plan but Enterprise. A shorter value still publishes, runs at the plan's minimum (`effectiveRefreshSeconds` on the status), and the deployment warns `tracker-refresh-below-plan-minimum`. |
| `maxPages`       | 60 search pages per cycle | 1–1000                                                                                                                                               |
| `window`         | Required               | Milliseconds. Includes `start` and excludes the optional `end`. Collection stops after the final cycle at the end; a later manual sync can refresh the retained posts. |

- **Credentials stay with the platform.** The platform operator configures the X wrapper credential on the collector. Campaign code does not receive that credential. A missing credential appears as a tracker error; the collector retries after configuration is fixed.
- **Removal keeps data and collection.** Removing a definition from the manifest does not delete its stored dataset or stop collection. A deployment publishes only the trackers it lists, so a removed tracker keeps polling until you pause or archive it.

Every cycle searches the whole window again, from the tracker's discovery start to now (or to `window.end` once that has passed), and stops after `maxPages` search pages. A long window with many matching posts can reach that limit on every cycle: the tracker then reports `partial` coverage, and its `error` field carries the warning "X search reached the configured page limit" even though the cycle succeeded. Raise `maxPages` or narrow the query or window. See [collection status](https://console.domino.run/docs/concepts/trackers#collection-status).

Search is limited by the provider's available history and results. Coverage **complete** means this cycle exhausted the provider's pagination and refreshed known posts successfully. It does not certify that every matching X post exists in the dataset. Page limits and unavailable known posts produce **partial** coverage.

A tracker created without a deployment (`trackers.create`, `domino trackers create`, or Console **New tracker**) takes the same fields and limits as `defineTracker`, with `source` named by the management API instead of a builder. Custom sources stay in code.

| `source` | Stored as | Settings |
| --- | --- | --- |
| `{ kind: "x-search", query }`, or `query` alone | `x.search({ query })` | The X search query |
| `{ kind: "discord-messages", text? }` | `discord.messages({ text })` | `text: true` stores each message's text |
| `{ kind: "discord-reactions" }` | `discord.reactions()` | None |
| `{ kind: "telegram-messages" }` | `telegram.messages()` | None |
| `{ kind: "telegram-reactions" }` | `telegram.reactions()` | None |
| `{ kind: "youtube-shorts" }` | `youtube.shorts()` | None |
| `{ kind: "tiktok-videos" }` | `tiktok.videos()` | None |
| `{ kind: "wallet-balances", chain, token }` | `wallet.balances({ chain, token })` | The chain's CAIP-2 ID and the token, null for a native coin; neither changes later (`--chain`, `--token` on `domino trackers create`) |

`create` takes exactly one of `query` and `source`. `refreshSeconds` and `maxPages` take the defaults above. An X tracker's first collection searches from `window.start`, and the answer is the tracker's status, with `origin: "data"`. A title, `refreshSeconds` or `maxPages` change leaves discovery where it was. An `update` with `source` replaces the source's settings: a different kind, or `query` on a tracker that is not an X search, fails with 422 `validation-failed`. Turning Discord message text off erases the text already stored, as publishing `discord.messages()` without `text: true` does. `managedTrackerSource(source)` from `@domino-sdk/relay/schema` reads a published source by these names.

Environment bindings are not definitions, so a promotion never copies them: a Discord or Telegram tracker promoted to live records nothing until live has its own Discord server or Telegram chats connected.

## Code or data owns each key

A key belongs either to code or to data, and only its owner changes the definition. Tracker status shows it as `origin`: `code` for a tracker a deployment publishes, `data` for one created in Console or with `trackers.create`, of any source in the table above.

| You try to | When the key belongs to | Result |
| --- | --- | --- |
| Deploy a `defineTracker` with that key | Data | 409 `tracker-owned-by-data` |
| `create` or `update` it | Code | 409 `tracker-owned-by-code` |
| `create` it again | Data | 409 `already-exists` |

Deployments never change or remove data trackers. To move a tracker between code and data, create one with a new key. Live refuses `create` and `update` with 422 `promotion-required`, from Console as well as from API keys.

## Record filters

`management.trackers.records(tracker, query)` and `trackers.summary(tracker, query)` accept these filters. They need a management credential with `read` permission.

| Filter                     | Behavior                                                                                                                                                                        |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `verified`                 | Omit to include everyone. `false` selects explicitly unverified authors; unknown verification is `null` and matches neither boolean. This boolean does not distinguish verification categories. |
| `member`                   | On an X tracker, the member's currently connected X account. On a custom tracker with `identities`, records whose author is one of the member's verified identities of those kinds, such as every wallet they currently link or their `app:<name>` user ID. A member with none matches nothing. Custom trackers without identities refuse it. |
| `minLikes`, `minViews`     | Engagement thresholds. Require an X tracker.                                                                                                                                    |
| `author`, `since`, `until` | Narrow by author and time.                                                                                                                                                      |
| `removed`                  | Selects by removal flag. Reads exclude removed records by default.                                                                                                                      |
| `cursor`                   | For the next page, repeat your filters and pass `cursor: page.nextCursor` when non-null.                                                                                        |

The X wrapper's `author.verified` value and additional provider fields are preserved. Summary totals cover all matching records, independent of pagination.

## Content revisions and observations

Pass `includeEvidence: true` to `trackers.records` to receive evidence references for X posts:

```text
import type { TrackerReader } from "@domino-sdk/relay/authoring";
import { launchPosts } from "./trackers/launch-posts";

// Pass management.trackers, or trackers from a hosted quest evaluator.
export async function captureEvidence(trackers: Pick<TrackerReader, "records">) {
  const page = await trackers.records(launchPosts, {
    includeEvidence: true,
  });
  return page.items.map(({ id, evidence }) => ({ id, evidence }));
}
```

| Reference | Changes when |
| --- | --- |
| `contentRevision` | The captured text, publication time, author ID, post flags, media type or URL, mentioned author IDs, video length, or post references change. |
| `observation` | The sample time or any captured provider data changes, including counters, author profile, verification, or removal status. |

Compare references only for the same X post in the same project and environment. Overlapping trackers share references for an identical sample. Returning to an earlier body reuses its revision; a revision is a body identity, not an edit count. Optional provider fields that are absent or malformed stay unknown in the stable body. Raw provider data remains in `record.data`.

Default reads omit `evidence`, preserving the existing response shape. Opted-in custom records return `evidence: null`. Hosted quest reads capture these references with their results for replay. These references do not trigger automatic qualification or rewards. Use the captured-content read below to retrieve a retained observation. An observation records when Relay sampled the provider; it does not prove the exact counters or author verification at publication time.

## Captured posts and account attribution

Call `trackers.content` with a record's `id` and `evidence.observation`. It returns that captured body, even if the current post has changed. The post must belong to the named X tracker in the selected project and environment. Custom trackers do not support this read.

In a hosted quest, Relay checks attribution to the attempt's member. The evaluator cannot supply another member ID:

```text
import type { TrackerReader } from "@domino-sdk/relay/authoring";
import { launchPosts } from "./trackers/launch-posts";

export async function readOwnedPost(trackers: TrackerReader) {
  const page = await trackers.records(launchPosts, { includeEvidence: true });
  const post = page.items[0];
  if (!post?.evidence) return null;

  const captured = await trackers.content(launchPosts, {
    id: post.id,
    observation: post.evidence.observation,
  });
  // null when the tracker has no such record or observation.
  if (!captured || captured.attribution.kind !== "linked" || captured.removed) return null;
  return captured.content;
}
```

The management SDK exposes the same read with a required `member` in the query and requires `read` permission:

```ts
import { createManagementClient } from "@domino-sdk/relay/server";
import type { TrackerContentQuery } from "@domino-sdk/relay/schema";

export function readCapturedPost(
  management: ReturnType<typeof createManagementClient>,
  query: TrackerContentQuery,
  member: string,
) {
  return management.trackers.content("launch-posts", { ...query, member });
}
```

| Attribution | Meaning at `attribution.capturedAt` |
| --- | --- |
| `linked` | The member's selected, verified X account has the captured author's stable ID. The result includes its issuer, subject, connection revision, and known link time. |
| `unlinked` | No unambiguous selected X account exists. Explicit disconnects take precedence over retained identity records. |
| `different-author` | The selected account's stable ID differs from the captured author. |

X verification badges and usernames do not prove account ownership. Legacy verified connections remain supported; an unknown link time is `null`. The returned `content` contains stable body fields, without engagement counters or mutable author profiles. Use `records` for observed engagement.

Hosted quest execution captures the entire result for replay, and attributes it to the X account the evaluation run pinned at its first X read. A scheduled run after defer or progress pins the current account. Edits, refreshes, account switches, and disconnects do not rewrite that captured result. In a hosted quest, a missing observation or a post outside the tracker returns `null`, and a custom tracker throws `CapabilityError` with code `author-error`. A fresh management read checks the current account selection; there, missing observations or posts outside the tracker return `404`, malformed queries `400`, and custom trackers `422`.

Attribution is evidence, not admission. A linked account does not establish enrollment, eligible publication dates, historical-content policy, or current post availability. The removal flag belongs to the requested observation. This read does not create an attempt or award rewards. Existing quests still apply their authored evaluation and ordinary reward rules.

## What a record contains

Each record has its original ID, occurrence time, first-seen time, observation time, author, removal flag, and provider data. X data retains content, media, references, author metadata, and available engagement metrics.

- **Missing metrics stay `null`.** Summary totals sum known values, and `missingMetrics` counts records missing any of the six metrics. Check freshness and coverage before treating a total as sufficient evidence.
- **Known posts are refreshed.** Known X posts are refreshed on the schedule in [How often Domino reads X](https://console.domino.run/docs/build/trackers#how-often-domino-reads-x), even when they disappear from search. A failed lookup does not prove deletion. It retains the last observation and reports partial coverage.
- **Empty search pages stop pagination.** X can return an empty page with a changing continuation cursor. Relay stops discovery for that cycle, retains collected records, refreshes known posts, and reports partial coverage with an empty-page diagnostic. Preview X search also stops at an empty provider page; its `more` flag stays true when an unconsumed continuation cursor remains. If your dataset stays empty, check that the query and tracker window include posts the provider returns. A cursor alone does not prove that more matching posts exist.
- **Overlapping X trackers share current facts.** Within one project and environment, trackers that collected the same X post read its latest observed content, author verification, and engagement. Each tracker keeps its own matches and first-seen times. A post collected by another tracker does not become a match in yours automatically. Each search still runs independently; concurrent known-post refreshes share a lease.
- **Pause stops collection.** An operator pauses a tracker, or Domino does when an X budget or a test window runs out (`pauseReason`). A paused tracker's retained posts can still show newer facts collected by another active tracker in the same project and environment. Pause does not freeze a dataset snapshot. Collection status and coverage describe each tracker's own runs.
- **Synthetic records are marked.** In test, an operator can send a test member's post, message or reaction into an X or Discord tracker ([Send X and Discord activity](https://console.domino.run/docs/use/quests#5-send-x-and-discord-activity)). Its record has `synthetic: true`, which no collected record has, and Console marks it **Synthetic**. The collector never asks X to refresh a synthetic post, and resetting the test member deletes its synthetic records.

## Point-in-time reads in a quest

Hosted evaluators have a read-only `trackers` capability. `TrackerReader` methods accept the handle returned by `defineTracker`, which preserves the record data type.

`trackers.records` and `trackers.summary` count records that occurred at or before `asOf`, which defaults to `ctx.eligibleAt`, each as its latest such observation. The attempt pins a watermark at its first tracker read, so every read of the attempt and of its retries sees the same stored observations. Pass `asOf: ctx.now` to count everything stored by then. A later attempt reads newer observations.

## Content quest admission

`content({ tracker, ...policy })` evaluates one collected content item at a time and completes once per item, so a content quest omits `completion`.

The collector retains unlinked authors. Each new record uses the shared tracker admission rules. Admission waits for one verified author account belonging to a project member, required enrollment, historical release, and prerequisites. Successful admission pins the current quest release, body revision and attribution. Skips are decided again under the quest's current rules. Username matches and verification badges do not establish ownership.

| Policy | Values and behavior |
| --- | --- |
| `enrollment` | `"none"` by default, or `"required"` for quest-specific enrollment. Repeated enrollment preserves the first server timestamp. |
| `since` | `"activation"` by default, `"window"` to include activity within the tracker window, or `"enrollment"` to count only from each member's enrollment. The last requires `enrollment: "required"`. |
| `historical` | `"claim"` by default, `"automatic"`, or `"staff"`. A post is historical when published before linking or required enrollment. Unknown legacy link times require this policy too. |

Publication dates must be within the tracker window: start inclusive, end exclusive. A historical claim cannot override dates, enrollment, ownership, prerequisites, or pause. Claims and reviewer retries decide admission under the current quest release. Once admitted, edits keep that release and beneficiary.

The server owns the completion key. Repeated discovery, metric refresh and account relinking cannot duplicate the initial reward for one quest and content item. Different quests can reward the same item independently. `ctx.tracker.content` contains the source's typed `body`, `revision` and `observation`; account attribution and enrollment remain on `ctx.tracker`. Budget timing uses `ctx.eligibleAt`, the record's original `occurredAt`.

Existing tracker records are not backfilled when a content quest is published. Pausing the quest blocks admission and new content evaluations. Removal does not reverse an existing award. Attempt retries keep their pinned content. Use a new quest key to change its tracker or switch between content and another trigger.

## Qualification after an edit

Relay queues a new evaluation when the captured body of an admitted post changes. Each evaluation keeps its own immutable body and observation, the release pinned at admission, and the original beneficiary. Metric and profile refreshes do not queue content evaluations. Reverting to an earlier body creates a fresh evaluation after an intervening evaluated revision; it does not reactivate an old review.

Before applying an evaluator or staff decision, Relay checks the current body and selected verified owner. A stale decision cannot award rewards. Disconnecting, relinking to another member, or removing the post blocks outstanding awards. Edits do not require another enrollment, claim, or staff admission approval. Pausing stops new evaluations; an already-running evaluation can finish if its content and ownership remain current.

Read the latest evaluation and retained history in a participant app:

```ts
import { createRelayClient } from "@domino-sdk/relay";

const relay = createRelayClient({ baseUrl: "/relay" });
const page = await relay.content.qualifications({ quest: "launch-post", content: "collected-x-post-id" });
const current = page.items.find(item => item.current);
if (page.nextCursor !== null) {
  const older = await relay.content.qualifications({ quest: "launch-post", content: "collected-x-post-id", cursor: page.nextCursor });
}
```

The API returns at most 100 evaluations, newest first. `content` is optional; omit it to read all of the signed-in member's posts for the quest. Continue with `cursor: page.nextCursor`, preserving any filters. Management uses `management.content.qualifications({ quest, member?, content?, cursor? })` with `read` permission. Omit `member` to read the quest's history across members. History remains visible to the original beneficiary after disconnect or relinking.

| Field | Meaning |
| --- | --- |
| `current` | Whether this is the latest evaluation for this quest and post. |
| `state` | `pending`, `qualified`, `unqualified`, `needs-review`, or `failed`; `superseded` means its decision was invalidated. The current evaluation can also be `stale`, `blocked`, or `unavailable`. |
| `reason` | Evaluation result or current reason the post cannot qualify. |
| `release`, `attempt`, `revision`, `observation` | Pinned policy and evidence references for this evaluation. |
| `reward` | One-time outcome: `unresolved`, `awarded` with a completion, or `excluded` after budget exhaustion. Its `attempt` identifies the evaluation that resolved the reward. |

Qualification and rewards are separate. The first eligible acceptance resolves the normal discrete reward. Later qualifying revisions retain that outcome without issuing more points, roles, referrals, or other completion effects. Budget exhaustion stays excluded even if budget becomes available later. An unqualified or unavailable post can still have an earlier award; automatic reversals are not implemented.

## Performance stage

A content quest's `performance` stage selects metrics declared by the source. X provides `likes`, `replies`, `reposts`, `quotes`, `views`, and `bookmarks`; a custom source provides its own names. The callback's metric type contains exactly that selection. It also receives `settings`, `member`, the pinned `tracker` with its content body, and `observedAt`. It has no managed review or provider capabilities.

Content approval queues performance evaluation without issuing rewards. Content review is reused until the body or verified account selection changes. After staff approve content, performance uses the latest eligible sample, including measurements collected during review.

Each performance attempt pins one complete sample. Unknown counters produce a waiting reason; a known zero reaches the callback. `maxAgeMs` defaults to one hour and accepts a positive integer up to 24 hours. Samples older than that limit or dated in the future wait for refresh. A newer sample, body edit, removal, or verified account change prevents an outstanding evaluator or staff decision from awarding against old evidence. Failed executions retain their sample through the existing automatic retry workflow.

Performance is a gate on the rewards selected by content evaluation or content review. A performance decision cannot replace that selection: it is `decision.accept()`, `decision.review({ reason })` or `decision.reject({ reason })`, and a result that carries `rewards` fails the evaluation. Both stages must accept before the first reward resolves. Budget timing uses the content record's occurrence time. Later threshold drops change current performance eligibility without reversing an award; later recoveries cannot award again or retry a budget exclusion.

### Read performance status and history

Use `content.performance` for one admitted post:

```ts
import { createRelayClient } from "@domino-sdk/relay";

const relay = createRelayClient({ baseUrl: "/relay" });
const query = { quest: "popular-launch-post", content: "collected-x-post-id" };
const page = await relay.content.performance(query);
const status = page.current;
if (page.nextCursor !== null) {
  const older = await relay.content.performance({ ...query, cursor: page.nextCursor });
}
```

`current` reports `content-pending`, `content-unqualified`, `waiting`, `pending`, `qualified`, `unqualified`, `needs-review`, `failed`, or `superseded`, with a reason. It identifies the content and performance attempts and retains the separate `reward` outcome. A waiting result may have no performance attempt yet. `items` contains up to 100 performance evaluations, newest first, with their pinned metric values, observation, sample time, content attempt, decision state, and reason.

Participant reads always use the session member. Management uses `management.content.performance({ quest, content, member?, cursor? })` with `read` permission. `content.qualifications` continues to return content evaluations only; its existing shape is unchanged. These APIs provide status and history. They do not trigger a claim or a new evaluation.

## Record data schemas

`customSource(namespace, { data })` requires a schema. `z` is zod, exported by `@domino-sdk/relay/authoring`. The schema does three things:

1. **Ingestion checks every record.** A record whose `data` does not match fails the whole batch with 422, and nothing in it is stored. The error body carries the stable `code` `invalid-record-data`, the `record` ID, the `path` that failed, such as `data.amount`, and a `reason`, such as `expected a string, got number`. A record marked `removed: true` is not checked: a removal retracts a record whatever its data. A refused batch records nothing, so you can send the corrected batch with the same action ID.
2. **Quests read typed data.** In a tracker-triggered quest's evaluator, `ctx.tracker.record.data` has the schema's type, and `trackers.records(tracker)` returns records with that type.
3. **Filters are checked.** A tracker trigger's `where` paths and values are [type-checked](#tracker-trigger-options) against it.

Relay stores the schema's JSON Schema form with the tracker, so it can check only what JSON Schema can express without running your code:

| Supported | Not supported |
| --- | --- |
| Objects (`z.object` accepts other keys, `z.strictObject` refuses them), `z.record`, arrays, strings with lengths, numbers and integers with bounds, booleans, `null`, literals, enums, unions, discriminated unions, intersections, `.optional()`, `.nullable()`, `z.unknown()`, and `.describe()` and `.meta()` titles, descriptions, defaults and examples | String formats and patterns (`z.email()`, `.regex()`), `.multipleOf()`, tuples, `.refine()`, `.transform()`, `z.never()`, `z.json()` (use `z.unknown()`), dates, bigints, other `.meta()` keys, a property named `__proto__`, and schemas reused by ID |

`customSource` throws when the schema uses something unsupported, naming the field. Check formats and custom rules in your evaluator instead. The schema describes data as your server sends it: a field with `.default()` stays optional, because Relay stores records unchanged.

Validation work is bounded, so one schema cannot stall ingestion:

- **Schema size.** A stored schema is at most 32 KB and 16 levels deep, and one object lists at most 256 properties, `required` keys, enum values or allowed key names.
- **Unions.** A union lists at most 16 branches. Inside a union's branch, a nested union can have at most one branch with properties, items, or a union of its own, such as `.nullable()` on an object; flatten anything wider into the outer union.
- **Steps.** Validation draws on a step budget per record and per batch, and every unit of work costs a step: each value checked against a schema, each key, `required` key and literal property, each enum lookup. A record that spends it fails the batch with 422 and the code `record-data-too-complex`. A 128 KB record of plain values uses about a quarter of a record's budget. Send fewer or smaller records per batch, or simplify the schema's unions. Trigger admission and `trackers.records(tracker)` apply the same budget; a record too costly to check is skipped or left out.

### Change a schema

Publish a new schema under the same tracker key. It applies to records ingested afterwards, and the deployment is refused with 422 when the new schema would break the `where` filter of a live quest on the tracker, including quests created in Console, naming each quest and filter. Change the filter in the same deployment, or keep the field.

Records stored before the change are not checked again at ingestion. Relay checks them when it uses them instead:

| Where | What happens to a stored record that no longer matches |
| --- | --- |
| Tracker trigger | The observation is skipped with `invalid-record-data`, and a staff retry fails the same way until the schema accepts its data. |
| Evaluator `ctx.tracker` | Rejected with a reason when the pinned data does not match the schema the quest was built with, so the evaluator never sees data of another type. |
| `trackers.records(tracker)` in an evaluator | Left out of `items` and counted in `excluded`. `total` and the cursor still count every record. Pass the `defineTracker` handle so the remaining items retain their declared data type. |

Keep stored records readable, such as by adding optional fields rather than changing a field's type.

A tracker published before record data has no schema. Ingestion refuses its records with 422 and the code `tracker-schema-required` until you publish it with `data`; its records stay readable.

## Source and identity kinds

- **Identities are checked at ingestion.** A record without an author, with a kind or role the source does not declare, with an account that leaves out its kind when the source declares several, or with an ID in another form is refused with 422, and nothing in the batch is stored.
- **The link is the verification.** `author.verified` is stored as `null`; the member's verified link is what attributes the account.
- **Reads return records as ingested.** Record reads return `author.kind` and `accounts` when the record named them; an account sent without its kind reads without it. `records(tracker, { member })` returns records whose author is one of that member's verified identities.
- **Read source kinds as an open set.** Published tracker definitions can contain a source added by a newer Relay. The response parser preserves an unknown source's `kind` and payload. Call `isKnownTrackerSource(source)` before reading provider-specific fields; `sourceIdentities` returns `undefined` and `sourceRoles` returns an empty list for unsupported sources. Authoring still accepts only supported source kinds. Console shows unsupported sources as "Tracker", lets you inspect records and the source definition, and leaves source controls to a newer Console.
- **Read identity kinds as an open set.** A published tracker's `source.identities` uses `OpenIdentityKind`, including kinds a newer Relay adds. Tracker status, lists, record pages, summaries, and deployment responses keep those kinds. Use `trackerDefinitionResponseSchema` to parse a published definition and `trackerDefinitionSchema` to validate a definition you author.
- **Changing the source needs a new key.** A published custom tracker's namespace, identities and accounts cannot change. Its [data schema](#record-data-schemas) can.
- **App identity kinds.** Name an app kind with `app:` and up to 40 lowercase letters, digits, `_` or `-`. Ingestion checks only that a record's kind is one the source declares; an ID is any string up to 256 characters.

## Tracker trigger options

`automatic({ tracker, on, where, beneficiary })` runs a quest for record observations, for the member behind one of the record's accounts:

| Option | Default | Meaning |
| --- | --- | --- |
| `on` | `"new-record"` for X, Discord and Telegram trackers, `"new-observation"` for other custom trackers | `"new-record"`: a record's first observation in this tracker. `"new-observation"`: every observation that becomes the record's current one (a new record, or a later `observedAt`, even with unchanged data). `"data-changed"`: a new record, or a current observation whose `data` differs from the record's previous one. |
| `where` | None | A condition on a dot-separated path in the record's `data`, or a list of conditions that must all hold: `{ path, equals }` with a string, number, or boolean, or `{ path, gt?, gte?, lt?, lte? }` comparing a number or an integer string, such as a base-unit balance, exactly. A bound is a number or `{ setting: "name" }`, one of the quest's integer or amount settings. |
| `beneficiary` | `"author"` | The role whose account the quest pays: `"author"`, `"inReplyTo"`, `"quoted"` or `"reposted"` on an X tracker, `"author"` or `"messageAuthor"` on a Discord reactions tracker, `"author"` or a role the custom source declares in `accounts`. To reward members an account picks up, pay `"reposted"` on a `from:<handle> filter:nativeretweets` tracker or `"quoted"` on `from:<handle> filter:quote`; see [Reward members when an account quotes or reposts them](https://console.domino.run/docs/build/trackers#reward-members-when-an-account-quotes-or-reposts-them). |

Observations the mode or filter does not select record nothing and admit no attempt. Older observations are kept as history and run nothing. `ingest` reports what one batch did in `admissions`: `admitted` and `skipped` count outcomes, and `filtered`, present only when nonzero, counts observations a quest's mode selected but its `where` did not match, per quest.

A `where` condition matches only when:

- **Every key of the path exists.** Each dot-separated key names a key of a JSON object; a missing key, or a key into an array or a scalar, never matches. `list.0` does not index arrays.
- **`equals` has the same type.** `{ path: "decimals", equals: 6 }` matches the number `6`, not the string `"6"`.
- **Compared values are numbers.** `gt`, `gte`, `lt`, and `lte` accept a JSON number or a string of digits with an optional leading `-` (up to 100 digits), compared exactly. Anything else, such as `"1.5"`, `"1e3"`, `true`, or `null`, never matches, whatever the bounds.
- **Every bound holds.** A `{ setting }` bound uses the release's value of that integer or amount setting. An amount setting (`settings.amount()`) holds a base-unit integer string, so a threshold such as 1 token with 18 decimals compares exactly. An amount setting never matches a fractional value, and an integer string never equals a fractional number bound.

A condition that never matches records no outcome, so check `filtered` after ingesting when a quest does not run.

TypeScript checks a filter against the tracker's data type, to five levels of nesting: a path must name a string, number, or boolean field (arrays end a path; a `z.record` field takes any key, and a `z.unknown()` field any path below it), `equals` must be a value that field can hold, a comparison needs a number or string field, and a `{ setting }` bound must name an integer or amount setting of the quest. A list holds at most 10 conditions. `automatic()` checks paths against the data schema at runtime too, and publication refuses with 422 a filter that reads a field the schema does not declare or compares a field that cannot hold the value.

| Rule | Behavior |
| --- | --- |
| Who it pays | The member behind the `beneficiary` account, through a verified link: the X post's author or the author of the post it replies to, quotes or reposts, a Discord message's author, a reaction's reactor or the author of the message reacted to, or a custom record's author or named account of any declared identity kind. A record never pays its author's member through another role: the observation is skipped when that account is the author's account, such as a reply to one's own post, or when both accounts belong to the same member, such as the same wallet on another chain or the member's X and Discord accounts. Quest code cannot choose another member. |
| What the evaluator gets | `ctx.tracker`, never null in a tracker-triggered quest's evaluator: `beneficiary` as the release names it (`"inReplyTo"` for reply quests, `"quoted"` or `"reposted"` for quote and repost quests), the pinned record (`id`, `data` with the tracker's data type, `occurredAt`, `observedAt`, `author`, `accounts`, …), `x` (for X trackers, the captured post `content` and its `evidence`; `null` for Discord and custom trackers), and `attribution` (`member`, and the `account` it paid through: `{ provider: "wallet", id, revision, linkedAt }`, `{ provider: "x" \| "discord", issuer, subject, revision, linkedAt }`, `{ provider: "member", id }`, or `{ provider: "app:<name>", subject, revision, linkedAt }`; later versions may add kinds). `eligibleAt` is the record's `occurredAt`: for a post, when it was published. |
| Who takes part, and from when | `enrollment: "required"` counts only members who enrolled in the quest (`relay.quests.enroll`, or `management.quests.enroll` from a server with `enroll` permission); `ctx.tracker.enrolledAt` is then the start of their current enrollment. `since` is `"activation"` (the default: records since the quest was published), `"window"` (every record in the tracker's window), or, with required enrollment, `"enrollment"`. `historical` says who releases activity from before the member linked the account or enrolled: `"staff"` (the default), `"claim"`, or `"automatic"`. See [Historical activity](https://console.domino.run/docs/concepts/trackers#historical-activity). |
| Completion | `once()`, `keyed(...)`, or `streak(...)`. `keyed` receives `ctx.tracker` to key each occurrence. Write `completion: ({ keyed }) => keyed(({ tracker }) => tracker.record.id)` for a typed, non-null `tracker`; an inline `keyed(...)` sees it nullable and untyped. `streak` counts the calendar day of each accepted observation's `occurredAt`; see [Reward consecutive days](https://console.domino.run/docs/build/trackers#reward-consecutive-days). After a `once()` quest completes, or a streak pays its milestone, later observations are skipped with `already-completed`. |
| Publication | The tracker must be a published X or Discord tracker, or a custom tracker with `identities`. The beneficiary must be a role the tracker's records have, a filter's paths fields of the tracker's data schema, and a filter's setting bound an integer or amount setting of the quest. Moving a quest to or from a tracker trigger, or to another tracker, needs a new quest key. |

With `on: "new-observation"`, every sample admits an attempt until a `once()` quest completes. Use `"data-changed"` and a `where` filter so unchanged or unqualifying samples cost nothing, or send samples when values change.

On a Discord tracker, record data has fixed types, so `where` paths such as `channelId` and `emoji.name` and `ctx.tracker.record.data` are type-checked. A `where` value is fixed in code; check a value operators choose in Console, such as a channel, in the evaluator.

## Codes and other secrets

Never put a secret, such as a code hidden at an event station, behind a text or URL quest ([text and URL submissions](verification.md#text-and-url-submissions)). Relay returns submitted `input` to the member who sent it ([reads that return submitted input](participant-api.md#reads-that-return-submitted-input)), and any signed-in member can submit through the hosted participant proxy (`handleHostedApp` forwards `POST /v1/me/attempts` and the submission routes), past any rate limit the app's own form applies. Relay has no option to hide `input` or refuse participant submissions for one quest.

Check the secret in the app's server and report what it proved to a custom tracker instead:

1. Declare `customSource(namespace, { identities: ["member"], data })` with a `data` field naming what was proven (a station key, never the code), and a quest with `trigger: automatic({ tracker, on: "new-record" })`, keyed per proven thing (`keyed(({ tracker }) => "station:" + tracker.record.data.station)`).
2. In a server route, `await signedInMember(request, env, { apiUrl })` from `@domino-sdk/relay/portal-proxy`; refuse a null member. Rate-limit guesses per member in the app (KV or your platform's limiter), then compare the code against an app secret.
3. On a match, `await hostedManagementClient(env).trackers.ingest(tracker, { actionId, records: [{ id: member + ":" + station, author: { id: member }, occurredAt, observedAt, data: { station } }] })`. The hosted app's key has `ingest`; the browser holds no key that can ingest. The stable record ID makes a repeated code one record, and `on: "new-record"` runs the quest once for it.

Ingestion never joins a member: a member who has not joined is skipped with `not-joined`. Report only members `signedInMember` returns (signing in to the app joins them). `packages/sdk/examples/station-code-quest.ts` has the tracker, a record helper and the quest; the [first quest guide](https://console.domino.run/docs/build/first-quest#codes-and-other-secrets) shows the Worker route.

## Trigger outcome codes

Each observation a quest's mode and filter select is recorded as exactly one outcome: **Admitted**, with its attempt, or **Skipped**, with the first rule that failed, as a stable `code` and a `reason` for people. Rules apply in this order:

| Code | Example reason | What to do |
| --- | --- | --- |
| `quest-paused` | Quest is paused | Resume the quest; later observations run. Retry this one if it should count. |
| `before-activation` | Occurred before the quest was published | Nothing; with the default `since`, publishing never backfills. |
| `outside-window` | Occurred outside the tracker's window | Read `window`: `"tracker"` means the tracker's collection window refused the record; `"quest"` means the quest or campaign availability window refused it. Nothing is retried automatically. |
| `future-record` | Occurred later than the decision | Nothing. |
| `record-removed` | Record is removed | Nothing; removed records never run a quest. |
| `invalid-record-data` | Record data does not match the tracker's data schema: data.amount expected a string, got number | The record was stored before the tracker's schema changed. Retry it once the schema accepts its data. |
| `content-unavailable` | Captured post is unavailable | Nothing; the post's captured body could not be read. |
| `no-account` | Post is not a reply; Discord did not name the message's author; Record has no recipient account | Nothing; the record has no account in the role the quest pays. |
| `beneficiary-is-author` | Reply is to the author's own post; Reaction is to the reactor's own message; The recipient account belongs to the record's author | Nothing; a record never pays its author's member through another role. An account other than the author's own is judged once it resolves to a member, after `unlinked-account` and `ambiguous-account`. |
| `unlinked-account` | No member has linked this wallet; No member has linked this Discord account | Ask the member to verify the account. Linking it decides the skip again. |
| `ambiguous-account` | Linked account is ambiguous | More than one member selected the account. Once one of them removes it, the skip is decided again. |
| `not-joined` | Member has not joined this project | The member signs in to your app once. Joining decides the skip again. |
| `member-banned` | Member is banned | Nothing; an operator banned the member (`members.ban`). An unban counts only what the member does afterwards, so the skip is never decided again. |
| `already-completed` | Member already completed this quest | Nothing; a `once()` quest completes once, and a streak pays one milestone. |
| `not-enrolled` | Member has not enrolled in this quest, or in the quest's campaign | The member enrolls in what `enrollment` names (`quest` or `campaign`). Enrolling decides the skip again. |
| `before-enrollment` | Occurred before the member enrolled | Nothing; the quest counts from enrollment. |
| `historical` | Occurred before the member linked this account; a reviewer can release it | Retry it to release it, or, under `historical: "claim"`, the member claims it. |
| `prerequisites-incomplete` | Prerequisites are incomplete | A required quest was not complete when the record occurred, or the member was not in a required group when it was decided. Adding them to the group later does not decide it again; retry it if it should count. |

Branch on `code`, not on `reason`: reasons are worded for people and name the account kind or role. For `outside-window`, use the optional `window` field to distinguish the tracker window from the quest or campaign window. Older Relay responses omit it. A quest budget is not a skip: the observation is admitted, and its attempt is rejected with `budget-exhausted` when the budget cannot cover another completion. Later versions may add codes; show an unknown one by its `reason`. A skip recorded before Relay stored skip codes, whose reason matches no rule, has the code `legacy`; read its `reason`, and retry it if it should count.

`management.trackers.triggerOutcomes(tracker, query)` filters by `quest`, `record`, `member`, `state`, and skip `code`, returns up to `limit` (default 25, at most 100) newest first, and pages with `cursor: page.nextCursor`. It needs `read` permission. `management.trackers.retryOutcome(tracker, { actionId, quest, record, observedAt })` needs `review`. Retrying an admitted outcome returns it unchanged. A retry that still cannot admit fails with 409: the current reason as `error` and its skip code as `code`, such as `already-completed`. A record that no longer matches the quest's filter fails with "The record no longer matches the quest's filter".

## Discord record fields

| Field | `discord.messages()` | `discord.reactions()` |
| --- | --- | --- |
| `id` | The message ID | `messageId:reactorId:emoji`, where `emoji` is a custom emoji's ID or the Unicode emoji itself |
| `author` | The poster's Discord user ID | The reactor's Discord user ID |
| `accounts` | None | `messageAuthor`: the Discord user ID of the message's author. Absent when Discord could not name it. |
| `occurredAt` | When the message was posted, or when Relay received it, whichever is earlier | When Relay received the reaction. Discord sends no reaction time. |
| `data` | `guildId`, `channelId`, `parentChannelId`, `messageId`, `replyTo` (the ID of the message it replies to, or `null`) and `publishedAt`, plus `text` with `text: true` | `guildId`, `channelId`, `parentChannelId`, `messageId`, `emoji: { id, name }`, `reactorRoles` (the reactor's role IDs when reacting) and `messagePublishedAt` |

- **Threads name their channel.** In a thread, `channelId` is the thread's own ID and `parentChannelId` the channel it belongs to (a forum post's is the forum). Outside a thread `parentChannelId` is `null`. It is absent, meaning unknown, when Discord's channel read failed and on records stored before Relay recorded it. A `where` on `parentChannelId` selects a channel's threads only; to count a channel and its threads, check both fields in the evaluator.
- **Message changes are observations.** An edit updates `data.text` on a tracker with `text: true`. A `new-observation` or `data-changed` trigger can select the edit and apply its `where` filter again; the default `new-record` mode does not. Deleting a message marks its record removed, records a `record-removed` outcome for each triggered quest, and erases its stored text. Reaction removals change nothing. Adding a removed reaction again is a new observation of the same record.
- **Every tracker of a kind gets the record.** Each Discord tracker of that kind receives it, in every project and environment whose Discord connection names the server and was made before the activity, when a member of that project linked one of its accounts. A paused tracker, or activity outside its window, stores nothing.
- **Text is erased when the option is dropped.** Publishing a `discord.messages()` tracker without `text: true` erases text from its current records, retained observations, and pinned attempts.
- **No ingestion or refresh.** `trackers.ingest` answers 403 for a Discord tracker, and a refresh answers 422.

## Engagement accounting

Publish an independent engagement rule with its content quest and points balance, or reference resources you already published. The quest controls current eligibility; its one-time reward remains separate.

An engagement rule's formula (`target`) is cumulative. Return a nonnegative safe whole number synchronously. The formula receives only the declared counters. It runs in an isolated sandbox without network or member capabilities; clocks and randomness are forbidden. `baseline: "historical"` counts eligible historical engagement using a zero-counter baseline. Growth-only enrollment baselines are not supported yet.

A manifest accepts at most 50 engagement entries. Paths are relative to the manifest; use `exportName` for a named export. `domino build` and `deploy --dry-run` bundle locally without running the authored module or contacting Relay. Engagement files require explicit entries; quest discovery globs do not select them. The rule's initial discovery cutoff is the actual publication time, not the earlier preview time. Removing an entry from the manifest does not delete its published rule.

Retain the deployment ID to retry publication. Each formula version is immutable. Deployment previews label an existing rule as unchanged or changed and carry any accounting adjustment for review. The older engagement-only publication methods remain supported but are deprecated; use deployments for new code. Publishing another key creates an independent rule and can pay separately for future discoveries; it does not replace an existing rule.

| Field | Meaning |
| --- | --- |
| `earned` | Highest eligible cumulative target, including previously earned points across a pause. |
| `accounted` | Growth already settled as paid or excluded. |
| `paid` | Points credited by this rule. |
| `excluded` | Growth permanently excluded by this rule's budgets. |
| `pending` | `earned - accounted`, awaiting an eligible daily settlement. |
| `counters` | Component-wise highest eligible measured counters used for scoring. |
| `baseline` | Counters subtracted from the formula at the current accounting boundary. |

The worker checks due accounts about once a minute. It processes the latest eligible sample; it does not replay older observations to reconstruct peaks during review or worker downtime. It retains eligible peaks between settlements: 100, 130, 120, then 135 likes produce at most 135 rewarded likes. Reports and performance gates continue to use actual sampled counters. Missing counters are not zero. The rule waits for every declared metric and a fresh observation; `maxAgeMs` defaults to one hour and may be at most 24 hours.

Days are those of the [project calendar](https://console.domino.run/docs/concepts/calendar), UTC unless you choose a project timezone. The first settlement becomes due at the next midnight after the first eligible sample. The daily settlement then runs at most once per calendar day, when fresh evidence and current qualification are available. Provider observations are not exact midnight measurements. Delayed work uses the observation and calendar day when it settles; it does not reconstruct missed daily buckets.

Points are dated by measurement, not by payment. Each settlement's point entries take effect when the sample that measured their growth was observed, so growth measured before midnight counts on that day even though it is paid after. When the growth between two samples spans the start or end of a [leaderboard](https://console.domino.run/docs/concepts/leaderboards) window over the rule's balance, or the boundary of one of the rule's budget periods, Relay splits it there in proportion to the time on each side, rounding down and giving the remainder to the later part. Each part is its own point entry, so no entry straddles a boundary. A settling leaderboard over engagement points also takes a [closing measurement](https://console.domino.run/docs/concepts/leaderboards#closing-measurement) at its cutoff and settles it off the daily schedule.

Budgets are independent per rule and project/environment. Each entry in `budgets` caps the points paid for the whole project (`scope: "project"`) or for each member (`scope: "member"`), per `day`, `week` (from Monday) or `month` of the project calendar, or over the rule's `lifetime`. Each point entry is charged to the period it takes effect in, which is when its growth was measured. Give each scope and period at most one budget. Without a budget, credits are unlimited, and a limit of zero excludes all growth. A partial budget pays the available amount and records the rest as excluded. Excluded growth never carries into tomorrow. The point entries, budget spend, account, and settlement receipt commit together. Engagement points credit the selected balance directly without another completion. They count as earnings of the qualifying quest for [referral bonuses](https://console.domino.run/docs/build/referrals#percentage-bonuses): a percentage bonus in the same ledger that covers that quest, or names no quests, accrues for the member's inviter on the paid amount in the same commit, subject to the bonus's limits.

Pausing the qualifying quest stops accrual and settlement. On resume, the first fresh eligible sample establishes a new baseline: already earned pending points remain, while growth through that observed boundary is skipped. Deleted content, changed ownership, pending review, and failed qualification block further accounting until resolved. Already credited points remain; automated reversals and restorations are not implemented.

### Read settlement history

```ts
import { createRelayClient } from "@domino-sdk/relay";

const relay = createRelayClient({ baseUrl: "https://relay.domino.run" });
const page = await relay.engagement.page({ rule: "launch-growth", content: "123456789" });
const pending = page.account?.pending ?? 0;
if (page.nextCursor !== null) {
  const older = await relay.engagement.page({
    rule: "launch-growth", content: "123456789", cursor: page.nextCursor,
  });
}
```

Participant history belongs to the originally admitted member and requires their session. Management can use `engagement.list()` and `engagement.page({ rule, content, member?, cursor? })` with `read` permission. Each page includes the rule, its last processed account checkpoint, and up to 100 settlements, newest first, including formula-boundary settlements. Receipts identify the observation, qualification attempt, growth, paid/excluded amounts, UTC day, and point entry: the latest one, when a boundary split the growth into several. An absent management account is `null`; another member's account is not visible to participants.

Explicit historical backfill, enrollment baselines, captured quality multipliers, automated corrections, and dedicated Console controls remain unavailable.

## Formula changes

A formula change keeps the same rule key, qualifying quest, balance, declared metrics, baseline policy, freshness policy, and budgets. An unchanged bundle keeps the existing version.

If the requested version, admission set, or accounting checkpoint changes before publication, the whole deployment returns `409` and publishes nothing. Create a fresh preview. If the publish response is lost, resume the same deployment ID to recover its committed receipt without requesting another version.

The preview reports known admissions, existing accounts, uninitialized accounts, pending transitions, and current earned/paid/excluded/pending totals. Totals are decimal strings so a project sum cannot overflow a safe JavaScript integer. The preview states `settlement: "deferred"`. Its totals describe retained accounting; they are not a promise of the next payout. Applying the change requests a version and does not itself credit points or reverse an award.

### The observed boundary

The supported boundary is `available-eligible-observation`. The worker uses the fresh, complete, qualified observation available when that post can transition. That observation may predate the request by up to the rule's `maxAgeMs`. This is an observed boundary, not a measurement at the exact publication time. Missing metrics, pending evaluation/review, paused quests, changed ownership, and scoring failures keep the post on its current version.

At the boundary, Relay:

1. Calculates the old formula through the component-wise highest eligible counters, retaining any prior earned peak.
2. Settles its pending growth under the existing budgets, recording paid and permanently excluded amounts.
3. Starts the new formula at those same counters, retaining the old earned/accounted totals.

These steps commit together. Formula boundaries can create an additional settlement during a calendar day; they use that day's existing budget spend. Daily settlement resumes at the next midnight of the project calendar. Neither caps nor excluded growth reset across versions.

| Change | Result after the boundary |
| --- | --- |
| Rate changes from one to two at 100 likes; next sample is 110. | Earn 20 new points. |
| Likes previously reached 100 but now read 80; then recover to 100. | No reward for recovery; the baseline retains 100. |
| A target capped at 100 changes to a cap of 200 when counters are 150. | No top-up for the old 150; 160 earns 10 new points. |

A later request supersedes versions that a post has not started. The worker transitions that post directly from its current version to the latest requested version; it never fabricates intermediate earnings. A stale in-flight score cannot apply a superseded request. Existing applied versions and receipts remain immutable.

New discoveries start on the version requested at discovery. A post discovered earlier retains its discovery-time formula even if admission or account initialization happens later, then transitions through an eligible boundary. The original rule publication cutoff remains unchanged, so formula changes do not backfill old content.

### Follow a pending transition

```ts
import { createRelayClient } from "@domino-sdk/relay";

const relay = createRelayClient({ baseUrl: "https://relay.domino.run" });
const history = await relay.engagement.boundaries({
  rule: "launch-growth", content: "123456789",
});
const waiting = history.current?.version !== history.requested.version;
if (history.nextCursor !== null) {
  const older = await relay.engagement.boundaries({
    rule: "launch-growth", content: "123456789", cursor: history.nextCursor,
  });
}
```

`current` identifies that account's governing formula, and `requested` identifies the latest requested formula. Each boundary item has `from`, `to`, the high-water `baseline`, the provider's `observedAt`, and its settlement receipt. Compare `observedAt` with `settlement.createdAt` to distinguish measurement time from application time. Participant reads require the original beneficiary's session.

Management can use the same `engagement.boundaries({ rule, content, member?, cursor? })` method and `engagement.ruleVersions(rule, { cursor? })` with `read` permission. Both histories return at most 100 items, newest first; pass a page's `nextCursor` back as `cursor` for the next. `engagement.list()` shows requested versions; `engagement.page()` shows the account's governing version and retains its existing response shape.

If a requested formula fails, a corrected request can supersede it before the account switches. If the current formula cannot score its old growth, the boundary stays pending; there is no forced skip or historical rewrite. Changing budgets or metrics, global exact-time cutovers, and fixed future-payout previews are not supported.
