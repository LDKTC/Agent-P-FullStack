---
name: electron
description: Electron-specific expertise — the main/renderer/preload process model and which code belongs in each, the non-negotiable security baseline (`contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`, a minimal `contextBridge` surface, CSP, `setWindowOpenHandler`/`will-navigate` guards, validated `shell.openExternal`), IPC done as `ipcRenderer.invoke`/`ipcMain.handle` with per-channel validated arguments instead of a generic "run this" bridge, keeping the main process off blocking work, correct userData/file paths, and packaging concerns (asar, native module rebuilds, code signing, auto-update). Use when `package.json` lists `electron`, `electron-builder`, `electron-forge`, or `electron-vite`, when a `main`/`preload` entry or `BrowserWindow` appears in the repo, or when asked directly (Thai or English) to สร้าง/แก้ไข desktop app ด้วย Electron. Not a replacement for frontend-dev's UI work or react's component rules — the renderer is still a web app and those still apply; this skill covers what is specific to running it as a desktop process with OS access.
---

# Electron

A desktop shell around a Chromium renderer and a Node main process. Inside
the renderer, everything in `skills/html-css/SKILL.md`,
`skills/react/SKILL.md`, and `skills/tailwind/SKILL.md` still applies —
it is a web UI. The main process is Node, so `skills/nodejs/SKILL.md`
applies there. What this skill adds is the part that has no web equivalent:
a process boundary that is also a **trust boundary**, because the renderer
runs remote-influenced content with an OS-privileged process on the other
side of it.

## Step 1 — Confirm Electron and find the three entry points

Check `package.json` for `electron`, `electron-builder`, `electron-forge`,
or `electron-vite`, and locate the main entry, the preload script(s), and
the renderer root. Read the existing `BrowserWindow` options before adding a
window — this is where the security posture of the whole app is set, and
copying a window's options forward is how an insecure setting spreads. Note
the Electron major version; the security defaults changed across versions
and older tutorials assume the unsafe ones.

## Step 2 — What belongs in which process

- **Main**: window lifecycle, menus, tray, dialogs, filesystem, child
  processes, auto-update, anything touching the OS. One instance, and it
  owns all privilege.
- **Preload**: the only bridge. Runs before the page loads with access to a
  limited Node surface, and exposes a **specific, named** API to the
  renderer via `contextBridge`.
- **Renderer**: the UI. Treat it as untrusted — it renders content that may
  include remote data, and any XSS there becomes a request across the
  bridge.
- Business logic that needs OS access lives in main and is invoked through a
  named IPC channel; don't move it to the renderer by widening the bridge.

## Step 3 — The security baseline (non-negotiable)

Every `BrowserWindow` gets:

```js
webPreferences: {
  contextIsolation: true,   // default since Electron 12 — never turn off
  nodeIntegration: false,   // default since Electron 5 — never turn on
  sandbox: true,
  preload: path.join(__dirname, 'preload.js'),
}
```

- `nodeIntegration: true` with any remote content is remote code execution.
  `contextIsolation: false` lets page script reach the preload's scope and
  prototype-pollute the bridge. If either appears in the repo, that is a
  finding for `security-auditor`, not a style preference.
- `webSecurity: false`, `allowRunningInsecureContent`, and disabled
  certificate-error handling are the same category — never used to "fix"
  CORS or a self-signed certificate in development, because they ship.
- Set a Content Security Policy on the loaded document (no
  `unsafe-eval`/`unsafe-inline`), and don't load application code over
  plain `http:` in production.
- Only load content you control. `loadURL` of a third-party site inside your
  privileged app puts their JavaScript one bridge away from your handlers;
  use the system browser instead.
- `setWindowOpenHandler` should deny by default and pass vetted URLs to
  `shell.openExternal`; add a `will-navigate` handler that blocks navigation
  away from your own origin.
- `shell.openExternal` with an unvalidated string is a command-execution
  primitive (`file:`, and OS-specific schemes). Allowlist `https:` (and
  `mailto:` if needed) and reject everything else.
- Never enable `webviewTag` or `enableRemoteModule` (removed in modern
  versions) unless there's no alternative, and never accept `webPreferences`
  from page content.
- Run Electronegativity (or the equivalent checklist) before shipping, and
  keep the Electron version current — an out-of-date Electron pins an
  out-of-date Chromium with published CVEs.

## Step 4 — IPC design

- Use `ipcRenderer.invoke` / `ipcMain.handle` for request-response, and
  `webContents.send` / `ipcRenderer.on` for main→renderer events. Avoid the
  fire-and-forget `send`/`on` pair for anything that needs a result.
