---
name: raycast
description: "Best practices and workflows for developing and modifying Raycast Extensions (React/Node) AND Raycast Script Commands (shell scripts with @raycast metadata). Use this skill whenever the user wants to create, update, or troubleshoot anything that runs inside Raycast — extensions, script commands, menu-bar commands, AI extension tools, deeplinks, quicklinks. Triggers include: 'make a Raycast command', 'Raycast script command', 'fix my Raycast script', '@raycast.title', registering a Script Directory, useCachedPromise, useFetch, useForm, popToRoot, MenuBarExtra, raycast:// deeplinks, calling AppleScript/Keyboard Maestro/shell/SSH from Raycast, or any task involving shell scripts under ~/raycast-scripts/. Use even when the user asks for a 'quick automation' or 'Raycast hotkey' without explicitly naming the type — Claude should pick script command vs extension based on the task."
---

# Raycast Development

This skill defines the preferred workflow and best practices for building, modifying, and troubleshooting Raycast **Extensions** (React/Node) and **Script Commands** (shell scripts with `@raycast.*` metadata).

## 0. Reference files — read the one that matches the task

The sections below are the rules. The exact signatures and doc facts live in `references/`, distilled 09-24-26 from developers.raycast.com and manual.raycast.com (API 2.5.0, @raycast/utils 2.3.0):

| 📄 File | Open it when you are… |
|---|---|
| `references/platform.md` | editing `package.json` (commands, modes, `interval`, arguments, preferences, `platforms`), using LaunchProps / background refresh / deeplinks, running `ray` CLI commands, debugging, building AI tools / `ai.yaml` / evals |
| `references/api-reference.md` | writing UI or calling `@raycast/api`: List, Grid, Form, Detail, Actions, MenuBarExtra, Toast/HUD/Alert, Clipboard, Cache, LocalStorage, Keyboard shortcuts, launchCommand, OAuth, AI.ask, WindowManagement, BrowserExtension |
| `references/utils-hooks.md` | picking or configuring a `@raycast/utils` hook (usePromise, useCachedPromise, useFetch, useExec, useSQL, useForm, useFrecencySorting…), pagination, optimistic `mutate`, runAppleScript, OAuthService, and changelog / migration / ESM questions |
| `references/script-commands-and-app.md` | writing a Script Command (every `@raycast.*` key, arguments, output per mode, exit codes), or using app features: aliases, hotkeys, Hyper Key, quicklinks, Dynamic Placeholders, `raycast://` URLs, MCP, Raycast AI skills, logs |

*Trigger:* you are about to write a Raycast prop, option, manifest key, or metadata key you have not checked in this session. *Discrimination:* the rules below are enough for workflow decisions; they are not enough for exact names and defaults (`filtering`, `keepPreviousData`, `refreshTime`…), which drift between API versions. *Action:* grep the matching reference file first. If it is missing or looks stale, ask the live docs: `curl -s 'https://developers.raycast.com/readme.md?ask=<question>'` (the answer comes back with excerpts), or grep `https://developers.raycast.com/llms-full.txt` (two parts, the second at `/llms-full.txt/1`) and `https://manual.raycast.com/llms-full.txt`. Then add what you learned to the reference file.

## 1. Decide: Script Command or Extension?

Pick the simplest tool that fits. Choosing wrong adds complexity (Extensions) or hits walls (Script Commands).

**Use a Script Command when:**
- The action is a single shell invocation (run a command, fire a magic packet, restart a service, SSH somewhere and run something).
- No UI is needed beyond a brief HUD-style success/failure toast, or one line of live text in root search (`inline` mode + `refreshTime`).
- No persistent state, no dropdowns populated from APIs, no multi-step forms. (A fixed dropdown argument is fine: `"type": "dropdown"` with `data`.)
- A single `.sh` (or `.py`, `.swift`, etc.) file is enough — no `npm install`, no build step.

**Use an Extension when:**
- The user needs a form, a list, a dropdown populated from an API, or any other interactive UI.
- The command has multiple steps that the user navigates between.
- It integrates with an external API and needs to manage tokens via Raycast preferences.
- It needs caching, debouncing, background refresh on a schedule, a menu-bar item, or tools Raycast AI can call.

**Before either, check the built-ins.** A URL with a slot is a **Quicklink** (`{argument name="query"}`, up to 3 arguments, any app deeplink works). Fixed text is a **Snippet**. One prompt run on selected text is an **AI Command**. None of these needs code.

When in doubt, lean Script Command first. Migrating up to an Extension later is easy; downgrading a half-built Extension to a Script Command feels like wasted work.

## 2. Script Commands

### Anatomy

A Script Command is a single executable file with `@raycast.*` metadata comments near the top. Minimal template:

```bash
#!/usr/bin/env bash

# Required parameters:
# @raycast.schemaVersion 1
# @raycast.title My Command
# @raycast.mode compact

# Optional parameters:
# @raycast.icon 🔧
# @raycast.packageName Personal
# @raycast.description Does the thing.
# @raycast.argument1 { "type": "text", "placeholder": "Optional input", "optional": true }
# @raycast.needsConfirmation false
```

**Modes — what each shows:**

| Mode | Shows | Use for |
|---|---|---|
| `compact` | LAST line of stdout in a toast | Best default for personal automations |
| `silent` | LAST line (if any) as a HUD after the window closes | Fire-and-forget |
| `inline` | FIRST line inside the root-search row, re-run every `refreshTime` | Live status (favorite it for a dashboard) |
| `fullOutput` | All stdout in a terminal-like view | Long output or long-running jobs |

Other languages work too — use `#!/usr/bin/env python3`, `#!/usr/bin/env node`, `.applescript`, etc. The metadata format is the same (`//` comments work too).

**Metadata keys the template leaves out** (full list: `references/script-commands-and-app.md`):
- `@raycast.needsConfirmation true` — a confirm alert before running. Use it on anything destructive.
- `@raycast.refreshTime 1m` — `inline` only, minimum `10s`. `inline` WITHOUT it silently behaves like `compact`. Only the first 10 inline commands refresh on their own.
- `@raycast.currentDirectoryPath ~/somewhere` — default is the script's own folder.
- `@raycast.iconDark` — dark-theme icon. `packageName` defaults to the script folder's name.
- Arguments: max 3; `type` is `text` / `password` / `dropdown` (dropdown needs `data: [{"title","value"}]`); `percentEncoded: true` for URL use; arguments are REQUIRED unless `"optional": true`; values arrive as `$1 $2 $3`, an omitted optional one as `""`.

### Registration

1. Place the script in a stable directory (e.g. `~/raycast-scripts/`).
2. `chmod +x` the file.
3. In Raycast: Settings → Extensions → `+` → **Add Script Directory** → point at the directory.
4. Raycast auto-picks up metadata edits — no restart needed.

**A script that never shows up** fails one of: all 3 required keys present (`schemaVersion`, `title`, `mode`), not executable, or its filename contains `.template.` (Raycast ignores those on purpose).

### Critical gotchas

These bite every time and aren't obvious from Raycast's documentation:

- **Minimal PATH.** Raycast runs script commands in a NON-login shell and only appends `/usr/local/bin` — `/opt/homebrew/bin` (Apple Silicon Homebrew) is missing. Tools like `wakeonlan`, `jq`, `ffmpeg`, `gh`, `kubectl` will fail with "command not found" even though they work in Terminal. Locate them explicitly (or `export PATH="/opt/homebrew/bin:$PATH"` at the top, or a `#!/bin/bash -l` shebang):
  ```bash
  TOOL=""
  if command -v wakeonlan &> /dev/null; then
    TOOL="$(command -v wakeonlan)"
  else
    for p in /opt/homebrew/bin /usr/local/bin; do
      [ -x "$p/wakeonlan" ] && TOOL="$p/wakeonlan" && break
    done
  fi
  ```

- **SSH calls need `BatchMode` and `ConnectTimeout`.** Without `-o BatchMode=yes`, a failed key auth falls back to a password prompt that hangs Raycast indefinitely. Without `-o ConnectTimeout=N`, an unreachable host blocks until the system-level timeout (~75 seconds). Pattern:
  ```bash
  ssh -o BatchMode=yes -o ConnectTimeout=5 user@host "command"
  ```

- **Errors: print the reason LAST, then exit non-zero.** A non-zero exit shows a failure toast, and in `compact`/`inline` the last output line becomes the error text: `echo "Mac Mini is asleep"; exit 1`. Chatty tools (`zip` without `-q`, progress bars) break `compact`/`silent`/`inline` — quiet them or use `fullOutput`.

- **Test from Terminal first.** In `compact` mode Raycast suppresses stderr by default. If a script silently fails when run via Raycast but works in Terminal, suspect a PATH or environment difference and add explicit absolute paths. Reproduce with a stripped env: `env -i HOME="$HOME" PATH=/usr/bin:/bin:/usr/local/bin ./script.sh`.

- **Permissions belong to Raycast.** Accessibility, Automation, and Full Disk Access prompts name Raycast, not Terminal. A script that works in Terminal can still need its own grant for Raycast.

- **Arguments are optional UI surface.** Hardcoding values in a CONFIG block at the top of the script is often cleaner for personal automations than exposing arguments. Use `@raycast.argument1` only when variability actually helps.

- **Hotkey- and refresh-triggered scripts must be idempotent** — "ensure X is on" beats a toggle that drifts out of sync.

### When NOT to use a Script Command

If the script ends up needing any of these, migrate to an Extension instead:
- Dropdown populated dynamically from an API.
- Form with multiple fields the user fills in.
- Cached/debounced state across runs.
- Branching UI based on results (a list of items the user picks from).

