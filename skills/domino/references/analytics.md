# Measure a project and build dashboards

Use this reference to answer how a project is doing, to pick the metrics that measure what it is for, and to build, check and explain dashboards. Command flags and result shapes are in [project analytics](automation.md#project-analytics), [metric catalog](automation.md#metric-catalog) and [dashboards](automation.md#dashboards); the `defineDashboard` fields and deploy refusals are in [authoring dashboards](authoring.md#dashboards).

## Answer "how is it going"

1. For the overview, run `domino analytics get --range 7d|30d|90d|all --json`. Pick the range the question implies; `30d` is the default.
2. For a narrower question (one quest, one program, a group of members, rejections by reason), run `domino analytics query --metric ID --json` with the breakdown or filter that asks it. List the catalog with `domino analytics metrics` when you don't know the ID.
3. Quote the read's numbers: the `total`, the `previousTotal` it compares with, and the rows that explain it. Every number in your answer comes from a read. When the question needs a number no read returns, such as a ratio of two metrics, say the catalog doesn't have it and propose it as a new metric; arithmetic on read results stays out of the answer.

An answer is done when each number it gives names the metric and range it came from.

## Choose metrics from the project's goals

Start from what the project is for, as its brief (CAMPAIGN.md or the app's project document) states it, and measure that. Common goals map to these metrics:

| Goal | Metrics |
| --- | --- |
| Growth through referrals | `referrals.qualified` and `referrals.qualificationRate`, each by `program`; `members.signUps` for the whole intake |
| Tracker-driven quests, such as staking or holding | `quests.completions` with a `tracker` filter, and `quests.attempts` with the same filter to see who starts |
| A points economy | `points.awarded`, `points.spent` and `points.net` for each ledger, and `points.membersEarning` for reach |
| Content and review load | `quests.needsReview` by `quest`, and `quests.rejections` by `reason` |
| Rewards that get claimed | `offers.redemptions` by `offer` |
| Engagement | `members.active` and `quests.completions` by `quest` |

The catalog is the source of truth: read `domino analytics metrics` for the live list, each metric's breakdowns, filters and charts, and whether it needs a ledger. Points metrics read one ledger each, so a project with two ledgers needs a panel per ledger.

## Build a dashboard

Build one when the operator wants a recurring view of a goal, or after you build a feature whose success they will want to watch, such as a referral program or a tracker quest. Offer it then, in the same handoff.

- **One dashboard per goal.** Give it the goal's name and a description that states the goal.
- **A few panels that each answer a question.** Three to six is typical. A `number` panel answers "how many"; a `line` answers "is it growing"; a `bar` or `table` with a breakdown answers "which one".
- **Breakdowns over pinned lists.** Use `breakdown: "quest"` or `breakdown: "program"` instead of one panel or filter per quest, so new quests and programs appear without editing the dashboard. Filter by key only when the goal is about those resources.
- **Rates as `number` or `line`.** A rate can't be drawn as `bar`, and it is empty below its `minimumDenominator`.

Write the definition with `defineDashboard`, list it under `dashboards` in relay.json, and follow [authoring dashboards](authoring.md#dashboards) for the fields.

### Check before you hand over

1. Run `domino deploy` in test. A refusal has a code and names the panel it is about; fix the definition and deploy again.
2. Run `domino dashboards get <key>` and read every panel.
3. Fix each panel `error`. `group-not-found` means the filter names a group this environment doesn't have: create the group or drop the filter. `panel-timeout` on a long range is worth naming to the operator.

The dashboard is ready when `domino dashboards get <key>` shows a result for every panel. Only then tell the operator it exists, with its key.

## Explain a dashboard

Describe each panel with its metric's definition from `domino analytics metrics`, in plain words, and say what its breakdown splits by. Name these where they apply:

- **A `group` filter** counts members who are in the group now, whatever their membership was when they acted. A member who leaves the group drops out of past buckets too.
- **A rate** is empty below its minimum denominator (10), so a quiet range shows no value rather than 0%.
- **Records count as they stand now:** a completion revoked or a redemption canceled later leaves the bucket it was in.
- **Live gets the dashboard by promotion.** It exists in test after a deploy; it reaches live when the operator promotes, with its archive state.

## Limits

- **No formulas.** A panel shows one catalog metric. If the catalog can't express the number the operator wants, say so and propose the metric (ADR 0021 keeps the catalog closed).
- **Group keys are local to an environment.** Promotion doesn't copy groups, so a panel filtered on a group that live doesn't have reads `group-not-found` there until the operator creates it.
- **Caps.** A dashboard holds at most 24 panels, and a project at most 20 dashboards that aren't archived.
- **Retire by archiving.** Removing a dashboard from relay.json doesn't delete it. Run `domino dashboards archive <key> --reason "…"` in test, and `domino dashboards unarchive <key> --reason "…"` to restore it.
- **Console doesn't show dashboards yet.** Operators read them with `domino dashboards list` and `domino dashboards get`.
