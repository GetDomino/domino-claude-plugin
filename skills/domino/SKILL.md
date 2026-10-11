---
name: domino
description: Set up a Domino project, integrate Domino into an existing app, find and adapt quest snippets, author quests, leaderboards, or referrals, supply external points, or verify and stage a participant experience with the Domino CLI and SDK.
---

# Build with Domino

Deliver the requested participant experience and evidence that its main action works. Domino owns verification, completion, points, and reward execution. The app owns its presentation and integration with participant identity.

## Unattended runs

Check the run mode once, at the start: read `agentMode` from `domino doctor --json`, which prints it even when doctor exits 1, or read `DOMINO_AGENT_MODE` from the environment before the CLI is installed. In `interactive` mode, ignore this section. `unattended` means whoever launched you set `DOMINO_AGENT_MODE=unattended` and nobody is present to answer; never set it yourself. In unattended mode, these rules replace every instruction below to ask, wait, or end the turn:

- **The brief is the confirmed brief.** Skip brand and campaign intake, questions, and confirmation, and never end a turn to wait for an answer.
- **Fill gaps, don't ask.** Where the brief leaves a consequential gap, such as rewards, timing, eligibility, or brand, choose the most conservative reasonable option. Record each choice in CAMPAIGN.md under "Assumed in an unattended run" and carry on.
- **No browser login.** Remote work uses `RELAY_MANAGEMENT_TOKEN`. On `AUTH_REQUIRED` or `UNATTENDED_LOGIN_UNAVAILABLE`, finish the local work and report the missing credential; don't run `domino login`.
- **No live publication.** Stop at hosted staging; live needs human review in Console.
- **Feedback stays local.** `domino feedback` saves each report to the `file` in its result instead of submitting it. Report as described below and name that file in the handoff.
- **The handoff is a report.** End with what you built, the staging URL, the checks and participant actions you ran, and every assumption. Ask no questions.

## Report tooling issues

