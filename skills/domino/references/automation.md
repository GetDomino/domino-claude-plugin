# CLI automation contract

Scripts, CI jobs and agents can rely on everything on this page: command names, options, exit statuses, error codes and the JSON result and error shapes. From 1.0 these follow semver. New commands, options, error codes and result fields can arrive in a minor release, so ignore fields you do not know. Human-readable output, messages and help text are prose and may change in any release; branch on `code` and the exit status, never on a message.

The machine-readable contract ships with the CLI in `node_modules/@domino-sdk/relay-cli/cli/contract/`. `contract.json` lists every command with its arguments, options and stability, plus the exit statuses, error codes, environment variables and scope precedence. `schemas/` holds JSON Schemas (draft 2020-12) for each command's result (`schemas/results/<command>.schema.json`), the error and the stderr events. `fixtures/` has an example of each. `index.mjs` exports the same schemas as zod objects by name (`schemas`), with the registry of exit statuses and error codes, for tooling that runs in Node.

## Client identification

Management API calls and CLI device sign-in calls send `X-Domino-Client: @domino-sdk/relay-cli/<version> (node)`, using the installed CLI's manifest version. The server records validated identities in public request logs. Client metrics use fixed family/runtime labels; exact versions do not create metric labels.

This header grants no access and does not pin an API version. CLI calls retain their existing JSON contracts and do not claim the SDK's `tolerant-v1` response capability. Server additions must remain compatible with older and unidentified clients.

## Run a command for automation

`domino storage list --json` returns `{ "stores": [...] }` in test, or live's stores with `--environment live`. Each store includes `kind`, `name`, `declared`, and `createdAt` (null until provisioned); active declarations also include `description`. Unknown future store kinds are preserved in results; consumers should tolerate them. Provider IDs are never returned. See [hosted storage](hosted.md#app-storage) for declaration and retention rules. The result schema and example ship as `storage-list.schema.json` and `storage-list.json` in the contract's result directories. With `publish`, `domino storage tables NAME`, `rows NAME TABLE [--cursor C]`, `keys NAME [--prefix P] [--cursor C]` and `get NAME KEY` print the management API's browse answers (`storage-tables`, `storage-rows`, `storage-keys`, `storage-get` schemas); `rows` pages by 50 and `keys` by 100, passing a page's `nextCursor` back as `--cursor` for the next page. `domino storage delete KIND NAME` removes a store the serving build no longer declares and asks for confirmation unless `--yes` is given; a declared store fails with 409.

Pass `--json`. The CLI then never prompts, prints the result as one JSON value on stdout, and reports a failure as one JSON object on the last line of stderr:

```sh
if result=$(domino deploy --preview --environment test --json 2>errors.log); then
  echo "$result" | jq .deployment
else
  status=$?
  tail -n 1 errors.log | jq -r .code
fi
```

