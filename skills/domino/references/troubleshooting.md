# Development and staging diagnostics

## Report a tooling issue

Send feedback about the CLI, SDK, guides, or your experience to Domino's Linear Triage queue. Sign in with `domino login` first, or use your existing `RELAY_MANAGEMENT_TOKEN`. You do not need a project or a Linear account.

```sh
domino feedback --title "Checkout failed after project creation" --body-file report.md
domino feedback --reporter agent --title "Deploy returned an unclear error" --body-file report.md --json
```

Write a short report with the command or action, reproduction steps, expected behavior, actual error, impact, and any workaround. Remove credentials, private participant data, and unrelated source or logs. The title accepts 1–200 characters; the report accepts 1–20000 characters. Use `--body "Report text"` for a short report or `--body-file -` to read stdin. Use `--dry-run --json` to inspect the report without sending it.

For feedback at task completion, report difficulties that better SDK or CLI support could have prevented, even if the task succeeded without errors. Explain what you were trying to accomplish, the extra steps or workaround you needed, and the helper, command, or clearer API that would have reduced that work. For example, describe having to make several manual API calls because the CLI lacked an operation needed for the task. Submit only difficulties you actually encountered and have not already reported. A general task recap or a report saying everything worked is unnecessary.

The CLI sends only the title, report text, reporter kind, CLI version, Node version, and platform. The server adds your authenticated Domino user and organization IDs. It does not automatically collect files, logs, environment variables, or conversation history. Feedback is sent only when the command runs.

Success returns `sent: true` and the Linear issue's `id`, `identifier`, and `url`. Keep that link in the handoff and submit each distinct issue once. Delivery errors exit 1. An unavailable destination means a Domino operator still needs to configure feedback delivery; keep the report for later. A network or Linear error can leave delivery uncertain: the command exits 3 with `OUTCOME_UNCERTAIN`, and Linear may already have the report, so do not submit it again. Reports are not queued automatically. At most ten submissions per user and organization are allowed per hour.