- Expose **narrow, purpose-named** functions across the bridge —
  `saveInvoice(data)`, not `invoke(channel, …args)` or `readFile(path)`. A
  generic passthrough hands the renderer everything the main process can do
  and makes the bridge pointless.
- Validate every argument in the main process handler: type, shape, and
  range. Treat IPC input exactly like an HTTP request body — same schema
  validation, same reasoning as `skills/typescript/SKILL.md`'s boundary rule.
- Any path from the renderer must be resolved and confirmed to sit under an
  allowed root; a bridged "read this file" is a full filesystem read
  otherwise.
- Check `event.senderFrame`/the sender's URL in handlers when multiple
  windows or frames exist with different privileges.
- Don't expose `ipcRenderer` itself, `require`, or any Node module on
  `window`.
- Keep secrets (API keys, tokens) in the main process, and never in renderer
  code or `localStorage` — a packaged app is an unpacked archive to anyone
  who wants to look. Anything that must remain secret belongs on a server,
  not in the app.

## Step 5 — Responsiveness and lifecycle

- The main process is single-threaded and blocking it freezes **every**
  window, including their animations and input. No sync filesystem calls, no
  CPU-heavy loops — same rules as `skills/nodejs/SKILL.md`, with a more
  visible symptom.
- Create windows with `show: false` and show them on the `ready-to-show`
  event to avoid a white flash; restore window bounds from persisted state
  rather than always centering.
- Handle `window-all-closed` and `activate` per platform (macOS apps stay
  alive with no windows), and use `app.requestSingleInstanceLock()` for a
  single-instance app.
- Write user data under `app.getPath('userData')`, never next to the
  executable — the install directory is read-only for a normal user on
  Windows and inside a signed bundle on macOS.
- Guard `will-quit`/`before-quit` to flush pending writes; a database or
  settings file written on quit needs the async work to actually complete.

## Step 6 — Build, packaging, and updates

- Pick one toolchain (electron-builder or Forge) and keep the config in the
  repo. Renderer bundling is a normal frontend build; the main/preload
  bundle has different externals and must not be tree-shaken as browser
  code.
- `asar: true` packages sources into one archive — it is not encryption and
  not obfuscation; it only stops casual browsing.
- Native modules must be built against Electron's ABI
  (`electron-rebuild`/the toolchain's hook), not the system Node. "Works in
  dev, crashes in the packaged app" is usually this.
- Code sign and notarize (macOS) / sign (Windows) — an unsigned app is
  blocked or warned about, and auto-update mostly requires signing.
- Auto-update must fetch over HTTPS from a source you control with signature
  verification (`electron-updater` or Squirrel). An unsigned or
  unauthenticated update channel is a remote code execution channel into
  every user's machine.
- Don't ship devtools-open, verbose logging of user data, or development
  URLs in the production build; gate them on the packaged flag.

## Step 7 — Self-check before reporting done

Confirm every `BrowserWindow` sets `contextIsolation: true`,
`nodeIntegration: false`, `sandbox: true`, and a preload, and that no
`webSecurity: false` or remote-content load slipped in. Confirm the
contextBridge surface is a list of named operations with no generic invoke,
and every `ipcMain.handle` validates its arguments and any path. Confirm
`setWindowOpenHandler`/`will-navigate` deny by default and `shell.
openExternal` is allowlisted. Confirm nothing synchronous or CPU-bound runs
in the main process during interaction, user data goes to
`app.getPath('userData')`, and native modules are rebuilt for Electron.
Then actually launch the packaged (not just the dev) app and drive one
real user task per `skills/dev-testing/SKILL.md`, treating an uncaught
renderer console error or a main-process exception as a failure.

## What not to do

- Don't set `nodeIntegration: true`, `contextIsolation: false`, `sandbox:
  false`, or `webSecurity: false` — not even "temporarily" in development.
- Don't expose `ipcRenderer`, `require`, `fs`, or a generic `invoke` through
  `contextBridge`.
- Don't trust IPC arguments, and don't accept a filesystem path from the
  renderer without resolving and bounding it.
- Don't load third-party web content in a privileged window, and don't pass
  an unvalidated URL to `shell.openExternal`.
- Don't embed API keys or secrets in the app — a packaged build is readable.
- Don't block the main process; every window freezes with it.
- Don't write files next to the executable instead of `userData`.
- Don't ship an unsigned build or an update channel without signature
  verification.
- Don't treat `asar` as protection, and don't leave the Electron version
  behind on an old Chromium.
- Don't restate renderer-side UI rules here — `frontend-dev`,
  `skills/react/SKILL.md`, and `skills/html-css/SKILL.md` own those.