- **stdout** carries only the result. With `--json` that is exactly one JSON value followed by a newline, including `[]` for an empty list and `null` when the API answered with no content. Keep stdout and stderr apart when parsing (no `2>&1`). A reader that closes stdout early, as `| head` does, ends the CLI with status 0.
- **stderr** carries progress, warnings, prompts and the error. With `--json`, lines before the error are prose or JSON events (see [Events](#events)); only the last line is the error.
- **Prompts** appear only when stdin and stderr are both terminals, `CI` is unset, and neither `--json` nor `--no-interactive` is given. A missing value that would be prompted for fails with `INPUT_REQUIRED` instead. `deploy`, `stage` and `secrets delete` ask for confirmation only when they can prompt; `--yes` skips it.

## Exit statuses

| Status | Outcome     | Meaning                                                                                                                                                                     |
| ------ | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0      | `success`   | The command completed. The result is on stdout.                                                                                                                             |
| 1      | `failed`    | The command failed and nothing it was asked to change is uncertain. `doctor`, `check` and `migrate` also print their result on stdout, so read both.                        |
| 2      | `failed`    | Usage error: unknown command or option, or a missing or invalid argument. The CLI stopped before contacting the API or changing files. Fix the invocation; do not retry it. |
| 3      | `uncertain` | A change was sent, but whether it took effect is unknown. Resume it with the error's `operation`, or check the state before retrying.                                       |
| 130    | `cancelled` | Cancelled by SIGINT (Ctrl-C) or at a prompt. If a change had already been sent, the status is 3 instead; see [Cancellation](#cancellation).                                 |
| 143    | `cancelled` | Cancelled by SIGTERM. If a change had already been sent, the status is 3 instead.                                                                                           |

Every error code maps to one status, except `CANCELLED`: 130 for SIGINT or a prompt, 143 for SIGTERM. `outcome` in the error and the exit status always agree.

## Errors

With `--json`, a failure's last stderr line is:

```json
{
  "error": "Publication of dep_1 was not confirmed: The API did not confirm the change (ECONNRESET). It may have taken effect. Resume with domino deploy --resume dep_1.",
  "code": "OUTCOME_UNCERTAIN",
  "outcome": "uncertain",
  "details": { "cause": "NETWORK_ERROR" },
  "operation": {
    "kind": "deploy.publish",
    "id": "dep_1",
    "scope": {
      "apiUrl": "https://relay.domino.run",
      "organization": "org_acme",
      "project": "spring",
      "environment": "test"
    },
    "resume": [
      "deploy",
      "--resume",
      "dep_1",
      "--api-url",
      "https://relay.domino.run",
      "--organization",
      "org_acme",
      "--project",
      "spring",
      "--environment",
      "test"
    ]
  }
}
```

| Field       | Present          | Meaning                                                                                                         |
| ----------- | ---------------- | --------------------------------------------------------------------------------------------------------------- |
| `error`     | always           | A message for people. It may change.                                                                            |
| `code`      | always           | One of the stable codes below.                                                                                  |
| `outcome`   | always           | `failed`, `uncertain` or `cancelled`, matching the exit status.                                                 |
| `status`    | API answers      | The management API's HTTP status.                                                                               |
| `apiCode`   | API answers      | The management API's own code, listed in [API error codes](errors.md). Unknown API codes are kept here; `code` still classifies them. |
| `details`   | some codes       | Code-specific data, such as the fields of `MANIFEST_INVALID` or the Git phase of `CHECKOUT_GIT_FAILED`.         |
| `operation` | resumable change | Only on `OUTCOME_UNCERTAIN` and `CANCELLED`: what was sent and how to finish it. See [Resume an uncertain change](#resume-an-uncertain-change). |

Without `--json` the same failure is one line, `Error: <message>`, followed by `Resume: domino …` when there is an operation.

A failure from the typed management client follows the same codes. A refusal is classified by its HTTP status, with the API's code in `apiCode`. An `apiCode` of `promotion-required` is classified as `ENVIRONMENT_UNSUPPORTED` instead. A change that was sent and then lost, timed out or cancelled, or that was answered with 500, 502 or 504, is `OUTCOME_UNCERTAIN`. Input the client refused before sending is `INVALID_ARGUMENT`.

Minor releases can add error codes and operation kinds, and the schemas accept them. Treat a code you do not know by its `outcome` and the exit status.

### Error codes

| Code                      | Exit | Meaning                                                                                                                                     |
| ------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `UNKNOWN_COMMAND`         | 2    | No command or subcommand has this name.                                                                                                     |
| `UNKNOWN_OPTION`          | 2    | The command does not accept this option.                                                                                                    |
| `INVALID_ARGUMENT`        | 2    | An argument, option value or piped input is invalid, there are too many arguments, or options conflict.                                     |
| `INPUT_REQUIRED`          | 2    | A required argument, option, subcommand or piped input is missing, and the CLI cannot prompt for it.                                        |
| `AUTH_REQUIRED`           | 1    | No credential. Run `domino login`, or set `RELAY_MANAGEMENT_TOKEN`.                                                                         |
| `AUTH_INVALID`            | 1    | The API rejected the credential (HTTP 401), or it belongs to another organization.                                                          |
| `PERMISSION_DENIED`       | 1    | The credential lacks permission for this action (HTTP 403).                                                                                 |
| `LOGIN_FAILED`            | 1    | Browser sign-in failed, expired, or was approved for another organization.                                                                  |
| `UNATTENDED_LOGIN_UNAVAILABLE` | 1 | Browser sign-in needs a person, and `DOMINO_AGENT_MODE=unattended` says none is present. Set `RELAY_MANAGEMENT_TOKEN` instead. |
| `SCOPE_INVALID`           | 1    | A scope value from the environment, relay.json or the checkout is malformed. `details.source` says where.                                   |
| `SCOPE_REQUIRED`          | 1    | Several organizations or projects match, or a resume lacks part of its pinned scope. Pass the option in `details.option`.                  |
| `SCOPE_NOT_FOUND`         | 1    | The credential cannot reach the selected project in the selected environment, or any project.                                               |
| `SCOPE_PLACEHOLDER`       | 1    | relay.json still names a starter's placeholder organization (`local`). Run `domino checkout`, or pass `--organization` and `--project`. `details.file` names the manifest. |
| `PROJECT_MOVED`           | 1    | The project moved to another organization. See [troubleshooting](troubleshooting.md#scope-and-installation).                                |
| `ENVIRONMENT_UNSUPPORTED` | 1    | The command does not run in this environment, such as `deploy --promote` in test, `promote` in live, or `deploy --resume` and `deploy rollback` in live. A definition change in live gets it too, with `apiCode: "promotion-required"` and `status: 422`; make the change in test. |
| `PROJECT_NOT_FOUND`       | 1    | No relay.json was found, or the `--config` file does not exist.                                                                             |
| `MANIFEST_INVALID`        | 1    | relay.json is invalid. `details` lists each field with a suggestion.                                                                        |
| `PROJECT_INVALID`         | 1    | The project lacks configuration the command needs, such as `app` in relay.json.                                                             |
| `BUNDLE_FAILED`           | 1    | Bundling the configured quests, types, engagement rules or images failed.                                                                   |
| `LOCAL_CONFLICT`          | 1    | Local files are in the way: a directory exists, a file would be overwritten, or changes are uncommitted.                                    |
| `CHECKOUT_GIT_FAILED`     | 1    | A Git step of `checkout` or `create` failed. See [troubleshooting](troubleshooting.md#scope-and-installation).                              |
| `CHECK_FAILED`            | 1    | `check` found blocking diagnostics, listed on stdout, or the app's check script failed.                                                     |
| `DOCTOR_UNHEALTHY`        | 1    | `doctor` found a check with status `error`. Its result is on stdout.                                                                        |
| `MIGRATION_BLOCKED`       | 1    | `migrate` found conflicts and wrote nothing. Its result is on stdout.                                                                       |
| `BAD_REQUEST`             | 1    | The management API rejected the request as invalid (HTTP 400).                                                                              |
| `NOT_FOUND`               | 1    | The resource does not exist (HTTP 404).                                                                                                     |
| `CONFLICT`                | 1    | The resource is not in a state that allows the change (HTTP 409).                                                                           |
| `REJECTED`                | 1    | The API refused the change (HTTP 422 or another 4xx).                                                                                       |
| `RATE_LIMITED`            | 1    | Too many requests (HTTP 429). Wait before retrying: `details.retryAfter`, when the API sent `Retry-After`, is how long in milliseconds.     |
| `SERVER_ERROR`            | 1    | The API failed while reading (HTTP 5xx), or answered 501 or 503. `details.retryAfter`, when present, is how long to wait in milliseconds.   |
| `PREFLIGHT_FAILED`        | 1    | Hosting is not ready for this project. The message lists the blocking checks.                                                               |
| `STAGE_BUILD_FAILED`      | 1    | The cloud build failed. Read it with `domino logs <details.build>`.                                                                         |
| `NETWORK_ERROR`           | 1    | The API could not be reached before any change was sent: DNS, connection, TLS or a redirect. Or a read's answer broke off.                   |
| `TIMEOUT`                 | 1    | A read did not finish within 30 seconds, retries included.                                                                                  |
| `INVALID_RESPONSE`        | 1    | The API answered a read with something the CLI cannot use.                                                                                  |
| `OUTCOME_UNCERTAIN`       | 3    | A change was sent, then the connection failed, timed out, was interrupted, or the API answered 500, 502 or 504. `details.cause` says which. |
| `CANCELLED`               | 130 or 143 | Interrupted or declined before any change was sent: 130 for SIGINT or a prompt, 143 for SIGTERM. For a signal, `details.signal` names it. |
| `NODE_VERSION_UNSUPPORTED` | 1  | Node is older than the CLI supports. `details.minimum` is the required major version, `details.current` the running one. Install that Node release and rerun. |
| `COMMAND_FAILED`          | 1    | Any other failure. Read the message.                                                                                                        |

## Retries and deadlines

The CLI sends every management request through the same transport as `createManagementClient` from `@domino-sdk/relay/server`, so the two share these deadlines, retries and failure classes; the CLI never follows a redirect. Every request has 30 seconds, retries and the answer's body included. The CLI sends a request again, at most three times in all, only when the API applies a repeat at most once: reads, `deploy` publishing its test preview, promotion to a live review request, a live deployment submission by preview ID, `stage` submitting its build request ID, `quests archive` and `quests unarchive` with their action ID, `groups create`, `groups add` and `groups remove` with their action ID, `webhooks create` and `webhooks rotate-secret`, every saved resource change (see [Resource commands](#resource-commands)) and every `--action-id` change, and project creation with its saved action ID. Creation also retries repository provisioning, which reserves one repository per project. It retries when no answer arrived, when the answer broke off after its headers, on HTTP 500, 502, 503 and 504, and on a 429 whose `Retry-After` fits in the time left. If an error body is cut off, its status still controls retries and its `Retry-After` is kept in `details.retryAfter`. It waits `Retry-After` when the API sent one, and otherwise up to 0.25 seconds, then up to 1 second. A failure that remains is the one reported. Other changes, including `domino api` with any method but GET, are sent once. An uncertain response includes `details.recovery: "reconcile"` unless it names a saved operation. Read the affected resource before deciding whether to repeat the change, because an `actionId` alone does not prove a route keeps receipts. A timeout, an unreadable answer and a refusal such as a 400 or 409 are never retried.

## Resume an uncertain change

A change the CLI cannot confirm exits 3 with `OUTCOME_UNCERTAIN`, after any retries above. When the error has an `operation`, run `domino` with its `resume` array as the arguments, adding `--json`. `operation.scope` pins all four scope values (API URL, organization, project and environment), and `resume` either repeats the scope as flags or reads it from the saved operation journal. It runs in the original scope from any directory and never contains a credential. Project creation uses an empty `scope.project` until the server assigns one; `operations show` then reports the created project separately.

A resume (`deploy --resume`, `stage --request-id`) runs only in a pinned scope: each of `--api-url`, `--organization`, `--project` and `--environment` must come from a flag or its `RELAY_*` variable. Otherwise it fails with `SCOPE_REQUIRED` and `details.missing` lists the flags to add; relay.json and the checkout's scope are never used for a resume. An operation is reported only when the CLI knew all four values when it sent the change.

| `operation.kind` | Started by      | `id`                   | Resuming                                                                                                           |
| ---------------- | --------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `deploy.publish` | `domino deploy` in test | the deployment preview | `deploy --resume <id>` publishes that test preview, or returns its committed result. It never creates a second release. Live requests are approved in Console and have no CLI resume command. |
| `deploy.promote` | `domino deploy --promote` in live | the promotion action UUID | `operations resume <id>` returns the pending promotion that action prepared, in the deployment record shape, or prepares it once. Approval still happens in Console. |
| `deployments.rollback` | `domino deploy rollback` in test | the rollback action UUID | `operations resume <id>` returns the deployment that action published, or rolls back once. |
| `promotions.prepare` | `domino promote` in test | the promotion action UUID | `operations resume <id>` returns the promotion that action prepared, or prepares it once. Approval still happens in Console. |
| `stage.build`    | `domino stage`  | the build request ID   | `stage --commit <sha> --request-id <id>` returns the build that request created, or creates it once. It needs no checkout: it skips the hosting preflight, the uncommitted-changes check, the missing-secret warnings and the wallet sign-in warning. |

A `deploy.publish` or `deploy.promote` operation sends `POST /deployments/:deployment/publish` or `/promote`, with the deployment in the path, whichever CLI release saved it.

### Recover after restarting the CLI

The CLI saves project creation, quest archive/unarchive, group creation and membership changes, resource changes, webhook endpoint creation and secret rotation, build submission, deployment publication, and live promotion before sending the mutation. The journal lives in `$RELAY_CONFIG_DIR/operations`, or `~/.config/domino/operations`. Keep that directory between CI invocations. Journal resume commands include `--directory` with the exact journal path; pass that option when using a copied journal elsewhere. Existing stage and deploy resume commands keep their pinned scope flags. Files are private to the local user and contain the exact request inputs, scope, actor and committed results, never authentication credentials. A webhook create or rotation result keeps its signing secret only until the CLI has written it to stdout once, when the operation completes (or the next `show` or `resume`, if that output failed); the record then keeps the result with `secretRedacted: true`, and `show` adds a `note` pointing to `domino webhooks rotate-secret`. Listing and error output omit request inputs.

```sh
domino operations list --json
domino operations show OPERATION_ID --json
domino operations resume OPERATION_ID --json
domino operations forget OPERATION_ID --json
domino operations prune --older-than 30 --json
```

`list` and `show` work offline. `resume` uses the saved API URL, organization, project, environment and payload; it ignores checkout configuration and `RELAY_*` scope variables. Explicit conflicting scope flags are rejected. Authenticate for the saved API origin as the original actor. A completed operation returns its cached committed result without another mutation. For a build, that result acknowledges submission; use `domino builds` or the existing `stage --request-id` path to inspect completion.

Nothing expires saved operations on its own. `forget` removes one record, pending or complete; a forgotten pending operation can no longer be resumed. `prune` removes completed operations last changed more than `--older-than` days ago (default 30; `0` removes every completed one) and answers `{ cutoff, removed }`. It leaves pending operations, and records another process holds or that cannot be read.

Use `--operation-id UUID` on `create`, `quests archive`, `quests unarchive`, `groups create`, `groups add`, `groups remove` or any saved resource change when your automation supplies the ID; a resource change sends it as its `actionId`. Reusing it with changed inputs, actor or scope fails with `CONFLICT`. Without this flag, a new invocation starts a new operation; use `operations resume` for recovery.

Creation saves the server-assigned project ID before repository setup. If setup or Git checkout fails, the error still includes `operation` and its exact resume command, even when the outcome is a known failure with exit 1. Resuming provisions the same project's repository and finishes its checkout. It clones into an operation-owned sibling directory, saves the finished checkout and renames it into place. An interrupted clone with local changes is preserved and reports `LOCAL_CONFLICT`; preserve those edits before resolving the conflict. A live process owns the operation until it exits; another resume reports `LOCAL_CONFLICT`. Cancellation keeps the journal, and the next process recovers a stopped owner's lock.

| Mutation | Recovery |
| --- | --- |
| `create` | Saved project-creation receipt, repository and checkout checkpoints. |
| `quests archive`, `quests unarchive` | Saved action ID, exact lifecycle request and scope. |
| `groups create`, `groups add`, `groups remove` | Saved action ID, exact request and scope (`group.create`, `group.member`). `groups update`, `archive` and `unarchive` set a state, so repeating one is safe. |
| Resource changes: `points award\|spend\|adjust\|set\|reverse`, `trackers create\|update\|ingest`, `leaderboards exclude\|include\|bind-exclusion-group`, `redemptions cancel`, `entitlements restore`, `completions restore`, lifecycle verbs and the rest in [Resource commands](#resource-commands) | Saved request and scope; the operation ID is the request's `actionId`, so a resume repeats the same request, which the server applies once. A new resource command saves its management client method, and `operation.kind` names it, such as `points.record`; a lifecycle verb, `trackers refresh`, `quests enroll|withdraw`, `leaderboard-settlements reject|retry-measurement`, `members unlink-identity|relink-identity|identity-sign-in` and the deprecated `members relink` save their route and body as `management.request` and print the API's answer as before. `point-ledgers update` sets a state, so repeating it is safe. |
| `stage` | Saved build request ID and commit; existing `--request-id` remains supported. |
| `deploy` test publication | Saved preview ID; existing `--resume` remains supported. |
| `deploy --promote` | Run `domino promotions list` before repeating. A pending promotion may already exist; approval and rejection happen only in Console **Live → Deployments**. |
| `promote` | Saved action ID; `operations resume <id>` returns the same promotion. `domino promotions list` shows whether a newer one superseded it. |
| `deploy --preview`, image uploads | Inspect `domino releases list` and the project's image/catalog state before rebuilding; publication is separate. |
| `secrets set`, `secrets delete` | Inspect `domino secrets list`; values are never read back. Confirm the intended value before replacing it. |
| `domains add`, `domains recheck`, `domains remove` | Inspect `domino domains list`. Adding a hostname the project already has fails with `already-exists`; re-checking twice within 10 seconds fails with `rate-limited`. |
| `billing checkout`, `billing topup`, `billing portal` | Inspect `domino billing show`. Each returns a link to a Stripe page; the project changes only once someone pays there. A repeated checkout in the same ten-minute window returns the same Checkout session, and one for a project whose subscription holds its plan fails with `subscription-exists`. |
| `billing change`, `billing cancel`, `billing resume` | Inspect `domino billing show`. Each is safe to repeat: asking for the plan the subscription is already on, or one already held for the renewal, changes nothing more, and cancelling or resuming twice leaves it as the first call did. `subscription`'s plan moves once Stripe reports the change, usually within a minute. |
| `feedback` | Inspect the Linear issue list for the submitted report before sending another. |
| `logout` | Inspect `domino whoami`; revoke or sign in again as appropriate. |
| `api POST/PUT/PATCH/DELETE` and management writes without documented receipts | Read the affected resource first. An `actionId` field alone does not make replay safe. For webhook writes, list endpoints and deliveries before recreating, rotating or replaying. |

These reconcile-only writes are sent once and are not entered into the replay journal.

## Cancellation

SIGINT (Ctrl-C) and SIGTERM stop the command. With no change outstanding they report `CANCELLED`, exit 130 for SIGINT and 143 for SIGTERM; while a change is outstanding they report `OUTCOME_UNCERTAIN`, exit 3. Git children started by this CLI are stopped before it exits on a signal. Recovery records remain on disk. Interrupting `stage` while it waits for the build reports `CANCELLED` with the `stage.build` operation: the build keeps running, and resuming returns it. Declining a confirmation or cancelling a prompt reports `CANCELLED` too; prompts come before any change. `domino dev` treats the first signal as a normal stop and prints its result; a second signal exits at once.

## Events

With `--json`, a command writes JSON events to stderr while it runs. Each has an `event` field:

- `{"event": "operation", "operation": {…}}`: written once the credential check passes, right before a resumable change is sent, after its recovery journal is saved, with the same `operation` a recoverable error would carry. Record it to resume the change if the CLI process dies.
- `{"event": "browser-login", "url": "…", "code": "…"}`: `domino login` is waiting for approval of this code at this URL.

Other stderr lines are prose. The error, when there is one, is always last.

## Results

Each command's result schema is `schemas/results/<command>.schema.json`, with spaces in the command replaced by `-`, such as `secrets-set`. `contract.json` names it for every command. `status`, `quests`, `releases`, `members list`, `builds`, `logs`, `whoami` and `api` print the management API's response body unchanged; their `route` in `contract.json` names the endpoint, and the schema lists only the fields the CLI relies on. The [resource commands](#resource-commands) print the typed result of the management client method they call, the same value `createManagementClient` returns, and `route` names its endpoint. `--help` and `domino help [command...]` with `--json` print `{"help": "…"}`, and `domino help` with an unknown command fails with `UNKNOWN_COMMAND`; `--version` with `--json` prints `{"version": "…"}`.

`domino members list [--q TEXT] [--needs-review] [--limit N] [--cursor C] --json` prints `{ items, total, nextCursor }` (`members-list` schema), the project's members most recently joined first with their `accounts` and activity counts. Pass `nextCursor` back as `--cursor` for the next page until it is `null`, keeping the same `--q` and `--needs-review`. It needs `read`; see [operator member reads](authoring.md) for what `--q` matches.

`domino quests replay <attempt> --json` fetches the attempt's capture (`GET /attempts/{attempt}/capture`, `publish` permission) and evaluates local quest code against it with `replayAttempt` from `@domino-sdk/relay/testing`. Captures exist for test-environment attempts only: with live selected the command fails with `ENVIRONMENT_UNSUPPORTED` before any request, and an attempt whose evaluation has not started fails with `CONFLICT` (`attempt-not-evaluated`); a capture taken mid-run has `recorded: null`. It replays the discovered quest whose key the capture names; pass `--quest FILE` (and `--export NAME` for a named export) for another module or for a quest created in Console, whose capture has no key. `--out FILE` also saves the capture; `--capture FILE` replays a saved one without contacting the API, and then `<attempt>` is optional. The result (`quests-replay` schema) is `{ attempt, quest, decision, recorded, changed, divergence, calls, out? }`: `decision` is what the code decides now, `recorded` what quest code decided in the captured run, `changed` whether they differ (kind, reason, rewards or completion key), `divergence` the first capability call the capture could not answer (the replay stops there, with `decision` and `changed` null), and `calls` the number of calls it answered. A changed decision still exits 0; branch on `changed`.

Member accounts in `members get`, `members list`, `members lookup` and the attempts' people carry `status` and `reason` only when Relay keeps the account's OAuth tokens to read it in the background: `status` is `connected` or `needs-reconnect`, and `reason` (`revoked`, `expired` or `missing-scope`) says why it needs reconnecting. See [operator member reads](authoring.md). `members get` also carries `creatorAccounts`: the member's linked [creator accounts](participant-api.md#creator-accounts), such as a YouTube channel, each with `provider`, `subject`, `revision`, `linkedAt`, `status` (`connected` or `needs-reconnect`, with `reason`) and the channel's `profile`; it is empty when the member linked none. Unlink one with `domino api POST /members/<id>/creator-accounts/unlink body.json` and a body of `provider`, `subject`, `revision` and `reason` (`review` permission).

Quest releases (`quests`, `releases`, `members get`, `attempts`) carry `announcements` when the quest [posts announcements](authoring.md#announcements), each with `provider` (`telegram` or `discord`), `action` (`post-message`), `chat` or `channel`, `text` and an optional `replyToTriggeringMessage`; it is omitted otherwise. A Discord channel setting's `source` carries `postable: true` when it is an announcement's channel.

Quest releases (`quests`, `releases`, `members get`, `attempts`) carry `retryCooldownMs` when the quest has a [retry cooldown](authoring.md#set-a-retry-cooldown) and `completionCooldownMs` when it has a [completion cooldown](authoring.md#set-a-completion-cooldown). Each quest in `members progress` and `members get` carries `retryAt` while that member's retry cooldown runs, and `nextAvailableAt` while a cooldown or an exhausted budget holds the quest until a known time. All are omitted otherwise. Releases also carry `authorBudgets` when the quest has a [per-giver budget](https://console.domino.run/docs/build/trackers#cap-what-each-giver-gives).

`domino reference --json` prints `{ topics }`, every reference shipped with the installed CLI, and `domino reference <topic> --json` prints `{ topic, markdown }`. Each capability's `reference` in `domino capabilities` names one of these topics. An unknown topic fails with `INVALID_ARGUMENT` and lists the topics.

The management client accepts additional metadata fields in a deployment record's `source` and discards those fields. It still validates the source kind (`direct`, `promotion` or `rollback`) and the deployment identifier required for promotion and rollback.

`leaderboards list` and `leaderboards history` return each board's definition as published, including `includeUnscored` when a board lists members without points.

`domino build --hosting-artifacts --json` prints the `--out` result: `file` is the `.domino-build` directory beside `relay.json`, `files` lists the artifacts written there (`quests.json` and `catalog.json`), and the counts follow. It refuses `--out` with `INVALID_ARGUMENT`.

The `build`, `check` and `deploy` result counts include `offers`. Build results and deployment previews also carry the authored `offers` array in their bundle, so automation that inspects publication contents can handle offers alongside point ledgers, quests and leaderboards. A quest or type built against an SDK runtime carries `runtime: { digest, sdkVersion }` and only its own code; the bundle's `runtimes` array holds each runtime's modules once. Write the bundle through unchanged: dropping `runtimes` makes Relay refuse a reference it has not stored. A manifest holds up to 1,000 quests and 200 quest types; more fails `domino build` and `domino deploy` with `BUNDLE_FAILED` before anything is sent.

A `deploy` preview or published result carries `calendar`: the project timezone change the deployment makes from relay.json's `calendar`, as `{ "timeZone", "effectiveAt" }`, or `null` when it makes none. A null `effectiveAt` means it keeps `timeZone` and cancels a pending change. Publishing makes exactly the change the preview showed; if a day boundary passed or a calendar change since then would make it differ, it fails with "Preview again" and publishes nothing. See the [project calendar](authoring.md#project-calendar).

A `deploy` preview or published result carries `warnings`: what the deployment publishes anyway but the operator should fix, as `{ "code", "message", "integration", "quests" }`. Code `integration-required` names the quests (by key) that need an unconnected integration, such as Discord; their attempts fail with that code until it is connected, and connecting it admits the failed attempts of automatic quests without a tracker again. Code `x-post-rewards-ungated` names the quests that reward X posts without an `inGroup` prerequisite (an X tracker trigger paying the post's author, or a content quest on an X tracker, with declared rewards or an engagement rule); X does not allow apps that reward users for posting, so add `inGroup(group)` with a group of approved creators. The list is empty when there is nothing to report, and a result from an older Relay omits it. Treat an unknown `code` as a notice its `message` explains. A warning never fails the command or changes its exit code; without `--json` the CLI also prints each one on stderr as `Warning: …`.

## Scope and credentials

Each scope value comes from, in order: a flag (`--api-url`, `--organization`, `--project`, `--environment`), then `RELAY_API_URL`, `RELAY_ORGANIZATION`, `RELAY_PROJECT` or `RELAY_ENVIRONMENT`, then the checkout's `domino.json` in the Git common directory, then the nearest `relay.json` (or `--config FILE`). The API defaults to `https://relay.domino.run`. `domino doctor --json` reports each value's source in `scopeSources`. `domino secrets`, `domino storage` and `domino schedules` are the exception for the environment: they use test unless the `--environment` flag says `live`, and ignore `RELAY_ENVIRONMENT` and configuration.

The credential is `RELAY_MANAGEMENT_TOKEN` when set, otherwise the token `domino login` saved for the API URL in `$RELAY_CONFIG_DIR/credentials.json` (default `~/.config/domino`).

### Running without a person

A headless agent or CI uses an organization automation key (`dak_…`) as `RELAY_MANAGEMENT_TOKEN`, or saves it with `domino login --token-stdin`. An organization administrator creates it in Console Settings → API keys; never ask for one in chat. It reaches every project of the organization or a chosen list, with chosen permissions (`read`, `publish`, `inventory`, `points`, `ingest`, `enroll`) and, separately, permission to create projects. A listed key also reaches the projects it creates. It works for `domino create`, `git push` through the credential helper, `domino stage`, authoring, test members and the [staging test toolbar](campaign-testing.md#check-participant-flows-on-staging), in the test environment only:

- Any write in live, including drafts, previews and secrets, is refused with `console-approval-required`. Ask the owner to publish from Console.
- Grants, the team, `/project` changes, `/tokens`, CLI sign-in approval and creating keys are refused with `credential-not-allowed` or `permission-required`.
- `domino create` without the key's create-projects permission fails with `permission-required`; a project outside its list fails with `scope-mismatch`.
- A project it creates connects a Discord server another project of the organization already connects, with `domino integrations discord connect --server <id>` (list them with `domino integrations discord servers`) and the key's `publish` permission. Installing Domino's bot into a new server takes an organization administrator in Console: stop and ask on `server-not-connected-in-organization`.

`domino whoami --json` prints `credential`: `{ kind: "automation", id, name, projects, createProjects, permissions }`, with `projects` null for every project. Its changes are audited as `automation:<id>`.

## Unattended mode

Whoever launches an agent with nobody present to answer it, such as a CI job, a scheduled run or a sandbox, sets `DOMINO_AGENT_MODE=unattended` in its environment. The agent cannot turn the mode on itself. `domino doctor --json` and `domino whoami --json` report it as `agentMode`: `unattended`, or `interactive` for any other value or none. In unattended mode:

- **Login.** Browser sign-in refuses with `UNATTENDED_LOGIN_UNAVAILABLE`, and a command with no credential fails with `AUTH_REQUIRED`. Supply `RELAY_MANAGEMENT_TOKEN`; `domino login` can still save a token it reads from that variable or `--token-stdin`.
- **Feedback.** `domino feedback` appends the report as one JSON line, with a `savedAt` timestamp, to `./domino-feedback.jsonl` or the path in `DOMINO_FEEDBACK_FILE`, and does not send it. It needs no credential. The result is `{ "sent": false, "saved": true, "file", "report" }`. `--submit`, or `DOMINO_FEEDBACK=submit`, sends it as usual.

## Claude and ChatGPT

Operators can connect Claude or ChatGPT to Domino's MCP server at `https://mcp.domino.run` and sign in with their Domino account; the guide is [Connect Claude or ChatGPT](https://console.domino.run/docs/agents/connect). Its tools act as that operator, through the same management operations as the CLI, so their roles apply:

- **Reads, in test and live:** `list_projects`, `get_project`, `list_quests`, `get_quest`, `list_campaigns`, `list_submissions`, `get_submission`, `find_member`, `get_member`, `get_leaderboard`, `list_offers`, `get_usage`, `list_analytics_metrics`, `query_analytics` and `list_audit_events`.
- **Writes, in test only:** `create_test_member`, `act_as_test_member`, `reset_test_member` and `review_submission`. In live they are refused with `console-approval-required`.

There is no tool for publishing, points, deployments, access or arbitrary management calls; use the CLI for those. When an owner asks how to check their campaign from a chat assistant, point them to the guide instead of building an integration.

## Stable and experimental commands

`domino --help` lists stable commands under **Commands** and experimental ones under **Experimental commands**; `contract.json` marks each with `stability`. An experimental command's options, output and result can change in a minor release. It never changes how a stable command behaves.

| Experimental command | Why                                                                                           |
| -------------------- | --------------------------------------------------------------------------------------------- |
| `dev`                | Its preview transport, progress output and result are still changing. Script `stage` instead. |

Commands the CLI runs for Git, such as `credential-helper`, are not part of the contract.

## Webhook operations

`domino webhooks` covers endpoint list/create/update/disable, secret rotation, delivery logs, and replay. `create` and `rotate-secret` print a signing secret; treat their JSON output as a credential. Other commands omit secrets. Both are saved operations: pass `--operation-id <uuid>` to supply the ID, sent as the `actionId`, and after an uncertain answer run `operations resume` to get the same endpoint and secret. Their journal records keep that secret only until it has been printed once: a resume of an operation that has not completed returns it, and once it has, `operations show` and `operations resume` answer `secretRedacted: true` instead, and repeating the command with the same `--operation-id` fails with `CONFLICT`. Rotate the secret to get a new one. A CLI session can read live data but cannot write in live. A live API key with `publish` can perform these operational writes. See [Webhook reference](https://console.domino.run/docs/reference/webhooks) for command arguments and response shapes.

Deprecated generic resource lifecycle commands also use the durable operation journal. `resources archive` and `resources unarchive` save `resource.lifecycle` with the selected kind, key or ID, desired archive state and reason before sending. `operations resume` reuses that scoped payload and action ID, including after a fresh process starts. `resources list` reports `archivedAt` as a timestamp or `null`, and lists offers in `offers`, which archive only with `offers archive|unarchive`; a successful mutation returns `{ resource, changed }`.


## Resource commands

Each family has `list` and `get` style reads and its changes, all built on the management client from `@domino-sdk/relay/server`. Every command takes the scope flags and `--json`; reads need `read`, and changes the permission their management method documents.

| Family | Reads | Changes |
| --- | --- | --- |
| Points | `points get MEMBER LEDGER`, `points list MEMBER LEDGER [--cursor]`, `point-ledgers list` | `points award\|spend\|adjust\|set\|reverse MEMBER LEDGER --reason …`, `point-ledgers create NAME`, `point-ledgers update LEDGER --name …`, `point-ledgers archive\|unarchive` |
| Trackers | `trackers list`, `trackers get`, `trackers records`, `trackers summary`, `trackers content`, `trackers outcomes`, `x lookup HANDLE` (an X handle's stable user ID) | `trackers create KEY [--source KIND]` (`--chain` and `--token` for `--source wallet-balances`) and `trackers update TRACKER` (trackers of the managed sources as data, in test), `trackers ingest TRACKER FILE`, `trackers retry-outcome`, `trackers pause\|resume\|refresh\|archive\|unarchive` |
| Leaderboards | `leaderboards list`, `leaderboards standings`, `leaderboards history`, `leaderboards exclusions`, `leaderboards exclusion-group`, `leaderboard-settlements list\|get\|export` | `leaderboards exclude\|include BOARD MEMBER --reason …`, `leaderboards bind-exclusion-group BOARD GROUP --reason …`, `leaderboard-settlements approve\|reject\|retry-measurement`, `leaderboards archive\|unarchive` |
| Referrals | `referral-programs list`, `referral-relationships list` (each relationship and qualification a [member import](imports.md) recorded names it in `import`), `referral-relationships history MEMBER` | `referral-programs qualify PROGRAM MEMBER [--disqualify]`, `referral-rewards resolve ENTITLEMENT`, `referral-relationships pause\|resume`, `referral-programs archive\|unarchive` |
| Offers and redemptions | `offers list`, `redemptions history [--member] [--cursor] [--limit]` | `redemptions cancel REDEMPTION --revision N`, `offers pause\|resume`, `offers archive\|unarchive OFFER --reason …` (`publish`, in test), `redemptions pause\|resume` |
| Rewards | — (read entitlements with `members get MEMBER`) | `entitlements restore ENTITLEMENT --revision N --reason …` (a voided entitlement; `review` permission) |
| Completions | — (read revoked ones in `revokedCompletions` of `members get MEMBER`) | `completions restore COMPLETION --reason …` (a revoked completion; `review` permission) |
| Profiles | `profiles settings` | `profiles configure FILE`, `profiles moderate MEMBER [--reset-name] [--remove-avatar] [--lock\|--unlock]` |
| Calendar | `calendar get` | `calendar configure --time-zone ZONE` |
| Members and groups | `members list\|get\|progress\|lookup\|identities\|counters` (deprecated: `identity-links\|oauth-accounts`), `groups list\|get\|members\|member` | `members unlink-identity\|relink-identity\|identity-sign-in` (deprecated: `relink`), `members moderate-counter`, `groups create\|update\|archive\|unarchive\|add\|remove` |
| Quests and releases | `quests list\|get`, `quests enrollments`, `releases list [--quest]\|get`, `attempts list [--quest]\|get`, `metrics get`, `usage active-members`, `usage real-people-in-test`, `usage plan`, `pauses list` | `quests pause\|resume\|archive\|unarchive\|enroll\|withdraw`, `attempts accept-anyway ATTEMPT --reason TEXT` (`review`) |
| Test members (test only, `publish`) | `test-members list`, `test-members history MEMBER [--since MS]` | `test-members create NAME`, `test-members act MEMBER QUEST [--input JSON] [--release RELEASE]`, `test-members pass MEMBER QUEST [--completion-key KEY]`, `test-members accept-anyway MEMBER ATTEMPT`, `test-members advance MEMBER (--days N \| --hours N \| --minutes N \| --to INSTANT)`, `test-members accept-referral MEMBER (--code CODE \| --inviter MEMBER)`, `test-members link-wallet\|unlink-wallet MEMBER --chain CHAIN --address ADDRESS`, `test-members link-account MEMBER --provider x\|discord (--id ID \| --username HANDLE)`, `test-members unlink-account MEMBER --provider x\|discord --id ID`, `test-members record MEMBER --tracker TRACKER [--data JSON] [--as author\|messageAuthor]`, `test-members reset MEMBER` |
| Dashboards | `dashboards list [--include-archived]`, `dashboards get KEY [--range 7d\|30d\|90d\|all]` (see [dashboards](#dashboards)) | `dashboards archive\|unarchive KEY --reason …` |
| Member imports (`publish`; reads need `read`) | `imports list`, `imports get IMPORT`, `imports rows IMPORT [--status STATUS]` | `import --file FILE [--source NAME] [--import IMPORT] [--dry-run]`, `imports apply\|cancel\|stop IMPORT`, `imports revert IMPORT --reason TEXT` (see [member imports](imports.md); live imports are applied in Console; a stopped or reverted import reads `applied` with `stoppedAt` or `revert` set) |

To simulate a whole campaign with the test-member commands, read [campaign testing](campaign-testing.md) (`domino reference campaign-testing`). For tracker collection options, record filters, trigger outcome codes and engagement accounting, read [trackers](trackers.md) (`domino reference trackers`). For quests that check a wallet's EVM contract events or Solana transactions, read [on-chain actions](onchain.md) (`domino reference onchain`).

Every change except `point-ledgers update`, the `groups update\|archive\|unarchive` state changes and the member import commands is saved before it is sent. Member imports resume by themselves: run `domino import` again with `--import IMPORT`, since a row with the same `ref` replaces the earlier one. Saved changes resume as described in [Recover after restarting the CLI](#recover-after-restarting-the-cli). The CLI checks the input with the SDK's own schema first: input it refuses exits 2 with `INVALID_ARGUMENT` and leaves nothing saved or sent. A change the API refuses outright on its first attempt (exit 1, `outcome: "failed"`) is not kept either, since nothing was applied; only a change that may have been applied stays saved for `operations resume`. When no environment comes from `--environment`, `RELAY_ENVIRONMENT` or relay.json, a change is saved for the only environment your credential reaches, such as a personal API key's own, and otherwise for `test`. Release history (`releases list` and `releases get`) also needs `publish`.

Quest and release records can include optional `presentation.steps`, `image`, `imageAlt`, and `requirements`. Steps retain their authored order and can carry an HTTPS link action. `configuration.presentation` records a Console or quest-instance override; `configuration.presentationDefaults` records the defaults used when resetting it. These fields also appear wherever attempt or member results include a release. Read the resolved `presentation` to render a quest; the configuration fields describe its authoring state. See [quest authoring](authoring.md) for the shapes. Older records can omit these fields.

A text setting in a release's settings can carry an optional `picker`, such as `"telegram-chat"` from `settings.telegramChat()`: Console then offers a picker for its value, which is still a string. Treat an unknown picker as plain text. Six pickers also constrain the value: `"amount"` (`settings.amount()`), a non-negative base-unit integer string, with an optional `decimals` field for showing it in whole tokens, `"chain"` (`settings.chain()`), `"solana"`, Solana mainnet's CAIP-2 ID or an EVM chain ID, and `"nft"` (`settings.nft()`), an NFT collection such as `"eip155:1/erc721:0x…"`, `"eip155:8453/erc1155:0x…/1"` or `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/nft:<collection mint>"`, whose EVM contract publication and Console edits also check through ERC-165, and whose Solana mint they check is a Metaplex collection NFT, `"contract-call"` (`settings.contractCall()`), a contract's CAIP-10 ID and a view or pure function signature returning one uint, int, bool or address, such as `"eip155:8453:0x… function stakedOf(address) view returns (uint256)"`, whose chain they also check, and `"contract-event"` (`settings.contractEvent()`), an EVM contract event such as `"eip155:8453:0x… Transfer(address indexed from, address indexed to, uint256 value)"`, whose chain publication and Console edits check reports a finalized or safe block, and whose address they check holds a contract, and `"solana-program"` (`settings.solanaProgram()`), a base58 Solana program address, which they check is an executable program. A release with such a setting may carry `solanaIdls`: by setting key, `{ program, source, idl }`, the program's Anchor IDL as JSON, stored from its on-chain IDL account when the setting was saved (`source: "chain"`) or pasted by an operator (`"pasted"`); treat other `source` values as unknown. `ctx.wallet.transactions()` decodes the program's instructions with it ([Solana transactions](onchain.md#solana-transactions)).

The bare `status`, `quests` and `releases` commands, and `groups show`, are deprecated and removed in 2.0.0; `contract.json` gives every deprecated command a `deprecated` note naming its replacement. They still print Console's former pages, and without `--json` the three page reads warn on stderr. Use `metrics get`, `attempts list`, `quests list` and `pauses list` for `status`; `quests list` and `quests get` for `quests`; `releases list` and `releases get` for `releases`; and `groups get` for `groups show`.

`usage active-members` reads the project's live active members this UTC month against its plan's `limit` (null when the plan has none), and the six earlier months, newest first (`management.usage.activeMembers()`, `GET /management/v1/usage/active-members`). An active member sent a signed-in participant request on a UTC day of the month after the day they joined; imports, tracker records and test members never count. See [active members](https://console.domino.run/docs/concepts/participants#active-members).

`usage real-people-in-test` reads the real people who joined the test environment against the plan's `limit`, and whether the limit is `enforced` (`management.usage.realPeopleInTest()`, `GET /management/v1/usage/real-people-in-test`). Test members never count. When it is enforced and full, new people's sign-ins to test fail with `limit-reached`; people who already joined still sign in. See [real people in test](https://console.domino.run/docs/concepts/participants#real-people-in-test).

`usage plan` reads the project's plan as it stands (`management.usage.plan()`, `GET /management/v1/usage/plan`), with `read` permission and whichever environment you select: `plan` and its `source` (`stripe`, `staff`, `grandfathered`, `trial`, or null on Free), whether it may serve live and why (`live.reason`: `plan`, `trial` or `grace` with `until`, or `free` or `trial-ended`), `liveState` while live is in notice or offline, the organization's one `trial` (`unused`, `active`, `grace` or `used`, and its `project`), whether the plan includes a custom domain, the AI builder balance by pool (`ai`), the Stripe `subscription` as `billing show` gives it to an administrator, and `enforcement`, whether plan limits are enforced. It buys nothing; see `billing` below for that. See [going live needs a plan](https://console.domino.run/docs/concepts/promotion#going-live-needs-a-plan).

`billing show` reads whether Stripe billing is open to you (`available`, and `reason`: `admin-required`, `staff-only` or `not-configured`) and, when it is, the project's subscription as Stripe last reported it (`management.billing.read()`, `GET /management/v1/billing`). `billing checkout --plan launch|growth|scale [--interval month|year]` returns a Stripe Checkout URL that subscribes the project, and `billing portal` the organization's Stripe customer portal (`management.billing.checkout()`, `management.billing.portal()`); open the URL in a browser. Both need an organization administrator's sign-in, never an API key. Until the billing launch they are open to Domino's own organizations only, and everyone else gets `billing-staff-only`.

`billing organization` (`management.billing.organization()`, `GET /management/v1/organization/billing`) reads the organization's billing for any of its members, with no project selected: each project you can read with its `plan`, `source`, `live`, `liveState` and Stripe `subscription`, the organization's Free projects against their limit (`freeProjects`), the free AI allowance its Free projects share (`freeAi`), the `trial`, `enforcement`, and in `billing` whether you may open the customer portal, as `billing show` says. An app key, or a Playground session fixed to one project, gets `credential-not-allowed`.

`billing change --plan launch|growth|scale [--interval month|year]` (`management.billing.changePlan()`, `POST /management/v1/billing/plan`) moves a paying project's subscription to another plan. A higher plan, or annual billing, applies now: Stripe invoices the prorated difference at once, and when that payment needs the customer (a bank's authentication, say), nothing changes until the returned `invoiceUrl` is paid. A lower plan, or monthly billing, is held for the renewal, so the period already paid keeps its plan; `billing show` names it under `scheduledChange`. Asking for the current plan drops a held change. `billing cancel` (`management.billing.cancel()`) ends the subscription when its period does; the project keeps its plan until then, and `billing resume` takes it back. `subscription.endedFor` is `project-deleted` while a deleted project's subscription runs out (restoring the project resumes it), and `project-transferred` when the project came from another organization whose subscription ends with its period: `billing checkout` then subscribes the project again with no charge until that period ends. `billing topup [--quantity 1-10]` (`management.billing.topup()`, `POST /management/v1/billing/topup`) returns a Checkout URL buying $10 packs of AI builder credit for the project, on any plan; the credit is added once Stripe reports the payment, once per purchase, and never expires. While an invoice is overdue, `billing show`'s `subscription.unpaid` gives when it first failed and the two deadlines: from `promotionsRefusedAt` every promotion in the organization is refused with `plan-required` (`reason: "unpaid"`), and at `offlineAt` the project's live goes offline until it is paid. A project without a subscription that holds its plan gets `no-subscription`, and a plan change while the subscription is set to cancel gets `subscription-canceling`. The same administrator and staff rules apply.

## Project analytics

`domino analytics get [--range 7d|30d|90d|all] [--ledger KEY]` reads how a project is doing in one environment; `--range` defaults to `30d`. It needs `read`. Without `--json` it prints a summary of every total and its change; `--json` prints the read itself, which `management.analytics.read({ range, ledger })` from `@domino-sdk/relay/server` also returns (`GET /management/v1/analytics?range=…&ledger=…`). When someone asks how a project is doing, quote these numbers instead of computing your own from other reads.

**Range.** The range ends now (`range.to`). `7d`, `30d` and `90d` are that many [project-calendar](https://console.domino.run/docs/concepts/calendar) days, today included; `all` starts on the day of the project's earliest dated record the read counts: a membership, attempt, decision, completion, entry in the chosen ledger, fulfilled redemption, accepted referral or active qualification. That is usually the first membership, but a completion imported with its original date can be older. `7d` and `30d` have one bucket per day; `90d` and `all` one per week, from Monday, with the first and last weeks cut to the range. Days start at local midnight; when the project's timezone changes, the day or week already open finishes in the old timezone, and each bucket's `key` names its timezone (`2026-10-08@Europe/Budapest`, `2026-W41@UTC`). `previous` is the range of equal length that ends where this one starts; it is absent for `all`.

**Metrics.** Each metric is `{ total, previousTotal, series }`: the range's total, the previous range's total (absent for `all`), and one value per bucket in `range.buckets`. Records count as they stand now: a completion revoked or a redemption canceled later leaves the bucket it was in.

| Metric | Counts | Dated by |
| --- | --- | --- |
| `members.signUps` | Members who joined the project, as the Members page counts them | When they joined |
| `members.active` | Distinct members with an attempt; the total counts each member once | Attempt creation |
| `quests.attempts` | Attempts | Attempt creation |
| `quests.completions` | Quest completions never revoked, as Console counts them | Completion |
| `quests.rejections` | Rejected attempts | Decision |
| `quests.needsReview` | Attempts the evaluator sent to staff review, whatever staff decided later | Attempt creation |
| `points.awarded` | Points in the ledger's award entries | Entry's effective date |
| `points.spent` | Points in the ledger's spend entries, as a positive number | Entry's effective date |
| `points.net` | Every entry's signed amount: awards, spends, adjustments (offer refunds and imported balances among them) and reversals | Entry's effective date; a reversal takes the date of the entry it reverses |
| `points.membersEarning` | Distinct members awarded more than 0 points; the total counts each member once | Entry's effective date |
| `offers.redemptions` | Fulfilled offer redemptions | Redemption |
| `referrals.accepted` | Accepted referrals | Acceptance |
| `referrals.qualified` | Active referral qualifications, one per program and friend. A suspended referral still counts, as it does toward inviter rewards, although Console labels it **Suspended** | First qualification |
| `referrals.acceptedQualified` | Of the referrals accepted in the range, how many have an active qualification now, in any program, each friend once, suspended ones included. Only `{ total, previousTotal }`, no series. `acceptedQualified.total ÷ accepted.total` is the share of the range's friends who have qualified since, never above 100%; a friend accepted before the range who qualifies inside it counts in `qualified` but not here | Acceptance |

A decision is dated on the wall clock. Attempt creation, completion and quest points are dated on the member's clock, which an [advanced test member](campaign-testing.md#3-run-the-days) moves up to 90 days ahead; activity dated after now is left out until then. An evaluator that asks for review on a quest with human review off rejects the attempt: it counts as a rejection, not as review.

**Quests.** `quests.byQuest` lists each quest with an attempt, completion or decision in the range, most attempted first, by quest ID with its current title:

- `attempts`, `completions` and `needsReview` as above, with `series.attempts` and `series.completions` per bucket.
- `decided`: attempts accepted, progressed or rejected in the range, by decision. `rejected`: those rejected.
- `rejectionRate`: `rejected ÷ decided`, or `null` below 10 decided attempts.
- `topReasons`: the 5 most common rejections. Rejections with a code group by it, whether an evaluator's reject code or one Relay writes (`budget-exhausted`, `defer-expired`, `defer-limit-exceeded`), and `reason` is the latest one's reason; a budget's reason names the budget, but its rejections still form one group. Rejections without a code group by exact reason. Equal counts list coded groups first.
- `rejectionsBySource`: `budget` is `budget-exhausted`; `expired` is `defer-expired` or `defer-limit-exceeded`; `staff` is a reviewer's rejection, of an attempt waiting for review or of a deferred one, and Relay's rejection of a quest whose completion an operator revoked; `evaluator` is every other rejection.

**Points.** The points section reads one ledger: `--ledger` by key or ID, archived ledgers included, or else the first ledger that is not archived. `points.ledgers` lists every ledger with `archived`, and `points.ledger` is the one read, or `null`, with zeros, when there is none. `spent` and `net` appear only when the ledger has ever had a spend entry (`hasSpending`). An unknown ledger fails with 404 `not-found`.

**Sections that depend on the project.** `offers` appears only when the project has an offer, with `byOffer` listing every offer, most redeemed first. `referrals` appears only when it has a referral program, archived ones included. `testEnvironment` appears only in test: `lastResetAt`, when a test member was last reset, which deletes that member's history, and `shiftedClockMembers`, how many have an advanced clock.

For range `all`, `members.signUps.total` and `quests.completions.total` equal `metrics get`'s `members` and `completions`, imported completions included, except for records dated after now (a test member's advanced clock) or not at all.

## Metric catalog

`domino analytics metrics` lists Domino's closed metric catalog, read from the API (`management.analytics.catalog()`, `GET /management/v1/analytics/catalog`): each metric's `id`, `definition`, `kind` (`count`, `distinct` or `rate`), the `breakdowns` and `filters` it accepts, the `charts` it allows, whether it needs a point `ledger`, and a rate's `minimumDenominator`. It needs `read`.

`domino analytics query --metric ID [--range 7d|30d|90d|all] [--breakdown DIMENSION] [--filter NAME=KEY[,KEY]]... [--ledger KEY] [--limit N]` reads one metric (`management.analytics.query({ metric, range, breakdown?, filter?, ledger?, limit? })`, `GET /management/v1/analytics/query?metric=…&range=…&filter.quest=a,b`). `--range` defaults to `30d` and buckets exactly as [project analytics](#project-analytics) does. `--filter` repeats, one per filter, and its keys are keys or IDs. Without `--json` it prints the total with its change and one line per breakdown row, with `Other` last; `--json` prints the result. There are no formulas or custom queries: answer a question the catalog can't express by saying so, not by combining metrics.

The result has `metric`, `kind`, `range`, `previous` (absent for `all`), `ledger` (the key of the ledger a points metric read) and `value` (`{ total, previousTotal, series }`). With no breakdown or filter, `value` equals the same metric in `analytics get`; `groups.*`, `trackers.records` and `leaderboards.settledParticipants` are only in the catalog. With `--breakdown`, `breakdown` has the `dimension`, its `rows` and `other`:

- **Rows.** Each row has `key` (a resource's key, or its ID when it has none; a group's key; a source; an outcome; or a reason's code or text), `title`, `archived`, `total`, `previousTotal` and `series`. Rows come by total, largest first, with an empty rate last; ties go to the larger denominator, then by title. A quest, offer, program, group, tracker or leaderboard breakdown has a row for every matching resource that is not archived, with zeros when it had no activity, and for an archived one with activity in the range or the range before.
- **`other`.** Rows past `--limit` (1 to 100, default 100) are summed into `other`, with `rows` saying how many. For a rate, `other` is the summed numerators over the summed denominators of the rows it holds.
- **`value`** covers every row, `other` included.

**Filters** narrow what is counted before it is split. A list matches any of its keys; different filters all apply. `quest`, `offer`, `program` and `leaderboard` keep those resources. `tracker` keeps activity on the quests whose current release is triggered by the tracker, and on `trackers.records` the tracker's own outcomes. `group` keeps members in the group now, whatever their membership was when they acted, and on `groups.joins` and `groups.leaves` the changes in that group.

**Rates** are fractions from 0 to 1, and `null` below 10 in the denominator: in the total, the previous total, each bucket and each row.

| Metric | Breakdowns | Filters |
| --- | --- | --- |
| `members.signUps` | none | `group` |
| `members.active` | none | `group` |
| `quests.attempts`, `quests.completions`, `quests.needsReview` | `quest` | `quest`, `group`, `tracker` |
| `quests.rejections` | `quest`, `source`, `reason` | `quest`, `group`, `tracker` |
| `quests.rejectionRate` | `quest` | `quest`, `group`, `tracker` |
| `points.awarded`, `points.spent`, `points.net`, `points.membersEarning` | none | `group`; `--ledger` is required |
| `offers.redemptions` | `offer` | `offer`, `group` |
| `referrals.accepted` | none | none |
| `referrals.qualified` | `program` | `program` |
| `referrals.qualificationRate` | `program` | `program` |
| `groups.joins`, `groups.leaves` | `group` | `group` |
| `trackers.records` | `outcome`, `tracker` | `tracker` |
| `leaderboards.settledParticipants` | `leaderboard` | `leaderboard` |

The counts are defined in [project analytics](#project-analytics). The catalog adds:

- `quests.rejectionRate`: rejected attempts ÷ attempts accepted, progressed or rejected, both by decision. Its quest rows are `analytics get`'s `byQuest[].rejectionRate`.
- `referrals.qualificationRate`: of the referrals accepted in each bucket, the share whose friend has an active qualification now, suspended referrals included; for a `program` row or filter, a qualification in that program. Every program row shares the denominator, so the rows don't add up to the total. With no filter its total is `acceptedQualified.total ÷ accepted.total`.
- `source` rows are `evaluator`, `staff`, `budget` and `expired`, as `rejectionsBySource` counts them. `reason` rows group by rejection code when there is one, otherwise by the exact text, titled with the latest rejection's reason; a code and the same text stay separate rows.
- `referrals.accepted` takes no program: a friend accepts an inviter, not a program.
- `groups.joins` and `groups.leaves`: members added to and removed from a group, by when the change happened. A change counts only when it changed the member's membership from their previous change in that group, so a first change counts only as a join. Repeats that changed nothing, and the undated rows Domino recorded for memberships older than its group history, don't count. Each change is one Console **Audit history** record (`group-member-added`, `group-member-removed`).
- `trackers.records`: tracker trigger outcomes, one per quest and record observation, by when the record was observed. `outcome` rows are `admitted` and `skipped`. A retry or re-decision overwrites an outcome, so each bucket shows today's outcome of its records, not the outcome they had then. They are the records a tracker's trigger outcomes list in Console.
- `leaderboards.settledParticipants`: the rows of leaderboard settlements, one per member and settlement, by the settlement's cutoff. Pending and approved settlements count; a settlement rejected later leaves its bucket. Settlements are periodic, so it draws as `number`, `bar` or `table`, not `line`.
- With `--range all`, a group, tracker or settlement metric starts at its own earliest record when that is older than the project's.

`--ledger` takes a key or ID, archived ledgers included, so an archived ledger's history stays readable.

| Code (422) | Meaning |
| --- | --- |
| `unknown-metric` | No metric has this ID. |
| `breakdown-not-accepted`, `filter-not-accepted` | The metric doesn't take this breakdown or filter. |
| `ledger-required`, `ledger-not-accepted` | A points metric needs `--ledger`; other metrics take none. |
| `invalid-limit` | `--limit` is outside 1 to 100. |
| `unknown-reference` | A quest, tracker, offer, program, leaderboard or ledger key names nothing; the error has `filter` and `key`. |
| `group-not-found` | No group has this key in the environment; the error has `group`. |

## Dashboards

A dashboard is a set of catalog-metric panels defined in code with `defineDashboard` and deployed from `relay.json` `dashboards` (see [dashboards](authoring.md#dashboards) for the definition and the deploy rules). To choose its metrics and check it before handing it over, read [analytics](analytics.md). Both reads need `read`.

`domino dashboards list [--include-archived]` (`management.dashboards.list({ includeArchived })`, `GET /management/v1/dashboards?includeArchived=true`) lists each dashboard's `key`, `title`, `description`, `position`, `archivedAt` and `updatedAt` (when its current version was published), ordered by `position` with dashboards without one last, then by title. Archived dashboards appear only with `--include-archived`.

`domino dashboards get KEY [--range 7d|30d|90d|all]` (`management.dashboards.read({ key, range })`, `GET /management/v1/dashboards/KEY?range=…`) reads one dashboard, archived ones included, over `--range`, or its `defaultRange` without one. Without `--json` it prints each section and each panel's total, breakdown rows and `Other`, or the panel's error: use it to check a dashboard you built. `--json` prints the read: the summary fields, `definition` (as stored, with `defaultRange` and each panel's `width` filled in), `range`, and `panels`, one per panel in reading order, each with its `key` and either `result` or `error`.

- **`result`** is the panel's [metric query](#metric-catalog) result for the dashboard's range, metric, breakdown, filters and ledger. Every panel reads as of the same instant. A broken-down `line` or `bar` panel reads the top 8 rows with the rest in `other`; a `table` panel up to 100.
- **`error`** is `{ code, message }` for a panel whose query failed: `group-not-found` for a group the environment lacks, `panel-timeout` for a query that ran past its time budget (each statement 2 seconds, and the read about 18 seconds, after which panels not yet read answer `panel-timeout`), `panel-failed` for any other failure, or another metric query code. The other panels still have results.

An unknown key is 404 `not-found`. `domino dashboards archive|unarchive KEY --reason TEXT` (`management.dashboards.archive|unarchive(key, { reason })`, `POST /management/v1/dashboards/KEY/archive|unarchive`) archives and restores one in test (`publish`), returning `{ dashboard, changed }`; live refuses it with `promotion-required`. Each is a saved change; see [lifecycle retries](#lifecycle-retries). Archiving takes it out of the default list; deploying its code again leaves it archived.

## Lifecycle retries

Resource-specific archive, unarchive, pause and resume commands use the matching management path verb, and `trackers refresh` its own. Each is a saved resource change: pass `--operation-id <uuid>` to supply its ID, and resume an uncertain one with `operations resume`. `--action-id <uuid>` instead sends that action ID without saving the change, as these commands did before they were saved; retry identical inputs with the same ID. Passing both fails with `INVALID_ARGUMENT`. `trackers pause|resume`, `groups archive|unarchive` and `schedules pause|archive|resume` take an optional `--reason TEXT` for the audit log; quest, offer and redemption pauses require one. Quest, offer and redemption pauses require the latest `--revision` (`pauses list` shows quest and redemption revisions, `offers list` offer revisions); a stale revision refuses the change. The retained `resources archive|unarchive` and `quests archive|unarchive` commands support the saved operation journal and `operations resume`. `webhooks disable` remains a deprecated alias of `webhooks pause`. `trackers resume` on an X tracker that Domino paused because its X budget is spent (`pauseReason: "x-budget-exhausted"` in `trackers get`) exits with the API's 409 `x-budget-exhausted` until `resumesAt`; one paused with `test-window-ended` resumes for another 14 days. `trackers create|update --refresh-seconds` below the plan's minimum still saves, and `effectiveRefreshSeconds` shows the interval the tracker runs at.

## Retry operator corrections

Quest enrollment and withdrawal, settlement reject and identity changes are saved resource changes with `--operation-id`, like the lifecycle commands above; `--action-id` sends them unsaved, and an `actionId` in a `members unlink-identity`, `relink-identity`, `identity-sign-in` or `relink` file counts as `--action-id`. Inventory rename/archive/unarchive/stock correction and webhook delete accept `--action-id`. Save that ID with the exact input before sending. After an uncertain response, retry the same command with the same ID, revision or creation-time fence and reason. A new correction uses a new ID. A reused ID with different input returns `action-id-reused`.

Read state with `quests enrollments`, `stock-items list`, `webhooks get` or `states`, `leaderboard-settlements get`, or `members identities` before preparing a correction. `members identities` marks an X, Discord or Telegram account a member import vouched for with `imported: true`. See [operator undo commands](authoring.md#operator-undo-commands) and [webhook deletion](webhooks.md#delete-an-endpoint) for arguments and refusal recovery. CLI session writes remain Test-only.

## Scheduled jobs

`domino schedules list --json` reads the active staged build in test, or the live app's build with `--environment live`. It returns `{ state, schedules }`; each schedule has its `declaration`, `nextRun`, `blocked` and nullable `lastRun`. A run includes `id`, `build`, `startedAt`, `scheduledAt`, `manual`, `status`, nullable `duration` in milliseconds, nullable `error` and `result`: the app's `{ outcome, summary?, metrics?, diagnostics? }`, null when it sent none and absent on runs from before results. Times are epoch milliseconds. `nextRun: null` means the schedule is stopped or blocked. The six control commands share this result shape. `schedules runs NAME --json` returns `{ runs }`, the schedule's latest 20 runs newest first, in the same run shape. All seven ship result schemas in the automation contract.

With `publish`, `schedules run NAME` queues one manual run, and `pause`, `archive` and `resume` change project-wide state, with an optional `--reason TEXT` for the audit log. `recover NAME --yes` acknowledges the previous job has stopped and permits future ticks. JSON mode never prompts, so recovery without `--yes` fails with `INVALID_ARGUMENT` before sending. A queued or blocked job refuses another run with `CONFLICT`; inspect it and recover only after checking its work has stopped. See [hosted jobs](hosted.md#scheduled-jobs).

## Badge catalog and member corrections

`domino badges list --json` returns the managed catalog, including `archivedAt`. `domino badges member MEMBER --json` returns `{ items: [{ badge, awardedAt }] }`. The create/update/archive/unarchive commands return a managed badge. Award/revoke commands return `{ badge, member, awarded, changed }`: `changed: false` means the member already had the requested presence. Each write uses the normal saved operation journal and supports uncertain-answer recovery. Award and revoke require `--reason`.

Rewards in `quests get`, `quests list`, `releases get`, `attempts get` and the other results that carry a reward bundle may include a group membership reward: a `kind: "tier"` reward whose `reward.group` is the group key. Treat `reward.badge` and `reward.group` as distinct from an ordinary tiered reward, which has neither. `groups members` and `groups member` list a member a quest reward added like any other; Console **Audit history** names the completion that added them. See [group membership rewards](authoring.md#group-membership-rewards).
