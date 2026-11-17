---
name: service-studio-ui
description: Drive the OutSystems Service Studio 11 desktop UI through the windows-mcp tools - open a module from the server/environment, add or remove dependencies (Manage Dependencies, e.g. System AbortTransaction, Forge/library actions), publish (1-Click Publish), close modules, and navigate to actions/screens. Use this whenever the user asks to open, load, close or switch to an O11 module, add a dependency/reference, refresh references, publish a module, or otherwise do something in Service Studio that the servicestudio MCP cannot do on its own - even if they just say "open <app> module" or "add AbortTransaction".
---

# Service Studio UI automation

**Local setup:** if `references/local-setup.md` exists, read it first. It holds this machine's monitor layout, scale factor, exact click positions and environment details, and those make every step faster. It's optional: without it, find each control on a screenshot using the descriptions below.

## Prerequisites

- Windows with **OutSystems Service Studio 11**, already signed in to the environment.
- The **windows-mcp** server registered in Claude Code. It provides the `mcp__windows-mcp__*` tools: Screenshot, Click, Type, Wait and so on. One way to install it:
  `pip install --user uv`, then `claude mcp add windows-mcp --scope user -- <path-to>\uvx.exe --python 3.13 windows-mcp serve --transport stdio`
- Optional: the **servicestudio** MCP (Service Studio's built-in MCP server) for reading and editing the open module's model. Pair it with the `servicestudio-mcp-oml` skill if you have it.

If the windows-mcp tools aren't available, say so and stop. Don't try to drive the UI another way.

## Which tool for which job

| Need | Use |
|---|---|
| Open a module from the environment, Manage Dependencies, publish, close, switch tabs | **windows-mcp** (UI driving) — the servicestudio MCP cannot do these |
| Read or edit the model of the *already open* module (actions, screens, entities, validation) | **servicestudio MCP** (`createSessionToken` once per session, then pass the token) |

The UI is slow and loads asynchronously, so most of the cost is waiting and verifying. The rules below exist to cut wasted screenshots and wrong guesses.

## Working with the screen

- **Find the monitor first.** Run `DisplayInventory` once, or check `local-setup.md`. Then take screenshots of only the display Service Studio is on (`Screenshot` with `display: [n]`). That's much faster than the whole desktop.
- **Scale coordinates.** Screenshots are often downscaled. The result reports a scale factor; multiply image coordinates by it before clicking.
- **Avoid full `Snapshot` with the UI tree.** It walks every open window (Word, browsers…) and often hits its element cap before reaching Service Studio. Use it only when a screenshot can't show you what you need.
- **Don't rely on `WaitFor text_exists`.** It doesn't see Service Studio's (Avalonia) dialog text reliably and times out even when the text is visible. Use `Wait` (2–10 s) followed by a `Screenshot` instead.
- **Focus.** If you paused to ask the user something, Service Studio may have lost focus, so your next click may only bring the window back to the front. Verify every important click on a screenshot.

## Flow 1 — Open a module from the server

1. Bring Service Studio to the front (`App` mode `switch`, name `Service Studio`) and take a screenshot.
2. On the environment tab (titled "<Environment> - <host>"), click **Search Modules**, the button at the bottom of the left panel under "Recent modules". This opens the **Open from Environment** dialog, which shows "Getting modules information…" for a few seconds.
3. Type the name into the search box at the top of the dialog, using `clear: true`. The list filters **2–4 s after** you type, so wait before reading the results. An unfiltered A–Z list just means it hasn't caught up yet.
4. **Resolve informal names.** Users say things like "the foo go module". Match the name without spaces or case (`FooOnGo`). If several modules match (`_BL`, `_Sync`, `_Theme`, `_CS`, …), open the one with **no suffix**, which is usually the app's main/UI module. Tell the user which one you opened and list the siblings, so they can redirect you in one word.
5. Double-click the row. A new tab opens and "Getting references from …" runs for about 15–30 s.
6. It's done when the progress dialog is gone and the toolbar at the top center shows **1-Click Publish** or **Open in browser**. The status bar then shows "<Module> uploaded on <date> by <user>"; include that in your report.

## Flow 2 — Add a dependency (Manage Dependencies)

1. With the module tab active, press **Ctrl+Q** (or click the plug icon in the toolbar under the Module/Edit menus).
2. The dialog first shows "Fetching Data…" in the left pane. This can take **30–60 s**, so wait it out with `Wait` + `Screenshot` rather than retrying.
3. Type the producer module into the **left** search box ("Search modules"). For System actions such as AbortTransaction, CommitTransaction or GetUserId, the producer is **`(System)`**. Click the producer row.
4. Type the element name into the **right** search box ("Search public elements"). Wait about 2 s, then take a screenshot. The tree collapses to the matching folder, such as Server Actions.
5. Tick the element's checkbox and confirm it's checked on a screenshot. A partly checked (–) parent folder is normal.
6. Click **Apply** at the bottom right of the dialog. "Refreshing References" runs for a few seconds.
7. It's done when the dialog is gone and the tab title starts with `*` (unsaved changes).

To remove a dependency, untick the element instead. Service Studio refuses to remove one that is still in use and lists the usages. Report that to the user rather than forcing it.

## Flow 3 — Publish

Publishing deploys to a shared environment that other developers use, so **ask before publishing** unless the user explicitly said to publish in this request. Adding a dependency is not consent to publish.

1. Click the green **1-Click Publish** button at the top center, or press F5.
2. The bottom TrueChange/publish panel shows progress. Poll with `Wait 10` + `Screenshot`.
3. Report success, or copy the error and warning text exactly as shown. If errors appear, you can read them in full with `getValidationMessages` from the servicestudio MCP.

## Flow 4 — Close a module

1. Click the **×** on the module's tab, in the tab strip at the top of the window.
2. If there are unsaved changes (`*` in the tab title), a prompt appears with **Save** (saves a local .oml file, does not publish), **Don't Save** (discards), and **Keep Open**. Neither Save nor Don't Save publishes. If the user didn't say which they want, ask them.
3. If you asked the user, see **Focus** above: your next click may only re-focus the window, so the prompt disappears but the tab stays open. Take a screenshot. If the tab is still there, click × again and pick the option again.
4. It's done when the tab is gone and the environment tab is showing.

## Flow 5 — Find an action, screen or entity

- Prefer the servicestudio MCP (`getActionNames`, `getScreenNames`, `getServerAction`, …) to read the model without clicking around.
- To show the user something in the UI, use the search box: the magnifier icon at the top right, or Ctrl+F. Type the name and double-click the result.

## Reporting

After each flow, report in one or two lines:
- what changed, naming the exact module or element you picked if you had to resolve an informal name
- whether anything is still unsaved or unpublished (a `*` in the tab title)
- any alternatives the user might have meant
