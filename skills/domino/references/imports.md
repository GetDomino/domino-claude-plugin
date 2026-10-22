# Import members from another platform

A member import brings members, their opening point balances, their quest progress and their referrals into one project and environment. It runs in two steps: a check (the dry run), which changes nothing, then apply. Every import command needs `publish` permission; `domino imports list|get|rows` need `read`. The human guide is [Import members from another platform](https://console.domino.run/docs/use/imports).

## Write the file

JSONL, one member per line. Map the old platform's quest IDs to the project's quest keys before writing it.

```json
{"ref": "legacy-1", "identity": {"kind": "app:legacy", "subject": "u1"}, "accounts": [{"kind": "x", "subject": "1001"}], "points": [{"ledger": "xp", "balance": 1200}], "quests": [{"quest": "daily-check-in", "actions": [{"at": "2026-09-01T10:00:00Z"}]}]}
```

| Field | Rules |
| --- | --- |
| `ref` | The member's ID on the old platform, 1–200 characters, unique in the import. A row with the same `ref` replaces the import's earlier row. |
| `identity` | What the member will sign in with: `{"kind": "app:<name>", "subject"}` (exchanged by the app login integration of that kind), `{"kind": "wallet", "chain": "solana" \| "eip155" \| "cosmos", "address", "network"?}` (`network` is required for Cosmos), or `{"kind": "x", "subject": "<numeric X user ID>"}`. |
| `accounts` | Up to 20 accounts the organization vouches for: wallets and X accounts in the same shapes, `{"kind": "telegram", "subject": "<numeric Telegram user ID>"}` and `{"kind": "discord", "subject": "<numeric Discord user ID>"}`. At most one X, one Telegram and one Discord account. They never sign in. |
| `points` | Up to 50 `{"ledger": key or ID, "balance": positive integer}`. |
| `quests` | Up to 200 `{"quest": key or ID, "actions": [{"at": epoch ms or ISO date-time, "key"?: string}]}`, 1–3660 actions each. |
| `referral` | Optional `{"inviter", "acceptedAt", "qualified"?}`. `inviter` is another row's `ref` (a string) or the sign-in `identity` of an existing member (an object, as `identity`). `acceptedAt` is epoch ms or an ISO date-time. `qualified` is up to 50 `{"program": referral program key or ID, "at"}`. |
| `consent` | Optional, experimental. Up to 10 `{"channel", "state": "granted" \| "withdrawn", "at": epoch ms or ISO date-time, "wording"?}`, one per channel. Only `email` is accepted. `at` must be in the past. `wording` is the text the member agreed to, verbatim, at most 2,000 characters, required for `granted`. |

## Run it

```sh
# Test: check, print the summary, change nothing.
domino import --file members.jsonl --source legacy-domino --dry-run --json
# The refused rows and why.
domino imports rows IMPORT --status refused --json
# Test: add the fixed file to the same import, check it again and apply the ready rows.
domino import --file members.jsonl --import IMPORT --json
```

`domino import` uploads rows in batches of at most 500 rows, 16 MB and 1,000,000 JSON values (Relay reads at most 64 MB and 2,000,000 values per `addRows` call and answers 413 past either; a larger row goes alone, and a row too large for one call, 63 MB or 1,990,000 values, fails before anything is sent), checks the import, and polls it until the check (and in test, the apply) finishes. It prints the import: `state`, `summary` (`members.created`, `members.existing`, `accounts`, `points` by ledger, `actions` by quest, `referrals.accepted` and `referrals.qualified` by program, `refusals` by code) and `progress` (rows applied and refused).

In live, `domino import` without `--dry-run` exits 2 with `INVALID_ARGUMENT`. Check it with `--environment live --dry-run`, then ask a person with publish permission to apply it in Console from **Members** > **Imports**. Applying a live import with a CLI session or API key is refused with `console-approval-required`.

Other commands: `domino imports list`, `domino imports get IMPORT`, `domino imports apply IMPORT` (test; waits until applied), `domino imports cancel IMPORT` (open or checked imports), `domino imports stop IMPORT` (applying imports), `domino imports revert IMPORT --reason TEXT` (applied or stopped imports). In code, use `management.memberImports.create|addRows|check|get|rows|apply|cancel|stop|revert|list`; `check` and `apply` return at once, so poll `get` until `state` leaves `checking` or `applying`. `check`, `apply`, `cancel` and `stop` take an optional `{ actionId }` before the call options; a retry with the same one answers with the first result instead of being refused for the import's new state.

An import's `state` is `open`, `checking`, `checked`, `applying`, `applied`, `stopped`, `reverting`, `reverted` or `canceled`; Relay may add states, so handle one you don't know. Read `state`, not `status`: `status` reads `applied` for a stopped, reverting or reverted import too, so filtering on it counts those as applied. A Relay older than `state` omits it.

## What applying does

- **A row's identity names the member.** If the identity already belongs to a member, the row adds to that member. Otherwise it creates one and joins them to the project. Rows never merge members.
- **Vouched accounts count as linked, from when the import attached them.** Attribution (X, Telegram and Discord tracker records), `ctx.x`, Discord and Telegram checks, and wallet checks trust them; earlier records are historical. Console labels them imported, and identity states carry `imported: true`, `source: "import"` and the import's ID as `import` (absent for accounts imported before Relay recorded it). The member's own X, Telegram or Discord link, or wallet signature, replaces the vouched link of that kind. Another member cannot later prove a vouched account while the member holds it, and signing in with a vouched Telegram or Discord account is refused with `sign-in-method-disabled`. An operator unlink (`members.identityStates.unlink`, `domino members unlink-identity`) of a vouched account or wallet the member never signed in with or proved releases it: its rows go, its owner can link it and sign in with it, and later imports and app-login vouching refuse it for that member with `identity-unlinked`. A revert finds it gone and keeps nothing.
- **Opening balances are `adjust` entries with `import` set.** They emit no `points.awarded`, count on balance leaderboards only, and appear with `imported: true` in the member's own point history.
- **Progress pays no rewards.** Each action is an accepted action under the quest's completion rule, at its `at` time, on one attempt with code `imported` (`codeSource: "relay"`): `once` completes once, `keyed` once per `key`, a streak counts each action's project-calendar day, and a counter counts distinct keys (an action without `key` counts alone; actions outside a calendar window's lateness are late). Completion limits and prerequisites count imported completions.
- **Referrals pay nothing at import.** The row writes the accepted referral and its qualifications at their times, with `import` set on each in `management.referrals.participants()`: no inviter or invitee reward, no milestone, no bonus, no `referral.qualified` webhook. Imported qualifications count toward the inviter's qualified total, so a milestone pays only when a later friend reaches its count; the friend's later awards pay the program's bonus as usual. A signup program is qualified only if the row names it. An inviter's row applies before the rows that name it by `ref`, in any file order.
- **Consent is recorded as `source: "imported"`.** Each channel gets a consent record with the row's `at` and `wording`, read back by `management.members.consent(id)` and `consentHistory(id)` (`domino members consent MEMBER`) and by the member's `relay.consent.get()`, and a `member-consent` event for destinations, in the row's transaction. A channel whose current record is the member's own choice (`source: "member"`) keeps it whatever the row's `at`; one whose current record is imported is replaced only by a later `at`; a row whose state and wording already match records nothing. An import never overwrites a choice the member made in Relay. A revert leaves imported consent in place.
- **Apply re-checks every row.** Each row applies once, in its own transaction; a row that became invalid since the check is refused instead.
- **Stopping ends an apply.** `domino imports stop IMPORT` lets the row being applied finish, applies no more, and leaves the import `stopped`, with `appliedAt` null and `stoppedAt` set, and `progress.applied` rows applied; the rest stay `ready`. A stopped import is final: `apply`, `check` and `cancel` refuse it with `import-status`, so put the remaining rows in a new import. Stopping a stopped import returns it unchanged; stopping any other status is refused with `import-status`. CLI sessions and API keys may stop a live import.
- **Reverting takes back what the applied rows granted.** `domino imports revert IMPORT --reason TEXT` (or `management.memberImports.revert(id, { reason })`) works on an `applied` or `stopped` import, one applied row at a time in the background; the CLI waits for it. For each row it adds a `reverse` point entry, naming the import, for each opening balance, taking only points the member still has free (the total less held points), as a completion's revocation does; points a member already spent stay spent and are listed, so no balance goes below zero. It revokes each imported completion with reason `import-reverted`, freeing its occurrence and returning the budget it used. It removes each wallet, account and app identity the row linked, unless the member signed in with it (or proved a vouched wallet) since. Members, their project membership and imported referrals stay. The import is `reverting`, then `reverted`, with `revert` set (`reason`, `requestedAt`, `requestedBy`, `rows` reverted so far, `revertedAt` and `outcome` once done); each applied row keeps status `applied` and reads `reverted: true` once reverted. `outcome` has `reversed` counts, `unreversedPoints` (`member`, `ledger`, `amount`), `keptLinks` (`member`, `kind`, `account`, `reason`) and `linksUntracked`: true when rows were applied before imports recorded their links, whose links all stay. The same `actionId` answers the import again, and rerunning the CLI command while it reverts waits for it; any other revert of a reverting or reverted import, or a revert of another status, is refused with `import-status`. CLI sessions and API keys may revert a live import.

A new import for a member already imported credits their opening balance again: check `members.existing` in the summary first.

## Refusal codes

| Code | Meaning |
| --- | --- |
| `account-linked-elsewhere` | Another member holds the identity or an account. Never merged. |
| `duplicate-account` | An earlier row of the import names the same identity or account. |
| `identity-unlinked` | The identity or wallet was unlinked from its member. |
| `member-banned` | The row's identity names a member who is banned (`members.ban`): an import adds nothing to a banned member. |
| `x-account-selected` | Two X accounts in the row, or the member already has another X account. |
| `telegram-account-selected` | Two Telegram accounts in the row, or the member already has another Telegram account. |
| `discord-account-selected` | Two Discord accounts in the row, or the member already has another Discord account. |
| `limit-reached` | More than 5 wallets of one chain family besides the sign-in wallet. |
| `unknown-ledger` | No active ledger with that key or ID. |
| `unknown-quest`, `quest-not-published` | No quest with that key or ID, or no active release in this environment. |
| `quest-not-importable` | The quest completes per content item. |
| `key-required` | A keyed quest's action has no `key`. |
| `key-used` | The quest's keys are unique across members, and another member's completion, or an earlier row of the import, already uses an action's key. A row naming its own member's key is not refused. |
| `future-action` | An action, `acceptedAt`, a qualification's `at` or a consent's `at` is in the future. |
| `sign-in-disabled` | The row's sign-in wallet or X account is linked to its member without sign-in. |
| `duplicate-entry` | The row names one point ledger, quest or referral program twice, by any of its names, or one consent channel twice. |
| `unsupported-consent-channel` | A consent names a channel other than `email`. |
| `consent-wording-required` | A `granted` consent has no `wording` (or only whitespace). |
| `consent-wording-too-long` | A consent's `wording` is over 2,000 characters. |
| `consent-wording-invalid` | A consent's `wording` contains a NUL or a lone surrogate. |
| `unknown-inviter` | The `inviter` ref names no row of the import, or no project member signs in with the inviter identity. |
| `self-referral` | The row names itself, or its own member, as the inviter. |
| `inviter-already-accepted` | The member already has an inviter; an import never replaces it. |
| `referral-cycle` | The inviter was invited, directly or through others, by the row's member (across the import's rows and existing referrals). |
| `inviter-refused` | The row its `inviter` names is refused, so the inviter is never imported. |
| `unknown-referral-program` | No referral program with that key or ID, or it is archived. |

Import commands also fail with `import-status` (409: the import is not in a state that allows the command, such as applying an import that is not `checked`) and `import-empty` (409: nothing to check or apply).
