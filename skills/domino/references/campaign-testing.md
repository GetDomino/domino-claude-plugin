# Test a campaign with test members

Simulate the campaign the way its participants will live it: several members, several days, through the real ledger in the test environment. Then fix what breaks and run the scenario again. A unit test of one evaluator (`runQuest`) proves one decision; a test-member scenario proves the campaign: enrollment, prerequisites, availability, streaks, budgets, review and rewards working together over time.

Work in the test environment with a `domino login` CLI session, a personal API key, or an organization automation key. The `domino test-members` commands always address test. Each step needs its own permission. An organization admin holds all of them; a project role holds only its own:

| Step | Permission | Role that holds it |
| --- | --- | --- |
| `domino test-members …` (create, act, pass, accept-anyway, advance, link-wallet, unlink-wallet, link-account, unlink-account, record, history, reset) | `publish` | editor |
| `domino groups add` | `publish` | editor |
| `domino points award` | `points` | editor |
| `domino quests enroll` | `enroll` | editor |
| `domino trackers ingest` | `ingest` | editor |
| Confirm a staff-confirmed quest | `confirm` | staff |
| Decide an attempt awaiting review | `review` | reviewer |

A personal API key carries only the permissions chosen when it was created. Before you start, check which steps your campaign needs. A refusal with code `permission-required` means the credential lacks that step's permission; ask the operator for it. Method details, limits and reset rules are in [test members](authoring.md#test-members) and the public guide, [Try quests with test members](https://console.domino.run/docs/use/quests#try-quests-with-test-members).

## The loop

Work through these steps in order. Each ends on its completion criterion.

### 1. Map the campaign

Read the published quests (`domino quests list --json`, `domino quests get QUEST --json`) and the quest source. For each quest, write down its trigger and input, completion rule (once, daily, streak, count), prerequisites, availability window, enrollment requirement, budget, review switch and rewards. Then list the thresholds and the data each evaluator reads.

Done when every quest has a row and every number in the code (a minimum, a limit, a day count, a follower count, a balance) is on your list.

### 2. Cast the members

Create one test member per persona the campaign must handle, not one per quest. A persona is a path through the campaign: the member who does everything on day one, the one who misses a day, the one exactly at each threshold, the one just below it, the one with odd input. Name members after their persona so the history reads like a story:

```sh
domino test-members create "Streak-breaker" --json
domino test-members create "Exactly 100" --json
domino test-members create "ゆき 🌸" --json
```

Names take 1–40 UTF-16 code units after trimming, so an emoji counts as two. Keep each member's `id` from the result.

Done when every threshold on your list has a member on each side of it, and every quest has at least one member expected to pass and one expected to fail.

### 3. Run the days

Act as each member, then move their clock. Each test member has their own clock: advancing one leaves the other test members where they are, and real members and live on real time. `QUEST` is the quest ID from `domino quests list`.

```sh
domino test-members act MEMBER QUEST --json                                    # claim
domino test-members act MEMBER QUEST --input '{"kind":"text","value":"gm"}' --json
domino test-members act MEMBER QUEST --wait --json                             # the settled attempt: status, reason, code
domino test-members advance MEMBER --days 1 --json
```

`act` answers the queued attempt. With `--wait` it reads the attempt until it is no longer queued, evaluating or deferred, and prints it with its `status`, `reason` and `code`; after `--timeout` seconds (default 60) it prints the attempt as it stands. Reading the attempt also needs `read`, which a personal API key may not carry.

Give a member the wallet a holding or balance quest reads with `domino test-members link-wallet MEMBER --chain solana|eip155|cosmos --address ADDRESS` (Cosmos also takes `--network cosmos:<chain ID>`). Set up the rest of what a quest gates on with the ordinary operator commands, which accept a test member's ID: `domino quests enroll QUEST MEMBER`, `domino points award MEMBER LEDGER --amount N --reason …` for a points-balance prerequisite, `domino groups add GROUP MEMBER --reason …` for a group prerequisite. For custom trackers whose source declares `identities: ["member"]`, send records with `domino trackers ingest TRACKER FILE`, naming the member as `"author": { "id": "MEMBER" }` and keeping real `occurredAt` and `observedAt`; Relay projects them onto that member's clock. For managed X and Discord trackers, send the member's activity with `domino test-members record` (see [X and Discord activity](#x-and-discord-activity)). There is no CLI command for staff confirmation or review; use `domino api` or the management client:

```sh
# confirm: body { "actionId", "quest", "member" } (management.attempts.confirm)
domino api POST /attempts confirm.json --json
# review: body { "actionId", "generation", "decision": "accept" | "reject", "reason" } (management.attempts.review)
domino api POST /attempts/ATTEMPT/review review.json --json
```

Take `generation` from `domino attempts get ATTEMPT --json`. Use a new `actionId` for each new decision, and the same one only to retry it.

To get a persona past a quest whose check you can't satisfy in test (a follow on X, an on-chain balance, a quiz) and on to the quests behind it, pass it: `domino test-members pass MEMBER QUEST [--completion-key KEY] --json` admits an attempt as `act` would, then accepts it as you without running the check. Prerequisites, availability, enrollment, a waiting attempt and the retry cooldown still refuse it, and the completion policy, budgets and rewards still apply: a progress quest counts one step, a streak quest today's action, and a keyed quest without `--completion-key` gets a fresh key. Automatic quests can't be passed; send the record that starts a tracker-triggered quest, and if its check rejects the attempt, `domino test-members accept-anyway MEMBER ATTEMPT --json` accepts that rejected attempt as you. Both answer the decided attempt. Use them only for checks you aren't testing: a passed quest proves nothing about its own check.

Submission returns a queued attempt; evaluation continues in the background. Read the outcome before acting again.

Done when every persona has walked its whole path, including the days it is meant to skip.

### 4. Read what happened

```sh
domino test-members history MEMBER --json    # attempts with eligibleAt, skipped tracker outcomes with code
domino members get MEMBER --json             # progress, completions, balances, enrollments
domino members progress MEMBER --json        # quest progress, attempts, balances; no quest code
domino attempts get ATTEMPT --json           # status, reason, code, progress
domino trackers outcomes TRACKER --json      # why a record admitted no attempt
```

Compare each persona against what the campaign promises. An outcome is a bug when it differs from the brief, even if the code did exactly what it says.

Done when every attempt and skipped outcome has an expected/actual verdict, and every balance matches a sum you computed by hand.

### 5. Fix, then prove the fix

Fix the quest or the campaign definition locally. Before publishing, replay the attempts the bug touched: `domino quests replay ATTEMPT_ID --environment test --json` runs your local quest code against a captured evaluation, so you see which past outcomes the fix changes before it reaches anyone. Replay a few attempts that went right, too. Then publish to test, create fresh members, and run the failing persona again. A member cannot go back in time, so start each re-run from a new member.

`domino test-members reset MEMBER` deletes a member. It refuses with 409 `test-member-reset-blocked`, naming the blocking `state`, while an attempt is queued, evaluating or deferred, and when the member touched shared or external state: a shared project budget, inventory, redemptions or a reward handover, reward or webhook deliveries, referral relationships, a leaderboard settlement or an engagement account. A successful reset also frees the member's wallet addresses and unlinks the accounts it connected while a staging site acted as it. A refused reset changes nothing; wait for evaluations to finish, or keep the member and create another.

Done when the failing persona passes on a fresh member, the replays change only the outcomes you meant to change, and the other personas still pass.

## What to watch for

These are the places campaigns break. Treat each as a prompt to design a persona, not as a checklist to tick: the right cases depend on what this campaign reads and promises.

### Odd handles and characters

Anything a member types or a provider returns reaches your evaluator as a string. Try names, text and handles with emoji, combining marks, right-to-left scripts, zero-width characters, leading `@`, mixed case, surrounding spaces and the maximum length.

- Text limits count UTF-16 code units, as JavaScript's `.length` does, so one emoji can use two of a `maxLength`. A text input that is only whitespace counts as empty.
- URL inputs must be `http` or `https` with no spaces or credentials. `allowedHosts` matches the hostname exactly and case-insensitively: `x.com` does not admit `www.x.com` or `twitter.com`.
- An evaluator comparing handles or keywords should decide on case, Unicode normalization and a leading `@` on purpose. X `username` arrives without `@`; members often type it with one.
- Display names render in Console, leaderboards and your app. Check that a 40-character name and a right-to-left name lay out without breaking the page.

### Thresholds and boundaries

