# Trackers reference

The contract behind [Track posts and activity](https://console.domino.run/docs/build/trackers): collection options, record reads and attribution, the rules a tracker trigger applies, its outcome codes, and engagement accounting. The guide has the tasks and examples; this page has the detail they rely on.

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

A tracker created without a deployment (`trackers.create`, `domino trackers create`, or Console **New X tracker**) takes the same fields and limits as `defineTracker`, with `query` in place of `source`. `refreshSeconds` and `maxPages` take the defaults above. The first collection searches from `window.start`, and the answer is the tracker's status, with `origin: "data"`. A title, `refreshSeconds` or `maxPages` change leaves discovery where it was.

## Code or data owns each key

A key belongs either to code or to data, and only its owner changes the definition. Tracker status shows it as `origin`: `code` for a tracker a deployment publishes, `data` for one created in Console or with `trackers.create`.

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
| `on` | `"new-record"` for X and Discord trackers, `"new-observation"` for custom trackers | `"new-record"`: a record's first observation in this tracker. `"new-observation"`: every observation that becomes the record's current one (a new record, or a later `observedAt`, even with unchanged data). `"data-changed"`: a new record, or a current observation whose `data` differs from the record's previous one. |
| `where` | None | A condition on a dot-separated path in the record's `data`, or a list of conditions that must all hold: `{ path, equals }` with a string, number, or boolean, or `{ path, gt?, gte?, lt?, lte? }` comparing a number or an integer string, such as a base-unit balance, exactly. A bound is a number or `{ setting: "name" }`, one of the quest's integer or amount settings. |
| `beneficiary` | `"author"` | The role whose account the quest pays: `"author"` or `"inReplyTo"` on an X tracker, `"author"` or `"messageAuthor"` on a Discord reactions tracker, `"author"` or a role the custom source declares in `accounts`. |

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
| Who it pays | The member behind the `beneficiary` account, through a verified link: the X post's author or the author of the post it replies to, a Discord message's author, a reaction's reactor or the author of the message reacted to, or a custom record's author or named account of any declared identity kind. A record never pays its author's member through another role: the observation is skipped when that account is the author's account, such as a reply to one's own post, or when both accounts belong to the same member, such as the same wallet on another chain or the member's X and Discord accounts. Quest code cannot choose another member. |
| What the evaluator gets | `ctx.tracker`, never null in a tracker-triggered quest's evaluator: `beneficiary` as the release names it (`"inReplyTo"` for reply quests), the pinned record (`id`, `data` with the tracker's data type, `occurredAt`, `observedAt`, `author`, `accounts`, …), `x` (for X trackers, the captured post `content` and its `evidence`; `null` for Discord and custom trackers), and `attribution` (`member`, and the `account` it paid through: `{ provider: "wallet", id, revision, linkedAt }`, `{ provider: "x" \| "discord", issuer, subject, revision, linkedAt }`, `{ provider: "member", id }`, or `{ provider: "app:<name>", subject, revision, linkedAt }`; later versions may add kinds). `eligibleAt` is the record's `occurredAt`: for a post, when it was published. |
| Who takes part, and from when | `enrollment: "required"` counts only members who enrolled in the quest (`relay.quests.enroll`, or `management.quests.enroll` from a server with `enroll` permission); `ctx.tracker.enrolledAt` is then the start of their current enrollment. `since` is `"activation"` (the default: records since the quest was published), `"window"` (every record in the tracker's window), or, with required enrollment, `"enrollment"`. `historical` says who releases activity from before the member linked the account or enrolled: `"staff"` (the default), `"claim"`, or `"automatic"`. See [Historical activity](https://console.domino.run/docs/concepts/trackers#historical-activity). |
| Completion | `once()`, `keyed(...)`, or `streak(...)`. `keyed` receives `ctx.tracker` to key each occurrence. Write `completion: ({ keyed }) => keyed(({ tracker }) => tracker.record.id)` for a typed, non-null `tracker`; an inline `keyed(...)` sees it nullable and untyped. `streak` counts the calendar day of each accepted observation's `occurredAt`; see [Reward consecutive days](https://console.domino.run/docs/build/trackers#reward-consecutive-days). After a `once()` quest completes, or a streak pays its milestone, later observations are skipped with `already-completed`. |
| Publication | The tracker must be a published X or Discord tracker, or a custom tracker with `identities`. The beneficiary must be a role the tracker's records have, a filter's paths fields of the tracker's data schema, and a filter's setting bound an integer or amount setting of the quest. Moving a quest to or from a tracker trigger, or to another tracker, needs a new quest key. |

With `on: "new-observation"`, every sample admits an attempt until a `once()` quest completes. Use `"data-changed"` and a `where` filter so unchanged or unqualifying samples cost nothing, or send samples when values change.

On a Discord tracker, record data has fixed types, so `where` paths such as `channelId` and `emoji.name` and `ctx.tracker.record.data` are type-checked. A `where` value is fixed in code; check a value operators choose in Console, such as a channel, in the evaluator.

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
| `already-completed` | Member already completed this quest | Nothing; a `once()` quest completes once, and a streak pays one milestone. |
| `not-enrolled` | Member has not enrolled in this quest | The member enrolls. Enrolling decides the skip again. |
| `before-enrollment` | Occurred before the member enrolled | Nothing; the quest counts from enrollment. |
| `historical` | Occurred before the member linked this account; a reviewer can release it | Retry it to release it, or, under `historical: "claim"`, the member claims it. |
| `prerequisites-incomplete` | Prerequisites are incomplete | A required quest was not complete when the record occurred, or the member was not in a required group when it was decided. Adding them to the group later does not decide it again; retry it if it should count. |

Branch on `code`, not on `reason`: reasons are worded for people and name the account kind or role. For `outside-window`, use the optional `window` field to distinguish the tracker window from the quest or campaign window. Older Relay responses omit it. Later versions may add codes; show an unknown one by its `reason`. A skip recorded before Relay stored skip codes, whose reason matches no rule, has the code `legacy`; read its `reason`, and retry it if it should count.

`management.trackers.triggerOutcomes(tracker, query)` filters by `quest`, `record`, `member`, `state`, and skip `code`, returns up to `limit` (default 25, at most 100) newest first, and pages with `cursor: page.nextCursor`. It needs `read` permission. `management.trackers.retryOutcome(tracker, { actionId, quest, record, observedAt })` needs `review`. Retrying an admitted outcome returns it unchanged. A retry that still cannot admit fails with 409: the current reason as `error` and its skip code as `code`, such as `already-completed`. A record that no longer matches the quest's filter fails with "The record no longer matches the quest's filter".

## Discord record fields

| Field | `discord.messages()` | `discord.reactions()` |
| --- | --- | --- |
| `id` | The message ID | `messageId:reactorId:emoji`, where `emoji` is a custom emoji's ID or the Unicode emoji itself |
| `author` | The poster's Discord user ID | The reactor's Discord user ID |
| `accounts` | None | `messageAuthor`: the Discord user ID of the message's author. Absent when Discord could not name it. |
| `occurredAt` | When the message was posted, or when Relay received it, whichever is earlier | When Relay received the reaction. Discord sends no reaction time. |
| `data` | `guildId`, `channelId`, `messageId`, `replyTo` (the ID of the message it replies to, or `null`) and `publishedAt`, plus `text` with `text: true` | `guildId`, `channelId`, `messageId`, `emoji: { id, name }`, `reactorRoles` (the reactor's role IDs when reacting) and `messagePublishedAt` |

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
