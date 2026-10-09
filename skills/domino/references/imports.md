# Import members from another platform

A member import brings members, their opening point balances and their quest progress into one project and environment. It runs in two steps: a check (the dry run), which changes nothing, then apply. Every import command needs `publish` permission; `domino imports list|get|rows` need `read`. The human guide is [Import members from another platform](https://console.domino.run/docs/use/imports).

## Write the file

JSONL, one member per line. Map the old platform's quest IDs to the project's quest keys before writing it.

```json
{"ref": "legacy-1", "identity": {"kind": "app:legacy", "subject": "u1"}, "accounts": [{"kind": "x", "subject": "1001"}], "points": [{"ledger": "xp", "balance": 1200}], "quests": [{"quest": "daily-check-in", "actions": [{"at": "2026-09-01T10:00:00Z"}]}]}
```

| Field | Rules |
| --- | --- |
| `ref` | The member's ID on the old platform, 1–200 characters, unique in the import. A row with the same `ref` replaces the import's earlier row. |
| `identity` | What the member will sign in with: `{"kind": "app:<name>", "subject"}` (exchanged by the app login integration of that kind), `{"kind": "wallet", "chain": "solana" \| "eip155" \| "cosmos", "address", "network"?}` (`network` is required for Cosmos), or `{"kind": "x", "subject": "<numeric X user ID>"}`. |
| `accounts` | Up to 20 X accounts and wallets the organization vouches for, in the same shapes. They never sign in. |
| `points` | Up to 50 `{"ledger": key or ID, "balance": positive integer}`. |
| `quests` | Up to 200 `{"quest": key or ID, "actions": [{"at": epoch ms or ISO date-time, "key"?: string}]}`, 1–3660 actions each. |

## Run it

```sh
# Test: check, print the summary, change nothing.
domino import --file members.jsonl --source legacy-domino --dry-run --json
# The refused rows and why.
domino imports rows IMPORT --status refused --json
# Test: add the fixed file to the same import, check it again and apply the ready rows.
domino import --file members.jsonl --import IMPORT --json
```

`domino import` uploads 500 rows per request, checks the import, and polls it until the check (and in test, the apply) finishes. It prints the import: `status`, `summary` (`members.created`, `members.existing`, `accounts`, `points` by ledger, `actions` by quest, `refusals` by code) and `progress` (rows applied and refused).

In live, `domino import` without `--dry-run` exits 2 with `INVALID_ARGUMENT`. Check it with `--environment live --dry-run`, then ask a person with publish permission to apply it in Console from **Members** > **Imports**. Applying a live import with a CLI session or API key is refused with `console-approval-required`.

Other commands: `domino imports list`, `domino imports get IMPORT`, `domino imports apply IMPORT` (test; waits until applied), `domino imports cancel IMPORT` (open or checked imports). In code, use `management.memberImports.create|addRows|check|get|rows|apply|cancel|list`; `check` and `apply` return at once, so poll `get` until the status is `checked` or `applied`.

## What applying does

- **A row's identity names the member.** If the identity already belongs to a member, the row adds to that member. Otherwise it creates one and joins them to the project. Rows never merge members.
- **Vouched accounts count as linked.** Attribution, `ctx.x` and wallet checks trust them; Console labels them imported. The member's own X link or wallet signature replaces the vouched link. Another member cannot later prove a vouched account.
- **Opening balances are `adjust` entries with `import` set.** They emit no `points.awarded`, count on balance leaderboards only, and appear with `imported: true` in the member's own point history.
- **Progress pays no rewards.** Each action is an accepted action under the quest's completion rule, at its `at` time, on one attempt with code `imported`: `once` completes once, `keyed` once per `key`, a streak counts each action's project-calendar day, and a counter counts distinct keys (an action without `key` counts alone; actions outside a calendar window's lateness are late). Completion limits and prerequisites count imported completions.
- **Apply re-checks every row.** Each row applies once, in its own transaction; a row that became invalid since the check is refused instead.

A new import for a member already imported credits their opening balance again: check `members.existing` in the summary first.

## Refusal codes

| Code | Meaning |
| --- | --- |
| `account-linked-elsewhere` | Another member holds the identity or an account. Never merged. |
| `duplicate-account` | An earlier row of the import names the same identity or account. |
| `identity-unlinked` | The identity or wallet was unlinked from its member. |
| `x-account-selected` | Two X accounts in the row, or the member already has another X account. |
| `limit-reached` | More than 5 wallets of one chain family besides the sign-in wallet. |
| `unknown-ledger` | No active ledger with that key or ID. |
| `unknown-quest`, `quest-not-published` | No quest with that key or ID, or no active release in this environment. |
| `quest-not-importable` | The quest completes per content item. |
| `key-required` | A keyed quest's action has no `key`. |
| `future-action` | An action is dated in the future. |
| `sign-in-disabled` | The row's sign-in wallet or X account is linked to its member without sign-in. |
| `duplicate-entry` | The row names one point ledger or quest twice, by any of its names. |

Import commands also fail with `import-status` (409: the import is not in a state that allows the command, such as applying an import that is not `checked`) and `import-empty` (409: nothing to check or apply).
