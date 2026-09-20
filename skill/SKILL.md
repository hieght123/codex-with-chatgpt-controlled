---
name: codex-with-chatgpt-controlled
description: Coordinate read-only planning and review with web ChatGPT from a Codex in-app browser. Local Codex keeps all write, terminal, test, and commit authority.
---

# Codex with ChatGPT Controlled Profile

Use this profile when web ChatGPT should review or plan against a workspace while local Codex performs implementation and verification. It does not grant web ChatGPT write, terminal, commit, credential, or arbitrary-path capabilities.

## Safety and scope

- Use only a connector scoped to the user-approved workspace.
- State that web ChatGPT will receive persistent read-only access to workspace files, search, Git status and diffs, and execution summaries before pairing a new workspace.
- Do not expose write, delete, shell, commit, credential, or arbitrary-path tools through the connector.
- Keep tokens, pairing codes, connector URLs, workspace identifiers, and absolute paths out of prompts, logs, screenshots, and repository files.
- Pin a reviewed browser runtime version. Do not pull source updates, rebuild dependencies, or update the runtime during an active collaboration; perform those actions only in a separately approved maintenance task.
- Do not change providers, accounts, network settings, or restart Codex as part of normal collaboration.
- Stop when the connector health check fails. Do not treat a running local process as proof of end-to-end readiness.

## Browser rules

- Control web ChatGPT only through the Codex in-app browser named `iab`.
- Do not use generic computer-use APIs, physical coordinates, or accessibility-tree clicks.
- Locate the composer by a stable selector such as `#prompt-textarea`.
- Use one browser action per tool call: fill the prompt, submit it with Enter, then read the reply.
- Before filling the composer, confirm that no response is generating and that a user draft will not be overwritten.
- Do not call browser handoff methods during automation. Keep the current tab as a deliverable until the read-only review is complete.

## Failure policy

- Allow no more than one retry for an individual browser action.
- Pass an explicit 60-second timeout to every browser operation. Do not rely on a shorter default timeout for cross-process calls.
- Bound total recovery time. When the budget is exhausted, enter the `C2C_TASK_KERNEL_UNAVAILABLE` circuit-breaker state instead of continuing in the background.
- If the browser runtime reports missing temporary assets or an unavailable kernel, make no further retry or reset attempt in that task. Open a fresh task and initialize a new browser runtime. Do not search the filesystem for temporary files.

## Review loop

1. Confirm an approved, read-only connector is ready for the intended workspace.
2. Ask web ChatGPT for a small, testable plan or review.
3. Let local Codex inspect, implement, and run relevant tests.
4. Send the resulting diff and verification summary for read-only review.
5. Report which actor planned or reviewed, which actor executed, the tests run, and remaining uncertainty.

## Codex desktop browser sequence

The caller supplies `CODEX_BROWSER_PLUGIN_DIR`, the directory containing Codex's bundled `browser-client.mjs`. This directory is installation-specific and must not be hard-coded in a public repository.

### Step 1: initialize and fill

```javascript
const fs = await import("node:fs");
const path = await import("node:path");
const { pathToFileURL } = await import("node:url");

const browserRoot = process.env.CODEX_BROWSER_PLUGIN_DIR;
if (!browserRoot) {
  throw new Error("CODEX_BROWSER_PLUGIN_DIR is required.");
}

const versions = fs.readdirSync(browserRoot)
  .filter((version) => fs.existsSync(path.join(
    browserRoot,
    version,
    "scripts",
    "browser-client.mjs",
  )));
if (versions.length === 0) {
  throw new Error("No supported Codex browser client was found.");
}

versions.sort((left, right) => left.localeCompare(
  right,
  undefined,
  { numeric: true, sensitivity: "base" },
));
const clientPath = path.join(
  browserRoot,
  versions.at(-1),
  "scripts",
  "browser-client.mjs",
);
const { setupBrowserRuntime } = await import(pathToFileURL(clientPath).href);

const agent = await setupBrowserRuntime();
const iab = await agent.browsers.get("iab");
await (await iab.capabilities.get("visibility")).set(true);

const tabs = await iab.tabs.list();
const existing = tabs.find((candidate) => candidate.url?.includes("chatgpt.com"));
const tab = existing ? await iab.tabs.get(existing.id) : await iab.tabs.new();
if (!existing) {
  await tab.goto("https://chatgpt.com/", { waitUntil: "domcontentloaded" });
}
await tab.markDeliverable();
globalThis.c2cTab = tab;

const precheck = await tab.playwright.evaluate(() => {
  const stop = document.querySelector(
    "button[data-testid='stop-button'], button[aria-label='Stop generating']",
  );
  const composer = document.querySelector("#prompt-textarea");
  return {
    isGenerating: Boolean(stop),
    hasComposer: Boolean(composer),
    draft: composer?.textContent?.trim() ?? "",
    turnCount: document.querySelectorAll(
      "[data-testid^='conversation-turn']",
    ).length,
  };
});

if (
  precheck.isGenerating
  || !precheck.hasComposer
  || (precheck.draft && !precheck.draft.startsWith("【C2C"))
) {
  throw new Error("Composer is not safe to overwrite.");
}

await tab.playwright.locator("#prompt-textarea").first().fill(
  "Review the approved workspace read-only. Return evidence-based findings only.",
);
nodeRepl.write("[C2C_PROMPT_FILLED]");
```

### Step 2: submit separately

```javascript
await globalThis.c2cTab
  .playwright
  .locator("#prompt-textarea")
  .first()
  .press("Enter");
nodeRepl.write("[C2C_SUBMITTED]");
```

### Step 3: read separately

```javascript
const reviewState = await globalThis.c2cTab.playwright.evaluate(() => {
  const stop = document.querySelector(
    "button[data-testid='stop-button'], button[aria-label='Stop generating']",
  );
  const replies = document.querySelectorAll(
    '[data-message-author-role="assistant"]',
  );
  const last = replies.length ? replies[replies.length - 1] : null;
  const markdown = last?.querySelector(".markdown, .markdown-body, .prose")
    ?? last;
  return {
    isGenerating: Boolean(stop),
    reply: markdown?.innerText?.trim() ?? "",
  };
});

if (!reviewState.isGenerating && reviewState.reply) {
  nodeRepl.write(reviewState.reply);
} else {
  nodeRepl.write("[C2C_GENERATING_WAIT]");
}
```

Do not combine these three steps into a single browser call.

