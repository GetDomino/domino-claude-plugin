# Existing apps

Before the first implementation, follow the Domino skill's existing-app campaign confirmation procedure. Ask for missing campaign basics, prepare the campaign and integration proposal, and obtain the owner's confirmation. Preserve the app's current design and components; brand discovery and visual approval are not prerequisites. Reuse an already confirmed brief in later sessions.

Inspect the app's framework, package manager, workspace boundaries, authentication, and deployment scripts. Identify where the requested participant experience belongs. Keep its repository, UI conventions, login, and hosting.

Use the Domino project selected by the user. If no remote project exists, create it in Console. `domino create` provisions a hosted repository and is not the existing-app integration path.

Use the project ID returned by Console. Domino assigns it when the project is created; the project name is a separate display label. Do not invent a project ID or derive it from the name.

From the intended package or project root, run `domino init --project <id>` to create relay.json and RELAY.md. Init verifies access and does not install packages or generate sample quests. If these files already exist, inspect the existing connection and continue from it. Init deliberately refuses to overwrite them.

Run `domino agents install` to register package-backed skill loaders for both Codex and Claude, or select one with `--agent codex` or `--agent claude`. It adds a marked guidance section to AGENTS.md and, for Claude, a pointer in CLAUDE.md while preserving existing project instructions. The loaders read `node_modules/@domino-sdk/relay-cli/cli/skills/domino/SKILL.md`; install the project-local CLI and dependencies first. Shared guidance follows the installed CLI version, so upgrades only require the agent to re-read the bundled skill.

For a project with copied skills, run the upgraded CLI's `domino agents install` once. Known unmodified copies become loaders, and existing reference links receive redirects to the package. If the installer names a customized skill or reference, move its project-specific rules into AGENTS.md, then replace that reported file with the corresponding file from the project-local CLI's `cli/skills/domino` directory and rerun installation. Preserve the custom rules before replacing anything. If a marked Domino section in AGENTS.md or CLAUDE.md was edited, move custom rules outside that section, remove only the marked section, and rerun installation. The installer checks all selected destinations before writing and supports retries. It does not fetch guidance from the network or change dependencies.

Add the SDK to the package that owns the authored modules using the app's package manager and an available Domino package distribution. Resolve SDK imports there before adding quest entries to relay.json. Quest files can follow the existing source layout.

For participant UI, inspect the installed `@domino-sdk/relay` and `@domino-sdk/relay/browser` exports. Use the supplied quest controllers for submissions, recovery, and changed rules. Keep authored quest keys and collection slots stable. Participants explicitly review changed rules before resubmitting.

Keep the application's existing login using `createParticipantBackend` on its server. Read [participant authentication and routing](participant-api.md) for the complete route example. Implement `authenticate(request)` using the app's existing server session verifier and return its stable user ID. Never accept a user ID from browser input as proof of identity.

An organization administrator creates an identity integration in Console under Settings → App login keys, for the selected project and environment. The user saves the shown key directly in the application's server secret settings as DOMINO_IDENTITY_KEY. The key is shown once; keep it out of agent chat, source control and browser bundles. CLI and personal management credentials cannot create these keys or substitute for them. Readiness requires the deployed API and SDK versions supporting `identity-exchange`; a legacy 404 is an upgrade issue.

Mount the adapter at `/relay/*` and use `createBrowserRelayClient({ baseUrl: "/relay" })`. Verify a signed-in session, one participant action, and that logging out of the application removes access. The adapter checks application authentication on every request and derives project scope from the integration key. No custom fetch wrapper or browser project headers are needed.

Read [authoring](authoring.md) when adding quests or collection slots. Run the app's normal checks and development server. Use its existing preview deployment process when available. `domino deploy` publishes Domino behavior; it does not deploy the app. Register participant offers in relay.json's `offers` list; build, check and deployment summaries include the number of offers in the authored bundle. Register ongoing content rewards under `engagement` in relay.json. Their tracker, qualifying quest, and points balance can publish in the same transaction. Preview reports new, unchanged, and changed formulas; resume the saved deployment ID to publish or recover its receipt. Publishing resends a request whose answer was lost, at most three times in all within 30 seconds, so resume only after `domino deploy` exits 3 with `OUTCOME_UNCERTAIN`. `domino stage` requires a supported Domino-hosted static repository, so it is not a general deployment command for existing apps.

When upgrading an existing integration, run `domino migrate` to inspect authoring changes, apply them with `domino migrate --write`, and run `domino check --json`. Keep each existing authoring identity value while renaming `id` to `key`; do not recreate projects or replace stored IDs. See [authoring upgrades](authoring.md#upgrade-existing-authoring-code) for diagnostics and manual cases.

### Add text and URL controls

For text or URL quests, upgrade the Relay SDK and render the corresponding interaction before adding `text` and `url` to `supportedInteractions` in `relay.json`. Text uses a textarea; URLs use a URL field. Read required, length and allowed-host constraints from the manual trigger, and send values through `relay.quests.open(view)`. Keep interrupted submissions resumable after reload. See [submission controls](https://console.domino.run/docs/reference/participant-api#submit-text-or-urls). The new campaign starter includes these controls; existing apps keep their current declared interactions until upgraded.