## 3. Extension Scaffolding & Setup

- **Preferred Method (Duplication)**: The fastest way to start a new extension is often duplicating the folder of an *existing* extension. If you do this, you must carefully update the `package.json`:
  - `"name"`
  - `"title"`
  - `"description"`
  - `"author"`
  - `"commands"` array (update `name`, `title`, and `description`).
- **Standard Method**: If not duplicating, use `npx @raycast/api@latest create`, or Raycast's **Create Extension** command, whose templates include Show List, Show Grid, Submit Form, Menu Bar Extra, Run Script (no-view + HUD), Show Typeahead Results, and AI.
- **Adding a command to an existing extension**: Raycast's **Manage Extensions** → your extension → **Add New Command** (or **Add New Tool**, ⌥⌘T) updates `package.json` and scaffolds `src/<name>.tsx` in one step.
- **Local Dev**: After creating/duplicating, run `npm install` then `npm run dev` (`ray develop`). This builds the extension, imports it into the user's Raycast app, and starts a watcher. The build is finished once you see `ready - built extension successfully`. The import **persists after you stop the watcher**, so for a one-off install you can stop `ray develop` after the first successful build. `npm run build` (`ray build -e dist`) only compiles to `dist/` and runs the TypeScript check — it does **not** refresh the imported dev extension, so after editing source, re-run `ray develop` to update what's loaded in Raycast.
- **One-shot import from a session (no watcher left running).** *Trigger:* you changed an extension and need Raycast to load it, but nobody will watch a dev terminal. *Discrimination:* `npm run build` does not reach Raycast; a `ray develop` left running keeps a watcher alive in the background. *Action:* start `npm run dev > <scratch>/dev.log 2>&1` in the background, wait with `until grep -qE 'built extension successfully|rror' <log>; do sleep 1; done`, then `pgrep -lf -- '<ext>/node_modules/.bin/ray develop'` and `kill <that PID>` (never a pattern kill). Confirm the install by reading `~/.config/raycast/extensions/<name>/package.json` or `cmp`-ing an asset.
- **A failed rebuild keeps the OLD code running.** *Trigger:* the user says a change "didn't apply" while `ray develop` is running. *Discrimination:* the watcher only swaps in a build that compiled; a TypeScript error leaves the last good build loaded, and the only sign is the command icon (bottom-left) in a failed state. *Action:* read the `ray develop` terminal output (or `npx ray build -e dist`) for the error before touching anything else.
- **Other CLI commands** (`npx ray …`): `lint`, `migrate` (codemods to the latest API — review the diff), `bundle -o <path>` (a `.rayext` file, a way to archive or share a personal extension without the Store). Details in `references/platform.md`.
- **`package.json` hygiene**: set `"platforms": ["macOS"]` whenever the extension uses AppleScript, Keyboard Maestro, or other macOS-only APIs. Never add a `"version"` field (Raycast has none). Keep `raycast-env.d.ts` gitignored and let the build regenerate it.
- **Renaming an existing extension**: changing the `name` field (and/or the source folder) makes Raycast treat it as a *new* development extension. The old entry lingers under Settings → Extensions pointing at the old path and must be removed manually, and preferences (`notionToken`, etc.) do **not** carry over to the new identity — they must be re-entered. Keep the command `name` in `package.json` in sync with its entry-point filename (`src/<command-name>.tsx`); `raycast-env.d.ts` is auto-generated from `package.json`, so let the build regenerate it rather than editing it by hand.
- **`Cannot find namespace 'Arguments'` (or `'Preferences'`) on build.** *Trigger:* `ray build` fails with TS2503 on `Arguments.X` / `Preferences.X`. *Discrimination:* the types exist — `raycast-env.d.ts` is generated at the extension root — but a `tsconfig.json` copied from an older extension only includes `src/**/*` (a common trap when one extension's tsconfig is copied into another). *Action:* set `"include": ["src/**/*", "raycast-env.d.ts"]`; check this whenever you duplicate a tsconfig.
- **ESM-only npm packages** need the whole extension converted to ESM (`"type": "module"`, `node16` module resolution, `.js` extensions on relative imports) — prefer a CommonJS alternative for a personal extension. Global `fetch` exists (API 1.94+); don't add `node-fetch`.

## 4. Extension UI & UX Requirements

When modifying or creating forms and commands, adhere to these standards:

- **Clean Inputs**: Do not set `defaultValue` with placeholder characters (e.g., `- ` for bullet points) unless explicitly requested. The user also dislikes placeholder ghost text — when a field needs explaining, use the `info` prop (a tooltip) rather than `placeholder`. Raycast's Store guidelines and ESLint rule (`@raycast/prefer-placeholders`) ask for placeholders — this skill's rule wins; turn that rule off in `eslint.config.js` if lint complains. The one exception: command `arguments` REQUIRE a `placeholder` in `package.json` — it is the field's only label, so make it one short noun ("minutes"), never a sentence (the box cuts text off at about 17 characters).
- **Timing and progress feedback for fire-and-forget commands**: end with a HUD that says what will happen and when (`⏱️ 5 min timer · ends 4:58 PM`), not just "Started".
- **Auto-Close on Success**: After successfully completing an action (like adding a Notion page, running a system script, etc.), the extension should disappear and return the user to the root Raycast search, rather than retaining the form on-screen.
  - Import: `import { popToRoot } from "@raycast/api";`
  - Execute: `await popToRoot({ clearSearchBar: true });`
  - In a `no-view` command, `showHUD(...)` closes the window itself. Call `await closeMainWindow()` BEFORE slow work (AppleScript, network) so the command feels instant.
- **Remembered values**: `storeValue` on a Form item restores the last submitted value next time — use it for fields the user repeats (a default database, a project). `<Form enableDrafts>` keeps an unsent form, but `popToRoot()` skips draft saving.
- **Validation**: use `useForm` + `FormValidation.Required` from `@raycast/utils` (spread `itemProps.x`) rather than hand-rolled `error` state. `onSubmit` never fires while any field shows an error.
- **Titles**: Title Case for command and action titles (`Copy to Clipboard`, `Open in Browser`); commands are `<Verb> <Noun>` (`Search Contacts`, `Create Task`). Don't set `navigationTitle` on a root command.
- **No flicker**: pass `isLoading` to the top-level view and don't render an empty list before data arrives; use `List.EmptyView` only for a real empty state.
- **Toasts for long work**: `const t = await showToast({ style: Toast.Style.Animated, title: "Saving…" })`, then set `t.style` / `t.title` to Success or Failure in place. A toast shown while the window is closed turns into a HUD.
- **Icon Changes**: If an extension's icon is modified (e.g., `extension-icon.png`), **Raycast must be fully restarted (⌘Q → reopen)** to reflect the change — the watcher rebuild alone is not enough. Always explicitly alert the user to "Restart Raycast" when you modify an icon. The icon file must be a **real PNG with alpha channel** (not a renamed JPEG) — use `sips -s format png source.jpg --out assets/extension-icon.png` to convert if needed. A dark-mode variant is just `icon@dark.png` next to `icon.png`. Changed list/asset icons (not the extension icon) refresh without a restart through the dev action that clears the local assets cache.
- **Icon transparency & the Antigravity JPEG trap.** When the user attaches an image in Antigravity chat, the platform silently converts it to JPEG, which **strips the alpha channel** — transparent areas become black. `sips -s format png` re-encodes the pixels but cannot restore lost transparency. To fix icons with baked-in black corners, use a Python flood-fill from the four corners (Pillow: `Image.convert("RGBA")`, then flood-fill near-black pixels reachable from `(0,0)`, `(w-1,0)`, `(0,h-1)`, `(w-1,h-1)` with `(0,0,0,0)`). Better yet: **ask the user for a filesystem path** to the original PNG instead of a chat attachment — e.g., "drop it in `~/Downloads/` and give me the path."

## 5. Extension Data Fetching & State

- **Pick the hook by data source** (all from `@raycast/utils`; options in `references/utils-hooks.md`):

  | Data comes from | Use |
  |---|---|
  | An async function (SDK call, Notion, files) | `useCachedPromise` — shows the last result instantly, then refreshes |
  | An HTTP endpoint | `useFetch` (headers/body in options, `mapResult` to reshape) |
  | A CLI / binary | `useExec(file, args, { parseOutput })` |
  | A local SQLite DB (Notes, Messages) | `useSQL` — return its `permissionView` first |
  | Result can't be JSON-serialized, or must never be stale | `usePromise` |
  | A no-view command or AI tool (no hooks allowed) | `withCache(fn, { maxAge })` / `executeSQL` / plain `await` |

- **`useCachedPromise` results must be JSON-serializable.** A `Date` comes back as a string on the next launch. Store ISO strings and convert when rendering.
- **Hooks already show a failure toast.** The default `onError` shows "Failed to fetch latest data" with a Retry action; pass `failureToastOptions: { title }` to reword it. Passing `onError` replaces the toast, so show your own inside it.
- **Search-as-you-type:** setting `onSearchTextChange` on a `List` turns OFF native filtering. Pass `searchText` into the hook's args, add `throttle`, and set `keepPreviousData: true` so the list doesn't flash empty between keystrokes. If you only want to watch the text, pass `filtering={true}` explicitly.
- **Pagination:** make the hook's function return `async ({ page, cursor }) => ({ data, hasMore, cursor })` and pass the returned `pagination` to `<List pagination={pagination}>`. Only page 1 is cached.
- **Mutations:** prefer `await mutate(apiCall(), { optimisticUpdate: (d) => … })` over calling `revalidate()` by hand — it updates the UI at once, rolls back on error, and revalidates after. Wrap it in try/catch: it rethrows.
- **Where to keep state:**

  | Store | For | Notes |
  |---|---|---|
  | `LocalStorage` / `useLocalStorage` | small durable user data | async, encrypted, `string \| number \| boolean` values |
  | `Cache` / `useCachedState` | disposable fetched data, instant first paint | sync, strings only, 10 MB LRU |
  | files in `environment.supportPath` | anything large | plain `fs` |

  The dev action "Clear Local Storage & Cache" wipes both of the first two.
- **Recently used first:** `useFrecencySorting(items, { key })`, calling `visitItem(item)` in the primary action.
- **Dynamic Forms**: For fields like Notion's `select`, `multi_select`, or integrations with existing playlists, dynamic `Form.TagPicker` or `Form.Dropdown` components should be populated via `useCachedPromise` rather than hardcoding.

## 6. Extension Automation & Local Execution

When an extension requires triggering local MacOS functionality (where external APIs fall short or aren't applicable):

- **AppleScript & Keyboard Maestro**: prefer `runAppleScript` from `@raycast/utils`, passing values as an `args` array instead of splicing them into the script text (no quoting bugs). The script reads them with `on run argv` / `item 1 of argv`. For structured results use `language: "JavaScript"` and return JSON; `humanReadableOutput` defaults to true and flattens lists. Default timeout is 10 s. Raw `execFileSync("/usr/bin/osascript", ["-e", script])` is still fine; never `execSync("osascript -e '...'")`.
  - *Example*: `await runAppleScript('on run argv\n tell application "Keyboard Maestro Engine" to do script (item 1 of argv)\nend run', [MACRO_ID])`
- **Handy system calls** (`@raycast/api`): `getSelectedText()`, `getSelectedFinderItems()`, `getFrontmostApplication()`, `open(target, bundleId)`, `showInFinder`, `trash` (moves to Trash, never deletes). The first two REJECT when nothing is selected or Finder isn't frontmost — wrap in try/catch and show a failure toast.
- **Clipboard Operations**: Always use the native `Clipboard` utilities from `@raycast/api` (e.g., `Clipboard.copy(text)`) rather than custom bash scripts. `{ concealed: true }` keeps secrets out of clipboard history; `Clipboard.copy({ file: path })` copies a file; `Clipboard.readText({ offset: 1 })` reads back into history.
- **Native Mac sounds** (for alarms and alerts; confirm the choice with the user before building): classic short alerts in `/System/Library/Sounds/*.aiff` (Glass, Hero, Ping, Submarine…), longer modern tones in `/System/Library/PrivateFrameworks/ToneLibrary.framework/Versions/A/Resources/AlertTones/Modern/*.m4r` (Circles, Chord, Pulse…). Preview with `afplay <file>` or System Settings → Sound → Alert sound. To loop an alarm until it's closed, play it from a Swift helper with `NSSound.loops = true`.
- **Commands calling commands**: `launchCommand({ name, type: LaunchType.Background, context })` — e.g. a form refreshing its menu-bar item after saving. The target reads `props.launchContext`.
- **Reconsider scope**: If the extension's *entire* purpose is to fire a single shell command, a Script Command is simpler. Extensions earn their weight when there's actual UI or state involved.

## 7. Native Binaries & Asset Bundling

When an extension shells out to a compiled binary (Swift, Go, Rust, etc.):

- **Put binaries in `assets/`.** Raycast copies `assets/` to the installed extension directory (`~/.config/raycast/extensions/<name>/assets/`). Files outside `assets/` (e.g., `swift/`, `bin/`) are **not copied** and will fail with "No such file or directory" at runtime.
- **Reference via `environment.assetsPath`.** Use `path.join(environment.assetsPath, "binary-name")` — never construct paths relative to the source tree with `..`.
- **Use `execFileSync`, not `execSync`.** The user's workspace path often contains spaces (e.g., `Raycast Extensions/Contacts`). `execSync(cmd)` runs through a shell and breaks on unquoted spaces. `execFileSync(path, args)` bypasses the shell entirely and handles spaces safely. (In a view command, `useExec(path, args)` does the same with caching.)
  ```typescript
  // ✅ Correct — handles spaces, no shell quoting needed
  const stdout = execFileSync(BRIDGE_PATH, ["--search", query], { encoding: "utf8" });

  // ❌ Breaks on paths with spaces
  const stdout = execSync(`${BRIDGE_PATH} --search "${query}"`, { encoding: "utf8" });
  ```
- **Ensure execute permission.** After compiling or copying the binary, run `chmod +x` on the asset.
- **Work that must outlive the command (timers, alarms, watchers).** *Trigger:* a command has to act minutes later ("remind me in 5 min", "alert when done"). *Discrimination:* a `no-view` command is unloaded as soon as its promise resolves, and background `interval` is approximate and at least 1m — neither can hold a precise countdown. *Action:* spawn a helper binary from `assets/` detached — `spawn(bin, args, { detached: true, stdio: "ignore" }).unref()` — then `showHUD` and return. The helper owns the wait (use a wall-clock deadline, e.g. `DispatchQueue.main.asyncAfter(wallDeadline:)`, so Mac sleep doesn't delay it) and any UI. Verify it survives: its PPID is 1 after the parent exits. Give the helper a headless `--selftest` flag so it can be verified without waiting.
- **Surface errors visibly.** Don't `catch` and return empty results — `throw` or show a `showToast(Toast.Style.Failure, ...)` (or `showFailureToast(error)` from `@raycast/utils`) so the user sees what went wrong instead of a blank list.

## 8. List Navigation Patterns

- **Detail panel (`isShowingDetail`)**: Shows contact/item metadata alongside the list. Good for "see everything at a glance" without leaving the list. Fields rendered as `List.Item.Detail.Metadata.Link` are clickable with the mouse but **not keyboard-focusable** — they're read-only display. Don't combine `accessories` with `isShowingDetail`; use `Metadata.TagList` for colored tags inside the panel.
- **Drill-in navigation**: Use `Action.Push` as the primary action to push a second `List` where each field is its own `List.Item` with actions. This gives full keyboard navigation (↑↓ between fields, Enter to act, Esc to go back). Combine both patterns: detail panel for visual scanning + Enter to drill in for keyboard interaction. Use `onPop` on `Action.Push` to `revalidate()` the parent when the child changed something.
- **Filter dropdown**: `searchBarAccessory={<List.Dropdown tooltip="…" storeValue onChange={…}>}` gives a ⌘P filter in the search bar that remembers its choice.
- **Keyboard shortcuts**: Add `shortcut` props to frequently-used actions. Prefer `Keyboard.Shortcut.Common.*` (`Copy`, `Open`, `Refresh`, `Remove`, `Pin`, `Edit`…) over hand-picked combos — they match the rest of Raycast. Shortcuts show automatically in the action panel (⌘K). Add shortcut hints as `Metadata.Label` items at the bottom of a detail panel for persistent visibility. Raycast ignores ⌘K, ⌘W, and ⌘Esc. A custom shortcut on the first two actions works but is not displayed (they are ↵ and ⌘↵, or ⌘↵ and ⌘⇧↵ in a Form).
- **Action ordering**: The **first** action in the `ActionPanel` becomes the primary action shown in the bottom bar and triggered by Enter. Order actions by frequency of use. Group with `ActionPanel.Section`; a submenu title ends with `…` (`Set Priority…`).
- **Destructive actions (delete, etc.)**: Use `confirmAlert` with `Alert.ActionStyle.Destructive` for irreversible operations. Style the action itself with `style={Action.Style.Destructive}` so it renders in red. After the mutation, call `revalidate()` (from `useCachedPromise`) to refresh the list. Pattern:
  ```tsx
  <Action
    title="Delete Contact"
    icon={{ source: Icon.Trash, tintColor: Color.Red }}
    style={Action.Style.Destructive}
    shortcut={Keyboard.Shortcut.Common.Remove}
    onAction={async () => {
      const confirmed = await confirmAlert({
        title: "Delete Contact",
        message: `Permanently delete "${name}"?`,
        primaryAction: { title: "Delete", style: Alert.ActionStyle.Destructive },
      });
      if (confirmed) {
        deleteContact(id);
        await showToast(Toast.Style.Success, `Deleted "${name}"`);
        revalidate();
      }
    }}
  />
  ```
- **Revalidating after mutations.** Destructure `revalidate` from `useCachedPromise` and call it after any write operation (delete, update, create) to refresh the list without a full remount — or use `mutate` (section 5).

## 9. Background, Menu Bar, and Launch Entry Points

- **Live info in root search**: a `no-view` command with `"interval": "10m"` in its manifest entry runs on a schedule; inside it, `updateCommandMetadata({ subtitle: "3 unread" })` updates the text next to the command. Branch on `environment.launchType === LaunchType.Background` to skip UI work. Use intervals of 1m or more. Scheduling is approximate, and Raycast adds a per-command toggle for it.
- **Test a background command** without waiting: the dev actions **Run in Background** on the command in root search; the built-in **Extension Diagnostics** command lists last runs. Errors show as a warning icon on the command.
- **Menu-bar commands (`"mode": "menu-bar"`)** — *Trigger:* writing or fixing a `MenuBarExtra`. *Discrimination:* they are not long-lived apps; Raycast loads, renders, and unloads them, and after a Raycast restart it restores the last render WITHOUT running your code. *Action:* set `isLoading` true while fetching and ALWAYS flip it to false (in `finally`), or the command never unloads; read the last value from `Cache` synchronously for an instant first paint; return `null` to hide the item; never put two identical `MenuBarExtra.Item`s at one level (their handlers misfire). An item with no `onAction` renders disabled, so it works as a section label. `alternate` gives an ⌥-held variant.
- **Arguments** (up to 3 per command, `text` / `password` / `dropdown`) show as inline fields in root search; read them from `props.arguments`, typed as `LaunchProps<{ arguments: Arguments.MyCommand }>`. All values are strings.
- **Aliases are invisible to extensions, and argument placeholders are static.** *Trigger:* the user wants help text that names an alias, or any text in the argument box that changes at runtime. *Discrimination:* there is no API that reads aliases (they live in Raycast's encrypted DB), `placeholder` is fixed in `package.json`, and the argument box cuts text off at about 17 characters. Only the command `subtitle` can change at runtime, via `updateCommandMetadata`, which updates the CALLING command only and takes effect after it runs. *Action:* keep the placeholder short ("minutes"); put the reminder in the subtitle, fed by a textfield preference that holds the alias.
- **Fallback commands**: read `props.fallbackText` to prefill from whatever the user typed in root search.
- **Deeplinks**: `raycast://extensions/<author>/<extension-name>/<command-name>?arguments=<url-encoded JSON>&context=<url-encoded JSON>&launchType=background`. Get the exact URL from the command's **Copy Deeplink** action (⇧⌘C) rather than building it by hand; inside code use `createDeeplink({ command, arguments })` from `@raycast/utils`. Raycast asks the user to confirm before running a deeplinked command. A Keyboard Maestro macro or Shortcut can `open` this URL.

## 10. Production Mode & Publishing

- **The "Development" label** is inherent to all extensions loaded via `ray develop`. There is **no local production install** — the only way to remove it is `ray publish`, which submits the extension to the Raycast Store (requires `ray login`, public listing, and review).
- **`package.json` metadata semantics**: The `title` field in the top-level `package.json` is what Raycast shows as the extension subtitle next to the command name (e.g., "Search Contacts  **Search the Contacts app**"). The `commands[].title` is the command name itself. Change `title` when the user asks to rename what appears next to the command. A command-level `subtitle` overrides it per command and is also searchable; command-level `keywords` add root-search synonyms.

## 11. Debugging

- **`console.log` output goes to the terminal running `npm run dev`**, not to Raycast. If nothing appears there, turn on the dev option "Use file logging instead of OSLog".
- **Dev-only behavior:** gate it on `environment.isDevelopment`, not `NODE_ENV`.
- **Breakpoints / live props:** `npm i -D react-devtools@6.1.1`, re-run `npm run dev`, open the command, press ⌘⌥D.
- **"Command Out of Memory"**: choose "Reload with Memory Reporting"; `captureMemorySnapshot(label)` marks points to compare. Usually an unpaginated list or a huge cached blob.
- **Raycast's own logs**: the "Reveal Raycast Logs" command, or `~/Library/Logs/com.raycast.macos`.
- **Runtime facts:** every extension runs in Raycast's own bundled Node (not Homebrew's), with the same stripped PATH as script commands. Extensions are not sandboxed for files, network, or child processes, and macOS privacy grants belong to Raycast.

## 12. Extension Preferences & Authentication

When building an extension that requires an external API (like Notion or Spotify):
- Add the required secrets/tokens to the `preferences` array in `package.json` (`"type": "password"`, `"required": true` — Raycast then blocks the command with a setup form until it is filled). Read them with `getPreferenceValues<Preferences.MyCommand>()`.
- **Put the setup steps in `help.md`** next to `package.json`: Raycast renders it beside the required-preferences form, so the instructions are there when the user needs them. On an auth error, offer an `Action` that calls `openExtensionPreferences()`.
- **User Instructions**: You must still proactively remind the user to configure the token when they first load the extension. For example, provide a short snippet:

  > ⚠️ **Integration Required**
  > You need to provide this extension with an API Token.
  > 1. Go to [Link to API dashboard]
  > 2. Create a new token.
  > 3. Open Raycast, run this new command, and paste the Token in the preferences when prompted.

- **OAuth instead of a pasted token**: `OAuthService.github|linear|slack|asana({ scope })` work out of the box with Raycast's hosted apps; `google|jira|zoom` need your own client ID. Wrap the command with `withAccessToken(service)(Command)` and read `getAccessToken().token`. Pass `personalAccessToken: prefs.token` to let a pasted token skip OAuth. Don't save source files mid-login while `ray develop` runs — the hot reload breaks the flow.

## 13. Notion-backed extensions

The user keeps a collection of small extensions that wrap a Notion database behind a Raycast form (one isolated subfolder + `package.json` per extension). The token lives in a `notionToken` password preference; the form reads it via `getPreferenceValues` and builds `new Client({ auth })`.

- **Per-extension config when duplicating**: update `name`, `title`, `description`, `author` (the user's Raycast handle), and the `commands` array. Then set the `databaseId` const in the `.tsx` — extract the 32-char hex id from the database URL (`notion.so/.../<id>?v=...`).
- **Fetch options dynamically, in one call.** Pull every dynamic field (tags, status, …) from a single `databases.retrieve` inside one `useCachedPromise` — don't issue one retrieve per field. Find properties by `type` (and name) rather than a hardcoded key, so a renamed column still resolves; locate the title property with `Object.values(db.properties).find(p => p.type === "title")`.
- **multi_select with create-new.** `Form.TagPicker` only selects from items you provide — it has **no** freeform entry. To let the user *create* new options, pair the picker (existing options, with `value`/`title` set to the option **name**) with a companion `Form.TextField` for comma-separated new names. On submit, merge + dedupe both lists and send `multi_select: names.map(name => ({ name }))`. Notion matches existing options by name and auto-creates any that don't exist — sending by **name** (not id) is what makes create-on-write work.
- **single select / status.** Use `Form.Dropdown` with a leading `<Form.Dropdown.Item value="" title="None" />` so the field defaults to unset; skip writing the property when the value is empty. Handle both the `select` and `status` property types: `{ select: { name } }` vs `{ status: { name } }`.

### Auth & the "Could not find database" error

`APIResponseError: Could not find database with ID … make sure the relevant pages and databases are shared with your integration` almost always means **the integration is not connected to the database**, not that the ID is wrong — Notion returns the same `object_not_found` message for "doesn't exist" and "not shared."

- Confirm the DB exists and the ID is right by fetching it through the user's own account (Notion MCP `notion-fetch`, or the Notion app) — that bypasses the integration's permissions.
- The fix is manual (an integration cannot grant itself access): in Notion open the database → **⋯ → Connections → Add connections** → select the integration whose token is in Raycast.
- Verify a token before the user pastes it into Raycast:
  ```bash
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Notion-Version: 2022-06-28" \
    "https://api.notion.com/v1/databases/<id>"
  ```
  `200` = valid + shared; `404` = not shared / wrong id; `401` = bad token.
- The token is stored only in Raycast's encrypted preferences — you can't set it from the shell, and never hardcode it in source. (Note the rename caveat in section 3: a renamed extension needs the token re-entered.)

## 14. AI Extensions (tools Raycast AI can call)

*Trigger:* the user wants to ask Raycast AI to do something with one of their extensions ("@contacts find…"), or mentions tools, `ai.yaml`, or evals. *Discrimination:* a tool is NOT a command — it never shows in root search; only the AI calls it. A prompt run on selected text is an AI Command (no code). *Action:*
- Add a `tools` entry in `package.json` (`name`, `title`, `description`) → `src/tools/<name>.ts` default-exports an async function taking ONE input object. JSDoc on the function and each input field is what the AI reads — say formats (ISO 8601 dates) and where IDs come from.
- Guard writes: `export const confirmation: Tool.Confirmation<Input> = async (input) => ({ message: … })`. Return `undefined` to skip it for safe inputs.
- Domain rules go in `ai.yaml` (`instructions: |`) next to `package.json`. Don't write "You are an assistant" — several extensions share one chat.
- Evals: capture a working run with the **Copy Eval** action in AI Chat, paste into `ai.yaml` `evals:`, run `npx ray evals`.
- Needs Raycast Pro. Share code between a command and a tool with `environment.entryPointType` (`"command"` / `"tool"`).
- Raycast AI also reads Agent Skills from `~/.claude/skills` (top level). It silently skips a skill whose `description` is over 1024 characters or whose folder name differs from its `name`.

## 15. Opening Claude Desktop (Claude Code) in a folder (added 09-17-26)

*Trigger:* the user wants a Raycast command that opens Claude Code in a folder/repo ("Raycast hotkey to open this project in Claude"). *Discrimination:* it is a script command, not an extension — one `open` of the desktop app's deep link `claude://code/new?folder=<percent-encoded absolute path>` (verified 09-17-26, Claude.app 2.110.1). Don't shell out to the `claude` CLI (that is the terminal app) and don't `open -a Claude <folder>` (lands in Cowork). *Action:*

```bash
#!/bin/bash
# @raycast.schemaVersion 1
# @raycast.title Open in Claude Desktop
# @raycast.mode silent
# @raycast.packageName Claude
# @raycast.argument1 { "type": "text", "placeholder": "absolute folder path" }
V="$1"; [ -d "$V" ] || { echo "not a folder: $V"; exit 1; }
open "claude://code/new?folder=$(V="$V" osascript -l JavaScript -e 'ObjC.import("stdlib"); function run(){return encodeURIComponent($.getenv("V"))}')"
```

Absolute paths only (`~` is not expanded). A "current Finder selection" variant needs `tell application "Finder"` — that triggers a one-time Automation permission prompt, so prefer the argument form or a fixed list of favorites unless the user asks. Prefill the first message with `&q=<encoded text>`.