For every number on your list, test exactly at it, one below and one above. Off-by-one shows up as `>` where the brief means "at least", or as a count that includes or excludes the current attempt.

- Follower and engagement thresholds: X `followersCount` and post metrics (`likeCount`, `viewCount` and the rest) can be `null` when the provider does not report them. Decide whether `null` means fail, review or pass, and test it.
- Count and streak completions: the attempt that reaches the target, the one after it, and a member who stops one short.
- Budgets and daily limits: the last reward a budget can pay and the first it refuses. Project budgets are charged in the period of the member's clock.

### Missing and partial data

Evaluators break on the record they did not expect, not on the happy one.

- X posts: `post()` resolves to `null` for a deleted or missing post. Replies, reposts and quotes carry `isReply`, `isRetweet`, `isQuoteTweet` and the `inReplyTo…`, `retweetedTweetId` and `quotedTweetId` IDs; `media` is empty when the provider's list is malformed. A repost's text is not the member's own words.
- X account reads (`x.connection()`, `profile()`, `follows()`, `reposted()`) resolve to `null` when the member has no linked X account. A test member has none until you link one with `test-members link-account` or its first `test-members record` into an X tracker links a synthetic one, so test the no-account branch with a member that has neither.
- Custom tracker records: send records with optional fields missing, empty strings, zero and negative amounts, a `removed` record, and the same record ID twice. The tracker's schema refuses some of these at ingest; the evaluator must handle the rest.
- Provider failures throw `CapabilityError` with code `provider-unavailable`. Relay retries the evaluation; a quest that treats the failure as a rejection punishes the member for an outage.

### Days, streaks and time

Time is where campaigns look right on day one and break on day three.

- Run streaks across a missed day and two submissions on one day. Advance with `--days 1` between days, not by waiting. Advancing keeps the member's time of day, so it cannot place an action just before or after midnight; test that edge in the evaluator with `runQuest`'s `now` and `timeZone`.
- Daily quests: act, advance one day, act again; then act twice without advancing and expect the second to be refused or to change nothing.
- Availability windows and campaign dates: advance a member past the end and confirm the quest closes; start one before a quest opens.
- The project calendar's time zone decides where a day ends. A campaign announced in one time zone and configured in another shifts every deadline.
- Deferred evaluations and outbox retries wait in real time. Advancing a member does not wake a deferred attempt early; read the attempt's `deferUntil` and wait.
- Each member can advance at most 90 days in total. Plan long campaigns as several shorter personas.

### Referrals

A test member accepts an invitation with `domino test-members accept-referral MEMBER (--code CODE | --inviter MEMBER)`, as a signed-in member accepting it would. `--inviter` names another test member and uses its invitation code, allocating one if it has none. The same refusals apply: `self-referral`, `inviter-already-accepted` for a second inviter, `invitation-closed` once the invitee has completed a quest, `referral-cycle` and `invitation-not-found`. Accepting the same inviter again changes nothing. A `signup` program qualifies the invitee on acceptance and pays both sides; a `questCompleted` program qualifies when the invitee completes that quest.

A run for an inviter, two invitees and a quest-qualified program:

1. Create three members. Have the invitees accept: `domino test-members accept-referral INVITEE --inviter INVITER`. Accept before any of the invitee's quest completions, or `invitation-closed` refuses it.
2. Pass or act on the qualifying quest as each invitee, advancing days between them when the program's timing matters.
3. Read the result: `domino referral-relationships list` for qualifications, `domino referral-relationships history INVITER` for the inviter's referral rewards, and `domino points get` for each balance. Check milestones at their exact counts.
4. Try the refusals: the inviter's own code, a second inviter, and an invitee who already completed a quest.

Once a test member has accepted an inviter or been accepted as one, `reset` refuses it with 409 `test-member-reset-blocked` (`referrals affect another member`), so create fresh members for each referral run. Check the referral UI itself on the staging site, signed in as a test member.

### Balance-gated quests