Report every distinct Domino CLI, SDK, documentation, or setup issue you encounter, even if you find a workaround. Use `domino feedback --reporter agent --title "Short summary" --body-file report.md --json` and follow the [reporting guidance](references/troubleshooting.md#report-a-tooling-issue). Include reproduction steps, expected and actual behavior, impact, and the workaround. Remove credentials and private participant data before submitting. Keep the returned issue link in your final handoff. If submission fails, retain the report and mention the delivery failure; do not repeatedly retry or let feedback block independent work.

Before the final handoff for every completed task, review what was difficult and submit any unreported difficulty that better Domino SDK or CLI support could address, even when all commands succeeded. Look for missing helpers, repetitive manual steps, confusing APIs, poor discoverability, or workarounds. Describe the concrete task, the extra work it required, and what SDK or CLI improvement would have helped. Keep this feedback focused on observed tooling difficulties; skip general task summaries, speculative feature requests, duplicate reports, and empty success reports. If there were no such difficulties, no completion report is needed.

## Discover and confirm the campaign

Read existing project instructions and saved decisions first. If the owner already confirmed the relevant campaign, reuse that evidence; reopen only decisions affected by a material change. Choose the procedure for the app being built.

### Existing apps: confirm the campaign

Preserve the app's current design, components, and UI conventions. Skip brand discovery, brand-name or website questions, visual samples, and brand approval unless the owner explicitly requests a redesign.

Ask only for missing campaign basics: goal, intended participants, desired actions, and reason to participate. If these are missing, ask a few short questions, end the turn, and wait for answers before setup or implementation. Use answers already supplied. Inspect the app to identify where the participant flow belongs, then propose the campaign and smallest integration that fits its existing UI. Resolve consequential rewards, timing, and eligibility gaps with the owner. Wait for explicit campaign confirmation before implementing, and save the confirmed brief and remaining open decisions in the app's established project document. Reuse prior confirmation when it covers the work. Continue with [existing apps](references/existing-app.md) for integration steps.

### New apps: discover the brand and campaign

Before implementing a new app, establish the brand direction and campaign brief with the owner. A starter, initial prompt, or Console brief is input to discovery, not confirmation. The procedure below applies to new apps only.

1. **Ask before researching or designing.** Check the supplied brief and prior owner answers for a brand name, official website or reference, campaign goal, intended participants, desired actions, and reason to participate. Ask for missing basics in a few short questions, then end the turn and wait for the owner's answers. For an empty or vague brief, ask both which brand this is for and what the campaign should achieve. The owner may confirm there is no website or established brand. A project name, repository name, starter, or sample quest is not evidence of brand identity or campaign intent. Create a new brand only when the owner explicitly requests it. If the basics are already supplied, summarize them for confirmation instead of asking the same questions again. Complete this intake before login, setup commands, research, or a visual proposal.
2. **Research and prepare the proposal.** After intake, inspect the owner's website, project documents, and supplied assets, including rendered appearance and styles. Record source URLs or file paths. If a source is inaccessible or identity remains ambiguous, ask the owner for a reference and wait rather than guessing. Show a concise brand interpretation and a small visual sample using the proposed colors, type, and voice. Draft the campaign goal, success measure, intended participants, desired actions, participant motivation, and journey from the owner's answers. Include timing, rewards, eligibility, and constraints where relevant. Suggest improvements tied to the goal and check Domino capabilities before promising supported behavior.
3. **Resolve consequential gaps.** Separate observed facts, recommendations, and assumptions. Ask only questions whose answers materially change the experience, a few at a time, with a recommended answer where possible. Use research and existing answers instead of giving the owner a questionnaire. Keep unconfirmed rewards, budgets, inventory, dates, and eligibility explicit as open decisions; do not turn guesses into participant promises.
4. **Confirm and save.** Present the proposed brand direction and campaign brief together for the owner to confirm or correct. End the turn and wait for their explicit response before implementation; silence and your own confidence are not confirmation. After intake, you may inspect and connect the project, research, write the draft brief, and prepare the visual sample. Keep app implementation and campaign configuration until after confirmation. Save the agreed direction, evidence of the owner's confirmation, sources, and remaining open decisions in CAMPAIGN.md, or the existing app's established brief document. Preserve the original request and link to the brief if it lives elsewhere.

Discovery is complete when the owner has confirmed the brand direction and campaign intent, including the goal, participants, actions, and motivation. Optional details can remain open if they do not affect the first implementation; record what must wait for those decisions. On subsequent sessions, read the saved brief and continue without repeating onboarding.

## Establish the project

Read the project's agent instructions and owner brief, such as CAMPAIGN.md, when present. Inspect package scripts and relay.json before selecting commands. Use the installed CLI through the project's package manager, such as `pnpm exec domino`; the examples below abbreviate this to `domino`.

Run `domino doctor --json` to inspect setup without running application code or contacting the service. Missing setup exits 1 with the diagnostic JSON on stdout and a `DOCTOR_UNHEALTHY` error on stderr. Invalid configuration exits 1 with only the error, such as `MANIFEST_INVALID`. Fix the reported prerequisite, then rerun the check.

Run CLI commands with `--json` in scripts and agents. Branch on the exit status (0 success, 1 failure, 2 usage error, 3 uncertain, 130 cancelled) and the error's `code`, never on its message. Exit 3 means a change may have taken effect: resume it with the error's `operation.resume` instead of rerunning the command. The [automation contract](references/automation.md) lists every code and result schema. An API refusal also carries the API's own code in `apiCode`, and SDK errors carry it in `code`; [API error codes](references/errors.md) lists them.

Use `domino login` for browser authorization when remote work is needed. As soon as the CLI prints the login details, send the user the code first, then the clickable sign-in link, in the same message. Always preserve this order, even if the CLI prints the link first. Use the exact code and URL from the current login attempt:

> Code: `<code from CLI>`
>
> [Open Domino to sign in](<URL from CLI>) and enter this code to approve the CLI connection.

Keep the login process running while the user approves in their browser. The CLI polls for approval and saves the credential itself; never ask the user to paste credentials into chat. Wait for the CLI to confirm success before continuing. If the attempt expires, run `domino login` again and send the new code first, then the new link.

Run `domino doctor --remote --json` after dependency installation finishes to verify access before remote operations. Inspect `scope` and `scopeSources`: flags override `RELAY_*` environment variables, then the Git common directory's `domino.json`, then `relay.json`. Checkout writes the untracked local connection file; starter values such as `local` and `campaign` in the manifest may therefore be valid defaults. Credentials live separately in the user config directory. For scope or setup failures, read [troubleshooting](references/troubleshooting.md).

Choose the reference that matches the task:

- Before starting or resuming work in a Domino-hosted repository, or sharing its staging preview, follow the [source and release workflow](references/hosted.md#source-and-release-workflow). Read [hosted projects](references/hosted.md) for checkout and app setup.
- For integration into an existing codebase, read [existing apps](references/existing-app.md).
- Before authoring a quest, read [authoring](references/authoring.md#find-a-snippet) and check the live snippet catalog for a suitable starting point. The same reference covers reusable types, collections, leaderboards, external points, and publication.
- To reward a wallet's on-chain action (an EVM contract event, or a Solana transaction that called a program), read [on-chain actions](references/onchain.md).
- To reward posting or tweeting on X, read [rewarding X posts](references/authoring.md#rewarding-x-posts) first. X does not allow apps that reward users for posting, so Domino pays only creators an operator approved.
- For standings, own rank, nearby participants, or pagination, read [participant API](references/participant-api.md#leaderboard-reads).
- Before calling a campaign ready, or to reproduce a reported campaign bug, simulate it with test members over several days: read [campaign testing](references/campaign-testing.md).
- To bring members, point balances or quest progress from another platform, read [member imports](references/imports.md).
- To read or change project data as an operator (points, trackers, leaderboards and settlements, referrals, offers and redemptions, profiles, the calendar, members, groups, quests and release history), use the typed `domino <noun> list|get|…` commands in [resource commands](references/automation.md#resource-commands) instead of `domino api`. Their changes are saved operations that resume after an uncertain answer.
- To answer how a project is doing (sign-ups, active members, attempts, completions, rejections, points, redemptions, referrals over time), run `domino analytics get --range 7d|30d|90d|all` and quote its numbers instead of computing your own; the definitions are in [project analytics](references/automation.md#project-analytics).
- To split or narrow one of those numbers (by quest, offer, program, rejection source or reason; for a group, tracker or quests), list the catalog with `domino analytics metrics` and read one metric with `domino analytics query --metric ID`; see [metric catalog](references/automation.md#metric-catalog). Don't combine metrics into a formula.
- To choose metrics that measure a project's goals, or to build, check or explain a dashboard, read [analytics](references/analytics.md).
- When an owner wants to check their project or try quests from Claude or ChatGPT, point them to Domino's MCP server: see [Claude and ChatGPT](references/automation.md#claude-and-chatgpt).

Before designing referrals, rankings, milestones, or rewards for another participant, run `domino capabilities --json`. Read `domino capabilities <id> --json` for the support boundary; its `cli` lists the `domino` commands that reach the capability. The output's `provenance` identifies a bundled CLI catalog, not a project-specific server probe. Treat an unlisted capability or a disagreement with the deployed API as unknown and inspect its contract before promising it. Check recurrence, daily limits, social verification, and prize fulfillment separately; points or leaderboards alone do not establish those mechanics.

For browser integration and submission recovery, read [participant API](references/participant-api.md). For invitation links, qualification, inviter bonuses, or referral milestones, read [referrals](references/referrals.md). For startup, staging, or version failures, read [troubleshooting](references/troubleshooting.md). When SDK, docs, and server behavior disagree, follow its upgrade guidance and check the public changelog before adding a workaround.

If no CLI is installed, install `@domino-sdk/relay-cli` and `@domino-sdk/relay` through the app's package manager. Starters install matching published npm versions of the SDK and CLI. Keep the lockfile and `.domino/starter.json` when diagnosing compatibility. For an older project with `.domino/*.tgz` dependencies or overrides, follow the registry migration in [troubleshooting](references/troubleshooting.md#check-for-an-sdk-upgrade); do not replace the archives.

## Verify the result

Run the app's checks and exercise the requested participant action through the running app. Verify the resulting progress or completion state. Keep fixture-provider results distinct from real external verification.

Complete the [tooling feedback review](#report-tooling-issues) before the final handoff, including after a successful task.

Return the working preview or local URL, what participant action you verified, and any remaining launch dependency. Report failed or blocked browser checks explicitly, including a missing browser executable. A successful build, installed skill, or healthy doctor result is not proof of participant behavior. Live publication requires human review in Console; hosted staging uses test data and real sign-in.