In an [unattended run](automation.md#unattended-mode) (`DOMINO_AGENT_MODE=unattended`), the command saves the report as one JSON line to `./domino-feedback.jsonl`, or to `DOMINO_FEEDBACK_FILE`, and sends nothing; it needs no sign-in. The result is `sent: false`, `saved: true`, and the `file` path; name that file in the handoff. Pass `--submit`, or set `DOMINO_FEEDBACK=submit`, to send it instead.

## Scope and installation

Read `scopeSources` next to `scope` in `domino doctor --json`. Each value comes from a command flag, a named `RELAY_*` environment variable, the checkout's local connection file, the manifest, or a default. Precedence is flags > environment > checkout connection > manifest. `domino checkout` writes `.git/domino.json`; linked Git worktrees read the file in their shared Git common directory. This untracked file contains API, organization, project, and environment identifiers, not credentials. Do not rewrite starter manifest values merely because they differ from the effective scope.

Use the `packageManager.installCommand` printed by checkout or doctor. It selects the declared version with a frozen lockfile. Wait for the install process to finish before diagnosing missing dependencies. Run checks from the project directory. Without a manifest, `doctor --remote` does not probe the remote and cannot explain why checkout failed.

Starters ship `relay.json` with the placeholder organization `local`. Outside a checkout nothing overrides it, so commands that reach a project refuse with `SCOPE_PLACEHOLDER`, and `domino doctor` reports a `scope` error. Run `domino checkout PROJECT` to clone the hosted project, which records its real scope, or pass `--organization` and `--project`; `domino whoami` lists both. `whoami`, `login`, `create`, `checkout`, `init` and `feedback` ignore the placeholder.

A CLI started on an older Node than its `engines` field allows exits with `NODE_VERSION_UNSUPPORTED` before running anything; `details.minimum` is the Node major version to install.

For `CHECKOUT_GIT_FAILED`, retain the original Git message and the structured `details`: failed phase, exit code, error classification, repository status reported by the API, project, and remote. A `ready` API status combined with a Git "Repository not found" error does not establish whether provisioning, permissions, or transport caused the failure. Retry the same checkout once. If it fails again, send the diagnostics to the operator and keep the existing Console project. Do not create a replacement, reinstall skills in a parent directory, or repeatedly run unrelated diagnostics.

For `PROJECT_MOVED`, an administrator moved the project to another organization. `details` names it: `organization`, `organizationName`, `project`, and `name`. Tell the user where it went, then run `domino login --organization <details.organization>` in the checkout. The user approves the sign-in in Console, and the command updates the checkout's organization in `.git/domino.json` or the manifest. Retry the original command afterwards. Do not edit the scope files by hand or create a replacement project. A `RELAY_MANAGEMENT_TOKEN` personal key belongs to the old organization; the user must create a new one in Console Settings → API keys.

## Capability disagreements

`domino capabilities` reads the catalog bundled with the CLI version in `provenance.cliVersion`. It does not request server capabilities; `serverVerified` is false even when project flags are supplied. Compare the installed version, the applicable public contract, and the actual deployed endpoint before changing the architecture. A successful management route does not by itself prove participant access or payout support. A missing route or catalog entry does not prove platform-wide non-support.

## Check for an SDK upgrade

If a documented request is rejected, an expected method is missing, or SDK and server behavior disagree, check for an upgrade before adding a workaround. During v0, Domino aims to preserve backward compatibility, but breaking changes can occur. Upgrade guidance is advisory; a newer release alone does not make a working project unhealthy.

1. Run the project's installed `domino doctor --json`. Record the CLI version and the installed SDK and CLI versions from `node_modules`. Current doctor reads installed packages, not the starter's historical package versions. A healthy doctor result does not establish server compatibility.
2. Read the [public changelog](https://console.domino.run/docs/reference/changelog). Review the entries between the installed and target versions for the affected feature, including migration commands and changed request shapes. Website references describe their published documentation revision; `domino reference` describes the installed CLI release.
3. Choose an available release that addresses the mismatch. Update the SDK and CLI to the same exact published npm release with the project's package manager. In older projects, replace `file:.domino/*.tgz` entries in `dependencies` and `devDependencies`, and remove the corresponding `pnpm.overrides` entries before installing. Preserve unrelated dependencies and overrides. After installation and checks pass, delete the unused SDK and CLI `.tgz` files and the obsolete `.domino/release.json` archive manifest. Do not replace archives or edit `node_modules`. If the required release is unavailable, report it.
4. Follow the release's migration instructions and review the diff. Regenerate the lockfile through the package manager. Package-backed skill loaders automatically use the installed CLI's guidance; re-read the bundled skill after upgrading. For older checkouts with copied skills, run the upgraded `domino agents install` once and follow the [guidance migration procedure](existing-app.md) if it reports customized files. Run `domino doctor --json` to verify the loaders and their package files are present.
5. Run the app's checks and `domino deploy --preview --environment test --json`. Retry the original failing action, then verify the affected participant flow. Report the before and after versions and any remaining failure.

If a build or typecheck fails on an API that no longer exists after an upgrade, read the [upgrade guide](https://console.domino.run/docs/reference/upgrade-to-v1) for that change's old and new code and a search command for affected code.

Doctor includes links to these instructions and the changelog. It does not query the latest package release or automatically change dependencies.

## Verification failures

Keep build results, browser rendering, and participant behavior separate. If a browser executable is missing or a check fails, report the exact blocked check and leave its behavior unverified. Do not turn a generated visual sample or a successful build into a claim that the app works in a browser.

## Preview diagnostics

Start with `domino doctor --json`. Add `--remote` to verify project access, publication permission, hosting configuration and repository readiness. Configuration checks do not prove a successful build or real participant sign-in. If cloud hosting is missing, a Domino operator must configure it; changing quest code will not fix it.

If participant requests answer 503 "Participant API is not configured", the app's `/relay/*` proxy received a management URL instead of the participant API origin. `domino dev` sends participant requests to the origin the API reports in `/preflight` (`participantApi`), even when `--api-url` points at Console's management-only `/relay` proxy; staged builds use `apiUrl` in relay.json. Check the `participant-api` result of `domino doctor --remote --json` and set relay.json `apiUrl` to the participant API origin it names, such as `https://relay.domino.run`. Upgrade a CLI older than this fix; it passed `--api-url` through unchanged.

`domino check` runs the app's checks, bundles authored entries, and parses each data definition (badges, point ledgers, offers, leaderboards, referrals, dashboards, trackers) with the schema Relay uses. `domino deploy --preview --environment test --json` validates the effective deployment against the server without publishing. Test and live data are separate; test does not imply external provider requests are fake.

`domino stage` preflights hosting before inspecting the commit. If configured, commit and push the app, then stage the exact commit. Use `domino logs <build-id>` for build failures. `domino stage` resends a build request whose answer was lost, with the same request ID, at most three times in all within 30 seconds. If it still exits 3, retry the uncertain build submission with the original request ID. Rebuilding cannot fix an unavailable hosting service.

If a staged wallet-first app's **Connect wallet** stays disabled and `relay.auth.providers()` answers `"wallets": []`, wallet sign-in is off in test. `domino stage` warns about it (`wallet-sign-in-off` in its JSON `warnings`) and `domino doctor --remote` reports a `wallet-sign-in` warning; run the `domino api PUT /participant-sign-in` command they print. It applies at once, without staging again.

If a staged app's routes answer 503 because a credential such as `APP_API_KEY` is missing, or `domino stage` warns that a required app secret is not set, run `domino secrets list` to see what is missing, set it with `printf %s "$VALUE" | domino secrets set NAME`, then stage again: a secret reaches only builds staged after it changes. `domino dev` never receives hosted secrets; put local values in `.dev.vars`. If `domino secrets set` answers 503 "App secrets are not configured on this deployment", a Domino operator must configure hosting; staging again will not fix it. If `domino stage` fails with "App secret setup failed" and the message says a secret can't be read with this deployment's key, the platform's sealing key changed: set that secret again, then stage.

For compatibility reports, include `domino --version`, `.domino/starter.json` when present, the lockfile, the command and its error. Exclude credentials and private participant data. Install the starter with its frozen lockfile. `starter.json` records the template, source revision, and original published package versions. The package manager checks downloaded package integrity against the lockfile.

If development reports unknown catalog fields at `/releases/batch`, the CLI is using an obsolete startup contract. Upgrade to the corrected CLI. Complete deployment bundles belong at `/deployments/preview`, followed by `/deployments/:deployment/publish` with the returned ID in the path. Do not strip fields from a user's bundle as a permanent workaround.

For existing-login failures, read `domino capabilities identity-exchange` and `domino reference participant-api`. Use a dedicated `di_` identity integration key from Console Settings → App login keys. Keys of a deleted project stop working until support restores it. A 401 means the key or participant session is invalid, expired, revoked, or belongs to another integration. A 400 rejects malformed input, including attempted scope or role overrides. A 429 asks the server to retry later. Older APIs return 404; upgrade the deployment. The Console `/relay` URL remains management-only. The user's application must use the participant API origin through `createParticipantBackend`.

## Hosted storage and SQL migrations

If a hosted binding is missing, declare it in `relay.json` `app.storage` and stage again; bindings in `wrangler.jsonc` alone are not imported. Use `domino storage list --json` to see the active build's declarations and retained stores. A local/hosted mismatch warning means the root Wrangler JSON/JSONC names differ from `relay.json`; mirror them for `domino dev`.

A failed SQL migration names its store and file and includes D1's error message. The build is not retried, the previous build keeps serving, and earlier migrations remain applied; only the failed file rolls back, matching Wrangler remote migrations. Correct it and stage again with the same filename. "Ran but did not record itself" means the file ends inside an unterminated `/* …` comment. "Nested layouts … are not supported" means the migrations directory holds folders instead of `.sql` files. A store-limit error means the project already owns 10 stores of that kind, including undeclared ones: reuse an existing name, or delete an unused one with `domino storage delete KIND NAME --yes` after confirming it holds nothing needed (`domino storage tables` or `keys`). Follow [storage migration recovery](https://console.domino.run/docs/build/storage#write-migrations). A Cloudflare provisioning refusal may mean the builder token lacks **Workers KV Storage: Edit** or **D1: Edit**; only a human operator changes those permissions.
