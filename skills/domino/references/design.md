# Design the participant app

This is the quality bar for any participant UI you build or restyle: a new app after the owner confirms the brand direction, a redesign the owner asked for, and the brand you choose in an unattended run. Each rule below can be checked in a screenshot, the CSS or the markup. Work through the sections in order and finish with the [self-review](#self-review-before-staging).

## Existing apps

Preserve the app's design. The rules here apply only to the Domino parts you add (quest cards, progress, sign-in prompts, rewards), and those parts use the app's own tokens, components and type. Add no second palette or font. The [states](#every-state-designed), [accessibility](#accessibility) and [self-review](#self-review-before-staging) sections still apply to what you add.

## Art direction

Name the direction in one line before writing styles: who the participants are, where they arrive from, and the mood. A conference booth crowd scanning a QR code between talks wants something loud, bold and fast to read. A DeFi points programme wants something precise, calm and dense with numbers. A creator's fan club wants something warm and personal. Then apply it everywhere: type, colour, layout, imagery, copy and the [celebration](#celebrate-completion) all follow the one direction.

The starters' default look is a base to replace. A finished app has its own type, an accent colour, and a layout that suits the campaign. The starter's default token values (the campaign starter's indigo accent, the loyalty starter's greyscale), the starter's sample copy, or "Domino" as the visible brand mean the design is unfinished.

In an unattended run (`DOMINO_AGENT_MODE=unattended`) building a new app with no brand in the brief, the direction is your call within the brief. Record the direction under "Assumed in an unattended run" in CAMPAIGN.md with one line of reasoning, for example: "Event booth at Devcon: high-contrast black and electric lime, condensed display type, because people read it at arm's length between talks." An existing app's brand is its current design: keep it, as [existing apps](#existing-apps) says.

## Brand as a system

Set the brand as tokens in one stylesheet, such as `src/styles.css` in the starters, and have components use only the token names. Tailwind CSS v4 exposes custom properties through `@theme`. Check: components use only role tokens, with no raw hex values, no palette classes such as `text-red-700` or `bg-zinc-100`, and no one-off sizes such as `text-[13px]`.

- **Type scale.** One display family and one text family at most, often the same one. Five or six sizes from one ratio (for example 14, 16, 20, 25, 31 and 39 px). Body text is at least 16 px on a phone, with line height around 1.5 for body and 1.1 to 1.25 for headings.
- **Colour roles.** Name colours by role, not by hue: background, surface, foreground, muted foreground, border, accent, the text on accent, and success, warning and danger. Give each role a light and a dark value. Every text and background pair you use meets [contrast](#accessibility) in both themes.
- **Spacing.** A scale on a 4 px base (4, 8, 12, 16, 24, 32, 48, 64). Related items sit closer together than unrelated ones.
- **Radius and elevation.** Two or three radii and two or three shadow levels, used the same way everywhere: cards share one radius, and dialogs sit above cards.

## Mobile first

Members arrive from a QR code, an X link or a Discord message, so they are on a phone. Design at 390 px wide first, then widen.

- One column at phone width, with no horizontal scroll at 320 px.
- The first screen at 390 × 844 says what the campaign is and shows the main action without scrolling.
- At desktop width, content keeps a readable maximum width (about 1,100 px for the page, 70 characters for paragraphs) instead of stretching edge to edge.

## Every state designed

Each quest card and page has a designed look for every state a participant can see, with copy written for it. Read quest states from `questStatus(view)` (see [React hooks](participant-api.md#react-hooks)) and give each `code` its own treatment.

| State | `questStatus` code | What the participant sees |
| --- | --- | --- |
| Loading | | A skeleton in the shape of the content it replaces, so the page doesn't jump when data arrives. |
| Signed out | | What the campaign offers, and one clear way to sign in. |
| Empty | | Why there is nothing yet and what to do next, such as "No quests are open yet. The first opens on 12 November." |
| Error | | What failed, in plain words, and a retry. |
| Ready | `ready`, `retryable`, `automatic`, `counter` | The input or action, with what the quest pays. An `automatic` quest has no input: say what counts. |
| Checking | `checking`, `in-progress`, `evaluator-deferred`, `evaluator-progress` | Progress on the action that was sent, with the input disabled. |
| In review | `in-review`, `staff-required` | That a person will check it, and roughly when. |
| Rejected | `rejected`, `counter-revoked` | The evaluator's `reason` and how to try again. |
| Cooldown | `retry-cooldown`, `completion-cooldown` | When it opens again, from `retryAt`. |
| Locked | `locked`, `connection-required`, `wallet-required`, `enrollment-required`, `outside-window`, `budget-exhausted`, `counter-budget-exhausted`, `streak-budget-exhausted`, `paused`, `closed` | What is missing, from `view.availability.missing`, or why the quest can't be done now. |
| Completed | `completed`, `streak` | The reward earned, and the next thing to do. |

`questStatus` codes are an open string: render a code you don't know with a neutral fallback.

## Celebrate completion

When a quest completes, mark the moment: the points or reward earned, a short animation or visual change in the campaign's style, and the next quest or the leaderboard. Keep it under two seconds, and dismissible or non-blocking.

Use motion only where it explains a change (a card moving from ready to completed, a balance counting up) or celebrates one. Interface transitions take 150 to 300 ms. Under `@media (prefers-reduced-motion: reduce)`, replace movement with an instant change or a plain fade.

## Accessibility

- **Contrast.** WCAG AA in both themes: 4.5:1 for body text, 3:1 for text of 24 px and up (or 19 px bold) and for borders and icons that carry meaning.
- **Tap targets.** Every button, link and input is at least 44 × 44 px, including icon buttons and the space around small text links.
- **Focus.** A visible `:focus-visible` outline on every interactive element, at 3:1 contrast against its background.
- **State without colour.** Each state has a label or icon as well as a colour.
- **Semantics.** Buttons are `<button>`, links are `<a>`, inputs have labels, and images that carry meaning have `alt` text.

## Self-review before staging

Before you stage, screenshot the running app and fix what looks unfinished. Use the screenshot tool you already have, such as a browser tool or Playwright.

1. Capture each page at 390 px and at a desktop width (1,280 to 1,440 px), in light and in dark (emulate `prefers-color-scheme`). If the app has only one theme by design, capture that one and say so.
2. Reach every state in the [table](#every-state-designed) that the app can show, and capture each. Use `createFakeRelay` ([test the UI](participant-api.md#test-the-ui)) for loading, empty, error and the quest states, or test members on a running `domino dev` preview ([before staging](campaign-testing.md#check-participant-flows-on-staging)). Hide the staging toolbar first (**Hide toolbar** in its `⋯` menu) so the captures show the live layout.
3. Check every capture against this reference: the art direction is visible, text is legible, nothing overflows or overlaps, spacing follows the scale, and no starter sample copy remains. Fix what fails and capture again.
4. Keep the final captures and list their paths in the handoff.

Done when every rule above holds in the final captures, the CSS or the markup. If you have no way to take screenshots, say so in the handoff and report the self-review as not done.
