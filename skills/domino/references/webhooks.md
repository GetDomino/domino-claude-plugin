# Outbound webhooks

Use `management.webhooks` from `@domino-sdk/relay/server`, or `domino webhooks`, to configure signed outbound activity. [Receive outbound webhooks](https://console.domino.run/docs/build/webhooks) covers receiver implementation and recovery. Console also provides endpoint controls, signing-secret rotation, delivery diagnostics and replay under **Settings → Webhooks**.

## Event envelope

Each JSON payload has `version: 1`, a UUID `id`, open string `type`, epoch-millisecond `createdAt`, `organization`, `project`, `environment`, and a `data` object. Unknown type strings must be accepted or ignored, because Relay can add event types. A subscription names exact types from `activityTypes`, exported by `@domino-sdk/relay/server`; creating or updating an endpoint with any other type is refused with `unknown-activity-type`, and the message lists the valid types. An update may keep a type the endpoint subscribed to before Relay checked types.

| Constant | Type | Data fields |
| --- | --- | --- |
| `completionCreated` | `completion.created` | `attempt`, `member`, `quest`, `completion`, `key`, `source` |
| `pointsAwarded` | `points.awarded` | `entry`, `member`, `balance`, `amount`, `completion`, `quest`, `source` |
| `reviewPending` | `review.pending` | `attempt`, `member`, `quest`, `completion` |
| `rewardDeliveryFailed` | `reward.delivery.failed` | `delivery`, `member`, `quest`, `completion` |
| `entitlementCollected` | `entitlement.collected` | `entitlement`, `member`, `tier` |
| `referralQualified` | `referral.qualified` | `program`, `member`, `inviter`, `completion` |

`completion` can be null where no completion caused the change; `quest` can be null on a points award.

`completion.created` fires for a quest attempt's completion and for each winner's completion when a leaderboard settlement is approved. `source` is `{ kind, id }` and names what paid the completion: `kind` is `quest` or `leaderboard` (`CompletionSourceKind` from `@domino-sdk/relay/server`), an open string that can gain values, and `id` is the quest or board ID. `key` is the occurrence the completion consumed: the quest's completion key, or the settlement ID for a prize. A leaderboard prize has `attempt: null`, and its `quest` carries the board ID, the same as `source.id`. Branch on `source.kind` rather than treating every `quest` as a quest ID. Events created before `source` and `key` existed omit them, so a replayed older delivery has neither. `points.awarded` covers positive point journal entries of kind `award`. Its `source` is `{ kind, id }` and names what paid the award: `quest` or `leaderboard` with the quest or board ID, `engagement` with the engagement rule's key, or `manual` with `id: null` for an operator's point award; `kind` (`PayoutSourceKind` from `@domino-sdk/relay/server`) is an open string that can gain values, and events created before `source` existed omit it. Because `id` can be null, do not parse this `source` with `completionSourceSchema`, which requires an ID. Manual adjustments and [member import](imports.md) opening balances have kind `adjust` and do not emit it; imported completions do not emit `completion.created`. Referral qualification emits when qualification changes to active, including an operator qualification. Events contain identifiers and values, without raw submission evidence, provider errors, or credentials.

## Management routes

All paths are under `/management/v1`. Scope comes from the management credential and project/environment headers, as for other management calls.

| Method and path | Server SDK | CLI |
| --- | --- | --- |
| `GET /webhooks` | `webhooks.list()` | `webhooks list` |
| `POST /webhooks` | `webhooks.create({ url, types, actionId? })` | `webhooks create <file> [--operation-id <uuid>]` |
| `PUT /webhooks/:id` | `webhooks.update(id, { url?, types?, enabled? })` | `webhooks update <endpoint> <file>` |
| `POST /webhooks/:id/disable` | `webhooks.disable(id)` | `webhooks disable <endpoint>` |
| `POST /webhooks/:id/rotate-secret` | `webhooks.rotateSecret(id, { actionId? })` | `webhooks rotate-secret <endpoint> [--operation-id <uuid>]` |
| `GET /webhooks/:id/deliveries` | `webhooks.deliveries(id, { cursor?, limit? })` | `webhooks deliveries <endpoint> [--cursor <cursor>]` |
| `POST /webhooks/:id/deliveries/:delivery/replay` | `webhooks.replay(id, delivery)` | `webhooks replay <endpoint> <delivery>` |

Reads require `read`; writes require `publish`. API keys are limited by their project, environment, and granted permissions. CLI sessions cannot write in live. Webhook configuration is an operational control, so a live API key with `publish` can write. Reviewer and staff roles cannot change endpoints or replay deliveries. Endpoint creation, changes, rotation, replay, and automatic disablement enter the management audit trail. Audit data omits signing secrets and endpoint URLs.

Create and rotate responses contain `{ endpoint, secret, previousValidUntil }`. Rotation normally retains the previous secret for 24 hours. If the previous secret cannot be decrypted after an infrastructure encryption-key replacement, rotation recovers immediately with `previousValidUntil: null`; install the new secret before replaying failed deliveries. A usable overlap blocks another rotation until it ends.

Create and rotate take an `actionId`; the SDK generates one when you omit it. A retry with the same `actionId` and input returns the first answer, including the same secret, instead of creating a second endpoint or being refused by the overlap. Once the endpoint's secret is rotated again, the earlier action's retry is refused with `secret-superseded`. The CLI saves both commands as operations: after a lost answer (exit 3), run `domino operations resume <id>`. The saved operation keeps the returned secret in the local journal (`RELAY_CONFIG_DIR/operations`, owner-only), where `domino operations show <id>` prints it; `operations list` omits it.

Other endpoint responses expose URL, subscribed `types`, `enabled`, `disabledReason`, `consecutiveFailures`, and timestamps. Delivery pages contain `items` and `nextCursor`, newest first by creation time and ID; pass `nextCursor` back as `cursor` for the next page. `before` and `next` are the deprecated names of the same cursor, removed in 2.0.0. Items expose the event, delivery ID, endpoint ID, status, attempts, response status, last error, creation and delivery times, and `replayOf`.

## Signature protocol

`Domino-Signature` has the form `t=<unix-seconds>,v1=<hex-hmac>`. During rotation it contains two `v1` fields. Each HMAC is SHA-256 over the UTF-8 string `<timestamp>.<raw-body>`, using the signing secret string as the key. The timestamp is refreshed for every HTTP attempt; the event ID and body stay unchanged.

`verifyWebhook({ body, signature, secret })` uses constant-time HMAC verification through Web Crypto, rejects timestamps more than 300 seconds in either direction, and parses the v1 envelope. `secret` accepts a string or array of secrets. `toleranceSeconds` can shorten the window to 0–300 seconds; `now` supplies epoch milliseconds for tests. Verification throws on failure. It does not store event IDs or perform business deduplication.

## Delivery limits and recovery

| Rule | Behavior |
| --- | --- |
| Scope | One project and either test or live; both send real requests |
| Delivery | At least once; no ordering guarantee |
| Subscription | 1–100 unique exact type strings, no wildcard; no historical backfill |
| Endpoints | At most 20 per project/environment |
| URL | HTTPS, at most 2,048 characters, no credentials or fragment |
| Network | Public IP addresses only; every DNS answer checked at creation/update and every delivery; the HTTPS connection is pinned to a checked address |
| Redirects | Not followed; 3xx counts as failure |
| Timeout | 10 seconds including DNS, connection, TLS, and response headers |
| Payload | At most 64 KiB; immutable across retry and replay |
| Retry | Up to 20 attempts, exponential backoff starting at 5 seconds, capped at 15 minutes |
| Success | Any 2xx response |
| Auto-disable | 20 consecutive failed HTTP attempts across the endpoint, or a URL that resolves to an unsafe address at delivery time |
| Disabled | No new event deliveries; queued deliveries become skipped when processed; in-flight requests can finish |
| Rotation | Both secrets sign for 24 hours; another rotation waits for the overlap to end |
| Delivery page | Default 50, maximum 100; `before` is a delivery ID from the same endpoint |
| Replay | Finished deliveries only, enabled endpoint required; retains event ID, creates a new delivery ID |

Private, loopback, link-local, metadata, multicast, reserved, and IPv4-mapped IPv6 addresses are rejected. DNS failures retry. A newly unsafe DNS answer permanently fails that delivery and disables the endpoint. HTTPS certificate and hostname verification stay enabled.

Use `management.webhooks.resume(id)` or `domino webhooks resume ID` after fixing the receiver; this clears the failure counter. Changing the URL also resets the counter. Outcomes from requests sent before a configuration edit cannot change its new failure counter. Updating `types` changes subscriptions for future events. A queued or replayed event keeps its original payload even if its type is no longer subscribed. Logs retain delivery summaries and events, not receiver response bodies. Project deletion pauses queued work; restore resumes it; project purge removes it.

Pause delivery with `management.webhooks.pause(id)` or `domino webhooks pause ID`. Both pause and resume require `publish`, as endpoint updates do. The old `webhooks.disable` SDK method and CLI command remain deprecated 1.x aliases of pause; the CLI contract marks `webhooks disable` with `deprecated`.

## Delete an endpoint

With publish permission, choose **Delete endpoint** in Console **Settings → Webhooks**, read its current revision and provide a reason. Deletion stops new delivery and retries, frees endpoint capacity and keeps read-only delivery history under **Deleted endpoints**. An HTTP request already in flight may finish. Deleted endpoints cannot be resumed, rotated or replayed; create a new endpoint to send again.

Use `management.webhooks.state(id)` for the current fence, then `management.webhooks.delete(id, { expectedRevision: endpoint.revision, reason })`. `management.webhooks.states({ cursor, limit })` pages active and deleted endpoints, and `deliveries(id, query)` retains their history. The CLI provides `domino webhooks get ID`, `domino webhooks states` and `domino webhooks delete ID --expected-revision N --reason TEXT`. Mutation inputs accept an optional action ID for identical retries. These controls retain the existing publish permission and credential rules in Test and Live.
