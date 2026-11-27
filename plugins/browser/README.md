# browser

Gives Claude the Playwright browser tools: open a page, read its accessibility tree, press keys, hover, click, measure and take screenshots.

```bash
claude plugin marketplace add mschieller/mstack
claude plugin install browser@mstack
```

## Why it exists

It runs the same server as the official Playwright plugin, `@playwright/mcp`. The official plugin starts it with `npx` in the repo's folder. npm then reads the repo's `package.json`, and when `devEngines.packageManager` names pnpm, npm exits with `EBADDEVENGINES` and the browser never starts.

This plugin starts the server with `npx --prefix` set to the plugin's own folder, which has no `package.json`. npm never reads the repo's `devEngines`, so the same command works in npm and pnpm repos. The working directory is still the repo, so screenshots and the `.playwright-mcp` folder land there.

## What uses it

The `accessibility-review` skill and the `accessibility-reviewer` agent in the [mstack](../mstack/README.md) plugin check rendered pages with these tools. They accept the tools of this plugin or of the official Playwright plugin.

Install one of the two. With both installed, Claude starts two browser servers, and the skill and the agent use this plugin's.

## What it needs

Node with `npx`. The first start downloads `@playwright/mcp`.
