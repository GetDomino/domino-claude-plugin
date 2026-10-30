# Verify member actions

Read this when a quest's `evaluate` checks what a member did. It covers the rules every evaluator reader follows (null results, errors and pinning), declared integrations, Discord membership, X, linked wallets and their balances, NFT holdings, contract state, Cosmos delegations, on-chain actions and Solana native stake, text and URL submissions, AI text review, image checks, staff confirmation, and entry after app-owned pass verification. [Authoring](authoring.md) covers the rest of a quest.

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
| `ctx.wallet.events(query)` | No linked EVM wallet | `provider-unavailable`, `author-error` (malformed event, unsupported chain, an `account` that is not an indexed address argument, a bad `where`, `since` over 30 days back, more than 3,000,000 blocks or 1,000 logs without `newest`, an `address` the member never linked) |
| `ctx.wallet.stake(query)` | No linked Solana wallet; a wallet without stake reads `"0"` | `provider-unavailable`, `author-error` (malformed vote account, an `address` the member never linked) |
| `ctx.wallet.transactions(query)` | No linked Solana wallet | `provider-unavailable`, `author-error` (malformed program, an account that is not an executable program, a `mint` that is not a token mint, `since` over 30 days back, an `address` the member never linked) |
| `ctx.trackers.records()`, `ctx.trackers.summary()` | Never: no match is an empty page or zero totals | `author-error` (unknown tracker, a filter the tracker cannot answer) |
| `ctx.trackers.content(reference, { id, observation })` | The tracker has no such record or observation | `author-error` (not an X tracker) |
| `ctx.images.*` | Never | `not-linked` (`{ x: "profile" }` without a linked X account), `author-error` (missing, undecodable or another attempt's image), `provider-unavailable` |
| `ctx.review.text()`, `ctx.review.photo()` (deprecated) | Never | `provider-unavailable`, `author-error` (input over the limits) |

`CapabilityError` is exported from `@domino-sdk/relay/authoring` with `code` (`"not-linked"`, `"provider-unavailable"`, `"author-error"`, `"integration-required"` or `"capability-limit"`; treat an unknown code, which a newer Relay can add, as a failure its `message` explains; it is never retryable) and `retryable`. Check `error instanceof CapabilityError && error.code === "provider-unavailable"` to decide yourself, such as sending the attempt to review, and rethrow everything else. Provider messages never reach quest code; `message` is Relay's own.

- **A failure that is not retryable is captured like a result**, or follows from the release and attempt alone, so a replay throws the same error. An uncaught `integration-required` ends the attempt as `failed` immediately. Relay releases that failed read and later captures so an explicit retry after repair can recheck them; earlier captured results and account pins remain fixed.
- **A retryable failure (`provider-unavailable`) is captured only once quest code moves past it.** If quest code handles it and moves past it, it is final for the evaluation run: once quest code makes another capability call, Relay captures it, so a replay throws it again and takes the same branch. If it escapes the evaluation (thrown, rethrown, or wrapped as a `cause`, in an `AggregateError` or a `SuppressedError`), Relay releases it, along with every capability result captured after it, and retries the attempt, asking the provider and those later calls again. To let a failure you caught be retried, rethrow it, or wrap it with `new Error("…", { cause: error })`; `throw new Error(error.message)` loses the link, so the failure stays captured and every retry replays it. A decision you return after catching it stands.
- Any other error (Relay itself unavailable, an expired evaluation) is not a `CapabilityError`; never treat it as an answer.

**Quest code that fails on its own fails the attempt once.** A bundle that throws while loading, such as a top-level `ReferenceError`, an `evaluate` that throws anything without an escaped capability failure in it (a plain `Error`, a `TypeError`, an invalid reject code), and an `evaluate` that returns no decision end the attempt as `failed` with code `evaluator-error`. The reason is the thrown error's name and message, such as `ReferenceError: seed is not defined`, cut to 1,000 characters. Relay does not retry it, because every run would fail the same way. Members read the reason too, so throw messages you are content to show them. An attempt keeps the release it started on, so retrying it runs the same code and fails again: fix the code and publish, and new attempts use it. `runQuest` rejects with an `EvaluatorError` (code `evaluator-error`, the same message, and what quest code threw as `cause`). Quest code that replaces `Response.json` so the evaluator answers more than 1 MiB or 10,000 JSON values fails the same way, with reason `The evaluator's answer is too large: …`; a description larger than 4 MiB or 100,000 values fails publication.

**An evaluation run makes at most eight capability calls**, shared by every reader: `ctx.x`, `ctx.discord`, `ctx.wallet`, `ctx.member`, `ctx.trackers`, `ctx.images`, `ctx.review` and `ctx.telegram`. The ninth call throws `CapabilityError` with code `capability-limit`, never retryable, without reaching Relay; uncaught, it fails the attempt with `evaluator-error`. A retry or a replay makes the same calls, so it fails the same way. Each page of tracker records is one call, so read with filters and `summary()` instead of paging through every record, and ask `ctx.member.completions` once for several quests instead of once per quest. The testing kit applies the same limit and exports it as `limits.capabilityCalls` from `@domino-sdk/relay/testing`. A scheduled defer starts a fresh run with eight calls of its own.

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

## Declared integrations

Import the `discord` and `x` modules from `@domino-sdk/relay/authoring` and list the ones the quest uses: `integrations: [discord, x]`. A module the SDK does not know can be named by provider, `{ provider: "name" }` (lowercase letters, digits and hyphens): the quest builds, its description lists it in `integrationModules` but not in `integrations`, and publishing refuses a module Relay does not have with 422 `unknown-integration-module`. Only declared provider readers appear in the evaluator context and its inferred type. Discord role rewards and resource selectors require the Discord declaration too. An X image reference requires X. Wallet, member, trackers, images, review and Telegram (`ctx.telegram`) stay available without provider modules. A quest may still declare Telegram, `{ provider: "telegram" }`: it publishes, and `ctx.telegram` reads the same. A new publication changes declarations for new attempts; existing attempts retain their pinned release.

## Verify Discord membership

Use the `discord-membership` snippet for a server membership check, with an optional role requirement. Connect the server first: Console **Integrations**, or `domino integrations discord connect`. Declare `integrations: [discord]` on the quest and call `await ctx.discord.member()` inside `evaluate`. There is no server ID setting. The result is `null` when the participant is not in the server or has not linked Discord, or `{ userId, roles, joinedAt, pending, nickname, displayName, boostingSince }`. Author code decides eligibility and the quest's declared rewards.

The platform supplies a read-only client bound to the project's configured server and the participant's Discord identity. Quest code cannot supply a user ID, server ID, access token, or arbitrary URL. For reads that need the project connection, a missing connection, unavailable bot configuration or removed bot access throws a non-retryable `CapabilityError` with code `integration-required`. Membership without a linked Discord identity still returns `null`. If the error escapes quest code, the attempt becomes `failed` immediately with that code and a participant-safe reason. Repair the connection before retrying; an explicit retry rechecks the failed read. Temporary Discord outages still use retryable `provider-unavailable`. These failures do not prove the participant is ineligible. Publication doesn't require the connection: a quest that needs Discord (declared, a role reward, or a Discord resource setting) publishes. A deployment's preview and result list it in `warnings` with code `integration-required`, and `domino deploy` prints the warning; other publication paths return none. When Discord becomes ready (first connection, or repair from needs attention), Relay admits a new attempt in the background for each member of an active automatic quest without a tracker that needs Discord whose latest attempt failed with `integration-required`; tracker-triggered attempts need an explicit retry. Role and channel selections are checked against the server only while it is connected. A quest that does not declare the integration gets `author-error`. Bot access problems appear in Console Integrations. Participants signing in with Discord need no extra permission flow; members using another login link their Discord identity once.

The evaluation run pins the participant's Discord account (or that they have none) and the server at its first Discord read and never rechecks them: linking, unlinking or reconnecting afterwards does not change that run, so only a new submission or scheduled deferred run uses a new connection. Reads, including absent membership, are captured and replayed on recovery. Membership screening (`pending`) and role requirements remain author decisions.

The campaign starter includes participant identity controls. Existing apps use the same APIs through `createParticipantBackend`; see [participant API](participant-api.md#connected-accounts). The managed bot is configured by the platform, never by a quest author. The injected Discord client supports `member()` and `message({ channelId, messageId })`. Discord messages and reactions reach quests as tracker records; see [Discord trackers](trackers.md#discord-trackers). Discord role grants are declared rewards, never evaluator writes. X reads use the separate `ctx.x` capability; see [X verification](#x-verification).

The release's `provider` setting names the AI review backend, and only `workers-ai` publishes; it does not select the evaluator for Discord, quizzes, or other authored checks. `humanReview` allows quest code to return a review decision. It does not automatically route Discord failures to staff. The Discord membership snippet returns accept or reject decisions and leaves provider outages as errors.

## X verification

Hosted evaluators expose `ctx.x.connection()`, `profile()`, `post({ postId })`, `follows({ userId })`, and `reposted({ postId })`. Use numeric X IDs as strings. Resolve a handle on the server with `management.x.lookup({ username })` or `domino x lookup HANDLE --json`: `{ id, username, name }` for the account that held the handle when X was last read, at most a day ago (a leading `@` is accepted), 404 `not-found` when none does; `read` permission, 30 a minute per user and 300 an hour per organization (429 `rate-limited`). It resolves the handle only; it does not show who owns the account, and handles can change hands, so store the `id`. `connection()` returns `{ userId }` or null and needs no provider credential. `profile()`, `follows()` and `reposted()` return `null` without a linked participant account, so check that before reading `false` as "does not follow"; `post()` needs no linked account and returns `null` for a post that does not exist. Provider reads need platform-configured `X_API_WRAPPER_API_KEY`. Declare `integrations: [x]` to enable these readers. X requires no project integration installation.

Profile, follow and repost checks bind to the participant's linked account. Profile fields are `id`, `username`, `name`, `bio`, `profileImageUrl`, `followersCount`, `verified`; unavailable optional fields are null. A post read returns content, author, metrics, nullable reply/quote/repost references (`inReplyToStatusId`, `inReplyToUserId`, `quotedTweetId`, `retweetedTweetId`), and `media` as `{ type, url }` in post order. Check profile pictures and post images with [image checks](#image-checks), not by fetching URLs. Post lookup does not assert authorship: compare `post.author.id` to `connection.userId`, check target references and the required time window. Missing metrics are null.

Reads are captured per evaluation run. The account is pinned at the run's first X read, so a change or disconnect afterwards does not affect that run. A new submission or scheduled deferred run can use another account and fresh evidence. Provider failures throw `CapabilityError` with code `provider-unavailable`, never negative evidence or acceptance. Quest code receives no X credential. These capabilities do not expose like checks or automatic X observation delivery. Combine them with URL/text submissions and text AI review below.

## Linked wallets

`ctx.wallet.list()` returns the member's wallets linked to the project: `{ chain, network, address, provedOn?, verifiedAt, revision, signIn }[]`, ordered by chain, newest first. An EVM smart-contract wallet carries `provedOn`, the `eip155:<chain ID>` that proved it: it is an account on that chain only, so `ctx.wallet.balance()`, `ctx.wallet.nfts()`, `ctx.wallet.events()` and `ctx.wallet.call()` on another EVM chain skip it, and naming its `address` there is an `author-error`. `runQuest` facts follow the same rule. `chain` is `"solana"`, `"eip155"` or `"cosmos"`; a Cosmos wallet's `network` is the CAIP-2 ID of the chain the app requested when linking it, such as `"cosmos:cosmoshub-4"`, and `null` for the others. `network` is not proof: Domino does not verify it, and any app can link an address under any `cosmos:<chain ID>`. Gate Cosmos wallets on the address: check its bech32 prefix against your own list of accepted chains, such as `linked.chain === "cosmos" && linked.address.startsWith("cosmos1")` for Cosmos Hub, and read the chain itself for balances or activity. The list is pinned at each evaluation run's first wallet read. Links and unlinks do not change that run, and its retries replay the same list. A scheduled run after defer or progress reads the current links. It returns `null` when the member linked no wallet, never an empty list; reject with a message that asks them to link one. Each call is one of the attempt's eight capability calls. The wallet a member signed in with is in the list with `signIn: true`; a member who signed up with a wallet may have no other account, so do not require one only to identify them.

`ctx.wallet.list()` returns all linked wallets or null; `ctx.wallet.balance({ chain: "solana" })` reads their total SOL balance. Pass a linked `address` to read one wallet. Wallet addresses belong to verified members only: an account is linked to at most one member.

## Wallet balances

`ctx.wallet.balance({ chain, token?, address? })` reads a fungible balance for the member's pinned linked wallets:

| `chain`                                                     | `token`                                                                    | Reads                                                      |
| ----------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `"solana"` (or `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp"`) | SPL or Token-2022 mint (base58); omit for SOL                              | Every linked Solana wallet, all token accounts of the mint |
| `"solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1"` (Solana devnet) | SPL or Token-2022 mint (base58); omit for SOL                              | The same wallets, on devnet                                |
| `eip155:<chain ID>` (CAIP-2), such as `"eip155:8453"`       | ERC-20 contract (`0x…`, checksummed or one case); omit for the native coin | Every linked EVM wallet, at the latest block               |
| `cosmos:<chain ID>` (CAIP-2), such as `"cosmos:cosmoshub-4"` | Bank denomination; required (such as `uatom` for the native coin)        | Wallets linked on that exact network, at one fixed block   |

The SDK types EVM and Cosmos chains as CAIP-2 IDs and validates their syntax; a Cosmos query always requires a denomination. Which chains the platform reads is a server list: `management.walletBalanceChains()` or `domino api GET /wallet-balance-chains` (`read` permission) returns `{ chains: [{ chain, name, testnet }] }`, with Solana mainnet and devnet (`testnet: true`), the Cosmos chains (Cosmos Hub, Osmosis and any other the platform operator configured), then EVM chains by chain ID. It returns `{ chain, token, decimals, total, wallets: [{ address, amount }], at, observedAt }`. `chain` is CAIP-2, and `token` is the canonical mint, EIP-55 address, or Cosmos bank denomination (null for a Solana or EVM native coin). `total` and each `amount` are integer strings in the token's smallest unit. `decimals` is null when token metadata does not provide it: send that to review or reject, never guess. `at` is `{ kind: "slot" | "block", number }`. Convert with `parseUnits(value, decimals)` (bigint; rejects excess precision) and `formatUnits(amount, decimals)`, both from `@domino-sdk/relay/authoring`: `BigInt(total) >= parseUnits("10", decimals)`. With a `settings.amount()` minimum, compare base units directly: `BigInt(total) >= BigInt(settings.minimum)`.

- `address` must be one of the member's pinned wallets of that family and, for Cosmos, that exact network (any letter case for EVM); otherwise the call throws `CapabilityError` with code `author-error` and message "Read only the member's linked wallets". Quest code cannot probe other addresses.
- No readable wallet returns `null`, never a zero total. Reject with a message asking the member to link one.
- Errors never mean ineligible. An unsupported EVM or Cosmos chain, a malformed token, a Solana account that is not a mint and no contract at the EVM address throw `author-error`; provider outages and network verification mismatches throw `provider-unavailable`.
- It reads the same pinned wallet list as `ctx.wallet.list()`. Each read is captured and replayed on retries, and uses one of the attempt's eight capability calls. EVM reads through dRPC are approximate `latest` reads. Cosmos reads verify the endpoint's chain ID and pin bank and metadata requests to one block.
- Domino chooses every endpoint: neither quest code nor project settings can supply an RPC or REST URL. For an unlisted chain or other on-chain state, feed a custom tracker source from the customer's own indexer, with `identities: ["wallet"]` records keyed by CAIP-10 account (see Tracker triggers), and read those records instead.
- Keep chain and token identifiers in settings, never in browser input. `settings.chain()` passes to `chain` as it is; an EVM `token` is typed `` `0x${string}` ``, so check a text setting with a type guard (`/^0x[0-9a-fA-F]{40}$/`) before passing it.
- An EVM `token` that is an NFT contract (ERC-165 reports ERC-721 or ERC-1155) throws `author-error` naming `ctx.wallet.nfts()`. No historical balances. When staking mints a receipt token (a liquid-staking token, ERC-4626 vault shares), read that token here; for stakes a contract keeps in a mapping use [`ctx.wallet.call()`](#contract-state), for Cosmos native staking [`ctx.wallet.delegations()`](#cosmos-delegations), and for Solana native staking [`ctx.wallet.stake()`](onchain.md#solana-native-stake).

Snippet `token-holder` installs a complete quest: chain (`settings.chain()`), token and minimum (`settings.amount()`, 6 decimals) settings, and the total across linked wallets compared in base units. The name discloses the evidence: a balance at verification time, not continued holding. The same quest is `packages/sdk/examples/token-holder-quest.ts`.

## NFT holdings

`ctx.wallet.nfts({ collection, address? })` counts the NFTs of one collection the member's pinned linked wallets of its family hold: EVM wallets for an EVM collection, Solana wallets for a Solana one. `collection` is a CAIP-19 asset type: `eip155:<chain ID>/erc721:<contract>` for an ERC-721 collection, `eip155:<chain ID>/erc1155:<contract>/<token ID>` for one ERC-1155 token ID (decimal, at most 2^256 − 1), or `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/nft:<collection>` for a Metaplex collection on Solana mainnet, named by its Core collection account's address or its Token Metadata collection NFT's mint (`solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1/nft:<collection>` on devnet, read from the same Solana wallets). An EVM chain is one `management.walletBalanceChains()` lists. Take it from a `settings.nft({ label, description?, default })` setting, typed `NftCollection`; never from member input. `isNftCollection()` and `parseNftCollection()` from `@domino-sdk/relay/authoring` check the syntax.

It returns `{ collection, chain, standard, total, wallets: [{ address, count }], at, observedAt }`: `collection` with an EVM contract EIP-55 checksummed, `standard` `"erc721"`, `"erc1155"` or `"metaplex"`, `total` and each `count` integer strings (ERC-721 `balanceOf(owner)`, ERC-1155 `balanceOf(owner, id)`, Solana the collection's Core assets, verified NFTs and verified compressed NFTs held), and `at` the approximate latest EVM block (`{ kind: "block", number }`) or the lowest Solana slot read (`{ kind: "slot", number }`). Holding at least one is `BigInt(total) >= 1n`.

- On Solana, for a Core collection, every Core `AssetV1` account a linked wallet owns whose update authority is `Collection(<collection>)` counts; only the collection's update authority or a delegate it approved can add an asset to it. For a Token Metadata collection, an NFT counts when a linked wallet has a token account (SPL Token or Token-2022) holding exactly 1 of a mint with 0 decimals and supply 1, whose Metaplex metadata names the collection with `verified: true` and whose token standard is not fungible. An unverified collection field is ignored: anyone can write one. Compressed NFTs (Bubblegum) count too, read through Helius's DAS API: a cNFT a linked wallet owns, not burnt, whose collection grouping is the collection with `verified: true`. Bubblegum v1 cNFTs sit in Token Metadata collections, v2 in Core collections; each wallet's count adds all kinds. Wallets are searched only in a collection that has cNFTs: a Core collection with the `BubblegumV2` plugin (read from its account), or a Token Metadata collection Helius reports cNFTs in (asked once, remembered a day if it has, 6 hours if not). A platform without its Helius key refuses a Core collection with the plugin (`not-configured`) and counts no cNFTs in Token Metadata collections.
- Publication, Console settings edits and unarchiving check every `settings.nft()` value, archived quests included: the syntax, then on EVM a chain Relay reads and that the contract reports the named standard through ERC-165 `supportsInterface`, and on Solana that the address is a Core `CollectionV1` account or a Metaplex collection NFT on the collection's cluster (a 0-decimal, supply-1 mint with a unique master edition, `max_supply` 0, and token standard `NonFungible`, `ProgrammableNonFungible` or unset, whose metadata has collection details, or, for a legacy unsized collection, that is not itself a verified member of another collection). A token contract, an ERC-1155 contract written as `erc721` (and the reverse), a fungible Solana mint, a mint without Metaplex metadata, a print or limited edition, a Core asset with no collection, and an NFT or Core asset that belongs to a collection (the message names that collection) are refused with `422 validation-failed` naming the setting's label and what is wrong. An RPC outage refuses with `503 unavailable`; retry. A platform without a dRPC key refuses every EVM collection with `503 not-configured`, since none can be checked.
- `address`, `null` and failures follow `ctx.wallet.balance()`: an address the member did not link, a malformed collection, a chain Relay does not read, and a contract that rejects the read throw `author-error`; no linked wallet of the collection's family returns `null`; outages throw `provider-unavailable`. Reads are captured, replayed on retries, and count toward the eight capability calls.
- `WalletReader.nfts` is optional in the type so that older test doubles compile; hosted evaluations always supply it, so call it as `ctx.wallet.nfts!(…)`, and in a context a test builds use `fakeWalletReader(facts)` from `@domino-sdk/relay/testing`. The testing kit answers it from `facts.wallet.nfts` (called with `{ collection, address }`), null while `facts.wallet.list` links no wallet of the collection's family; a run's `capabilityCalls` lists NFT reads as `wallet-nfts`.
- No token ID listing or Cosmos NFTs.

Snippet `nft-holder` installs a complete quest: a `settings.nft()` collection, accepted at a total of at least one across the linked wallets of its family.

## Contract state

`ctx.wallet.call({ call, args?, address? })` calls one `view` or `pure` function of an EVM contract once per pinned linked EVM wallet, every call at the same block, through dRPC on the chains `management.walletBalanceChains()` lists. Use it for state a contract keeps per address, such as `stakedOf(address)`, `getVotes(address)` or `userInfo(uint256 pid, address user)`. When staking mints a receipt token, read the token with `ctx.wallet.balance()` instead.

`call` is the contract's CAIP-10 ID, a space, and the function's human-readable signature: `"eip155:8453:0x… function stakedOf(address account) view returns (uint256)"`. Take it from a `settings.contractCall({ label, description?, default })` setting, typed `ContractCall`; never from member input. The function must be `view` or `pure`, take elementary parameters (`address`, `bool`, `string`, `bytes`, `bytes1`…`bytes32`, `uint8`…`uint256`, `int8`…`int256`; `uint` and `int` mean 256 bits), and return exactly one `uint`, `int`, `bool` or `address`. Arrays, tuples, structs and several return values are refused. `isContractCall()`, `parseContractCall()` and `parseContractFunction()` from `@domino-sdk/relay/authoring` check the syntax.

`args` lists the function's arguments in order (default `[]`). `{ wallet: true }` stands for each wallet's address and may fill at most one `address` parameter; the other values come from settings or code, never from member input. Integers are safe-integer numbers or decimal strings within the type's range, `bool` a boolean, `address` a checksummed or one-case hex address, `bytes` and `bytesN` `0x` hex of the right length.

It returns `{ call, chain, contract, returns, total, wallets: [{ address, value }], at, observedAt }`: `call` canonical (contract EIP-55 checksummed, parameter names dropped, `uint` as `uint256`), `returns` the return type, each `value` a decimal string for an integer, a boolean for `bool`, or an EIP-55 address, `total` the integer values' sum as a decimal string (null for `bool` and `address`), and `at` the block every call read (`{ kind: "block", number }`). Reaching a minimum is `BigInt(total) >= BigInt(settings.minimum)`.

- Publication, Console settings edits and unarchiving check every `settings.contractCall()` value, archived quests included: the syntax, and that Relay reads the chain. A wrong value is refused with `422 validation-failed` naming the setting's label.
- A revert throws `author-error` with the node's reason (such as `execution reverted: pool does not exist`); so do no contract at the address and return data that does not decode as the declared type. Arguments that do not fit the parameters, and an `address` the member did not link, throw `author-error` too. No linked EVM wallet returns `null`; an RPC outage, or a platform without its dRPC key, throws `provider-unavailable`. Reads are captured, replayed on retries, and count toward the eight capability calls.
- `WalletReader.call` is optional in the type so that older test doubles compile; hosted evaluations always supply it, so call it as `ctx.wallet.call!(…)`. The testing kit answers it from `facts.wallet.call` (called with `{ call, args, address }`), null while `facts.wallet.list` links no EVM wallet; a run's `capabilityCalls` lists contract calls as `wallet-call`.
- No Cosmos contracts, history or events. State a Solana program keeps per wallet (decoded with its Anchor, Codama or Shank IDL) is [`ctx.wallet.programAccount()`](onchain.md#solana-program-accounts).

Snippet `contract-state` installs a complete quest: a `settings.contractCall()` function called with the wallet as its only argument, accepted when the `uint` values reach a `settings.amount()` minimum.

## Cosmos delegations

`ctx.wallet.delegations({ chain, validator?, address? })` reads the native staking delegations of the member's wallets linked on one Cosmos SDK chain that `management.walletBalanceChains()` lists, through the same platform-owned REST endpoint as Cosmos balances, at one fixed block. `validator` is a validator's operator address (bech32 with a `…valoper` prefix, such as `cosmosvaloper1…`) to count only delegations to it; omit it to count every validator.

It returns `{ chain, denom, validator, total, wallets: [{ address, amount }], validators: [{ validator, amount }], at, observedAt }`: `denom` the chain's staking denomination (`bond_denom`, such as `uatom`), every amount an integer string in its base units, `total` the sum over `wallets`, `validators` what the wallets delegate to each validator (only `validator` when given; none at zero), and `at` the block (`{ kind: "block", number }`). Delegating at least a minimum is `BigInt(total) >= BigInt(settings.minimum)`.

- Every request after the latest block is pinned to its height with `x-cosmos-block-height`. An endpoint that cannot pin the staking queries answers at its own height: when every answer reports the same height, that block is recorded in `at`; answers at different heights are an outage.
- `address`, `null` and failures follow `ctx.wallet.balance()` for Cosmos: only wallets linked on that exact network are read; a chain Relay does not read, a malformed validator and an address the member did not link throw `author-error`; no wallet on the chain returns `null`; outages throw `provider-unavailable`. Reads are captured, replayed on retries, and count toward the eight capability calls.
- `WalletReader.delegations` is optional in the type so that older test doubles compile; hosted evaluations always supply it, so call it as `ctx.wallet.delegations!(…)`. The testing kit answers it from `facts.wallet.delegations` (called with `{ chain, validator, address }`), null while `facts.wallet.list` links no wallet on the chain.
- Bonded delegations only: no unbonding delegations, rewards, history or governance votes, and no EVM native staking. Solana native staking is [`ctx.wallet.stake()`](onchain.md#solana-native-stake).

Snippet `cosmos-staker` installs a complete quest: a Cosmos chain, an optional validator and a minimum delegation in base units.

## On-chain actions

EVM contract events (`ctx.wallet.events()`), Solana transactions (`ctx.wallet.transactions()`), Solana program accounts (`ctx.wallet.programAccount()`) and Solana native stake (`ctx.wallet.stake()`) are in [on-chain actions](onchain.md). Snippet `onchain-action` installs a complete EVM event quest, and `solana-action` a quest for a transaction a linked Solana wallet signed that called a program, optionally receiving a minimum of a token.

An app's server can also submit a member-signed Solana transaction that carries a quest's memo with `management.attempts.submitSolanaTransaction()` (experimental): Relay admits a member-submitted `manual()` quest without input for the member whose linked wallet signed it, and the evaluator reads the transaction as `ctx.action`, null for every other attempt. See [quests completed from a signed Solana transaction](onchain.md#quests-completed-from-a-signed-solana-transaction). Attempt captures carry `ctx.action` too, so `replayAttempt` reads it.

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

Treat every submission as an untrusted claim. Validation checks the input shape, not truth, authorship, ownership, or safety of the destination. Relay never fetches a submitted URL automatically. Do not put a submitted string into an HTML renderer or a fetch request. Never use text or URL input for a secret such as a station code: the member reads it back and can submit guesses directly; see [codes and other secrets](trackers.md#codes-and-other-secrets).

For X posts, use `xPostId` from `@domino-sdk/relay/schema` to parse the URL, then call `await ctx.x.post({ postId: id })`. The parser returns `null` for unsupported hosts and paths; `post()` returns `null` for a missing post (capture and provider failures: [X verification](#x-verification)). The URL's username is not proof of authorship. Check the returned post's content and author against your eligibility rules, or request human review. Participants must link X first. Compare the returned author ID with `userId` from the non-null result of `await ctx.x.connection()` to check authorship.

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
| Provider timeout  | 12 seconds, including response reads; after 5 seconds without an answer, a second identical request starts and the first answer wins |
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

Each call uses one of the evaluation's eight capability calls and is captured for replay. AI inputs are re-encoded JPEGs without metadata; the model is told to ignore instructions inside images but can still misjudge them. A screenshot is a participant's claim: compare extracted values with verified data, such as `ctx.x.profile().username`, and send `null` or uncertain results to staff. Missing reference images, undecodable images and missing X pictures throw `CapabilityError` with code `author-error`; provider failures throw `provider-unavailable`. `review` usually answers in about 2 seconds and `extract` in 3 to 5; both share text review's provider timeout, so a member's waiting screen should allow about 5 seconds for a review and 10 for an extraction, plus the upload. To fail AI-generated or edited pictures, say so in the criteria, for example "It must be a genuine photograph: fail AI-generated or digitally edited images." Deploying an existing key replaces its image for later checks; removing it from `relay.json` does not delete it. Checks describe the image at verification time and do not prove retention. `ctx.review.photo()` is deprecated and will be removed in 2.0.0: review an uploaded photo with `ctx.images.review({ images: [{ evidence: ctx.input.photo }], criteria })`. Snippets `x-profile-picture`, `unique-photo` and `community-screenshot` install complete quests with tunable thresholds; the same quests are SDK examples in `packages/sdk/examples`. See [check images](https://console.domino.run/docs/build/images).


### Evaluator protocol

The current `defineQuest()` bundle declares `protocol: 2`, and `describe()` emits `evaluatorProtocol: 2`. Relay verifies that they agree during description and chooses the matching execution adapter. Protocol 2 is the first stable contract and supplies the pinned `ctx.calendar`. Protocol-less, pre-release protocol-1 and unknown protocol versions are refused before evaluation. Rebuild and republish pre-v1 bundles with the current SDK; changing their version field alone is not an upgrade. Do not strip either field when transforming a bundle. SDK v1 upgrade details: [evaluator protocol](https://console.domino.run/docs/reference/upgrade-to-v1#evaluator-protocol).

Relay stores the declared protocol with each release and runs the matching host adapter, so a published bundle keeps its context and capability behavior as the host evolves. The SDK writes the declaration; normal publication needs no configuration. Future stable protocol versions keep their adapters for the supported lifetime of their published bundles.

## Staff confirmation

Staff confirm a `manual({ actor: "staff" })` quest for a member in Console with **Confirm activity** on the member's page. A server or script can do the same with `management.attempts.confirm({ quest, member })`, where `quest` is the quest's ID or key (`POST /management/v1/attempts` with `{ actionId, quest, member }`), using a management credential with `confirm` permission. Relay answers 202 with the queued attempt, and 200 with the same attempt when the action ID is repeated. Any other quest is refused with 403 "Choose a staff-confirmed quest". The member must already exist in that project and environment: an unknown member ID is refused with 404 "Member not found" (`not-found`) and creates nothing. A member who exists but has not joined the project joins it. Create members in test with test members; in live, members exist once they sign in. In live, only a Console session or a management key may confirm; a CLI sign-in cannot confirm in live.

## Entry after app-owned pass verification

Author the entry quest with `manual({ actor: "staff" })` and no evidence input. The participant app verifies its event pass on its server, resolves the signed-in member, and calls `management.passes.verify(...)` (`POST /management/v1/passes/verify`) with `{ provider, event, subject, member, quest }`, where `quest` is the entry quest's ID or key. Use a server-only management key with `confirm` permission in the intended project and environment. Relay records unique pass ownership and submits the entry attempt; it does not decide whether the external ticket is valid. The member must already exist in that project and environment; an unknown member ID is refused with 404 `not-found`. Repeating the same pass for the same member reuses the durable attempt. A different member cannot claim that pass, and one member cannot link a second pass for the same provider and event.
