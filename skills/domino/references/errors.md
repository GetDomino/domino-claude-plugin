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

## Routing, scope and identity exchange

| Code                                 | Meaning                                                                                         |
| ------------------------------------ | ----------------------------------------------------------------------------------------------- |
| `route-not-found`                    | The API does not serve this path or method.                                                     |
| `PROJECT_MOVED`                      | The project moved to another organization the signed-in operator can access. `moved` names it. |
| `member-not-found`                   | No member holds the account used for a management lookup.                                      |
| `account-ambiguous`                  | Several members hold the lookup account. `members` lists their IDs.                            |
| `TRACKER_SCOPE_DENIED`               | The API key cannot ingest into this tracker.                                                    |
| `INVALID_IDENTITY_EXCHANGE`          | The identity exchange body is malformed.                                                       |
| `INVALID_PARTICIPANT_REQUEST`        | The identity integration's participant request is malformed.                                   |
| `IDENTITY_INTEGRATION_ACCESS_DENIED` | The identity integration credential cannot make this request.                                  |
| `APP_IDENTITY_UNLINKED`              | An operator unlinked this app identity from its member.                                         |
| `APP_IDENTITY_CONFLICT`              | The app identity belongs to another member.                                                     |

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
| `invalid-cursor`                                                               | The page cursor no longer reads: start again from the first page.                  |
| `stale-session`                                                               | The participant session changed since the app read it.                              |
| `quest-updated`, `stale-quiz`                                                  | The quest or quiz changed since the participant loaded it.                          |