- Points-balance prerequisites (`balanceAtLeast`) and group prerequisites are testable now: award points or add the member to the group, then act. Test the member exactly at the minimum, one below, and one who drops below after spending.
- Token and wallet balances: link an address with `domino test-members link-wallet`, and quest code reads it as it reads a wallet a member linked. `ctx.wallet.list()` returns it, and `ctx.wallet.balance()` reads its real on-chain balance. So pick addresses whose holdings put each persona where you want them: at the threshold, just below it, holding on another chain, holding nothing. Give one member several addresses, up to five per chain family, to test a quest that sums or picks among them. For a balance you can't find on chain, such as an exact decimal amount, test the evaluator with `runQuest` and `facts.wallet`.
- A linked address stays the test member's until you unlink it or reset the member; both free it for anyone to link. An address another member already holds is refused with `account-linked-elsewhere`, so give each persona its own address. The address's real owner can take it back: if they sign in with that wallet, or connect it with a signature, in the same project and environment, it moves to them and leaves the test member's `wallets`. Link another address to continue. Read `wallets` in the `test-members list` result before a run, so a vanished address doesn't pass for a failing quest.
- Linking a Cosmos address again with another `--network` moves it to that chain ID instead of adding a second entry.
- Tracker records that name a linked address are attributed to the test member, and records the address authored before you linked it are decided again. A tracker that watches wallets therefore reacts to the link itself.

### Real X and Discord accounts

Quests that read `x.profile()`, `follows()`, `reposted()` or a member's Discord roles need an account X and Discord know. Link a real, public account to the test member without OAuth:

```sh
domino test-members link-account MEMBER --provider x --username solflare --json
domino test-members link-account MEMBER --provider discord --id 80351110224678912 --json
domino test-members unlink-account MEMBER --provider x --id 1263464405834899456 --json
```

- Quest code reads the linked account like one a member linked: `x.connection()` returns its ID, and the profile, follower, repost and role reads ask X and Discord about it. Pick accounts whose public state puts each persona where you want it: following the target, not following it, holding the role, missing it.
- `--username` resolves the X handle through X; a handle no account holds is refused with `not-found`, and a busy X lookup with `rate-limited`. Discord accounts link by ID only.
- Linking replaces the member's synthetic or previously linked account of that provider. A member that connected an account itself on a staging site refuses with `conflict`.
- An account another member holds is refused with `account-linked-elsewhere`. When the account's owner signs in with it or connects it in the same project and environment, it moves to them and leaves the test member's `accounts`. Read `accounts` in the `test-members list` result before a run.
- Tracker records the account authors are attributed to the test member, and `test-members record` posts as it.
- Unlinking or reset frees the account for anyone to link.
- Relay never writes to a linked account. A Discord role reward to a test-linked account creates no delivery and is never sent; the attempt's history says why. Prove role rewards with a real member or `runQuest`.

### X and Discord activity

`domino test-members record MEMBER --tracker TRACKER --data JSON` writes one synthetic post, message or reaction into a managed X or Discord tracker. Attribution, the quest's `where` filter and its trigger run as they do for collected activity, so this proves the quest's trigger end to end.

```sh
domino test-members record MEMBER --tracker posts --data '{"content":"gm #launch","likeCount":12}' --json
domino test-members record MEMBER --tracker chat --data '{"channelId":"1300000000000000001","text":"gm"}' --json
domino test-members record MEMBER --tracker reactions --data '{"emoji":{"name":"⭐"}}' --json
domino test-members record MEMBER --tracker reactions --as messageAuthor --json   # a reaction to MEMBER's message
```

`--data` uses the stored record's paths, the same ones the quest's `where` reads: `content`, `author.username`, `isReply` and the counts (`likeCount`, `viewCount` …) for X; `channelId`, `replyTo` and `text` for messages (`text` only when the tracker stores text); `channelId`, `messageId`, `emoji`, `reactorRoles` and `messagePublishedAt` for reactions. Relay sets IDs and real time. The answer's `admissions` counts the tracker's quests that admitted, skipped or filtered out the record; `test-members history` lists skips with their code.

What to watch for:

- Every condition of the filter, one at a time: a record that matches, then one that misses by a single field (another channel, another emoji, `likeCount` one below the bound, the text with different case). A filtered record leaves no outcome, only the `filtered` count.
- A reaction quest that pays `messageAuthor`: send with `--as messageAuthor`. A reaction the member makes has no message author a quest can pay, and is skipped with `no-account`.
- Discord message text: a tracker without `text: true` refuses `text` with `validation-failed`, and a quest filtering on `text` never matches its records.
- The record's time is now. A tracker window that excludes now refuses it; a quest whose window excludes the member's projected time skips it with `outside-window`.
- A record names the member's linked account of the tracker's provider when it has one. Otherwise it names a synthetic account linked on the first record, which is what `x.connection()` then returns. X doesn't know a synthetic account, so `profile()`, `follows()` and `reposted()` answer as X does for an unknown account; link a real account (above) or test those checks with `runQuest` and `facts.x`.
- Unlinking the synthetic account from the member makes later records skip with `unlinked-account`, the path a member who disconnected X takes.

