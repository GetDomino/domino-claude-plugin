# Domino plugin for Claude

[Domino](https://domino.run) runs community campaigns: quests, points, leaderboards, referrals and rewards. This plugin adds the Domino skill to Claude. It teaches Claude to set up a Domino project, integrate Domino into an existing app, author and test quests, and stage the participant experience, all through the Domino CLI.

## Setup

The skill drives the `domino` CLI from the npm package [`@domino-sdk/relay-cli`](https://www.npmjs.com/package/@domino-sdk/relay-cli), which needs Node.js 24. This plugin version comes from CLI 1.56.0. Install that CLI and the matching SDK in your app with its package manager, for example:

```sh
npm install @domino-sdk/relay@1.56.0 @domino-sdk/relay-cli@1.56.0
```

Then run `npx domino login`, which signs you in to your Domino account in the browser. The skill in this plugin matches that CLI release, so upgrade the CLI and this plugin together.

To try the plugin in Claude Code from a clone of this repository, start Claude Code with it loaded:

```sh
git clone https://github.com/GetDomino/domino-claude-plugin.git
claude --plugin-dir ./domino-claude-plugin
```

## What it runs and sends

The plugin contains instructions only: no hooks, MCP servers or executables. Following the skill, Claude runs the `domino` CLI in your project. The CLI calls Domino's API (`relay.domino.run`) for the signed-in account, and `domino feedback` sends the tooling reports Claude writes to Domino. Read the CLI's own output before approving each command.

## Privacy and license

Domino's [Privacy Policy](https://domino.run/privacy-policy) and [Terms of Service](https://domino.run/terms-of-service) cover the data the CLI sends. The plugin is distributed under the terms in [LICENSE](LICENSE).

## Source

This repository is generated. The skill's source lives in Domino's private CLI repository, and each CLI release replaces this repository's contents, so changes made here are overwritten. Documentation is at [console.domino.run/docs](https://console.domino.run/docs).
