# Participant API

For a v0 app, use the [SDK v1 upgrade guide](https://console.domino.run/docs/reference/upgrade-to-v1) before adapting components. Preserve action IDs across uncertain writes and browser recovery. Update custom attempt renderers for [counter progress](https://console.domino.run/docs/reference/upgrade-to-v1#counter-progress), and keep a fallback for unknown statuses.

Development uses the hosted project test environment through `domino dev`. Use real participant sign-in; local simulated participants and offline runtime credentials are no longer supported. Photo, Discord, X and wallet capability checks call real providers. Verify those flows with the configured test accounts and report a missing credential or provider check explicitly instead of substituting a simulated result.

Create `createBrowserRelayClient({ baseUrl: "/relay" })` from `@domino-sdk/relay/browser`. The browser client uses cookies and durable submission recovery. React apps, including every starter, use `@domino-sdk/relay/react` instead (see [React hooks](#react-hooks)). With `app.routing="transparent"`, the app explicitly mounts its participant integration at `/relay/*`; hosting does not intercept or reserve that prefix. Existing apps without that setting retain the gateway participant proxy. Management tokens are not participant credentials.

## Client conventions

Every participant client method takes an optional last argument, `{ session?, signal?, timeout? }`.

- **Credentials and transport.** The browser client sends the member's session cookie. Elsewhere, pass `token` to `createRelayClient` for a fixed bearer token, or `getToken: async () => token | null` for one that changes; not both, which is refused with `invalid-input`. `getToken` is read before every attempt, within the call's deadline and signal; when it fails or returns a value that is not a token, the call rejects before sending, with code `credential` (kind `network` when it failed with a `TypeError`, such as an unreachable token service, and `client` otherwise). When it returns `null`, the call sends no `Authorization` header, even one set in `headers`. `fetch` replaces the global `fetch` for every call (a test double, a proxy's or an edge runtime's own). `headers` adds headers to every call (an invalid name or value is refused with `invalid-input` when the client is created); the client's own authorization, scope, session, CSRF and content-type headers replace one of the same name.

- **Session.** Pass the session ID the app read (`(await relay.auth.session())?.id`) as `session` to bind a call to it. If the member's session changed since, Relay refuses the call with code `stale-session` and changes nothing. Wallet linking, collection passes and `quests.retry` always bind to a session; without `session` they read the current one first.
- **Paging.** Paged reads take `{ cursor? }` beside their filters and return `{ items, nextCursor }`. Pass `nextCursor` back as `cursor` for the next page; it is null on the last. Cursors are opaque strings. `paginate(read)` from `@domino-sdk/relay` walks every item: `for await (const entry of paginate((cursor) => relay.referrals.history({ cursor })))`. It reads the next page only when the loop needs it, and stops at a null `nextCursor`; it works for the management client's paged reads too. `me.activity.list` is a feed whose cursor never runs out, so read it with `me.activity.observe` instead. The paged reads are `referrals.history`, `me.pointHistory`, `me.leaderboardHistory`, `leaderboards.standings`, `profiles.completions`, `quests.outcomes`, `content.qualifications`, `content.performance`, `engagement.page` and `engagement.boundaries`.
- **Deadlines and cancellation.** A call has 15 seconds to finish, retries and reading the body included. Pass `timeout` in milliseconds to `createRelayClient` for every call, or to one call; a value that is not a positive, finite number is refused with `invalid-input`. A call bound to a session that reads the current session first (see Session above) shares its deadline with that read; `wallets.link` gives its verification a fresh deadline after the member signs. `signal` cancels the call at once, including retry waits and pending wallet address or signature requests, and nothing more is sent. Cancelling never undoes a write already sent: that write stays `uncertain`, with its `operationId` when it has one.
- **Retries.** The client sends a call again, at most three times in all, only when Relay applies a repeat at most once: reads, and writes with an `operationId` (`offers.redeem`, `me.claim`, `quests.enroll`, `quests.claim`, `campaigns.enroll`, `campaigns.withdraw` and `referrals.accept` by their action ID, evidence uploads by their upload ID, `wallets.verify` by its challenge), plus `wallets.challenge`, whose unused nonce expires. Every attempt sends the same body and session. It retries when no answer arrived, when the answer broke off after its headers, on 500, 502, 503 and 504, and on a 429 with a `Retry-After` that fits in the time left. It waits `Retry-After` when Relay sent one, and otherwise up to 0.25 seconds, then up to 1 second. A timeout, a cancellation, an unreadable answer, input refused before sending and any other refusal are never retried, and neither is any other write: `profiles.update`, `profiles.uploadAvatar` (Relay keeps no receipt for an avatar's upload ID, so a replay after a newer upload would bring the older picture back), wallet and email sign-in, which consume their challenge. An error status whose body breaks off or is cancelled is a `network`, `timeout` or `aborted` failure that keeps its `status` and `retryAfter`; the received status still controls whether to retry. An error body that is not JSON stays `http`. When retries run out, the last failure is the rejection; a write any attempt of which may have taken effect stays `uncertain`. `retryAfter` is the wait in milliseconds Relay's `Retry-After` asked for, when it sent one.
- **Errors.** Every rejection is a `RelayError`, which extends the `RequestError` all Domino clients share. `kind` says what happened: `http` (Relay answered with an error), `network` (the connection failed, including a body cut off after the headers), `timeout` (the call's deadline passed), `aborted` (your `signal`), `invalid-response` (a success answer that is not JSON, or not the shape this SDK version expects), or `client` (refused before sending: invalid input, a signed-out member, a `getToken` that failed, with code `credential`, or a wallet signer that failed or that the member refused, with code `wallet-signer` and the wallet's own error as `cause`). Calls that reach Relay report input that fails validation as a rejected promise with code `invalid-input`; only `response` throws at once, for a path outside `/v1`. `status` is set whenever Relay's response headers arrived, and never invented. `code` is a machine-readable string, set on every refusal Relay answers; a code this SDK does not know is kept with Relay's message, and `RelayErrorCode` lists the known ones, such as `not-signed-in`, `stale-session`, `invalid-input`, `invalid-response`, `quest-updated` and `prerequisites-incomplete`. Branch on `code`, never on the message: [API error codes](errors.md) lists Relay's codes. `uncertain` is true only for a write that may have taken effect: the connection failed, timed out or was cancelled after sending, the body was cut off, Relay answered 500, 502 or 504, or a success answer could not be read. Reads never are, and 503 and 429 are refusals: Relay answers 503 only when it did nothing, such as during maintenance, and sends `Retry-After` with a 503 or 429 when it knows how long to wait. `recovery` is then `replay` when Relay keeps a receipt for the call's `operationId` and `reconcile` otherwise (read the state first). Avatar uploads return `reconcile` even with an upload ID: Relay keeps no durable receipt, so replaying an older upload could replace a newer avatar. Paged reads answer a malformed cursor with 400 and code `invalid-cursor`: start again from the first page. Test errors with `isRelayError(error)` (or `isRequestError`) from `@domino-sdk/relay` rather than `instanceof`, which fails when an app bundles two copies of the SDK.
- **Types.** `createRelayClient` and `createBrowserRelayClient` return a `RelayClient`. A minor release can add methods to it and fields to `RelayError`, so don't implement the client or subclass the errors yourself. It can also add fields to results such as `Offer` and values to `Known*` unions such as `KnownRelayErrorCode`: give a `switch` over one a default branch. A type's `@compat` doc tag says how it may grow.

## Retry after an uncertain answer

Keep one action helper for a logical write until you know its result. A generated key protects
the client's automatic retries; calling the mutation again without reusing that key starts a
different write and can spend points twice.

```ts
import { createRelayClient, isRelayError, type Redemption } from "@domino-sdk/relay"

const relay = createRelayClient({ baseUrl: "/relay" })
const redemption = relay.action<Redemption>()
const offer = "supporter-offer"

async function redeem() {
  try {
    return await redemption.run((actionId) => relay.offers.redeem(offer, { actionId }))
  } catch (error) {
    if (isRelayError(error) && error.uncertain) {
      // Keep `redemption`. Show its operationId and a retry button when recovery is "replay".
      // The button calls redemption.retry(), keeping the original offer and action ID.
      console.error(error.operationId, error.recovery)
    }
    throw error
  }
}
```

`relay.action()` and `createRelayAction()` from `@domino-sdk/relay` are the same helper.
`run(callback)` supplies an action ID and shares one pending request across concurrent calls.
After an uncertain error, both `retry()` and another `run(...)` use the saved callback and key;
the new callback is ignored until the old write succeeds. A definite initial refusal ends the
action, so the next `run` gets a new key. A refusal on a later retry cannot settle an earlier
uncertain write. The helper waits any remaining `retryAfter` before sending a retry. Keep cache
refreshes and other follow-up work outside the callback so their failure cannot replay a write.
After an authoritative read confirms a definite outcome for a reconciliation-only write, call
`settle()` to finish that action. It returns `false` while a request is still running. Never
settle an action whose outcome remains uncertain.

| Error | Next action |
| --- | --- |
| `uncertain === false` | The request was refused. Fix the input or wait as instructed, then start another action. |
| `uncertain` and `recovery === "replay"` | Offer `retry()` with the same key, payload, and session. Show `operationId` if the participant needs help. |
| `uncertain` and `recovery === "reconcile"` | Read the current state first. The helper refuses replay; an ID alone does not guarantee a receipt. |

The helper retains state in memory. It does not survive a reload or process restart. Persist an
explicit `actionId` when your flow must survive those, and keep the original payload and session
with it. Relay keeps quest submissions, so they need none of this; use `resume` to recover
them. Sign-in and wallet consent still use their existing one-time challenges.

React apps can use `useRelayAction<Result>(scope)` from `@domino-sdk/relay/react`. It returns
`{ run, retry, settle, reset, status, isPending, data, error }`. Calls resolve
`{ ok: true, value }` on success or `{ ok: false, error }` on failure; a call made while another
runs resolves `{ ok: false }` without an error. This distinguishes a successful `null` value from
failure. `status` is `idle`, `pending`, `success` or `error`, and `reset()` clears `status`,
`data` and `error` without forgetting a retained uncertain write. Pass the session or client as
`scope` to isolate pending work when the participant or project changes, and bind the write to
that session with the mutation's request options. The action itself tracks the write in flight,
so two clicks before React renders again send one write. The campaign starter's offer cards use
it directly and show a retry button after an uncertain redemption. `useMyProfile()`,
`useAccounts()`, and `useWallets()` also expose `settle()`. Use it only after their refreshed
reads confirm the earlier write's definite outcome.

The SDK's mutation hooks build on it and share one shape: `mutate(variables)`, which never
rejects, `mutateAsync(variables)`, which resolves the result or rejects with the `RelayError`,
and `status`, `isPending`, `data`, `error`, `reset()` and `settle()`. After an uncertain failure,
the next `mutate` replays the retained write with its key and original variables instead of
starting another. `useRedeemOffer()` has this shape and reads offers and balances again before
`mutateAsync` resolves. `useRelayMutation(mutation, { invalidates })` gives your own writes the
same shape: `mutation` receives the variables and `{ actionId, session }`.

```tsx
import { relayKeys, useRelay, useRelayMutation } from "@domino-sdk/relay/react"

export function AcceptInvite({ code }: { code: string }) {
  const relay = useRelay()
  const accept = useRelayMutation(
    (code: string, { actionId, session }) =>
      relay.referrals.accept({ code, actionId }, { session }),
    { invalidates: [relayKeys.referrals] }
  )

  return (
    <button disabled={accept.isPending} onClick={() => accept.mutate(code)}>
      Accept
    </button>
  )
}
```

The keyed mutations `offers.redeem`, `quests.enroll`, `quests.claim`, `campaigns.enroll`,
`campaigns.withdraw`, `referrals.accept`, and `me.claim` all accept an omitted `actionId` and
generate one. `me.upload(file, { actionId })` also generates a key, sends it as the evidence
upload ID, and safely replays the same file. `profiles.uploadAvatar(file, { actionId })` accepts
the same spelling, retaining `uploadId` as an alias, but returns `reconcile` after uncertainty
because the server keeps no durable avatar receipt. `profiles.update` accepts an optional
`actionId` for identifying the write in errors and also requires reconciliation.

## Read evolving responses

Capability response readers also ignore added fields at every depth. `capabilityResponses` from `@domino-sdk/relay/schema` provides explicit supported/unsupported parsing for image and wallet balance results. Unknown wallet anchor kinds and image formats keep the raw result. Evaluator methods retain their existing types and evaluator protocol.

SDK calls send `X-Domino-Client: @domino-sdk/relay/<version> (<runtime>)`, with `browser` for browser calls and `server` for server calls. This identifies the installed package and grants no access.

Response readers ignore added fields at every depth. Existing methods retain their closed 1.x return types. Use `relay.response(path, schema, options?)` for an extensible response. It sends `X-Domino-Response: tolerant-v1` and returns `{ status, operationId, result }`.

```ts
import type { RelayClient } from "@domino-sdk/relay"
import { offerListResponseSchema } from "@domino-sdk/relay/schema"

declare const relay: RelayClient

const { result } = await relay.response("/v1/offers", offerListResponseSchema)

for (const item of result.items) {
  if (item.kind === "supported") {
    console.log(item.value.title)
  } else {
    console.log("This offer requires an application update")
  }
}
console.log(result.unsupportedCount)
```

Entity lists retain unsupported items as `{ kind: "unsupported", raw }` at their original positions and report `unsupportedCount`. An unknown requirement, reward, provider or policy makes the whole enclosing entity unsupported. Do not act on it or treat the list as fully supported when that count is nonzero.

Use `questExperienceResponseSchema`, `memberProgressResponseSchema`, `leaderboardListResponseSchema` and `redemptionResponseSchema` for their respective routes. `participantResponses` provides tolerant envelopes for every named participant response. `responseSchema(knownSchema)` and `responseListSchema(entitySchema)` build individual and list envelopes. Malformed known data still fails validation.

For a write, persist `options.operationId` and the matching action ID in `options.input` before sending. Unsupported financial results retain the raw result and HTTP status; an unknown status is never interpreted as fulfilled or canceled. Reconcile it before starting another operation. `response` does not create a durable submission or an action ID for you.

Older clients and legacy methods in the newest SDK still require their supported wire shapes. Installing a new SDK alone does not opt a legacy method into new closed variants.

## Sign in

Call `relay.auth.providers()` to discover configured providers, email availability, and `wallets`: the wallet families this project enabled for sign-in (`"solana"`, `"eip155"`, `"cosmos"`). For OAuth, call `relay.auth.start({ provider, mode: "sign-in" })` and navigate to its returned `url`. For email, call `emailStart(email)`, then `emailVerify({ challenge, code })` using the returned challenge. Both refuse a field they do not know with `invalid-input`. The positional forms `start(provider, mode)` and `emailVerify(challenge, code)` still work in 1.x but are deprecated. For a wallet, call `relay.auth.wallet.signIn(signer)` (see [Wallet sign-in](#wallet-sign-in)). `auth.session()` returns the current session or null; `sessionMember(session)` from `@domino-sdk/relay` is its member ID, or null for a signed-out visitor or a principal kind this SDK does not know. `auth.signOut()` clears the session. `relay.me.get()` returns the signed-in `member`, its `avatarSeed` for `defaultAvatar(avatarSeed)`, and the member's own verified sign-in email addresses: `emails` lists each email-code address, then the verified email of each Google or Apple account enabled for sign-in, and `email` is the first of them for "signed in as". A member who signs in only with a wallet, X, Discord, Telegram or an app login has none: `email` is null and `emails` is empty. Accounts connected only for activity are not listed, and no participant route returns another member's email. It also returns the member's `clock` (see [The member's clock](#the-members-clock)). A Relay older than these fields omits them.

OAuth providers and email are platform settings. Wallet sign-in is off until the project enables it: `domino api GET /participant-sign-in` shows the project's sign-in for the selected environment, and `domino api PUT /participant-sign-in body.json` with `{ "wallets": ["solana", "eip155", "cosmos"] }` sets it (publish permission). These three family codes are the complete accepted set; `cosmos` allows Cosmos wallets on any chain the app chooses. When porting a wallet-first community, enable it in the test environment first (`domino stage` and `domino doctor --remote` warn when an app that signs in with a wallet has none enabled in test), then check the `hosted-test-auth` entry of `domino api GET /preflight` or Console launch readiness. A project whose participants have no way to sign in answers 503 "Participant sign-in is unavailable".

Email codes and OAuth consent belong to the participant. Let the user complete sign-in in their browser, then verify the resulting session and action. When that has not happened, report sign-in as unverified.

### Telegram sign-in

Telegram is not an `AuthProvider` in SDK 1.x, so `providers`, `auth.accounts()`, `auth.start()` and `auth.disconnect()` never name it; its own fields and methods sit beside them ([guide](https://console.domino.run/docs/build/telegram)).

- **Methods.** `relay.auth.providers().telegram` is `{ login, miniApp }`: whether Domino configured Telegram Login, and whether this project names a Mini App bot. Older Relay versions omit it.
- **Telegram Login.** `relay.auth.telegram.start("sign-in" | "connect")` returns the `url` to navigate to, as `auth.start()` does. `useSignInOptions().telegram` (React; null when not configured) and `signInOptions(relay).telegram` (browser; absent when not configured) are the Telegram Login option to render beside `options`.
- **Mini App.** `relay.auth.telegram.miniApp.signIn(initData)` (`POST /v1/auth/telegram/mini-app`) signs in, or up, the Telegram user who opened the project's Mini App and returns the `Session`; `miniApp.connect(initData)` links them to the signed-in member. `telegramMiniAppInitData()` reads the launch data, or returns null outside a Mini App. Relay checks Telegram's Ed25519 signature for the bot the project named in Console (**Settings → Sign-in**), accepts launch data at most five minutes old from the app's own origin, and takes each launch once: `sign-in-method-disabled` (no bot), `invalid-signature` (not signed for that bot), `challenge-expired` (too old, or used), `forbidden` (another origin). React: `useSignIn().withTelegramMiniApp(initData)`.
- **Account.** `relay.auth.telegram.account()` (`GET /v1/auth/accounts/telegram`) is the member's Telegram account, shaped like an `auth.accounts()` entry with `provider: "telegram"`; `subject` is the numeric Telegram user ID. `relay.auth.telegram.disconnect()` unlinks it. React: `useTelegramAccount()` with `connect()`, `connectMiniApp(initData)`, `disconnect()` and `isConnected`.
- **Identity.** Telegram Login and Mini App sign-in reach one identity, the numeric Telegram user ID, so a participant who used both is one member.

### Creator accounts

A creator account is the platform account a member publishes from, a YouTube channel or a TikTok account, which managed tracker sources read with the member's own permission ([concept](https://console.domino.run/docs/concepts/participants#creator-accounts)). It is never a sign-in method, and SDK 1.x keeps it beside the closed provider lists.

- **List.** `relay.auth.creatorAccounts.list()` (`GET /v1/auth/creator-accounts`) is `{ accounts }`: one entry per provider Domino configured, each `{ provider, configured, status, subject, revision, profile }`, with `status` `disconnected`, `connected` or `reconnect`. `provider` is an open string; `knownCreatorAccountProviders` (`@domino-sdk/relay/schema`) lists `"youtube"` and `"tiktok"`. Unconfigured providers are absent.
- **Link.** `relay.auth.creatorAccounts.connect("youtube")` (`POST /v1/auth/start` with `mode: "connect"`) returns the `url` of Google's consent screen, where the member picks the channel; the subject is that channel's ID (`UC…`), never the Google account. `connect("tiktok")` returns TikTok's consent screen; the subject is the account's `open_id`, never the @username, and its `profile` has `name` and `imageUrl` only. It needs a member session: `mode: "sign-in"` answers `invalid-request`. Linking another account on the same platform replaces the current one.
- **Unlink.** `relay.auth.creatorAccounts.disconnect(provider)` (`POST /v1/auth/accounts/disconnect`). Records the account authored stay.
- **React.** `useCreatorAccounts()` with `connect(provider)`, `disconnect(provider)` and `isConnected(provider)`.
- **Operators.** `management.members.get(member).creatorAccounts` and `management.members.creatorAccounts(member)` list linked ones with `status` `connected` or `needs-reconnect` and a `reason`; `management.members.unlinkCreatorAccount(member, { provider, subject, revision, reason })` needs `review` and answers the rest.

## Existing application login

Mount this handler on the application's **server** at `/relay/*`. Use the app's existing session verifier. Its subject must be a stable, non-recycled account ID, not an email or browser-submitted value.

```ts
import { createParticipantBackend } from "@domino-sdk/relay/portal-proxy"
import { verifyAppSession } from "./your-existing-server-auth"

export const participant = createParticipantBackend({
  baseUrl: "https://relay.domino.run",
  origin: "https://app.example.com",
  integrationKey: process.env.DOMINO_IDENTITY_KEY!,
  authenticate: async (request) => {
    const user = await verifyAppSession(request)
    return user ? { subject: user.id } : null
  },
})
// Wire your framework's GET and POST /relay/* handlers to participant(request).
```

A Console organization administrator selects the project and environment, opens Settings → App login keys, and registers the exact HTTPS application origin. Save the key directly in server secret settings. It is shown once and never expires unless an expiry is chosen at creation (`expiresInDays`). Rotate with overlap: "Add secret" (`identityIntegrations.addSecret(id)`) issues a second secret while the first keeps working; once the old secret's "Last used" stops changing, "Revoke" it (`revokeSecret(id, secret)`). Rotation preserves participant identities and sessions. "Remove expiry" (`removeSecretExpiry(id, secret)`) makes a secret permanent; when an integration's last secret expires, every session it issued ends. The deprecated `rotate(id)` replaces every secret at once. "Disconnect" invalidates its sessions too. Recreating an integration creates a new identity namespace; rotate the existing integration to preserve identity continuity. An optional **Identity kind** (`identityKind: "app:<name>"`, set only at creation, kept by rotation) links each exchanged subject to its member as an `app:<name>` identity in the project and environment, which custom tracker records can name (`customSource(ns, { identities: ["app:<name>"] })`) and `ctx.member.identities()` lists. A kind belongs to the integration that first claimed it, also after a disconnect, which keeps the links attributing: a new integration takes a kind that has identities only with `adoptIdentitiesFrom: <old integration ID from identityIntegrations.kinds()>` and a `reason` (audited), else 409; adopted subjects sign in as their members. Reviewers unlink and relink an app identity with `management.members.identityStates.unlink` / `relink(member, { identity: { kind: "app", appKind, subject }, expectedRevision, reason })` (deprecated: `unlinkIdentity` / `relinkIdentity(member, { kind, subject, reason })`): unlinking stops attribution, ends the member's sessions for that subject, and makes the exchange answer 409 `APP_IDENTITY_UNLINKED` (passed on by `createParticipantBackend`) until relinked; a subject the integration already signs in as another member than its link answers 409 `APP_IDENTITY_CONFLICT`. Each unlink and relink raises `revision`; a relink sets a new `linkedAt`.

The adapter authenticates the app's session on every request. It exchanges the verified subject using the dedicated integration key, reuses a matching participant session, and keeps its token in a Secure, HttpOnly cookie. Account switching cannot reuse another user's participant session. Application logout removes access on the next request. Use the application's login/logout UI; this adapter does not expose Domino email, OAuth or wallet sign-in. Account linking still works for the signed-in member: `relay.auth.start({ provider, mode: "connect" })` links an X or Discord account (`"sign-in"` mode answers 403 `Use application sign-in`), `relay.auth.accounts()` and `relay.auth.disconnect(provider)` list and unlink accounts, and `relay.wallets.link(signer)` and `relay.wallets.unlink(wallet)` link and unlink wallets.

Every refusal the adapter creates has `{ error, code }`: `unauthenticated` when the app session is missing, `forbidden` for an origin or CSRF mismatch, and `route-not-found` for a path or method outside the participant API. If identity exchange fails, the adapter keeps Relay's `code` and `Retry-After` header. It passes through the two 409 app-identity messages above, but uses a generic message for every other exchange failure so provider or configuration detail does not reach the browser.

The browser still uses `createBrowserRelayClient({ baseUrl: "/relay" })`. Requests must reach the server with the configured HTTPS origin. Mutation requests require the matching Origin and X-Relay-CSRF headers, supplied by the browser client. Never cache these responses.

`doctor --remote` checks management and hosting configuration, not the app's session verifier or integration key. Verify `auth.session()`, a participant action, and access after application logout. CLI credentials and personal management tokens are not identity integration keys.

## Server routing and scope

The adapter uses `POST /v1/auth/exchange` with `{ subject, previousToken? }` and the integration key as a server bearer credential. Scope and the member role come from the registered integration, never from request input. It adds a server-only integration header when proxying participant requests. A plain exchange result is not a substitute for configuring that routing; use the adapter.

`https://console.domino.run/relay` proxies management requests only. Use the participant API origin for the adapter. For Domino-managed sign-in in a hosted preview, staging or live app, call `proxyParticipant(request, { baseUrl: "https://relay.domino.run", fetch })` from an app-owned route. It sends the app origin by default; the API resolves it against its registration and still validates sessions and CSRF. The live app's origin (`https://<project-slug>.domino.page`, once a promotion made its build live) selects the project's live environment, so members sign in to live there; preview and staging origins select test. Participant requests that no registered app or identity integration addresses receive 404. While a project's live environment is offline because its live trial and grace ended without a plan, every participant request that selects live, through the live app's origin, an identity integration or an OAuth flow, receives 403 `live-offline`; test still answers ([going live needs a plan](https://console.domino.run/docs/concepts/promotion#going-live-needs-a-plan)). The proxy deliberately excludes browser Authorization and organization/project/environment selectors.

## React hooks

`@domino-sdk/relay/react` wraps the browser client for React 19. Render `<RelayProvider>` once around the app; it creates the `/relay` client (or takes `client`/`baseUrl`), follows the signed-in member's [activity feed](#participant-activity), and keeps one query cache for the whole app. Build UI on these hooks instead of calling the client from components.

Every read hook returns `{ status, data, error, refresh }`:

- `status` is `loading`, `ready` or `error`. Hooks that need a member also return `signed-out`, and read nothing while nobody is signed in.
- `data` holds the value once `ready`. `error` is a `RelayError`: why the read failed, or, while `ready`, why the latest refresh failed; the last value stays in `data` until the next read succeeds. A 404 (a hidden profile, a board this viewer cannot see) is `error` with `error.status === 404`.
- `refresh()` reads again now and never rejects.

The cache works the same for every hook:

- Components that ask for the same read share one request and one value.
- A change reads what it affects again. While a member is signed in, the provider reads their activity feed every 5 seconds (one small request) and reads again only what each event changes, in every component that shows it (see the table below). A quest submission (`useQuest`) reads the feed and then the experience, so a completion Relay accepts at once shows its points and rewards straight away. An accepted invitation reads the referral summary and referral progress, and `useWallets`, `useAccounts` and `useMyProfile` changes their own list and the reads that follow from it. After a change made with `useRelay()` directly, call `useInvalidate()`.
- The provider reads the whole quest experience every 30 seconds, for changes no event announces (a quest published or opening) and as the fallback when the feed is unavailable. When Relay has no feed, a new completion the experience shows reads balances, point history, leaderboards, profiles, referral progress, rewards and offers again instead. Pass `polling={{ activityMs, snapshotMs }}` to change either interval.
- Polling (the activity feed, the experience, leaderboards, profiles, the referral summary, Discord role deliveries) pauses while the tab is hidden and resumes on its schedule when it is shown; the feed then catches up on what happened meanwhile. A focused window reads the feed again; for a signed-out visitor it reads the session, in case they signed in from another tab. Until the session is known, no read starts, public ones included. `useAccounts`, `useWallets` and `useMyProfile` also read again when the window regains focus.
- On a staging site, the test toolbar fires `domino:test-state-changed` on `window` after it changes test data under the page (a day advanced, a record sent, an attempt settled). The provider then reads the session and every query again, `useMe()` and its `clock` included. The event's `detail` is described in [The member's clock](#the-members-clock).
- The cache belongs to one participant of one client. Signing in or out, another account, project or environment, or a new `client` or `baseUrl` cancels every read in flight and starts an empty cache, so one member never sees another's values.

| Feed event | Reads again |
| --- | --- |
| `attempt.accepted` | The experience, quest outcomes, rewards, balances and point history, leaderboards, profiles, referral progress and offers |
| `attempt.rejected`, `attempt.review` | The experience and quest outcomes |
| `points.awarded` | Balances and point history, leaderboards, offers and `useDeliveries` |
| `entitlement.granted`, `entitlement.collected` | The experience (its `entitlements`), `useEntitlements` and `useDeliveries` |
| `delivery.updated` | `useDeliveries` |
| `referral.qualified` | Referral progress and profiles |
| A kind this SDK does not know | The experience |

| Hook or helper | Returns |
| --- | --- |
| `useSession()` | The session in `data` (null for a signed-out visitor) and its `member`. It does not depend on the experience read. |
| `useExperience()` | `data.quests`, `data.collections` (hidden quests removed), and `data.entitlements`, `signed-out` while nobody is signed in, Discord role `deliveries`, and `refresh()`. |
| `questGroups(data, order)` | Console collections in operator order, then unplaced quests sorted by `order`. |
| `useQuest(view)` | The latest view plus `submit`, `resume`, `retry`, `restart`, `isPending`, `error` (a `RelayError`), `notice`, `saved`, and `draft`, following the recovery rules below. |
| `useQuestAction(view)`, `questActionState(quest)` | `useQuest(view)` plus three display flags; `questActionState` derives them from a `useQuest()` result. `open`: show the input, because the participant can submit now or is starting a new submission after a failed one. `counted`: the submission just sent was accepted on a repeatable quest, so say so and clear the form for the next round. `automatic`: the server owns every run, so offer no input, resume, retry or restart. |
| `questStatus(view)`, `questStatusText(view)` | `questStatus` gives a `tone` and a stable `code` (`ready`, `checking`, `in-review`, `rejected`, `retry-cooldown`, `completion-cooldown`, `locked`, `streak`, …) and the evaluator's `reason` for a rejection, to localize. `retry-cooldown` and `completion-cooldown` also carry `retryAt`; both helpers take an optional `now`. Codes are an open string: render one you do not know with a neutral fallback. `questStatusText` adds English `label` and `detail`. |
| `canSubmit`, `isFinished`, `isAutomatic`, `canRepeat`, `isCoolingDown`, `progressOf` | Display rules for actions and counts. `canSubmit(view, now?)` is false while a retry or completion cooldown runs, and `isCoolingDown(view, now?)` says whether the retry cooldown runs; compare `view.nextAvailableAt` with the current time for either. `progressOf(quests)` gives `done` and `total`: locked and paused quests count toward `total`; archived and repeatable ones are left out. |
| `completionCountText(view)` | A repeatable quest's completion count in English ("Completed 3 times"), or null. For your own words, read `view.completionCount`. |
| `roleDeliveryStatus(deliveries, quest)`, `roleDeliveryLabel(status)` | A quest's Discord role as `pending`, `failed`, `granted` (an open string) or null, and its English label. |
| `useSignInOptions({ cosmos })` | Every sign-in method as one list: configured OAuth providers, installed wallets of the enabled families, and email, each with `id`, `label`, `kind` and `icon`; `start()` (email: `sendCode`, `verifyCode`); `isPending`, `error`, `reset()`, `codeSent`. Options keep their identity across renders and their actions never reject: failures land in `error`. `unavailable` reports `cosmos-chains-missing` (the app named no Cosmos chains, so none are offered), `cosmos-chains-invalid` (a chain in `cosmos` is not one, such as `"cosmoshub-4"` without `cosmos:`; other methods still load) and `no-wallet-installed` (with a message for the participant). `methodsError` says why the methods could not be read, leaving email alone. `options` is `null` until the methods are read; OAuth and email come first and installed wallets join when discovery finishes. Plain client: `signInOptions(relay, { cosmos })` from `@domino-sdk/relay/browser`, whose actions reject on failure. |
| `useSignInMethods()`, `useSignIn()`, `useSignOut()` | Configured providers, email and the project's `wallets` in `data`; email codes, OAuth redirects and `withWallet(signer)` with `isPending`, `error` and `reset()`; and sign-out in the mutation shape (`mutate()`), which drops the member's cached values once Relay confirms it. |
| `useMe()`, `defaultAvatar(seed)` | The signed-in `member`, `avatarSeed`, the member's own `email` (or null) and `emails`, and their `clock` in `data`; `defaultAvatar` draws the generated avatar as an SVG data URI. |
| `useAccounts()` | Connected accounts in `data`, with `connect(provider)`, `disconnect(provider)`, `isConnected(provider)`, `isPending`, and `actionError` until `resetAction()`. |
| `useWallets()` | The member's linked wallets for this project in `data`, with `link(signer)`, `unlink(wallet)`, `isPending`, and `actionError` until `resetAction()`; see [Wallets](#wallets). |
| `useWalletDiscovery({ cosmos })` | EVM, Solana and Cosmos wallets installed in the browser, each with a ready `signer`. Cosmos wallets only for the app's chains in `cosmos` (`{ chainId, chainInfo? }` or `"cosmos:<chain ID>"`, one or an array). |
| `useSignInWallets(methods.data?.wallets, { cosmos })` | Installed wallets of the project's enabled families, each with its `network` (family) and `signer` for `withWallet`; Cosmos wallets only for the app's chains in `cosmos`. `null` until discovery finishes, empty when none is installed. |
| `useLeaderboard(id, { limit, intervalMs })` | A board's page in `data`: rows, the participant's own row and neighbours, read again every 30 seconds by default. |
| `useMyProfile()`, `avatarSrc(avatar)` | The member's own profile in `data` with `update(input)` and `uploadAvatar(file)`, each resolving true once saved, with `isPending` (a failure lands in `actionError`); `avatarSrc` gives any profile avatar's `<img src>`. See [Profiles](#profiles). |
| `useProfile(id, { intervalMs })` | A profile's name and avatar in `data`; the uniform 404 is `error` with status 404. |
| `useLeaderboardPlacement(board, id)`, `useProfileReferral(id, program)` | One read keyed by a profile, in `data`. |
| `useProfileCompletions(id, { limit })` | A profile's completions of quests that opt in: `data.items`, `data.nextCursor`, `loadMore()`, `loadingMore` and `loadMoreError`. |
| `useBalances()`, `usePointHistory(ledger)` | The member's balances in `data.items`, read again whenever one of their quests completes; one ledger's own entries, newest first, in `data.items` with `loadMore()`. See [Point balances and history](#point-balances-and-history). |
| `usePointLedgers()`, `useTieredRewards()` | The project's ledgers and tiered rewards with their keys, in `data`. |
| `useOffers()` | The project's offers with public or member-specific availability in `data`. For a signed-in member, each offer's `missing` lists the prerequisites in its `requires` they have not met, in the shape quest `missing` uses; redeeming while it is not empty fails with `prerequisites-incomplete`. |
| `useRedeemOffer()` | The mutation shape: `mutate(offer)`, `mutateAsync(offer)`, `isPending`, `error` and the redemption in `data`; a successful redemption reads offers and balances again. |
| `useInvitation(param)` | Keeps a `?ref=` code through sign-in and accepts it once signed in: `pending`, `accepting`, `accepted`, `rejected` (Domino refused it), or `failed` (retry with `accept(code)`); both carry the `RelayError` in `error`. |
| `useReferralSummary({ path, param, intervalMs })` | The participant's code and per-program progress in `data`, and the share `link`, read again every 60 seconds by default so friends who qualify show up. |
| `useRelayQuery(key, read, options)`, `useMemberQuery(…)` | A cached read of your own in the same shape. `key` is an array of JSON values, or `null` to read nothing yet; `read` receives `{ signal, session }` to pass as the client method's last argument. `useMemberQuery` waits for a member. See [Query options](#query-options). |
| `useRelayPages(key, read)`, `useMemberPages(key, read)` | A paged read (`{ cursor }` in, `{ items, nextCursor }` out) as one list, with `loadMore()`, `loadingMore` and `loadMoreError`. A `null` key or `enabled: false` reads nothing. `useMemberPages` waits for a member. |
| `useRelayMutation(mutation, { invalidates })` | A write of your own in the mutation shape; see [Retry after an uncertain answer](#retry-after-an-uncertain-answer). |
| `useInvalidate()`, `relayKeys` | `invalidate([relayKeys.leaderboards, ["my-app"]])` reads every query under those key prefixes again; no keys means everything. |
| `useSetQueryData()` | `setData(key, value)` or `setData(key, (current) => next)` changes a cached value at once, such as an optimistic update, and cancels a read in flight for the key. Returns false for a key nobody reads. Invalidate the key once the change settles. |
| `useRelay()` | The client, for calls no hook covers. |

### Query options

`useRelayQuery` and `useMemberQuery` take these options; every one is optional.

| Option | Effect |
| --- | --- |
| `enabled` | `false` reads nothing, like a `null` key: for a read that waits on another value. The query stays `loading`, or shows `initialData`, until it is enabled. |
| `intervalMs` | Reads again this often while the page is visible. |
| `refetchOnFocus` | Reads again when the window regains focus. |
| `staleMs` | A component that shows the value again after nobody did for this long reads it again. Default 30000; `Infinity` never does. |
| `select` | Derives what this hook returns from the shared cached value, such as `(ledgers) => ledgers.map((l) => l.key)`. Pass a stable function. |
| `suspense` | Suspends the component until the first value arrives, for a `<Suspense>` boundary, and throws a failed first read to the nearest error boundary. To try again, call `invalidate([key])` before resetting the boundary. A `signed-out` member query returns as usual. |
| `initialData` | `useRelayQuery` only: a value to show before the first read. It counts as fresh for `staleMs`, and seeds the cache of the first participant the provider learns of; another participant's cache starts empty. Member reads take no `initialData`: their values come only from reads made with the member's session. |

### Server rendering and hydration

`<RelayProvider>` and every hook render on the server without reading anything: the provider starts reading when it mounts in the browser, so read hooks render `loading` on the server. To render a public value on the server, read it in your framework's server loader (a Next.js server component or `getServerSideProps`, a Remix or React Router `loader`), pass it to the component, and give it as `initialData`:

```tsx
import type { LeaderboardPage } from "@domino-sdk/relay"
import { useRelay, useRelayQuery } from "@domino-sdk/relay/react"

// The loader read `board` on the server and passed it as a prop.
export function Standings({ id, board }: { id: string; board: LeaderboardPage }) {
  const relay = useRelay()
  const standings = useRelayQuery(
    ["my-app", "board", id],
    (options) => relay.leaderboards.standings(id, {}, options),
    { initialData: board }
  )

  return <p>{standings.status === "ready" ? standings.data.items.length : "…"}</p>
}
```

The server and the browser's first render both show `board`, so hydration matches, and the browser does not read it again until it is older than `staleMs`. On the server, a `suspense` query without `initialData` throws, so React renders its `<Suspense>` boundary's fallback and fills it in the browser.

## Hosted app Worker

A hosted app's Worker handles Domino's routes first. `hostedAppRequest(request, env)` from `@domino-sdk/relay/portal-proxy` restores the public URL behind the `domino dev` tunnel; `handleHostedApp(request, env, { apiUrl })` answers `/relay/*` (the participant proxy, or the local API during development) and `domino dev` control routes, and returns null for the app's own routes. In a server route that acts for the participant, `await signedInMember(request, env, { apiUrl })` from the same entry returns the signed-in member's ID, or null for a visitor or a session that is not a member's. It asks Domino through the app's participant proxy with the request's cookies; never trust a member ID the browser sends. `serverBoundary()` from `@domino-sdk/relay/vite` fails a Vite build when browser code imports `server/`, `@domino-sdk/relay/server` or `@domino-sdk/relay-cli`. `dominoHosting({ command })` from the same entry returns `cloudflare` options for the app's own `cloudflare()` call and Vite `server` settings for `domino dev` and hosting (see the hosted reference).

### Where the app runs

`getHostEnvironment()` from `@domino-sdk/relay/browser` resolves to `{ environment: "test" | "live", kind: "staging" | "preview" | "live" }`, which Domino answers for the page's host at `/__domino/environment`, or to `null` where the host does not answer (a local dev server, a host outside Domino, server rendering). It asks once per page load. Gate test-only tools on `environment === "test"`, never on hostnames; a Worker's server code reads `DOMINO_ENVIRONMENT` instead.

### The member's clock

Relay judges a member's activity on their own clock. A test member's clock runs ahead when the staging toolbar's **Next day** or `domino test-members advance` moves it; every other member's clock is real time. An app that decides what to show by date (a campaign phase, which quests are open) compares against the member's time, so it shows what Relay judges on.

`relay.me.get()` (`GET /v1/me`) and `useMe()` return `clock: { now, offsetMs }`: `now` is the member's current time in epoch milliseconds when Relay answered, and `offsetMs` how far their clock runs ahead of real time. `offsetMs` is 0 for a live member and for a test member whose clock never moved, so app code needs no environment check. `memberTime(me, at?)` from `@domino-sdk/relay/browser` returns the member's time at the real instant `at` (default `Date.now()`), so the clock keeps ticking without another read. A Relay older than `clock` omits it, and `memberTime` then returns real time.

```ts
import { createBrowserRelayClient, memberTime } from "@domino-sdk/relay/browser"

const relay = createBrowserRelayClient({ baseUrl: "/relay" })
const me = await relay.me.get()
const phase = memberTime(me) < Date.parse("2026-10-17T00:00:00Z") ? "warm-up" : "event"
```

On a staging site the test toolbar fires `domino:test-state-changed` on `window` after it changes test data under the page. Its `detail` is `{ change, actingAs }`: `change` is `"clock"` (the member's clock moved), `"member"` (the toolbar switched or created a member; the page reloads right after), `"reset"` (the member was deleted, so `actingAs` is null until the new member's `"member"` event), `"record"` (a record sent), `"attempt"` (an attempt decided, or one the toolbar follows settled) or `"account"` (a wallet or account linked or unlinked), and `actingAs` is the test member the page is signed in as, or null. Treat a `change` you don't know like any other: the toolbar may add kinds. `RelayProvider` reads `/v1/me` again on the event; an app without it reads `relay.me.get()` again itself. The toolbar's own `/__domino/toolbar/*` endpoints are internal; read the clock from `/v1/me`.

## Display and complete quests

Use `relay.quests.observe(listener)` for the participant's experience. Render the observation's state; dispose the observer when the view unmounts. Read `relay.quests.experience()` for a single snapshot. Open a returned QuestView with `await relay.quests.open(view)`.

The controller's `kind` determines its input:

| Kind   | Action                                                                |
| ------ | --------------------------------------------------------------------- |
| claim  | `submit()`                                                            |
| photo  | `submit(file)` with a File                                            |
| quiz   | `submit(answers)` with question IDs mapped to choice IDs              |
| status | Observe only; staff and automatic actions are not participant buttons |

Submission results are `submitted` with an attempt, or `quest-updated` with `latest`, `originalDraft`, and a compatible `draft`. On quest-updated, show the new requirements and ask the participant to submit again. Submission receipt is not completion: verification or human review can still be pending.

## Recover without duplicate actions

Relay keeps each quest submission, so recovery works after a reload, in another tab or on another device signed in as the same member, and nothing is stored in the browser. The handle opens the member's submission for the quest, saves the draft into it and submits it; submitting again returns the same attempt. `controller.status()` returns `empty`, `needs-review`, `pending` (a draft saved but not submitted), or `submitted` with attempt ID. `needs-review` means the saved draft needs the participant's review, not that a moderator is reviewing it. Use `resume()` after a reload or a lost response: it submits a saved draft, or returns the attempt a submitted one admitted. A member has one unfinished attempt per quest: until it is decided (including while it waits for human review), another submit is refused with `attempt-pending`. After a rejection, a quest with a retry cooldown refuses the next submit with `retry-cooldown` until `error.retryAt`, and after an accepted completion a quest with a completion cooldown refuses it with `completion-cooldown` the same way; the draft stays saved. A draft expires 7 days after its last change. `discard()` clears an open draft. `status()`, `resume()`, `discard()` and `submit(input)` take `{ signal?, timeout? }` as their last argument (the quiz's `submit(answers, { release?, signal?, timeout? })`), on quest controllers and `me.submission` handles alike. Every request the call sends (open, save, upload, submit, retry, and a controller's read of the latest quest after `quest-updated`) gets the `signal` and what is left of `timeout`, one deadline in milliseconds for the whole call; without it each request has the client's `timeout`. A cancelled or timed-out call keeps the saved draft, so `resume()` continues it. A submission an older SDK version saved in the browser is finished first, by its saved operation: a confirmed `operation-retired` response (`RelayErrorCode.OperationRetired`) then produces `needs-review` with `reason: "operation-retired"`, including after reload. Explain that the old operation cannot continue and offer an explicit `discard()` action. Discard clears only that saved submission; show the current quest and wait for a separate submit action. Never automatically create a replacement operation. Use `relay.quests.retry(view)` only for a failed verification attempt. Do not invent a new submission to recover an uncertain response.

Read earned rewards with `relay.rewards.entitlements()` (`GET /v1/me/entitlements`) or `useEntitlements()`. Both return `{ items }`, including the earned tier, current tier, status, pickup allocation when assigned, and collection timestamps. The experience payload also includes `entitlements`. These reads include voided rewards so an app can explain a reversal. A collected reward with `recovery` needs organizer follow-up; do not present it as a new collectible reward.

Use `relay.rewards.deliveries()` (`GET /v1/me/deliveries`) or `useDeliveries()` for `{ items }` across reward kinds. A `points` item includes its ledger reference, awarded `amount`, and `reversed` amount. A `tier` item contains the current `entitlement`, one item per entitlement rather than one per historical tier improvement. An `integration` item includes provider/action and current delivery state; the list holds the member's newest 500 integration deliveries, and `relay.me.rewardDeliveries({ cursor })` pages through all of them. Reconciliation state takes precedence over historical delivery success, and provider diagnostics are omitted. Unknown kinds become `kind: "unknown"` with `reportedKind` and their JSON payload; statuses remain open strings. Display an unfamiliar state without offering a fulfillment action. Delivery reads are snapshots, ordered by the most recent known state timestamp, with undated available entitlements last.

The reward hooks share a cache scoped to the participant session and return `signed-out` without making member reads when nobody is signed in. They read again when the activity feed reports a change to them (see the [event table](#react-hooks)), and otherwise every 30 seconds, pausing in hidden tabs. Pass `QueryOptions` to change polling. `relayKeys.rewards` invalidates both reads.

Use `relay.me.observeProgress(listener)` for quest completion and rewards (entitlements). It carries no points balance: read that with `relay.me.balances()` (see [Point balances and history](#point-balances-and-history)). Dispose observers on unmount. Sign-in changes clear old participant observations. Render 401 as a sign-in request, 403 as an access issue, and transient failures with a retry that preserves the saved draft. 

### Storage failures

SDK versions before Relay kept submissions saved each one to IndexedDB, with Web Locks across tabs. The browser client (`createBrowserRelayClient`, which passes `browserSubmissionStore()`) finishes such a saved submission by its saved operation before it uses Relay's, and drops a record left after a submit. Only while it finishes one can a call reject with a `client` `RelayError` of code `submission-storage` (storage missing, blocked or unreadable: call `status()` or `resume()` again) or `submission-storage-quota` (no room: free space, then `resume()`). A browser without IndexedDB has nothing saved to finish, and submits through Relay.

## Leaderboard reads

```ts
import { createBrowserRelayClient } from "@domino-sdk/relay/browser"

const relay = createBrowserRelayClient({ baseUrl: "/relay" })
const boards = await relay.leaderboards.list()
const page = await relay.leaderboards.standings("weekly", { limit: 25 })
for (const row of page.items) {
  console.log(row.participant, row.rank, row.name, row.score)
}
console.log("My rank:", page.me?.rank)
console.log("Nearby:", page.nearby)
if (page.nextCursor) {
  const nextPage = await relay.leaderboards.standings("weekly", {
    limit: 25,
    cursor: page.nextCursor,
  })
}
```

Use `client.leaderboards.list()` and `client.leaderboards.standings(id, { limit, cursor })` for project-scoped standings. Results include `items`, `me`, `nearby`, resolved `period`, `calculatedAt`, `lastEntryAt`, and `nextCursor`. Equal scores share ranks. Restart pagination on HTTP 409; scores or eligibility may have changed. Private boards require a participant session. Public boards are explicitly published with public access. Rows show board-specific aliases and generated names unless the project makes [profiles](#profiles) visible to the viewer: then `row.profile` holds the member's name and avatar and `row.name` equals `row.profile.name`. Both the participant proxy and existing-app bridge support these GET routes.

Never expose management credentials to implement a leaderboard. Points writes and moderation use the scoped management API from trusted servers or Console. Cross-project balances, prize delivery, and finalized competition results are not supported by the leaderboard API.

Render `row.rank` as returned, rather than using the array index; ties share ranks such as `1, 2, 2, 4`. Use `row.participant` as the board-specific row key. `me` is null for anonymous or unranked participants, and `nearby` contains the member plus up to two rows on either side. No qualifying points means no ranked row, unless the board sets `includeUnscored`: then every eligible member has a row, and those without points share a rank after everyone who scored. Display an empty state when `items` is empty.

Cursors expire after 15 minutes. On HTTP 409, discard accumulated pages and fetch the first page again. `calculatedAt` is the calculation time; `lastEntryAt` is the latest received balance entry, not evidence that an external provider has synchronized. `row.profile` is null by default, for hidden members, and for viewers the project's profile settings exclude.

## Profiles

A profile is a member's name and avatar in one project and environment. Profiles are private until an operator enables them in Console **Settings → Profiles** (`private`, `participants` or `public`); the member always reads and edits their own.

```ts
import { avatarSrc, defaultAvatar } from "@domino-sdk/relay/avatar"
import { createBrowserRelayClient } from "@domino-sdk/relay/browser"

const relay = createBrowserRelayClient({ baseUrl: "/relay" })
const mine = await relay.profiles.mine()
await relay.profiles.update({
  displayName: "Ada",
  avatar: { source: "generated" },
})
const file = new Blob([], { type: "image/png" })
await relay.profiles.uploadAvatar(file)
const profile = await relay.profiles.get(mine.id)
const src = avatarSrc(profile.avatar)
const fallback = defaultAvatar(profile.avatar.seed)
const placement = await relay.leaderboards.placement("season-1", mine.id)
const completions = await relay.profiles.completions(mine.id, { limit: 20 })
const referral = await relay.profiles.referral(mine.id, "invite-friends")
```

- **`mine()`** returns `id`, `name`, `avatar`, `displayName` (null for the default), `avatarChoice`, `avatarProvider`, `hidden`, `locked`, the project's `visibility`, and `options`: the connected X, Discord and Google accounts, each with `provider`, `name`, `givenName` and `imageUrl`. `givenName` is the first name Google sent (null for X, Discord, and accounts without one); greet the member with it instead of asking. Only the member's own `mine()` carries `givenName` and Google names: `get(id)` and leaderboard rows show a Google name only if the member sets it as their `displayName`, because Google can't be a default name.
- **`update(input, { session })`** changes `displayName` (1–40 visible characters; null resets), `avatar` (`{ source: "account", provider }`, `{ source: "generated" }`, or null for the default), and `hidden`. Omitted fields stay. A locked profile answers 403 for name and avatar changes; `hidden` always works. An account the member has not connected answers 422.
- **`uploadAvatar(file, { uploadId, session })`** takes JPEG, PNG or WebP up to 5 MB and 25 MP (a wrong type or size is refused before sending, with code `invalid-input`) and stores a 512×512 WebP without metadata. The client never resends it on its own. Resending the same `uploadId` and file after an uncertain failure stores no second picture, but only before another upload: replaying an upload that a newer one replaced makes the older picture current again. It answers 503 when image storage is not configured, before any upload. A failed PUT is 502 or 504 because the object may already exist. Read the current profile before deciding whether to upload again.
- **`get(id)`** reads a profile by its public `profile_…` ID: `id`, `name` and `avatar` only. There is no composed profile page: combine the reads below. Every read keyed by a profile answers the same 404 "Profile not found" for a missing, hidden or not visible profile.
- **`leaderboards.placement(board, id)`** returns `rank`, `score` (both null while unranked or excluded) and the ranked `total` on one board, after the board's access (401 to signed-out visitors unless the board is public).
- **`completions(id, { limit, cursor })`** returns `{ items, nextCursor }`: completions grouped by quest, most recent first, with `completions` and `lastCompletedAt` (null for old undated completions); `limit` 1–100, default 20. Only quests whose newest release sets `profile: { visible: true }` in `defineQuest` appear.
- **`referral(id, program)`** returns `invited`, `qualified` and `milestones` in one program (ID or key), never codes or accounts. Referral programs are for signed-in participants: signed-out visitors get 401.
- **Avatars** carry `source` and `seed`. Uploaded and account pictures are URLs; `seed` draws the generated one with `defaultAvatar(seed)`, the fallback when a hotlinked picture fails. The seed equals `relay.me.get().avatarSeed`: an HMAC of the member under a project secret, unchanged by linking wallets or accounts.

Routes: `GET`/`POST /v1/me/profile`, `POST /v1/me/profile/avatar`, `GET /v1/profiles/:id`, `GET /v1/profiles/:id/completions`, `GET /v1/profiles/:id/referrals/:program`, `GET /v1/leaderboards/:board/profiles/:id`, and the public `GET /avatars/<file>.webp`. The participant proxy and existing-app bridge admit them; through `createParticipantBackend` every request needs the app's own sign-in. X, Discord and Google names and pictures fill in when a member next signs in or connects the account.

Operators: `management.profiles.settings()`, `management.profiles.configure({ visibility, defaults, expectedRevision, reason })`, where `defaults` is the ordered list of accounts that fill in names and pictures (`[]`, `["x"]`, `["discord"]`, `["x", "discord"]` or `["discord", "x"]`; `[]` is generated) (publish permission; live refuses it from every credential with `promotion-required`), and `management.profiles.moderate(member, { resetName, removeAvatar, lock, reason })` (review permission). With the CLI: `domino api GET /profile-settings`, `domino api POST /profile-settings body.json`, `domino api POST /members/<id>/profile body.json`. The member detail includes `profile`, and Members search matches profile names and IDs.

## Redeem offers

Call `relay.offers.list()` (`GET /v1/offers`) to show declared stock and availability. Signed-out visitors get public availability; signed-in participants also get `limit-reached` after reaching the offer's per-member limit. Treat `available`, `out-of-stock`, `limit-reached`, and `paused` as an open display state and disable redemption unless it is `available`.

Call `relay.offers.redeem(offer.id)` (`POST /v1/offers/<id>/redeem`) only for a signed-in participant. The client supplies the action ID; pass `{ actionId }` as the second argument when the app must persist it across process restarts. A successful answer is the durable redemption, including the points price and tier reward that were fulfilled. In React, `useOffers()` reads the list and `useRedeemOffer().redeem(offer.id)` performs the write, then refreshes offers and balances. Reuse the returned error: `offer-out-of-stock`, `offer-limit-reached`, `offer-paused`, and `insufficient-points` are expected participant outcomes.

Do not subtract points or stock optimistically. Relay reserves both, grants the entitlement, and settles the spend in one backend transaction. A staff-handover reward appears in the participant's entitlements for later pickup; an automatic reward is already collected.

## Point balances and history

`relay.me.balances()` (`GET /v1/me/points`) returns the signed-in member's balance in each of the project's point ledgers, in the project's ledger order, zero where they have no entries: `{ items: { ledger: { id, key, name }, balance, available }[] }`. `balance` is the current balance, every entry counted; `available` leaves out points that [reservations](https://console.domino.run/docs/concepts/points#entries-balances-and-available-points) hold, so show `available` where the member spends points. Find your ledger by key (`items.find((b) => b.ledger.key === "points")`) and show `ledger.name`.

`relay.me.pointHistory({ ledger, cursor? })` (`GET /v1/me/points/:ledger/history`) pages through the member's own entries in one ledger, named by key or ID, 100 a page: `{ items, nextCursor }`. Entries come in the order Relay recorded them, newest first, which is not always `at` order. Each entry is `{ id, amount, kind, at, quest, reverses }`:

- `amount` is signed: spending and reversals are negative.
- `kind` is an open string. The kinds this SDK knows are in `PointEntryKind` from `@domino-sdk/relay`: `award`, `spend`, `adjust` and `reverse`. Show an entry of a kind your code does not know by its amount alone. Referral payouts are adjustments: `adjust` with `quest: null`.
- `at` is when the entry took effect, in epoch milliseconds. A backdated award records now with an earlier `at`, and a reversal keeps the `at` of the entry it reverses.
- `quest` is the quest that paid it, through a completion or an engagement settlement, or null.
- `reverses` is the entry a reversal undoes.

Entries carry no operator reason or actor. An unknown ledger is 404 with code `point-ledger-not-found`, a member who has not joined the project is 404 with code `not-joined` (both in `RelayErrorCode`), and a malformed cursor is 400 `invalid-cursor`.

Both reads are the session's own member only: neither takes a member, so no participant can read another's points. They need a member session (401 `not-signed-in` without one) and are scoped to the session's project and environment. The participant proxy and existing-app bridge admit them, GET only. Operators read any member's balance and full history with `management.points.balance(member, ledger)` and `management.points.history(member, ledger)`.

In React, `useBalances()` returns the `{ items }` answer in `data` (`signed-out` while nobody is signed in) and reads again whenever one of the member's quests completes. `usePointHistory(ledger)` returns `data.items` and `data.nextCursor` with `loadMore()`; an unknown ledger or a member who has not joined is `error` with `error.code` `point-ledger-not-found` or `not-joined`.

`relay.me.leaderboardHistory({ board, cursor? })` (`GET /v1/me/leaderboards/:board/history`) pages through the member's own entries that count toward their score on one board, named by key or ID, 100 a page, newest recorded first: `{ leaderboard, period, items, nextCursor }`, with items shaped like `pointHistory` entries. The board's own scoring filter selects them (its ledger, window, `quests` and calculation; spending is left out of `earned` boards and imported opening balances count on `balance` boards only), so every page together sums to the member's score. `period` is the window the first page counted; the cursor keeps it while paging. A member the board excludes, or one outside its group, gets empty `items`, as their placement is unranked. It needs a member session on every board, public ones included (401 `not-signed-in`). An unknown board is 404 `not-found`, an archived one 404 `leaderboard-archived`, and a malformed cursor, or one from before the board was republished, 400 `invalid-cursor`. React: `useLeaderboardHistory(board)`, with `loadMore()`, reads again when one of the member's quests completes.

## Referral invitations and progress

Use `relay.referrals.summary()` for the signed-in member's invitation code, attributed and qualified counts, milestone progress, and earned/settled/outstanding accounts. Use `accept({ code, actionId })` after sign-in and before the first quest completion. Preserve the code through sign-in and reuse input on uncertain retries. `history({ cursor? })` reads paginated payout activity as `{ items, nextCursor }`. Read [referrals](referrals.md) for authoring, qualification, corrections, and verification. The participant proxy and existing-app bridge route these operations with session-derived identity.

## Connected accounts

Project integrations and participant identities are separate. An organization administrator connects Discord in Console **Integrations**, selects a server, and installs the managed bot. Each project/environment has one server binding. See [project integrations](https://console.domino.run/docs/build/integrations).

Participants who sign in with Discord already have the identity needed for verification. If they use another login, keep their current member signed in, call `relay.auth.start({ provider: "discord", mode: "connect" })`, and navigate to the returned `url`. This requests only `identify`, not membership-reading access. `relay.auth.accounts()` lists the provider's configuration, link status, subject, and identity revision. It never returns credentials. Quest views expose required participant identities in `connections`, derived from their declared integrations.

To choose a different identity, first call `relay.auth.disconnect("discord")`. This unlinks the active Discord identity in the project and environment. Ownership, sign-in eligibility, and earned rewards stay intact; another member cannot claim the identity. It does not disconnect the project's Discord server. Both operations retain historical verification facts.

The campaign starter includes these account controls. Apps using `createParticipantBackend` retain their own login and use the same APIs. The adapter preserves OAuth binding cookies and its session during return navigation; the application's own login cookie must also support top-level OAuth returns. If the app session changes during consent, start connection again.

After changing an account or integration during a failed attempt, use the quest controller's `discard()` to clear the saved failed submission, then submit again. The starter labels this **Start a new submission**. Discard is allowed after a confirmed failed or rejected attempt, or a saved operation explicitly retired by the server. A keyed quest controller can also discard a confirmed accepted browser submission to begin another occurrence. Server history and completion keys remain intact. Active attempts cannot be discarded. Use `resume()` when the connection has not changed to preserve captured facts.

## Wallets

A signed-in member links Solana, EVM and Cosmos SDK wallets to the current project by signing a challenge. `relay.wallets.link(signer)` reads the session, asks the signer for its address, requests a challenge, has the wallet sign the returned message, and verifies it. It returns the `LinkedWallet`: `chain` (`"solana"`, `"eip155"` or `"cosmos"`), `network` (a Cosmos wallet's chain as `cosmos:<chain ID>`, the one the app requested and not verified by Domino; otherwise null), the canonical `address` (base58, EIP-55 checksummed, or lowercase bech32 with the chain's prefix), `verifiedAt`, `revision` and `signIn`.

Signers come from `@domino-sdk/relay/browser` and need no wallet SDK:

| Wallet                                         | Signer                                                                                                                    | Discovery                                                                                                                                    |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| MetaMask, Rabby and other injected EVM wallets | `evmSigner(provider)`: EIP-191 `personal_sign`                                                                            | `discoverEvmWallets()`: EIP-6963 announcements, else `window.ethereum`                                                                       |
| Phantom, Solflare, Backpack                    | `walletStandardSolanaSigner(wallet)` (Wallet Standard `solana:signMessage`) or `solanaSigner(provider)` (legacy injected) | `discoverSolanaWallets()`: Wallet Standard first, then `window.phantom.solana`, `window.solflare`, `window.backpack.solana`, `window.solana` |
| Keplr, Leap, Cosmostation                      | `cosmosSigner(provider, chain)`: ADR-36 `signArbitrary` on the app's chain                                                | `discoverCosmosWallets(chain)`                                                                                                               |

```ts
import {
  createBrowserRelayClient,
  discoverCosmosWallets,
} from "@domino-sdk/relay/browser"

const relay = createBrowserRelayClient({ baseUrl: "/relay" })
const [keplr] = discoverCosmosWallets({ chainId: "cosmoshub-4" })
if (keplr) await relay.wallets.link(keplr.signer)
const wallets = await relay.wallets.list()
```

Domino has no Cosmos chain list: `chain` is `{ chainId, chainInfo? }` or `"cosmos:<chain ID>"`, any chain ID of 1 to 32 of `[-_a-zA-Z0-9]`, with any bech32 prefix. When the wallet answers that it does not know the chain, the signer calls `experimentalSuggestChain(chainInfo)` and retries if `chainInfo` (Keplr's `ChainInfo`, same `chainId`) was given; without `chainInfo` it throws "<wallet> doesn't know the Cosmos chain <chain ID>. Add the chain to the wallet, or pass its chainInfo so the wallet can suggest it.", and with a wallet that lacks `experimentalSuggestChain` it throws "<wallet> doesn't know the Cosmos chain <chain ID> and cannot add chains. …". Other refusals, such as a rejected connection, pass through unchanged and suggest nothing. In React, `useWalletDiscovery({ cosmos })` lists installed wallets and `useWallets().link(wallet.signer)` links one and refreshes every `useWallets()` in the app. 32-byte Cosmos module and contract addresses are refused ("Use a Cosmos account address, not a contract"). Ethermint-key wallets (Evmos, Injective) are refused, and so are smart-contract wallets: only EOA signatures verify.

- A member has at most five active linked wallets per chain family in a project; the five Cosmos wallets can be on any chains, and sign-in wallets do not count. A wallet is its address: one key on chains with different bech32 prefixes is separate wallets, and relinking an address on another chain ID with the same prefix moves it to that chain. Linking a sixth answers 409 "Unlink a wallet before linking another (5 EVM wallets at most)". Relinking an active wallet refreshes it.
- `relay.wallets.unlink({ chain, address })` removes a wallet from the project's list and answers `{ ok: true }`, also for unknown or already unlinked wallets. The address stays bound to the member in the project and environment: another member can never link it, and the member can link it again.
- An address linked by one member fails for another with "This account belongs to another member. Accounts were not merged."
- Linking is not sign-in: `signIn` is false for linked wallets, and a linked wallet cannot sign in unless an operator enables it.
- Lists are per project and environment. Quests read them through `ctx.wallet.list()`, pinned once per attempt.
- `relay.wallets.challenge(input, { session })` and `verify({ challenge, signature, publicKey? }, { session })` are the two steps `link` runs, for apps with their own signing flow; pass the same `session` to both. Both retry transient failures (see [Client conventions](#client-conventions)): a repeated verify of the same challenge, signature and session returns the wallet it linked, and a repeated challenge only issues another. Challenges expire after 5 minutes, and the request `Origin` must be a registered app origin.
- Token balances are evaluator-only: quests read them with `ctx.wallet.balance()` (see the authoring reference), and the participant API has no token balance route. To show a chain name for a quest's `eip155:<id>` setting, look it up from your server in `management.walletBalanceChains()` (`read` permission; a hosted app's own key has it); `formatUnits(amount, decimals)` formats amounts.

Operators see linked and unlinked wallets on the member in Console, and a reviewer can unlink or relink one with a reason (`management.members.identityStates.unlink` / `relink(member, { identity: { kind: "wallet", chain, address }, expectedRevision, reason })`, or `domino members unlink-identity` / `relink-identity`; deprecated: `management.members.unlinkWallet`). The change is in the audit log.

### Operator OAuth unlink

Read `management.members.identityStates.list(member)` (`GET /members/<id>/identity-states`, `read` permission) for every identity the member proved: wallets, X, Discord and Telegram accounts, and app identities. Each has its selector fields, `revision`, `linked`, `linkedAt`, `signIn` (wallets only), `canUnlink` and `canRelink`. An X, Discord or Telegram account shares its provider's selection `revision`, since the member selects one account per provider. The deprecated `management.members.oauthAccounts(member)` reads only the current X and Discord selections.

Call `management.members.identityStates.unlink(member, { identity: { kind: "oauth", provider, issuer, subject }, expectedRevision, reason })` with the account you reviewed (`POST /members/<id>/identity-states/unlink`, `review` permission; deprecated: `unlinkOAuthAccount(member, { provider, subject, revision, reason })`). The reason is required and limited to 500 characters. Console provides the same action under a member's **Identities**. The member must belong to the selected project; the unlink affects the project and environment. Identity ownership and sign-in grants remain, as do previously admitted activity and rewards. The member can connect again; another member cannot take ownership.

If the selected subject or revision changed, the write fails with 409. Read the selections again and review the change before retrying. To retry an unlink whose response was lost, send the same `actionId` with the same input: it answers the first result. A new `actionId` after the unlink succeeded is refused with 409, as the account is no longer linked at that revision; the deprecated `unlinkOAuthAccount` instead treats a repeat as harmless while its disconnected revision remains current. Proof callbacks started before the unlink cannot restore the old selection.

### Operator Telegram unlink

A Telegram account is `{ kind: "telegram", subject }` in `identityStates`, unlinked and relinked with the same rules and audit as X and Discord. Console shows it under **Identities**. The deprecated `management.members.telegramAccount(member)` and `unlinkTelegramAccount(member, { subject, revision, reason })` read and unlink only the current selection.

### Wallet sign-in

Where the project enabled the wallet's family, `relay.auth.wallet.signIn(signer)` asks the signer for its address, requests a sign-in challenge (`POST /v1/auth/wallet/start`), has the wallet sign it, and verifies it (`POST /v1/auth/wallet/verify`), which sets the session cookie and returns the `Session`. A new address creates a new member; the same wallet signs them in again, and signing in links it to the project as a sign-in wallet (`signIn: true`). `auth.wallet.start(input)` and `auth.wallet.verify({ challenge, signature, publicKey? })` are the two steps for apps with their own signing flow. Verify is not retried: the participant starts again. React: `useSignInOptions({ cosmos })` offers only installed wallets of enabled families, and Cosmos wallets on the app's chains; each wallet option's `start()` signs in. The starters' sign-in components use it, including for projects whose only method is wallet sign-in; they name no Cosmos chains until you add them to the `useSignInOptions()` call in `src/components/sign-in.tsx`.

Under `RelayProvider`, a session signed in with `useSignIn().withWallet(signer)` or a wallet option's `start()` follows an EVM wallet. When the wallet switches to another account (`accountsChanged`, compared case-insensitively) or reports none (locked or disconnected), the provider signs out as `useSignOut()` does: the member hooks report `signed-out` and the app shows its sign-in again. It never signs the new account in by itself. The same account, including the event wallets send on first connect, changes nothing. After a reload the provider finds the wallet again by name, checks `eth_accounts` without a prompt, and keeps following it for the tab's session. Solana and Cosmos wallets are not followed. Without React, `evmSigner(provider).watchAccounts(listener)` reports the wallet's accounts now and on each change, selected account first, and returns the function that stops it; call `relay.auth.signOut()` when it reports another account.

```ts
import {
  createBrowserRelayClient,
  discoverSolanaWallets,
} from "@domino-sdk/relay/browser"

const relay = createBrowserRelayClient({ baseUrl: "/relay" })
const { wallets } = await relay.auth.providers()
const [phantom] = discoverSolanaWallets()
if (phantom && wallets.includes("solana"))
  await relay.auth.wallet.signIn(phantom.signer)
```

- 403 "Wallet sign-in is not enabled for this project" when the wallet family is off, also for a challenge issued before it was turned off.
- 409 "This wallet is linked to an account but is not a sign-in method. Sign in with your original method." for a wallet linked while signed in. 409 "Sign out before switching accounts" with a session.
- 429 after 10 challenges per address or 30 per client IP in 10 minutes. A challenge lasts 5 minutes, allows 5 verify attempts, and works once; a wrong signature leaves it usable.
- Identity-integration apps cannot use wallet sign-in; their participants can still link wallets.
- Unlinking does not stop a sign-in wallet from signing in. A reviewer can disable or enable sign-in for a member's wallet with a reason (`management.members.identityStates.setSignIn(member, { identity: { kind: "wallet", chain, address }, enabled, reason })`, or `domino members identity-sign-in`; deprecated: `management.members.setWalletSignIn`), in the project and environment; sessions already issued end within 8 hours.
- Seed shared avatars with `avatarSeed` from `relay.me.get()`, never with an address: `walletAvatarSeed(wallet)` is for a member's own wallet list only.

## Resource IDs

Treat resource IDs as opaque strings and store the complete value. New generated IDs keep their resource prefix, such as `member_`, `session_`, or `attempt_`, followed by 22 random alphanumeric characters. Existing IDs remain valid. Do not parse the suffix as a UUID.

## Repeatable and hidden quests

Quest views include `completion`, `completionCount`, `visibility`, `campaign`, `window`, and `budgets`. For keyed quests, `completed` means at least one past completion; it is not terminal. Use `availability` for the current action and display the count separately. Filter hidden quests from participant lists and collections, including direct quest routes. The campaign starter does this. Visibility is not an authorization boundary.

`availability.kind === "locked"` includes typed `missing` prerequisites. Completion entries name the quest and either lifetime or project-calendar period scope. Balance entries include `minimum` and current balance. Group entries name the group. `availability.kind === "blocked"` carries a stable `reason`: `budget-exhausted`, `connection-required`, `wallet-required`, `paused`, `outside-window`, or `enrollment-required`, plus `resetsAt` when a calendar budget or future window has a known next boundary. An `outside-window` result can include `window: "quest"`; tracker outcomes use `"tracker"` for their own collection window, and older responses omit the field. Each `budgets` entry gives its ID, scope (`member` or `project`), period, current remaining capacity, and reset time (`null` for lifetime). Budgets with `scope: "author"` are left out: they cap what each record author gives, not what the member can earn. Use `questStatus(view)` for a localizable code or `questStatusText(view)` for English.

`retryAt` (Unix milliseconds) is set while the quest's retry cooldown runs after the member's rejected attempt; `availability` stays `available`, but a submit before then fails with `retry-cooldown` and the same `retryAt` on the `RelayError`. Show a countdown from it. `useQuest` renders again when the cooldown ends. See [retry cooldowns](authoring.md#set-a-retry-cooldown).

`nextAvailableAt` (Unix milliseconds) is when the member may next submit, while a retry cooldown, a [completion cooldown](authoring.md#set-a-completion-cooldown) or an exhausted budget's reset holds the quest: the latest of these. Use it for countdowns. It is absent when the member can submit now, or when nothing will reopen the quest (a used-up lifetime budget, a window that closes first). A submit before then fails with `retry-cooldown` or `completion-cooldown`, or the budget's block.

Automatic quest controllers remain status-only. Do not offer an ordinary submission or retry button, including after completion or execution failure. Their admitted attempts recover on the server. Tracker-triggered quests can expose enrollment, the member's outcomes and historical claims ([below](#automatic-quest-enrollment-outcomes-and-claims)); content quests also have the separate content release flow. Keyed manual controllers may start another submission after a confirmed success; uncertain submissions still resume their original action ID.

## Automatic quest enrollment, outcomes and claims

Tracker-triggered quests expose these participant operations through both hosted proxies and existing-app identity integrations. No input names a member: Relay uses the session's.

| SDK method              | Public route                  | Input                                              | Answer                    |
| ----------------------- | ----------------------------- | -------------------------------------------------- | ------------------------- |
| `relay.quests.enroll`   | `POST /v1/quests/enrollments` | `{ actionId, quest }`                              | `{ quest, member, enrolledAt }` |
| `relay.quests.outcomes` | `GET /v1/quests/outcomes`     | `{ quest?, state?, code?, cursor? }`               | `{ items, nextCursor }`   |
| `relay.quests.claim`    | `POST /v1/quests/claims`      | `{ actionId, quest, tracker, record, observedAt }` | the outcome               |

- **Enroll** in a quest whose trigger has `enrollment: "required"`. Repeating an active enrollment keeps its `enrolledAt`; re-enrolling after operator withdrawal starts a new consent time. A quest without required enrollment refuses with code `enrollment-not-required`, a paused quest with `quest-paused`. Enrolling decides the member's `not-enrolled` skips again shortly after.
- **Outcomes** are the decisions Relay attributed to the member, newest first, 50 a page: `{ quest, tracker, record, observedAt, state, code, reason, attempt, claimable, createdAt, updatedAt }`. `state` is `admitted` or `skipped`; `code` is an open string (known ones in `trackerSkipCodes` from `@domino-sdk/relay/schema`; show an unknown one by `reason`). A malformed cursor is 400 `invalid-cursor`. In React, `useQuestOutcomes({ quest?, state?, code? })` pages it.
- **Claim** an outcome whose `claimable` is true: a `historical` skip on a quest with `historical: "claim"`. Relay decides it again with historical activity released and answers the outcome, now `admitted`; a rule that still fails rejects with its skip code (such as `prerequisites-incomplete`), and a skip that is not claimable with `not-claimable`. Another member's outcome is 404 `not-found`. Keep the action ID for an uncertain retry.

After enrolling or claiming in React, call `invalidate([relayKeys.outcomes])` from `useInvalidate()`.

## Campaign enrollment

A campaign enrollment covers every current and future quest carrying the campaign key. It is separate from a tracker-trigger quest's own enrollment above.

| SDK method | Public route | Input | Answer |
| --- | --- | --- | --- |
| `relay.campaigns.enroll` | `POST /v1/campaigns/enrollments` | `{ actionId, campaign }` | `{ campaign, member, enrolledAt, withdrawnAt, withdrawalReason }` |
| `relay.campaigns.withdraw` | `POST /v1/campaigns/enrollments/withdraw` | `{ actionId, campaign }` | `{ campaign, member, enrolledAt, withdrawnAt, withdrawalReason }` |

The participant routes always take the member from the signed-in session. The first enrollment time is durable; an exact or later repeat returns it. Withdrawal sets `withdrawnAt`, blocks new campaign admissions, and preserves existing attempts, completions, and rewards. Calling `enroll` again clears the withdrawal fields without changing `enrolledAt`. Enrollment is allowed before a campaign opens but not after it ends. A campaign with `enrollment: "none"` returns `enrollment-not-required`. A trusted server uses `management.campaigns.enroll({ actionId, campaign, member })` or `management.campaigns.withdraw({ actionId, campaign, member, reason })` with `enroll` permission. Refresh the experience after either action so every campaign quest receives its new availability.

## Content quest enrollment and historical claims

Content quests expose separate participant operations through both hosted proxies and existing-app identity integrations:

| SDK method                     | Public route                     | Input                                  |
| ------------------------------ | -------------------------------- | -------------------------------------- |
| `relay.content.performance`    | `GET /v1/content/performance`    | `{ quest, content, cursor? }`          |
| `relay.content.qualifications` | `GET /v1/content/qualifications` | `{ quest, content?, cursor? }`         |

Content quests use the shared `relay.quests.enroll`, `relay.quests.outcomes` and `relay.quests.claim` operations above. Claims use the outcome's `tracker`, `record` and `observedAt`; no input names a member. Relay uses the signed-in session. A claim releases already-collected content only when its current quest policy permits claims and all other admission rules pass. It cannot override ownership, enrollment, dates, pause or prerequisites. Keep the action ID and input for identical retries.

Read the member's outcomes to show the skip reason or admitted attempt rather than sending an ordinary quest submission. Reviewer retries use the management API and are unavailable through participant proxies. Qualification and performance work for X posts and custom sources that declare a content body and metrics. See [content admission](https://console.domino.run/docs/build/trackers#admit-x-posts-automatically) for policy and current limits.

Read `content.qualifications` to follow edits and evaluation history. Pages contain at most 100 evaluations, newest first; continue with `cursor: page.nextCursor`, preserving filters. `current` identifies the latest evaluation per post. Display its `state` and `reason` separately from `reward.state` (`unresolved`, `awarded`, or `excluded`). A post can become unqualified or unavailable while retaining an earlier award. Edits keep the original release and member, and stale reviews cannot issue rewards. History stays with the original beneficiary after account relinking; member overrides are rejected. These status reads do not submit or claim content.

For quests with a performance stage, read `content.performance({ quest, content, cursor? })`. `current` separates content pending/rejected, metric waiting, execution/review, and qualified/unqualified states. Show its `reason` separately from its one-time `reward` outcome. `items` retain up to 100 metric samples and their decisions, newest first; preserve quest/content filters when continuing with `cursor: page.nextCursor`. Missing or stale counters wait, and a later threshold drop does not reverse an earlier award. Content reviews remain in `content.qualifications`; body changes and changes to the verified account selection rerun those checks.

### X verification accounts

For X quests, keep the participant signed in and call `relay.auth.start({ provider: "x", mode: "connect" })`, then navigate to the returned URL. `relay.auth.accounts()` exposes the X link status; `relay.auth.disconnect("x")` unlinks it. X sign-in also links the identity. Provider configuration must be enabled by the platform operator.

An app must offer this connection action itself: X evaluator reads do not add an automatic quest `connections` requirement. The quest checks `x.connection()` and can explain that linking is needed. After linking, changing accounts, or completing a previously failed follow/repost action, start a new submission. Existing captured facts remain attached to their attempt. X wrapper credentials belong only to the backend.

## Discord reward delivery

Call `relay.me.rewardDeliveries()` to read role deliveries for the signed-in member. The backend route is `GET /v1/reward-deliveries`; it is available through the participant proxy and existing-app identity integration. The response is a page of up to 500 of this member's deliveries, newest first (`limit` sets fewer); pass its `nextCursor` back as `relay.me.rewardDeliveries({ cursor })` for older ones until it is `null`. Records include `quest`, `completion`, `roleId`, `action`, and `status` of `pending`, `delivered`, or `failed`. `action` is an open string, `grant-role` today; show a delivery with an action you do not know by its status. A grant still pending when an operator revokes its completion fails. Use optional `reconciliation` for current membership: `pending`, `granted`, `removed`, `preserved` or `failed`. Original `status` remains delivery history. Relay removes its own assignment when no accepted completion across any project/environment still grants it; preexisting and explicitly preserved assignments stay. Display delivery independently of quest completion, since Discord can be temporarily unavailable after points have been awarded. Do not display internal `lastError` details to participants.

## Removed demo shortcuts

`auth.demoSignIn` and the campaign template endpoint are no longer supported. The participant client has no staff reads: `reviews.list` and `members.list` were removed because the participant API accepts only member sessions. List members from server code with the management client's `members.list({ q, limit })` or `members.get(id)`, and review pending attempts in Console **Activity**. Use managed sign-in or a registered identity integration. The participant client has no `me.linkPass`, and the participant API has no pass route (`/v1/me/pass` is 404). To verify passes, an app adds a route of its own, ahead of its `/relay/*` handler, and verifies the pass on its server, then calls `POST /management/v1/passes/verify` with a server-only management key carrying `confirm` permission. Send `{ provider, event, subject, member, quest }` and the usual organization, project, and environment headers; `quest` is the quest's ID or key. Resolve the member from the authenticated participant session, never from browser input. The quest must be staff-confirmed with no evidence (403 otherwise; an unknown quest is 422). The member must already exist in that project and environment (404 `not-found` otherwise). Relay binds ownership uniquely per provider/event in that project, preserves the authenticated management actor, and retries the same entry attempt. There is no built-in event-code verifier.

## Resource references

Store returned resource IDs as opaque strings. A quest's `quest` field and a collection's `id` are server-assigned. Authored resources also expose their stable `key`; `relay.quests.get(key)` and `relay.quests.collection(key)` resolve it. Console-created quests have no authoring key, so use their returned IDs. Continue supplying stable action IDs for retries; those values do not become resource IDs.

References inside a read are IDs too: a reward's or account's `balance`, a referral `program`, and `qualification.quest`. Never compare them with an authoring key, and never render them. Match a top-level resource by its `key` (`definition.key === "friends"`), then compare IDs. For ledgers, call `relay.pointLedgers.list()` (`GET /v1/point-ledgers`) or `usePointLedgers()`, which return `{ id, key, name }`; take the ID for your key and display `name`. Tiered rewards in entitlements and reward definitions carry their stored ID in `reward.id`: call `relay.tieredRewards.list()` (`GET /v1/tiered-rewards`) or `useTieredRewards()` for `{ id, key }` and match the key you gave `defineTieredReward`. Both reads are open to signed-out visitors, and the hooks return the list in `items`. Quest view rewards are typed: `{ kind: "points", label, amount, ledger: { id, key, name } }`, `{ kind: "tier", label, reward: { id, key, title }, tier }`, or `{ kind: "integration", label }`. Read `amount` rather than parsing `label`. A group membership reward (`groupMembership(group)`) is a `tier` reward whose `reward.group` is `{ key, title }`: its key is the group key (groups have no stored ID) and `label` is the title the quest's code gives the group. In tests, give fixtures IDs that differ from their keys, as Domino's answers do.

### Daily engagement history

Use `relay.engagement.page({ rule, content, cursor? })` (`GET /v1/engagement/history`) with the participant session. History remains with the originally admitted beneficiary; another member's account returns not found and member overrides are denied. The page includes the rule, the last processed account checkpoint, and up to 100 daily settlements newest first. Continue with `cursor: page.nextCursor`, preserving rule/content. Display `pending`, `paid`, and permanently budget-`excluded` points separately. Receipts include their observation, qualification attempt, earned/growth/paid/excluded amounts, UTC day, and nullable point entry. This read does not trigger settlement; participant APIs cannot publish engagement rules.

Use `relay.engagement.boundaries({ rule, content, cursor? })` (`GET /v1/engagement/boundaries`) to follow formula changes. `current` is the account's governing version; `requested` is the latest requested version, which may still be pending. Up to 100 items report `from`, `to`, high-water `baseline`, provider `observedAt`, and the settlement receipt. Continue with `cursor: nextCursor`. Boundaries are available fresh observations, not exact publication-time measurements. A boundary may pay in addition to that day's daily settlement, sharing its budget. The original beneficiary owns this history; member overrides are denied. `engagement.page()` keeps its response shape and includes boundary credits in its receipt history.

## Submit text or URLs

Render a textarea for `view.interaction === "text"` and a URL input for `"url"`. The manual trigger carries `constraints` for required input, text lengths, or allowed hosts. Use the quest controller so the value is validated before it is saved, and the backend checks the pinned publication again.

```ts
import type { QuestView } from "@domino-sdk/relay"
import { createBrowserRelayClient } from "@domino-sdk/relay/browser"

const relay = createBrowserRelayClient({ baseUrl: "/relay" })
async function submitValue(view: QuestView, value: string) {
  const controller = await relay.quests.open(view)
  if (controller.kind === "text" || controller.kind === "url") {
    return controller.submit(value)
  }
}
```

Low-level handles are `relay.me.submission(quest, { input: "text", release })` and the equivalent with `input: "url"`. Call `submit(value)` once, then `resume()` after a lost response or reload. Relay keeps the value as the submission's draft before it is submitted. A failed evaluation retries with the same values and release. A definitive `invalid-submission` response admits no attempt and leaves the draft open: call `submit(value)` again with the corrected value, which replaces it.

Show validation errors next to the form. When a publication changes, show the updated quest and the recovered draft for review before submitting again. Declare `text` and `url` in `relay.json`'s `supportedInteractions` only after the app renders them. Existing photo handles, uploads, and saved records keep their format.


## Participant activity

Read committed asynchronous outcomes with `relay.me.activity.list({ cursor?, start?, limit? })` (`GET /v1/me/activity`). The response is `{ member, items, nextCursor }`. Events cover accepted, rejected and review-pending attempts, point awards, granted or upgraded entitlements, collected entitlements, delivery updates and referral qualification. Each event has an immutable `id`, a `kind` and an `at` timestamp. Provider diagnostics, review notes and other members' referral identities are omitted.

Read the next page with `nextCursor`, even when `items` is empty. Without a cursor the feed starts at the member's first event; `start: "latest"` instead returns no events and the cursor of the newest one, to follow only what happens from now on. Pass `cursor` or `start`, not both. The cursor stays usable for later polling and belongs to one member, project and environment. Pages default to 50 events with a maximum of 100, in transaction order. A transaction that is still running can delay newer events; committed events are never skipped because an older transaction finishes late. Activity begins when the activity-feed migration is installed. Earlier state remains readable through progress, rewards and point history; the migration does not reconstruct historical events.

```ts
import { createRelayClient } from "@domino-sdk/relay"
const relay = createRelayClient({ baseUrl: "https://your-app.example" })
const watch = relay.me.activity.observe((event) => {
  if (event.kind === "attempt.accepted") {
    console.log(event.quest, event.attempt)
  }
})
// Dispose when the owner unmounts.
watch.dispose()
```

The observer defaults to five-second cursor polling, drains full pages immediately, pauses while the page is hidden and resumes on schedule when visible. Each poll is one request bound to the participant session: Relay refuses it when the session changed, and the observer then reads the session again and resets its cursor. Pass `cursor` to resume, `start: "latest"` to skip earlier events (also after a session change), `intervalMs` to change polling, and `onState` to handle loading, session, ready and error observations. Persist a cursor from the ready state's `progress.nextCursor` only after processing its events. Callback failures leave the prior cursor in place so the page can be retried; use event IDs to make your own effects idempotent.

In React, `<RelayProvider>` already follows the feed from the latest event and reads again the hooks each event changes; you do not need `useActivity` to keep data current. Mount `useActivity({ onEvent })` once for app-wide notifications. It returns the same loading/ready/error/signed-out states as other member hooks, retaining the most recent 100 events in `data.items` and exposing `refresh()`. It removes the previous participant's data on session changes. Unknown event kinds become `kind: "unknown"` with the original `reportedKind` and JSON fields. Treat them as informational until the app supports them. Snapshot hooks remain available for current state; activity reports changes and does not replace authoritative reward or quest reads.

Prerequisite-only automatic decisions appear in `relay.quests.outcomes()` and `useQuestOutcomes` alongside tracker outcomes. Such a row has an `automatic` outcome ID, null `tracker`, `record` and `observedAt`, and `claimable: false`. Use `automatic` for its list key; only claim rows with `claimable: true`. Show unknown skip codes through their `reason`. Balance changes schedule asynchronously after commit, so refresh outcomes/progress to observe the resulting attempt.

## Render quest and collection presentation

`GET /v1/me/experience`, `relay.quests.experience()` and `relay.quests.get(id)` return optional `view.presentation.steps`, `image`, `imageAlt` and `requirements`. Steps retain author order and have `{ title, description?, action?: { kind: "link", url, label } }`. Render link actions as HTTPS links, and submit evidence through the quest controller separately. Images use HTTPS or a same-site path. Account/wallet hints describe what to link; combine them with the existing `view.connections` for declared X/Discord integrations.

Collections return optional `description`, `image` and `imageAlt` next to `title`, with quests in published Console placement order. `useExperience().data` retains these quest and collection fields; `useQuest(view).view` retains quest presentation after an update; `questGroups()` retains collection metadata and placement order. All fields are optional, so apps can render older quests and collections without placeholders.

## Badges

Read the active project catalog, the signed-in member's awards, or a visible profile's awards:

```ts
import { createRelayClient } from "@domino-sdk/relay"

const relay = createRelayClient({ baseUrl: "/relay" })
const profileId = "profile_example"
const catalog = await relay.badges.list()
const own = await relay.me.badges()
const visible = await relay.profiles.badges(profileId)
const firstStep = own.items.find(item => item.badge.key === "first-step")
```

`own.items` and `visible.items` contain `{ badge, awardedAt }`. Badge metadata includes `id`, `key`, `name`, `description` and `image`. Archived definitions remain on existing awards but disappear from the active catalog. Profile reads follow project visibility and the member's hidden setting; owners can read their own profile's badges.

React hooks use the same reads under your existing `RelayProvider`:

```tsx
import { useBadgeCatalog, useBadges, useProfileBadges } from "@domino-sdk/relay/react"

function Achievements() {
  const awards = useBadges()
  if (awards.status !== "ready") return null
  return <ul>{awards.data.items.map(({ badge }) => <li key={badge.id}>{badge.name}</li>)}</ul>
}
```

`useBadgeCatalog()` reads the active catalog; `useProfileBadges(profileId)` follows profile visibility. `useBadges()` reads again after quest completion. Call the hook's `refresh()` to reread after an operator correction. Badge rewards in quest experience reads carry the catalog record at `reward.badge` inside the stable `kind: "tier"` envelope.

HTTP reads are `GET /v1/badges`, `GET /v1/me/badges` and `GET /v1/profiles/:profile/badges`. The catalog is public within project scope; own reads require a member session. Profile reads apply the same authorization as other profile endpoints.
