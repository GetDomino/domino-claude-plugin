# API error codes

Every refusal from the participant API (`/v1`) and the management API (`/management/v1`) carries a stable machine-readable `code` beside its human-readable message. Branch on `code` and the HTTP status, never on the message: messages are prose and may change in any release.

## The error body

```json
{
  "error": "Complete the required quests first",
  "code": "prerequisites-incomplete",
  "missing": [
    {
      "kind": "completion",
      "quest": "join",
      "title": "Join the waitlist",
      "scope": "ever"
    }
  ]
}
```

| Field        | Meaning                                                                                                   |
| ------------ | --------------------------------------------------------------------------------------------------------- |
| `error`      | A message for people. It may change.                                                                      |
| `code`       | A stable code: one of those below, or a code a newer Relay added.                                         |
| other fields | Structured detail some refusals add, such as `missing` prerequisites or the `members` of an ambiguous match. |

Each client reads the same body:

- **Participant client:** a `RelayError` with `code` and, for prerequisites, `missing`. `RelayErrorCode` from `@domino-sdk/relay` names the codes participant apps branch on most.
- **Management client:** a `ManagementError` with `code`, and the other fields in `details`. `managementErrorSchema` from `@domino-sdk/relay/server` parses the body of a raw `api.raw()` answer.
- **CLI:** with `--json`, the error's `apiCode` (see the [CLI automation contract](automation.md)).

`code` is an open string. Relay adds codes in minor releases, so handle a code you do not know by its HTTP status. A few codes from earlier releases are upper case, such as `PROJECT_MOVED` and the identity exchange's `INVALID_IDENTITY_EXCHANGE`; they keep their spelling.

## Keep uncertain writes pending

After refreshing and confirming the outcome of a write that requires reconciliation, call the
helper's `settle()`. Console offers **I have confirmed the result** for that acknowledgment.
Do not acknowledge a result that is still unknown; starting another write can apply it twice.