The [participant API](participant-api.md#client-conventions) describes the participant client's reload behavior.

## Sign in or use another credential

| Code                            | Meaning                                                                                         |
| ------------------------------- | ----------------------------------------------------------------------------------------------- |
| `not-signed-in`                 | The call needs a signed-in member.                                                              |
| `session-expired`               | The Console or CLI session expired. Sign in again, or run `domino login`.                      |
| `sign-in-expired`               | A CLI sign-in request expired or was already approved. Run `domino login` again.               |
| `invalid-credential`            | The API key, service key, hosted app key or identity integration key is expired, revoked or not valid here. |
| `credential-not-allowed`        | This action needs another kind of credential, such as a Console session or `domino login`.     |
| `permission-required`           | The credential lacks the permission or administrator role the action needs.                    |
| `console-approval-required`     | Approving, rejecting or rolling back a live deployment, and approving or rejecting a promotion, needs a Console session; a `domino login` session cannot write in live. |
| `scope-mismatch`                | The credential belongs to another organization, project or environment.                        |
| `organization-required`         | Choose an organization first.                                                                   |
| `organization-not-provisioned`  | The organization is not set up for Domino Console.                                              |
| `test-environment-required`     | The action works only in the test environment, such as development, staging, test members and preparing a promotion. |
| `operator-required`             | Acting as a test member on a staging app needs a Console sign-in. A password staging session names no operator. |
| `live-environment-required`     | The action works only in live: deployment review, rejection, promotion and rollback, and deciding or rolling back an environment promotion. |
| `promotion-required`            | 422: live refuses a definition change, such as a quest, its settings, a point ledger, a badge or archive state, and a code deployment (`domino deploy --environment live`, `management.deployments.preview`), from every credential. Make the change or deploy in test, then `domino promote`. See [test and live](https://console.domino.run/docs/concepts/environments). |
| `promotion-empty`               | 409: live already matches test, so there is nothing to promote.                                 |
| `promotion-changed`             | 409: test or live changed after the promotion was prepared; approval changed nothing. Prepare a new promotion. |
| `promotion-decided`             | 409: the promotion was already applied, rejected or superseded.                                 |
| `plan-required`                 | 403: approval would take a project live that has no plan, while the organization's one live trial is running on another project, in grace or used. `trial` says which; live did not change. Subscribe the project (`billing.checkout`, where billing is open to you), or ask Domino support to set its plan, then approve again; `usage.plan` reads the plan and the trial. With `reason: "unpaid"` (and `unpaidSince`), a payment for one of the organization's plans has been overdue 14 days: pay it in the customer portal (`billing.portal`). See [going live needs a plan](https://console.domino.run/docs/concepts/promotion#going-live-needs-a-plan). Adding a custom domain answers it too, with `feature: "custom-domain"` and the project's `plan`, when the plan is below Growth; see [custom domains](https://console.domino.run/docs/concepts/hosting#custom-domains). |

Participant sign-in adds its own:

| Code                       | Meaning                                                                                   |
| -------------------------- | ----------------------------------------------------------------------------------------- |
| `already-signed-in`        | Sign out before signing in as someone else.                                               |
| `invalid-code`             | The email code is wrong. The member can type it again.                                    |
| `code-expired`             | The email code expired, was used or had too many attempts. Request another.              |
| `challenge-expired`        | The wallet challenge expired, was used or changed. Start wallet sign-in again. For Telegram Mini App sign-in, the launch data is over five minutes old or already signed in once: reopen the Mini App. |
| `invalid-signature`        | The wallet signature does not match the challenge, or Telegram did not sign the Mini App launch data for the project's bot. |
| `sign-in-method-disabled`  | The account or wallet is linked but is not a sign-in method, or wallet or Telegram Mini App sign-in is off for the project. |
| `account-linked-elsewhere` | The account belongs to another member. Accounts are never merged.                         |
| `account-not-linked`       | Sign in with the provider, or link the account, before verifying.                         |
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
| `invitation-not-found`                 | No member has this invitation code.                                                           |
| `inviter-already-accepted`, `invitation-closed` | The member already accepted another inviter, or already completed a quest.           |
| `self-referral`, `referral-cycle`      | A member cannot invite themselves, directly or through a cycle of invitations.                |
| `referral-suspended`                   | The referral program or relationship is suspended.                                            |
| `pass-expired`, `pass-replaced`, `pass-used` | The collection pass expired, was replaced by a newer one, or was already used.          |
| `pass-already-linked`                  | The pass is linked to another account, or the member already has a linked pass.               |
| `out-of-stock`, `insufficient-points`  | No units remain, or the balance cannot cover the change.                                      |
| `point-ledger-not-found`               | No point ledger has the requested key or ID.                                                    |
| `leaderboard-archived`                 | The leaderboard is archived.                                                                  |
| `budget-exhausted`                     | A completion payout would exceed its budget. The payout batch is rolled back.                  |

Offers have their own refusal codes:

| Code                  | Meaning                                                               |
| --------------------- | --------------------------------------------------------------------- |
| `offer-not-found`     | No published offer has the requested key or ID.                       |
| `offer-paused`        | The operator paused the offer.                                        |
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
| `action-id-reused`                                                                   | The action ID was already used with different input. Use a new ID for a new action.         |
| `record-conflict`                                                                    | A tracker record's observation time was already used with different data.                    |
| `tracker-schema-required`                                                            | Publish the custom tracker with a data schema before ingesting records.                       |
| `invalid-record-data`                                                                | A record does not match the custom tracker's data schema; detail identifies the field.        |
| `record-data-too-complex`                                                            | A record or schema is too complex to validate within the limit.                               |
| `already-exists`                                                                     | The name or key is taken.                                                                    |
| `limit-reached`                                                                      | A count limit, such as app secrets, webhook endpoints, stores or linked wallets. Creating a project, or transferring a Free project in, answers 403 with `limit: "free-projects"`, `count` and `maximum` when the organization already holds its 10 Free projects: delete one or put one on a plan. See [Free projects](https://console.domino.run/docs/use/projects#free-projects). |
| `image-archived`, `tracker-archived`, `quest-type-archived`, `point-ledger-archived`, `group-archived` | Unarchive the resource first.                                                                |
| `tracker-paused`                                                                     | Resume the tracker's collection first.                                                       |
| `x-budget-exhausted`                                                                 | 409: the tracker paused because the project's monthly X budget (in test, the organization's) is spent. It resumes on its own next month, or sooner on a plan that allows more. |
| `tracker-owned-by-code`                                                              | 409: the tracker is defined in code; change it in code and deploy, or create a tracker with another key. |
| `tracker-owned-by-data`                                                              | 409: a deployment declares a tracker created in Console or with `trackers.create`; change it there, or give the tracker in code another key. |
| `webhook-disabled`                                                                   | Enable the webhook endpoint first.                                                           |
| `unknown-activity-type`                                                              | A webhook subscribes to an event type Relay never sends. The message lists the valid types.  |
| `secret-superseded`                                                                  | The webhook secret this action ID issued was rotated since. Rotate the secret again to get a new one. |
| `review-required`, `deployment-rejected`                                             | Review the draft, or submit the deployment for review, first; or the deployment was rejected. |
| `already-approved`                                                                   | The leaderboard settlement was already approved.                                             |
| `build-running`, `build-finished`, `build-superseded`, `nothing-staged`              | A build is running, already finished, no longer current, or nothing is staged.              |
| `repository-required`, `repository-not-ready`, `preview-expired`                     | Create the hosted repository first; it is still provisioning; or restart `domino dev`.       |
| `ai-allowance-exhausted`                                                             | 402: the project's AI builder balance is empty, so the Playground cannot start. The message names the allowance and when it renews. |
| `billing-staff-only`                                                                 | 403: Stripe Checkout and the customer portal are open only to Domino staff for now. Nothing to fix on your side. `billing.read()` and `billing.organization()` say whether billing is open to you before you try. |
| `subscription-exists`                                                                | 409: the project already has a subscription that holds its plan. Change its plan (`billing.changePlan`) or cancel it (`billing.cancel`) instead. |
| `no-subscription`                                                                    | 409: the project has no subscription that holds its plan, so there is nothing to change, cancel or resume. Subscribe it first (`billing.checkout`). |
| `subscription-canceling`                                                             | 409: the subscription is set to end when its period does. Resume it (`billing.resume`) before changing its plan. |
| `no-customer`                                                                        | 409: the organization has no Stripe customer yet, so there is no customer portal (`billing.portal`, `billing.organizationPortal`). Subscribe a project first (`billing.checkout`). |
| `test-member-reset-blocked`                                                          | Relay cannot safely reset the test member; `state` names the blocking record.                 |
| `test-member-clock-limit`                                                            | The test member's clock cannot advance beyond 90 days.                                       |
| `attempt-not-rejected`                                                               | 409: only a rejected attempt can be accepted anyway; this one is accepted, progressed, failed, needs review or is still being checked. Retry a failed attempt (`attempts.retry`) and review one that needs review (`attempts.review`). |
| `import-status`                                                                      | The [member import](imports.md) is not in a state that allows this, such as applying one that is not `checked`. |
| `import-empty`                                                                       | The member import has no rows to check, or no ready row to apply.                            |

## Operator corrections and retired operations

`inventory-in-use` refuses item archive while any retained offer, including a paused offer, or pickup variant references it. The response's `offers` and `pickups` arrays identify the blockers. Offers have no archive operation yet, so their backing items cannot be archived. Reconfigure pickup references away from the item first. `inventory-archived` refuses new allocations, restocking and pickup configuration against an archived item; unarchive it first.

| Code                               | Meaning                                                                                         |
| ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| `completion-revoked`               | The completion was already revoked.                                                             |
| `completion-grant-history-missing` | Relay cannot safely revoke an older completion until Domino restores its reward grant history. |
| `entitlement-status`               | The reward entitlement changed state before a correction could apply.                           |
| `operation-retired`                | The operation belongs to the retired pre-1.0 contract; read current state and create a new one. |

New operator corrections also return:

| Code | Recovery |
| --- | --- |
| `inventory-in-use` | Remove pickup references first. Enabled and paused offers retain stock references and currently have no archive operation, so their backing items cannot be archived. |
| `inventory-archived` | Unarchive the item before configuring pickup, restocking or making a new allocation. |
| `group-in-use` | The group is a leaderboard's exclusion group, or a quest that is not archived rewards it with `groupMembership`. The response's `leaderboards` array names those boards; bind them to another exclusion group (`leaderboards bind-exclusion-group`). Its `quests` array names those quests; remove the reward or archive the quest. Then archive the group. |
| `enrollment-not-found` | Read the member's quest enrollments before withdrawing. |
| `enrollment-changed` | Refresh enrollment; a withdrawal or new consent changed its revision. |
| `webhook-changed` | Refresh endpoint state before deleting. |
| `settlement-changed` | Refresh the frozen settlement before rejecting. |
| `settlement-decided` | The window was already approved or rejected; read its retained decision. |
| `settlement-closing` | The settlement is still taking its closing measurement (its `closing` field is set); approve it once it freezes. |
| `settlement-measured` | Every item was measured; there is nothing to measure again. Approve or reject the settlement. |
| `account-changed` | Refresh the retained identity choices; disconnect a different current OAuth selection before relinking. |
