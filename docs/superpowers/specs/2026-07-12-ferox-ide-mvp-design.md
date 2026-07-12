# ferox-ide MVP — Design

Date: 2026-07-12
Status: SUPERSEDED — 2026-07-12 decision: full VS Code extension ecosystem is a hard requirement, which requires the Node extension host. Chosen path: use VSCodium instead of building ferox-ide. This spec and its plan (docs/superpowers/plans/2026-07-12-ferox-core-editor.md) are retained for reference only.

## Goal

A lean, low-memory code editor for macOS with a 1:1 VS Code UI, built on
Tauri (Rust backend) + Monaco, with an embedded Claude Code terminal panel.
Target footprint: ~100–200MB total (vs Electron's 400+).

## Non-goals (MVP)

- Extensions/plugin system, LSP, debugging, multi-window, Windows/Linux,
  settings sync, deep Claude Agent SDK integration.

## Constraints & decisions

- **Platform**: macOS only. WKWebView (system webview) — GPU-accelerated by
  default (Core Animation/Metal); xterm uses the WebGL addon.
- **Process model**: one owned Rust process (window, fs, PTY, git, search).
  WKWebView's OS-managed helper processes are unavoidable on macOS; accepted.
- **No Node anywhere.** Claude Code runs as a PTY child (`claude` CLI).
- **UI**: replicate VS Code 1:1 — metrics, colors, icons, behaviors.
  Microsoft branding/name/logo excluded; all reused assets are MIT
  (Codicons, Monaco, xterm.js, theme format).

## Stack

- Frontend: React + TypeScript + Vite, Zustand (one store per domain:
  editor, git, terminal, settings), Tailwind, `@monaco-editor/react`,
  `@xterm/xterm` + WebGL addon, Codicons + Seti-style file icons.
- Backend: Rust/Tauri; crates: `portable-pty`, `git2`, `notify`, `tracing`;
  bundled `ripgrep` binary for search; system `git` for push/pull/fetch.

## UI (VS Code 1:1)

- **Activity bar** (48px): Explorer, Search toggles.
- **Side bar** (300px default, draggable): file tree (lazy-loaded,
  watcher-refreshed) or ripgrep results panel.
- **Editor area**: tab bar (35px, dirty dots, close hover behavior),
  breadcrumbs bar, Monaco editor; git-modified files can toggle a Monaco
  DiffEditor view.
- **Right panel** (toggle ⌘⇧C): Claude Code — xterm pane running `claude`
  in the project root.
- **Bottom panel** (toggle ⌃`): general shell terminal(s), xterm + WebGL.
- **Status bar** (22px): branch + ahead/behind, dirty count, cursor position.
- **Quick pick**: basic ⌘P (files) and ⌘⇧P (commands) palette.
- **Theming**: VS Code theme JSON → CSS variables + Monaco `tokenColors`.
  Dark+ and Light+ shipped; any VS Code theme file drops in.

## Rust command surface (Tauri IPC)

All commands return `Result<T, String>`. Streams use Tauri events keyed by
session id. No state in Rust beyond live handles (PTYs, watcher).

- **fs**: `read_dir`, `read_file`, `write_file`, `create_entry`,
  `rename_entry`, `delete_entry`; `notify` watcher → `fs:changed` events
  (tree refresh, changed-on-disk tab handling).
- **pty**: `pty_spawn(kind: shell|claude, cwd)`, `pty_write`, `pty_resize`,
  `pty_kill`; output streamed per session. Never tie process lifetime to
  stdin.
- **git** (`git2`): `status`, `diff_file` (original/modified pair for
  DiffEditor), `stage`, `unstage`, `commit`, `branch_info` (name,
  ahead/behind). Push/pull/fetch shell out to system `git` (user's existing
  credential/SSH setup).
- **search**: `search(query, opts)` spawns ripgrep `--json`, streams parsed
  hits; in-file find/replace is pure Monaco.

## Data flow

UI invokes command → Rust returns result or streams events → Zustand store
updates → React renders. Uniform for all domains.

## Error handling

- Command failures → VS Code-style toast notifications (bottom-right),
  message verbatim; never silent.
- Save conflict (file changed on disk) → Overwrite / Discard / Compare
  dialog.
- PTY exit → banner in terminal with restart action.
- ripgrep "no matches" is not an error.
- Backend logs via `tracing` to a log file.

## Testing

- Rust unit tests: git layer (temp repos), ripgrep JSON parsing, fs
  commands.
- Vitest: store logic (tab lifecycle, dirty tracking, git status mapping).
- Manual smoke before milestones: open folder → edit/save → diff → stage +
  commit → `claude` panel → project search. Verified with screenshots.

## Alternatives considered

- **Electron / VS Code fork / VSCodium**: memory-heavy; workbench welded to
  Chromium + Node. Rejected.
- **Electrobun**: Bun backend re-adds a JS runtime (+50–100MB), immature.
  Rejected.
- **GPUI native rewrite**: best memory, loses Monaco (months rebuilding the
  editor). Rejected for this product; revisit only if footprint targets
  tighten below what a webview allows.
- **Svelte/Solid/vanilla frontend**: savings are noise next to Monaco;
  React has the best-trodden integrations. Rejected.
