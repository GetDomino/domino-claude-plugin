# Build a referral program

Run `domino capabilities referrals --json` against the installed release. Native referrals own invitation codes, project-scoped attribution, qualification, fixed rewards, milestones, and one-level inviter bonuses. The participant app owns invitation presentation and retaining the code through sign-in. Use the existing project ledger and publication flow.

## Author and publish

```text
import { defineReferral, points, questCompleted } from "@domino-sdk/relay/authoring"
import { community } from "../../ledgers"
import welcome from "../../quests/welcome.quest"

export default defineReferral({
  key: "friends",
  title: "Invite friends",
  qualification: questCompleted(welcome),
  rewards: { inviter: [points(community, 25)] },
  milestones: [{ key: "five", count: 5, rewards: [points(community, 250)] }],
  bonuses: [
    {
      key: "share",
      ledger: community,
      basisPoints: 1000,
      quests: [welcome],
    },
  ],
})
```

`defineReferral` comes only from `@domino-sdk/relay/authoring`; the referral schemas are in `@domino-sdk/relay/schema`. Register `{ "entry": "src/referrals.ts" }` in `relay.json` under `referrals`; `exportName` selects a named export. `domino check` bundles definitions without executing author code locally. Preview and publish with the existing project deployment commands. Publishing code-defined behavior to live through Console is unavailable. Omitted programs remain published; `enabled: false` stops future qualification and bonuses. Keep program and rule keys stable.

Choose qualification explicitly: `signup` qualifies when a signed-in participant accepts an invitation; `questCompleted(quest)` requires the imported quest handle; `external` requires the trusted management qualification command. Each program counts distinct qualified friends. Existing points and tier rewards are accepted for inviter, invitee, and milestones. Each referral tier reward uses its own retained staff-handover entitlement. Different friends can earn separate items; requalification reuses the original entitlement. Discord role rewards are refused with `Referral programs can grant points and tiered rewards; Discord roles need a quest`; grant the role from the qualifying quest instead. If a program published by an older Relay release contains a Discord role, reads and unrelated deployments still work. Relay skips that reward when a friend qualifies and records `referral-reward-skipped` in Audit history. Archive the old program and publish a supported replacement.

`basisPoints: 1000` means an additional 10%; the invitee retains their full award. Omitted quest filters include all quests in that balance. A quest's earnings include daily engagement-rule payouts for that quest. Leaderboard prize points are not quest earnings and earn no bonus. Optional `inviterGroup` restricts eligibility to inviters in that [group](https://console.domino.run/docs/concepts/groups), which operators create and fill in Console **Groups** or with `management.groups`. Optional `durationDays` and `maxPoints` limit each invited friend's bonus. Fractions accumulate per friend, rule, and balance. Percentage rules must not overlap for the same source; group names cannot prove disjointness. Fixed rewards and milestones may coexist with a bonus.

## Integrate the participant flow

```ts
import { createBrowserRelayClient } from "@domino-sdk/relay/browser"

const relay = createBrowserRelayClient({ baseUrl: "/relay" })
const summary = await relay.referrals.summary()
const invitation = new URL("https://app.example.com/join")
invitation.searchParams.set("ref", summary.code)

// After the invited friend signs in, using the code retained by the app:
const code = "ref_code_from_invitation"
await relay.referrals.accept({ code, actionId: "accept-invitation" })
const activity = await relay.referrals.history()
```

Use the configured participant proxy or identity bridge. Domino derives the invited member from their authenticated session. Accept before their first completed quest. The first accepted inviter is permanent; self-referrals, cycles, and cross-project codes fail. Link visits alone do not establish attribution. Retry uncertain requests with the same action ID and payload. Do not discard the retained invitation until acceptance succeeds or the user dismisses a terminal error.

Render attributed and per-program qualified counts, milestone progress, and earned/settled/outstanding point accounts. Reward and account `balance` values and `program` values are IDs: find your program by `definition.key` and your ledgers through `relay.pointLedgers.list()`, then compare IDs (see [resource references](participant-api.md#resource-references)). Settled points can include amounts offset against correction debt. Fetch older history with `referrals.history({ cursor: activity.nextCursor })` while `nextCursor` is non-null. These reads reveal no invitee identities or detailed activity.

## External activity and moderation

On a trusted backend, construct `createManagementClient` from `@domino-sdk/relay/server` with the project/environment scope and server key. Use `management.referrals.qualify(program, member, { qualified: true, reason })` after verifying an external condition, and the same call with `qualified: false` for invalidation. `management.referrals.pause(member, { reason })` suspends a member's referral; `resume(member, { reason })` restores it, and `management.referrals.resolveReward(entitlement, { reason })` marks a reward held for review reconciled. Each takes an optional `actionId`, generated when omitted; reuse it to retry. These commands require `publish`; reads require `read`. `management.referrals.participants({ cursor? })` reads the project's referral relationships, 100 at a time and newest first, with the members' qualifications and accounts and the rewards waiting for review; pass a page's `nextCursor` back as `cursor` for the next. `management.referrals.history(member, { cursor? })` pages one member's referral point history the same way. The commands work in live with a personal API key that has `publish`, and Console **Audit history** records each one under the key's owner. A CLI sign-in changes nothing in live.

External `management.points.record` awards generate a bonus only with both `referralEligible: true` and a matching bonus rule with `external: true`. They require `points` permission. Preserve the source event's action ID. Source points and referral bonuses are atomic. Adjustments, absolute totals, spending, and referral payouts do not generate bonuses.

Use `pause(member, { actionId, reason })` or `resume(member, { actionId, reason })` to stop or resume future payouts while preserving attribution. Console Referrals exposes these controls, qualifications, payment history, and outstanding corrections. No missed payouts accumulate during suspension or ineligibility.

Reversals use the original rate. A fully reversed qualifying quest award revokes qualification; non-point invalidation requires staff action. Revoking the qualifying completion (`management.completions.revoke`) moves a quest qualification to another accepted completion of the quest, or unqualifies the participant and reconciles the inviter's rewards. Correct available points first and offset outstanding corrections against future referral payouts in that balance. Ordinary future earnings are not seized. If an operator reverses a positive referral payout through `management.points.record`, Relay lowers the referral account's settled amount too, so later unqualification does not recover the payout twice. An exact retry returns the first result; another reversal may only take the original payout's remainder. Restore the original fixed/milestone entitlement when eligibility returns, cancelling correction debt before restoring recovered points. Do not issue duplicate rewards. Invalidated tier entitlements require staff review before handover; `resolve-reward` clears that review after the original reward has been reconciled.

New rates and eligibility apply when an award is accepted, even if its effective date is backdated. Qualifying quest points count; earlier earnings do not. Requalification does not replay past percentage earnings. Fixed rewards already issued retain their payload and milestone threshold.

## Prove the flow

Use test members: `domino test-members accept-referral INVITEE --inviter INVITER` accepts as a signed-in member would (see [campaign testing](campaign-testing.md#referrals)). Share and accept a code, complete the qualifying quest, and verify both balances and qualified counts after reload. Repeat the acceptance and source event to prove deduplication. Check self-referral rejection, a second inviter, an ineligible source, suspension, and a reversal after the inviter spends their bonus. Verify desktop and mobile UI states, including unqualified and outstanding-correction states. Report the project, environment, program, observed balances, and any unverified external condition. Simulated quest providers prove the flow, not the external activity.

Badge achievements are quest rewards. Referral reward bundles accept points and tier entitlements; attach `badge(...)` to the qualifying quest when the same activity should mark an achievement. See [badges](https://console.domino.run/docs/reference/authoring#badges).