Reset removes the member's synthetic accounts and records with everything else, and unlinks its linked accounts.

### What test members cannot reach yet

| Quest needs | With test members | Instead |
| --- | --- | --- |
| Photo submission | Refused | `runQuest` with `facts.review.photo` answers `pass`, `fail` and `unclear` |
| X profile, follower and repost reads, Discord roles, for a state no public account has | Link a real account with `test-members link-account`; a synthetic account is unknown to X and Discord | `runQuest` with `facts.x` or `facts.discord` |
| Accepting a referral code | Not possible | Two signed-in test participants, on real time |

Report each gap you had to route around in your handoff, so the operator knows which paths were proven end to end and which only at the evaluator.

## Check participant flows on staging

`domino test-members` proves the ledger. To prove the app's own screens (sign-in state, profile pages, referral links, the quest UI), sign the staging site in as a test member and use it. Stage a build first ([hosted projects](hosted.md#source-and-release-workflow)).

1. **Get a staging link.** `domino api POST /staging/link body.json` with `{ "path": "/" }` returns `{ url, expiresAt }`: a one-minute URL that opens staging once. Mint a new one per browser session. It needs `read` and the test environment.
2. **Open it.** Navigate your browser to `url`. The staging host answers 303 to `path` and sets the `__Host-domino-staging` cookie, valid 12 hours. Without a browser, request `url` without following redirects and keep the cookie from `Set-Cookie`.
3. **Act as a test member.** The page shows the test toolbar when the link names an operator. Use its member picker, or call the toolbar's endpoints on the staging origin with the staging cookie, `X-Domino-Toolbar: 1` on every request, and the staging origin as `Origin` on every POST:
   - `GET /__domino/toolbar/state`: the operator, the project's test members, and who the page is signed in as.
   - `POST /__domino/toolbar/members` with `{ "name" }` creates a test member.
   - `POST /__domino/toolbar/session` with `{ "member": "<id>" }` signs the page in as that member and sets `relay_session`; `{ "member": null }` signs out. Reload the page afterwards. The session is an ordinary 8-hour member session.
   - `POST /__domino/toolbar/members/<id>/advance` with `{ "days" }`, `GET …/<id>/quests`, `POST …/<id>/passes` with `{ "quest" }`, and `GET …/<id>/history?since=<ms>` match `domino test-members advance`, `pass` and `history`, which you can use instead.
4. **Walk the flow** in the browser as the member, then read the outcome with `domino test-members history`.

Who the link names decides whether step 3 works. Each toolbar request checks again that the operator still holds `publish` in test, and that the key behind the link is still active:

| Credential that minted the link | Acts as | Toolbar |
| --- | --- | --- |
| `domino login` CLI session, Console | You | Works while you hold `publish` in test. |
| Personal API key (`ak_`) | You | Only when the key itself holds `publish`; ends when the key is revoked or loses it. |
| Organization automation key (`dak_`) | `automation:<id>` | Only when the key holds `publish` and reaches the project; ends when the key or the secret that minted the link is revoked or expires. |
| Hosted app key, service key, Claude or ChatGPT | Nobody | Views staging; the toolbar answers 403 `operator-required`. |

Toolbar actions are audited as the operator: your user, or `automation:<id>`. A revoked or expired key answers 401 `unauthenticated`; mint a new link with an active credential. A human reviewer without Console access signs in with the staging password instead (Console **Settings → Staging access**); a password session views staging but never gets the toolbar.

## Report

End with a short table per persona: what it did on which simulated day, the expected and actual outcome, and the balances. List the bugs you fixed with the persona that found them, the replays you ran, and the paths you could only test with `runQuest`. Leave the members you used for inspection, or reset them when the operator wants a clean list; a member whose reset is refused stays in the list with the `state` that blocked it.