Use `createRelayAction()` from `@domino-sdk/relay` to retain the callback and key for a management
write, such as `action.run((actionId) => management.points.record({ actionId, kind: "award", member,
balance, amount: 10, reason: "Welcome" }))`. Keep the helper until a definite result. Its
`retry()` sends the original input again only when the error has `recovery: "replay"`; it waits
any remaining `retryAfter`. A new key starts a new write even if its other fields are identical.
See [Retry after an uncertain answer](participant-api.md#retry-after-an-uncertain-answer) for
helper lifetime, reconciliation, and React usage.

Console shows uncertain writes with their operation ID. Offer controls and redemption refunds,
member point changes and reversals, stock changes, and entitlement corrections offer **Retry the
same change**. The retry uses the original input and key, even if the form has since changed.
Writes without replay-safe receipts instruct the operator to refresh and inspect the result first.

## Codes by status

A refusal with nothing more specific to say answers with its status's code:

| Status | Code                     | Status | Code                |
| ------ | ------------------------ | ------ | ------------------- |
| 400    | `invalid-request`        | 413    | `payload-too-large` |
| 401    | `unauthenticated`        | 415    | `unsupported-media-type` |
| 403    | `forbidden`              | 422    | `validation-failed` |
| 404    | `not-found`              | 429    | `rate-limited`      |
| 405    | `method-not-allowed`     | 500    | `internal-error`    |
| 409    | `conflict`               | 502    | `upstream-failed`   |
| 410    | `gone`                   | 503    | `unavailable`       |
|        |                          | 504    | `upstream-timeout`  |

A refusal that follows a write which may have taken effect answers 502 instead of 503 and keeps its code, such as `unavailable`: decide on retrying from the status.

**Invalid input names its fields.** A request body or query that fails validation answers 400 `invalid-request` with the fields it refused in `error`, such as `Invalid request: actionId, records.0.occurredAt` (at most five, then "and N more"). The message names fields only, never the values sent. The SDK clients check input before sending and reject with `invalid-input`, giving each field and its reason on one line: `Invalid management request: actionId: use 1–100 letters, digits, _ or - (no :, . or spaces)`. `actionId` follows that rule, so build a retry-stable one from a hex hash rather than joining IDs with `:`.

**Large request bodies.** A management write counts the JSON values in its body (each array, object, key, string, number and literal) and refuses one with too many with 413 `payload-too-large` before reading it: more than 2,000,000 on bulk writes (catalog deploys, previews and drafts, deployment previews, release batches, tracker record ingest and member import rows, each up to 64 MB) and more than 500,000 on other writes. Split the batch. A Relay server handles one body over 4 MiB or 500,000 values at a time; another one waits up to 10 seconds, then answers 503 `unavailable` with `Retry-After` before doing anything. The CLI and the SDK clients send any call again after such a 503, within the call's deadline.

## Routing, scope and identity exchange

| Code                                 | Meaning                                                                                         |
| ------------------------------------ | ----------------------------------------------------------------------------------------------- |
| `route-not-found`                    | The API does not serve this path or method. A caller no management credential identified counts it toward the refusal limit, as a refused credential: after 60 refusals within 10 minutes from one address, its unknown paths and its management, MCP and Git requests answer 429 `rate-limited` until the 10 minutes end ([troubleshooting](troubleshooting.md)). |
| `PROJECT_MOVED`                      | The project moved to another organization the signed-in operator can access. `moved` names it. |
| `member-not-found`                   | No member holds the account used for a management lookup.                                      |
| `member-erase-blocked` | 409: `members.erase` waits while the member has an attempt waiting for evaluation or review, or a reward handover in progress; `state` says which. Decide the attempt or finish the handover, then erase. Nothing changed. |
| `member-banned` | 403: an operator banned the member (`members.ban`), so their attempts, completions, invitation acceptances and referral qualifications are refused; `relay.me.get()` answers `eligible: false`. The member is given no reason. A banned inviter's code answers `invitation-not-found` instead. |
| `member-not-banned` | 409: `members.unban` named a member with no ban in force. Nothing changed. |
| `member-ban-restoring` | 409: `members.ban` waits while the member's last unban is still restoring their completions (`ban.unban.restore.status: "running"`); ban again once it finishes. `ban` names the ban. |
| `member-erased` | 409: the member was erased (`members.erase`), so nothing new can be recorded for them: points, badges, groups, campaigns, consent and profile changes are refused. |
| `account-ambiguous`                  | Several members hold the lookup account, or the signers of a submitted Solana transaction. `members` lists their IDs. |
| `identity-ambiguous`                 | An identity command named an X or Discord account by subject, and the member holds it through several issuers, none selected. Name it with the `{ kind: "oauth", provider, issuer, subject }` selector from `members.identityStates.list`. |
| `TRACKER_SCOPE_DENIED`               | The API key cannot ingest into this tracker.                                                    |
| `INVALID_IDENTITY_EXCHANGE`          | The identity exchange body is malformed.                                                       |
| `INVALID_PARTICIPANT_REQUEST`        | The identity integration's participant request is malformed.                                   |
| `IDENTITY_INTEGRATION_ACCESS_DENIED` | The identity integration credential cannot make this request.                                  |
| `APP_IDENTITY_UNLINKED`              | An operator unlinked this app identity from its member.                                         |
| `APP_IDENTITY_CONFLICT`              | The app identity belongs to another member.                                                     |
| `ACCOUNT_VOUCHING_OFF` | The exchange sent `accounts`, but the app login key's **Vouches accounts** setting is off. Nobody was signed in. |

## Reload, then try again

The state changed since the caller read it. Nothing was written. Read the current state, then retry or show it.

| Code                                                                           | What changed                                                                       |
| ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| `catalog-changed`                                                              | The quest catalog: a deployment or another save moved it on.                       |
| `draft-changed`                                                                | A catalog draft was saved or published elsewhere.                                  |
| `quest-changed`                                                                | A quest's release or settings.                                                     |
| `preview-changed`                                                              | The project since its deployment preview: preview again.                           |
| `engagement-changed`                                                           | An engagement rule since its preview.                                              |
| `integration-changed`                                                          | A project integration.                                                             |
| `profile-changed`, `upload-replaced`                                           | The member's profile settings; a newer picture upload replaced this one.           |
| `balance-changed`                                                              | A point balance since the caller read its total.                                   |
| `inventory-changed`, `entitlement-changed`, `redemption-changed`               | Pickup inventory, a reward entitlement or a redemption.                            |
| `operation-changed`, `resource-changed`, `review-changed`                      | A pause control, a resource's lifecycle, or an attempt awaiting review.            |
| `account-changed`, `store-changed`, `build-changed`, `repository-changed`      | A connected account, an app store, a build's artifacts, or a hosted repository.    |
| `offer-changed`                                                               | An offer changed since the operator read its revision.                              |
| `catch-up-changed`                                                            | Automatic-quest catch-up changed since its preview.                                 |
| `trigger-changed`, `filter-changed`                                            | A tracker trigger or its record filter no longer matches the recorded outcome.      |
| `invalid-cursor`                                                               | The page cursor is malformed, expired, or from other filters: start again from the first page. |
| `stale-session`                                                               | The participant session changed since the app read it.                              |
| `quest-updated`, `stale-quiz`                                                  | The quest or quiz changed since the participant loaded it.                          |

The [participant API](participant-api.md#client-conventions) describes the participant client's reload behavior.

## Sign in or use another credential

[Access](https://console.domino.run/docs/concepts/access) describes each kind of credential, the roles behind it and what it may do in live.

| Code                            | Meaning                                                                                         |
| ------------------------------- | ----------------------------------------------------------------------------------------------- |
| `not-signed-in`                 | The call needs a signed-in member.                                                              |
| `unauthenticated`               | A participant backend (`createParticipantBackend`) found no app session. The SDK ends the member's session on it, as on `not-signed-in` (`RelayErrorCode.Unauthenticated`). |
| `session-expired`               | The Console or CLI session expired. Sign in again, or run `domino login`.                      |
| `sign-in-expired`               | A CLI sign-in request expired or was already approved. Run `domino login` again.               |
| `invalid-credential`            | The API key, service key, automation key, hosted app key or identity integration key is expired, revoked or not valid here. |
| `credential-not-allowed`        | This action needs another kind of credential, such as a Console session or `domino login`. An automation key administers nothing.     |
| `permission-required`           | The credential lacks the permission or administrator role the action needs, such as an automation key creating a project without its create-projects permission, or a viewer (dashboards only) reading members. |
| `console-approval-required`     | The action needs a person signed in to Console: approving or rejecting a promotion, publishing, rejecting or rolling back a live deployment, creating or scoping a live personal API key (whichever environment the CLI selects), or a live write that a `domino login` session, an MCP connector or an automation key may not make. See [what each credential may do in live](https://console.domino.run/docs/concepts/access#what-each-credential-may-do-in-live). |
| `scope-mismatch`                | The credential belongs to another organization, project or environment, or the automation key does not reach this project. Finishing a plugin's OAuth connection answers it for anyone but the person who started it, or another project or environment than the one it started in. |
| `organization-required`         | Choose an organization first.                                                                   |
| `test-environment-required`     | The action works only in the test environment, such as development, staging, test members, clearing app storage and preparing a promotion. |
| `operator-required`             | Acting as a test member on a staging app needs a staging link minted by Console, a CLI session, or a personal or automation key that holds `publish`. A password staging session, and a link an app's key or an MCP client minted, name no operator. See [check participant flows on staging](campaign-testing.md#check-participant-flows-on-staging). |
| `live-environment-required`     | The action works only in live: deployment review, rejection and promotion, and deciding or rolling back an environment promotion. Deployment rollback works in both: in test it publishes at once. |
| `promotion-required`            | 422: live refuses a definition change, such as a quest, its settings, a point ledger, a badge or archive state, and a code deployment (`domino deploy --environment live`, `management.deployments.preview`), from every credential. `domino deploy rollback --environment live` gets it from the CLI before anything is sent. Make the change, deploy or roll back in test, then `domino promote`. See [test and live](https://console.domino.run/docs/concepts/environments). |
| `promotion-empty`               | 409: live already matches test, so there is nothing to promote.                                 |
| `promotion-changed`             | 409: test or live changed after the promotion was prepared; approval changed nothing. Prepare a new promotion. |
| `promotion-decided`             | 409: the promotion was already applied, rejected or superseded.                                 |
| `plan-required`                 | 403: approval would take a project live that has no plan, while the organization's one live trial is running on another project, in grace or used. `trial` says which; live did not change. Subscribe the project (`billing.checkout`, where billing is open to you), or ask Domino support to set its plan, then approve again; `usage.plan` reads the plan and the trial. With `reason: "unpaid"` (and `unpaidSince`), a payment for one of the organization's plans has been overdue 14 days: pay it in the customer portal (`billing.portal`). See [going live needs a plan](https://console.domino.run/docs/concepts/promotion#going-live-needs-a-plan). Adding a custom domain answers it too, with `feature: "custom-domain"` and the project's `plan`, when the plan is below Growth; see [custom domains](https://console.domino.run/docs/concepts/hosting#custom-domains). |

Participant sign-in adds its own:

| Code                       | Meaning                                                                                   |
| -------------------------- | ----------------------------------------------------------------------------------------- |
| `already-signed-in`        | Sign out before signing in as someone else.                                               |
| `invalid-code`             | The email code is wrong. The member can type it again.                                    |
| `code-expired`             | The email code expired, was used, was replaced by a newer code or had too many attempts. Request another, and enter the latest code. |
| `challenge-expired`        | The wallet challenge expired, was used or changed. Start wallet sign-in again. Linking by transaction: the member has no open challenge for the address, or it is over 7 days old; request a new one. For Telegram Mini App sign-in, the launch data is over five minutes old or already signed in once: reopen the Mini App. |
| `invalid-signature`        | The wallet signature does not match the challenge, or Telegram did not sign the Mini App launch data for the project's bot. |
| `transaction-not-final`    | Linking by transaction: the transaction is not finalized yet. Send the same signature again in a few seconds. |
| `transaction-not-found`    | Linking by transaction: Solana mainnet has no transaction with this signature. |
| `transaction-failed`       | Linking by transaction: the transaction failed, so it proves nothing. Execute the memo again. |
| `proof-mismatch`           | Linking by transaction: no Memo v2 instruction carries one of the member's open challenges for the address with the address as signer. |
| `challenge-used`           | Linking by transaction: another transaction already used the challenge. Request a new one. |
| `sign-in-method-disabled`  | The account or wallet is linked but is not a sign-in method, or wallet or Telegram Mini App sign-in is off for the project. |
| `account-linked-elsewhere` | The account belongs to another member. Accounts are never merged.                         |
| `account-not-linked`       | Sign in with the provider, or link the account, before verifying.                         |
| `staging-access-required`  | 403 on a participant request that names a staging address (`<slug>--staging.domino.page`) in `X-Relay-App-Origin` without the staging pass: staging's participant API answers only browsers signed in to staging, through the staging address. An app's own `proxyParticipant` route on staging forwards the pass; calling the API directly with a staging origin does not. A request with an app login key (`X-Relay-Integration-Key`, as `createParticipantBackend` sends) is selected by the key and needs no pass. Use a preview or the live app for public testing. |
| `live-offline`             | 403 on any participant request to live: the project's live trial and grace ended without a plan, so live is offline. Test still answers. A plan brings live back. |
| `limit-reached`            | 403 with `limit: "real-people-in-test"`, `count` and `maximum`: the test environment already has its plan's number of real people, so a person who never joined test cannot sign in to it. People who joined, live sign-ins and test members are unaffected. See [real people in test](https://console.domino.run/docs/concepts/participants#real-people-in-test). |

## Discord verification configuration

An attempt that cannot verify Discord because the project has no ready connection or the bot's access was removed becomes `failed` with `code: "integration-required"` when quest code lets the capability error escape. Its `reason` explains the configuration problem without provider details. Relay does not automatically retry this deterministic failure. Connect or repair Discord in Console **Integrations**, in the same project and environment, before an explicit retry. `domino attempts get <attempt>` includes the code and reason. Publication doesn't refuse these quests: they publish, and a deployment's preview and result name them in `warnings` with code `integration-required` (printed by `domino deploy`). Connecting or repairing Discord admits the automatic attempts that failed this way again, for automatic quests without a tracker; tracker-triggered attempts need an explicit retry. Temporary outages remain retryable `provider-unavailable` capability errors.

## Participation

| Code                                   | Meaning                                                                                       |
| -------------------------------------- | --------------------------------------------------------------------------------------------- |
| `prerequisites-incomplete`             | The member has not met the quest's prerequisites. `missing` lists them.                       |
| `quest-closed`                         | The quest is archived or retired and takes no new participation.                              |
| `quest-paused`, `collection-paused`    | An operator paused the quest, or reward collection.                                           |
| `staff-confirmation-required`          | Staff confirm this quest; the member cannot submit it.                                         |
| `invalid-submission`                   | The supplied text, URL, answer or evidence does not meet the quest's input rules.              |
| `attempt-pending`                      | The member's previous attempt on this quest is not decided yet: one unfinished attempt each.  |
| `transaction-not-final` | 409 with `Retry-After`: `management.attempts.submitSolanaTransaction` found no finalized transaction by that signature yet. A transaction finalizes about 13 seconds after it lands; send the same signature again. An unknown signature answers this too, so stop retrying after a deadline. Each read counts against the project environment's budget of 1,500 per 10 minutes (3,000 across Domino); past it the route answers 429 `rate-limited` before reading, and a failed read answers 502 `unavailable`. |
| `transaction-failed` | 422: the submitted Solana transaction failed on chain. |
| `transaction-not-found` | 404 from `management.solana.transactionFixture` (`domino quests fixtures`): the cluster has no transaction with a signature, as for one from the other cluster. The same call answers 409 `transaction-not-final` for one not finalized yet and 422 `transaction-failed` for a failed one; see [on-chain](onchain.md#solana-transactions). |
| `transaction-not-for-quest` | 422: the submitted Solana transaction carries no Domino quest memo, more than one, or one naming another quest. |
| `wallet-not-linked` | 404: no signer of the submitted Solana transaction is a wallet a member of the project and environment has linked. Domino never links a wallet from a transaction; the member links it in the app, then the app submits the signature again. |
| `quest-not-transaction-completable` | 422: only a member-submitted `manual()` quest without input can be completed from a Solana transaction. |
| `station-unknown` | 422: the quest has check-in `stations` and the transaction's memo `reference` names none of them, or names none at all. A member's own submission of such a quest, without a transaction, is refused with it too. |
| `station-closed` | 409: the station the memo names was outside its `opensAt`/`closesAt` window at the transaction's block time. |
| `station-revoked` | 409: an operator revoked the station the memo names, with `management.quests.stations.revoke()` or in Console. It applies when the transaction is submitted, whenever it was signed; `restore()` lifts it. |
| `retry-cooldown`                       | The member's last attempt was rejected and the quest's retry cooldown runs until `retryAt`.   |
| `completion-cooldown`                  | The member's last accepted completion is within the quest's completion cooldown, until `retryAt`. |
| `submission-closed`                    | The quest submission was already submitted, discarded or expired; open a new one.             |
| `attempt-not-evaluated`                | The attempt has no evaluation run yet, so there is nothing to replay.                          |
| `retry-unavailable`                    | Nothing failed that can be retried, or the quest recovers on its own.                         |
| `not-joined`                           | The member has not joined this project.                                                       |
| `outside-window`                       | The quest or campaign is outside its participation window.                                     |
| `enrollment-required`                  | Join the campaign before participating.                                                        |
| `enrollment-not-required`              | This quest or campaign does not take enrollment.                                               |
| `not-claimable`                        | The tracker outcome is not historical activity the member can claim.                           |
| `profile-locked`                       | An operator locked the profile. The member can still hide it.                                 |
| `unsupported-consent-channel` | Relay records consent for this channel only on a later version. Only `email` is accepted. |
| `consent-wording-required`, `consent-wording-too-long`, `consent-wording-invalid` | A grant needs the wording the member saw: plain text of at most 2,000 characters, without NUL characters or lone surrogates. |
| `invitation-not-found`                 | No member has this invitation code, or its member is banned.                                  |
| `inviter-already-accepted`, `invitation-closed` | The member already accepted another inviter, or already completed a quest other than a referral program's qualifying quest. |
| `self-referral`, `referral-cycle`      | A member cannot invite themselves, directly or through a cycle of invitations.                |
| `referral-suspended`                   | The referral program or relationship is suspended.                                            |
| `pass-expired`, `pass-replaced`, `pass-used` | The collection pass expired, was replaced by a newer one, or was already used.          |
| `pass-already-linked`                  | The pass is linked to another account, or the member already has a linked pass.               |
| `out-of-stock`, `insufficient-points`  | No units remain, or the balance cannot cover the change.                                      |
| `point-ledger-not-found`               | No point ledger has the requested key or ID.                                                    |
| `leaderboard-archived`                 | The leaderboard is archived.                                                                  |
| `budget-exhausted`                     | A completion payout, or restoring a completion whose revoke returned its budget, would exceed `budget`. Nothing changes. |

Offers have their own refusal codes:

| Code                  | Meaning                                                               |
| --------------------- | --------------------------------------------------------------------- |
| `offer-not-found`     | No published offer has the requested key or ID.                       |
| `offer-paused`        | The operator paused or archived the offer.                            |
| `offer-changed`       | The offer changed since the operator read it.                         |
| `offer-limit-reached` | The member reached this offer's redemption limit.                     |
| `offer-out-of-stock`  | This offer has no stock left.                                         |

Analytics queries (`domino analytics query`, `management.analytics.query`) answer 422 with these codes; see [metric catalog](automation.md#metric-catalog):

| Code                                            | Meaning                                                                                  |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `unknown-metric`                                | No metric in the catalog has this ID. `domino analytics metrics` lists them.             |
| `breakdown-not-accepted`, `filter-not-accepted` | The metric doesn't take this breakdown or filter.                                        |
| `ledger-required`, `ledger-not-accepted`        | A points metric needs a point ledger; other metrics take none.                           |
| `invalid-limit`                                 | The breakdown limit is outside 1 to 100.                                                 |
| `unknown-reference`                             | A quest, tracker, offer, program, leaderboard or ledger key names nothing; `filter` and `key` say which. |
| `group-not-found`                               | No group has this key in the environment; `group` names it.                              |

A deployment, preview or staging activation whose badge, point ledger, offer, leaderboard, referral, dashboard or tracker fails its schema answers 422 `validation-failed` naming the kind, key and field, such as `Offer "ladder" is invalid: stock: Invalid input: expected number, received undefined`. `domino check` reports the same failure first as a `DEFINITION_INVALID` diagnostic with the entry `file`, `export` and `path`.

Dashboard deployments (`domino deploy` with `dashboards` in relay.json) answer 422 with these codes; see [dashboards](authoring.md#dashboards):

| Code                                                                       | Meaning                                                                                       |
| -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `unknown-metric`, `breakdown-not-accepted`, `filter-not-accepted`          | A panel names a metric the catalog doesn't have, or a breakdown or filter its metric doesn't take. A `number` panel with a breakdown is `breakdown-not-accepted`. |
| `breakdown-required`                                                       | A `table` panel has no breakdown.                                                             |
| `chart-not-accepted`                                                       | The panel's metric can't be drawn as its chart, such as a rate as `bar`.                      |
| `ledger-required`, `ledger-not-accepted`                                   | A points metric needs `ledger`; other metrics take none.                                      |
| `unknown-reference`                                                        | A quest, tracker, ledger, offer, referral program or leaderboard key names nothing in the environment.    |
| `referral-program-required`                                                | A panel reads a referral metric and the project has no referral program.                      |
| `duplicate-key`                                                            | Two dashboards share a key, or two panels in one dashboard do.                                |
| `too-many-panels`, `too-many-dashboards`                                   | A dashboard has more than 24 panels, or the project would have more than 20 dashboards that aren't archived. |

A dashboard read gives a failing panel its own `error` instead of failing: `group-not-found`, `panel-timeout` when its query ran too long, or `panel-failed` for any other failure. Unarchiving a dashboard when the project already has 20 that aren't archived answers 422 `too-many-dashboards`.

## Configuration and project state

| Code                                                                                 | Meaning                                                                                      |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `not-configured`                                                                     | A required service or API URL is missing or invalid. Ask the operator responsible for that Domino installation or application to configure it. |
| `custom-domains-unavailable`                                                         | 503: custom domains aren't enabled on this Domino deployment yet. Nothing to fix on your side. |
| `invalid-hostname`                                                                   | The custom domain is not a hostname Domino serves: a root domain such as `example.com`, a wildcard, an IP address, a Domino domain, a malformed name, or a blocked name. Attach a subdomain such as `www.example.com`. |
| `domain-suspended`                                                                   | 409: Domino suspended this custom domain, so it cannot be removed. Contact Domino support. |
| `integration-required`                                                               | Connect the integration, such as Discord, in Console **Integrations**. Deployments report it as a warning, not an error. |
| `server-not-connected-in-organization`                                               | 409: no other project of the organization connects that Discord server, ready, in this environment. An organization administrator installs the bot in Console **Integrations**; `domino integrations discord servers` lists the servers you can connect. |
| `integration-connected`                                                              | 409: the project already has a Discord connection, even one needing attention. An organization administrator changes or repairs it in Console. |
| `server-not-reachable`                                                               | 422: Domino's bot can no longer see that Discord server. An organization administrator reconnects it in Console **Integrations**. |
| `bot-permissions-exceed-installer` | 422: Domino's shared bot already holds a permission in that Discord server that the person installing it lacks, such as Manage Roles. Installation connects nothing, and the install page says so. The server's owner, an Administrator, or someone holding all of the bot's permissions there installs it. |
| `action-id-reused`                                                                   | The action ID was already used with different input. Use a new ID for a new action.         |
| `record-conflict`                                                                    | A tracker record's observation time was already used with different data.                    |
| `tracker-schema-required`                                                            | Publish the custom tracker with a data schema before ingesting records.                       |
| `invalid-record-data`                                                                | A record does not match the custom tracker's data schema; detail identifies the field.        |
| `record-data-too-complex`                                                            | A record or schema is too complex to validate within the limit.                               |
| `already-exists`                                                                     | The name or key is taken.                                                                    |
| `limit-reached`                                                                      | A count limit, such as app secrets, webhook endpoints, stores or linked wallets. Creating a project, or transferring a Free project in, answers 403 with `limit: "free-projects"`, `count` and `maximum` when the organization already holds its 10 Free projects: delete one or put one on a plan. See [Free projects](https://console.domino.run/docs/use/projects#free-projects). |
| `image-archived`, `tracker-archived`, `quest-type-archived`, `point-ledger-archived`, `group-archived` | Unarchive the resource first.                                                                |
| `tiered-reward-archived`                                                             | 422: a deploy or promotion would make a quest, offer or leaderboard prize newly grant an archived tiered reward, named in `reward`. Unarchive it (`domino tiered-rewards unarchive KEY --reason "…"`) or remove it from that code. |
| `tracker-paused`                                                                     | Resume the tracker's collection first.                                                       |
| `x-budget-exhausted`                                                                 | 409: the tracker paused because the project's monthly X budget (in test, the organization's) is spent. It resumes on its own next month, or sooner on a plan that allows more. |
| `tracker-owned-by-code`                                                              | 409: the tracker is defined in code; change it in code and deploy, or create a tracker with another key. |
| `tracker-owned-by-data`                                                              | 409: a deployment declares a tracker created in Console or with `trackers.create`; change it there, or give the tracker in code another key. |
| `webhook-disabled`                                                                   | Enable the webhook endpoint first.                                                           |
| `unknown-activity-type`                                                              | A webhook subscribes to an event type Relay never sends. The message lists the valid types.  |
| `unknown-integration-module`                                                         | Publishing a quest that declares an integration module Relay does not have. The message names it. |
| `secret-superseded`                                                                  | The webhook secret this action ID issued was rotated since. Rotate the secret again to get a new one. |
| `review-required`, `deployment-rejected`                                             | Review the draft, or submit the deployment for review, first; or the deployment was rejected. |
| `already-approved`                                                                   | The leaderboard settlement was already approved.                                             |
| `build-running`, `build-finished`, `build-superseded`, `nothing-staged`              | A build is running, already finished, no longer current, or nothing is staged.              |
| `repository-required`, `repository-not-ready`, `preview-expired`                     | Create the hosted repository first; it is still provisioning; or restart `domino dev`.       |
| `preview-not-found`                                                                  | 404 from a preview link (`POST /staging/link` with `preview`): the URL is not a running `domino dev` preview of this project. Start `domino dev` and pass the preview URL it prints, without a path. |
| `commit-not-found`                                                                   | 422 from `domino stage`: the project's hosted repository has no commit with that SHA, a mistyped SHA or one not pushed yet. No build was started. Push the commit, then stage again. |
| `unavailable` from a starter                                                         | 503 from repository setup or `GET /starters/{id}` (`domino starters export`): this Domino deployment has not uploaded that starter, or its stored copy failed the integrity check. Nothing to fix on your side; tell a Domino operator. An ID that is not an offered starter answers 404 `not-found`. |
| `ai-allowance-exhausted`                                                             | 402: the project's AI builder balance is empty, so the Playground cannot start. The message names the allowance and when it renews. |
| `billing-staff-only`                                                                 | 403: Stripe Checkout and the customer portal are open only to Domino staff for now. Nothing to fix on your side. `billing.read()` and `billing.organization()` say whether billing is open to you before you try. |
| `unknown-plugin`                                                                     | 404: Relay has no plugin by that ID. `connections.list()` names the plugins available to the project. |
| `invalid-config`                                                                     | 422: a connection's config breaks the plugin's schema; `path` names the failing field (empty for the whole object). The connection's `configSchema` is the schema. |
| `invalid-secrets`                                                                    | 422: `connect` lacks a value for a secret field, or a call names a field the plugin doesn't have; `field` names it. |
| `already-connected`                                                                  | 409: the plugin is already connected in this environment. Change it with `connections.update`, or disconnect it first. |
| `not-connected`                                                                      | 409: the plugin is disconnected in this environment. Connect it first. |
| `connection-changed`                                                                 | 409: the connection's `revision` moved on since the `expectedRevision` you sent; the error carries the current `revision`. Read it again before changing it. |
| `oauth-required`                                                                     | 409: the plugin connects through its vendor's OAuth, not with entered secrets. Start it with `connections.authorize` (`domino connections authorize <plugin>`). |
| `oauth-not-supported`                                                                | 409: the plugin connects with entered secrets, so it has no OAuth to authorize. Use `connections.connect`. |
| `oauth-flow-expired`                                                                 | 409: the one-time code that finishes a plugin's OAuth connection expired (five minutes), was already used, or is unknown. Start again with `connections.authorize`. |
| `plugin-not-configured`                                                              | 409: this deployment lacks the plugin vendor's OAuth app, so nothing can connect it. `plugins.list()` shows `oauth.configured: false`; Domino has to set it up, there is nothing to fix on your side. |
| `plugin-managed`                                                                     | 409: the plugin connects through its own flow, not with settings, such as Telegram's chats through Domino's bot. Connect it under Integrations in Console, or through its own API (`management.telegram.connect()`). |
| `dry-run-not-supported`                                                              | 409: the plugin has no dry run: connect it with its secrets or through its OAuth. |
| `dry-run-test-only`                                                                  | 409: a dry run is for the test environment. Connect the vendor's account in live. |
| `not-a-destination`                                                                  | 409: the plugin syncs nothing to a vendor, so it has no activity, resync or dry-run log. |
| `subscription-exists`                                                                | 409: the project already has a subscription that holds its plan. Change its plan (`billing.changePlan`) or cancel it (`billing.cancel`) instead. |
| `no-subscription`                                                                    | 409: the project has no subscription that holds its plan, so there is nothing to change, cancel or resume. Subscribe it first (`billing.checkout`). |
| `subscription-canceling`                                                             | 409: the subscription is set to end when its period does. Resume it (`billing.resume`) before changing its plan. |
| `no-customer`                                                                        | 409: the organization has no Stripe customer yet, so there is no customer portal (`billing.portal`, `billing.organizationPortal`). Subscribe a project first (`billing.checkout`). |
| `test-member-reset-blocked`                                                          | Relay cannot safely reset or purge the test member; `state` names the blocking record. A purge names the member in `member`, and for `referrals affect another member` the unlisted member in `outside`: add it to the list. |
| `test-member-clock-limit`                                                            | The test member's clock cannot run more than 90 days ahead of real time.                     |
| `test-member-clock-past`                                                             | `advance` with `to` names an instant at or before the test member's clock; it only moves forward. |
| `attempt-not-rejected`                                                               | 409: only a rejected attempt can be accepted anyway; this one is accepted, progressed, failed, needs review or is still being checked. Retry a failed attempt (`attempts.retry`) and review one that needs review (`attempts.review`). |
| `attempt-superseded`                                                                 | 409: `attempts.acceptAnyway`, and `testMembers.acceptAnyway` with `rules: "live"`, accept only the member's latest attempt on the quest; the error names the newer attempt, and `attempts.get` answers it as `supersededBy`. |
| `quest-archived`                                                                     | 409: `attempts.acceptAnyway`, and `testMembers.acceptAnyway` with `rules: "live"`, refuse an attempt on an archived quest. Unarchive the quest first.                                     |
| `attempt-not-decided`                                                                | 409: `attempts.redactEvidence` deletes evidence only once the attempt is accepted, progressed, rejected or failed; this one, or another attempt that submitted the same photo, still waits for evaluation or review. Decide it first. |
| `evidence-redacted`                                                                  | 409: staff deleted the attempt's evidence (`attempts.redactEvidence`), so retrying it (`attempts.retry`, or the member retrying their failed attempt) and `attempts.capture` have nothing left to evaluate or replay. `attempts.acceptAnyway` still accepts a rejected one. |
| `import-status`                                                                      | The [member import](imports.md) is not in a state that allows this, such as applying one that is not `checked`. |
| `import-empty`                                                                       | The member import has no rows to check, or no ready row to apply.                            |

## Operator corrections and retired operations

`inventory-in-use` refuses item archive while any retained offer, including a paused or archived offer, or pickup variant references it. The response's `offers` and `pickups` arrays identify the blockers. An offer's backing item therefore cannot be archived; archive the offer instead. Reconfigure pickup references away from the item first. `inventory-archived` refuses new allocations, restocking and pickup configuration against an archived item; unarchive it first.

| Code                               | Meaning                                                                                         |
| ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| `completion-revoked`               | The completion was already revoked; or, from `attempts.acceptAnyway`, the member's completion of this occurrence was revoked and the occurrence kept, so it can't be completed again. |
| `already-completed`                | From `attempts.acceptAnyway`: the member already has this completion (a once-only quest, or the same completion key), so accepting the attempt would pay nothing. |
| `key-used`                         | From `attempts.acceptAnyway`: the quest's completion keys are unique across members (`keyed(fn, { across: "members" })`), and another member's completion holds this attempt's key. Revoke that completion with `freeOccurrence: true` first. |
| `completion-not-revoked`           | Restore refused: the completion isn't revoked.                                                  |
| `occurrence-taken`                 | Restore refused: the revoke freed the occurrence and `completion` holds it now.                 |
| `receipt-conflict`                 | Experimental. `completions.recordReceipt` refused: the completion already has a receipt with this `key` and different values. Receipts are write-once; record the new transaction under another key. |
| `receipt-limit`                    | Experimental. `completions.recordReceipt` refused: the completion already holds 20 receipts.    |
| `completion-grant-history-missing` | Relay cannot safely revoke an older completion until Domino restores its reward grant history. |
| `entitlement-status`               | The reward entitlement changed state before a correction could apply.                           |
| `operation-retired`                | The operation belongs to the retired pre-1.0 contract; read current state and create a new one. |

New operator corrections also return:

| Code | Recovery |
| --- | --- |
| `inventory-in-use` | Remove pickup references first. Every offer, paused or archived, keeps its stock reference, so archive the offer instead of its backing item. |
| `inventory-archived` | Unarchive the item before configuring pickup, restocking or making a new allocation. |
| `group-in-use` | The group is a leaderboard's exclusion group, or a quest that is not archived rewards it with `groupMembership`. The response's `leaderboards` array names those boards; bind them to another exclusion group (`leaderboards bind-exclusion-group`) in test, and promote to change live's binding. Its `quests` array names those quests; remove the reward or archive the quest. Then archive the group. |
| `enrollment-not-found` | Read the member's quest enrollments before withdrawing. |
| `enrollment-changed` | Refresh enrollment; a withdrawal or new consent changed its revision. |
| `webhook-changed` | Refresh endpoint state before deleting. |
| `settlement-changed` | Refresh the frozen settlement before rejecting. |
| `settlement-decided` | The window was already approved or rejected; read its retained decision. |
| `settlement-closing` | The settlement is still taking its closing measurement (its `closing` field is set); approve it once it freezes. |
| `settlement-measured` | Every item was measured; there is nothing to measure again. Approve or reject the settlement. |
| `settlement-not-approved` | Only an approved settlement can be paid on Solana; approve it first. |
| `pool-not-token` | The settlement has no pool, or its pool is in points, which approval already paid. Only a token pool is paid on Solana. |
| `decimals-mismatch` | The mint's decimals differ from the pool unit's. Pass the mint the pool is counted in (SOL has 9 decimals). |
| `insufficient-balance` | The payer cannot cover the unpaid shares, fees, rent for new token accounts and its own rent-exempt minimum; the message says how much is short. Fund the payer's wallet or its associated token account and build again. |
| `unsupported-mint` | The Token-2022 mint has a transfer fee, transfer hook or other extension a push payout cannot pay exactly through. Pay it another way. |
| `account-changed` | Refresh the retained identity choices; disconnect a different current OAuth selection before relinking. |
