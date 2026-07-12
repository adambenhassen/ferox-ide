# ferox-ide Plan 1: Scaffold + Core Editor

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A working Tauri app on macOS: open a folder, browse a VS Code-style file tree, edit files in Monaco tabs, save. Follow-up plans add terminals/PTY, git, search.

**Architecture:** One Rust/Tauri process exposes thin fs commands; React + Zustand UI in WKWebView renders a VS Code 1:1 shell (activity bar 48px, sidebar 300px, tabs 35px, status bar 22px) with Monaco as the editor.

**Tech Stack:** Tauri 2, Rust, React 18 + TypeScript + Vite, Zustand, Tailwind, `@monaco-editor/react`, `@vscode/codicons`.

## Global Constraints

- macOS only; WKWebView system webview.
- No Node in the shipped app (Node is dev-time only for Vite).
- All Tauri commands return `Result<T, String>`; UI surfaces errors as toasts, never silently.
- VS Code Dark+ colors via CSS variables (`--vscode-*` names matching VS Code theme keys).
- Spec: `docs/superpowers/specs/2026-07-12-ferox-ide-mvp-design.md`.

---

### Task 1: Scaffold Tauri + React + Vite project

**Files:**
- Create: entire scaffold via `create-tauri-app` at repo root (`src/`, `src-tauri/`, `vite.config.ts`, `package.json`, `src-tauri/tauri.conf.json`, `src-tauri/Cargo.toml`)

**Interfaces:**
- Produces: running dev app (`npm run tauri dev`), repo initialized in git.

- [ ] **Step 1: Initialize git and scaffold**

```bash
cd /Users/adam/repos/ferox-ide
git init
npm create tauri-app@latest . -- --template react-ts --manager npm --yes
npm install
npm install zustand tailwindcss @tailwindcss/vite @monaco-editor/react monaco-editor @vscode/codicons
```

- [ ] **Step 2: Configure Tailwind v4 in Vite**

In `vite.config.ts` add:

```ts
import tailwindcss from "@tailwindcss/vite";
// plugins: [react(), tailwindcss()]
```

Create `src/index.css` with:

```css
@import "tailwindcss";
@import "@vscode/codicons/dist/codicon.css";
```

- [ ] **Step 3: Verify dev app launches**

Run: `npm run tauri dev`
Expected: window opens showing the template page; no console errors.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "chore: scaffold tauri + react + vite app"
```

### Task 2: Rust fs commands

**Files:**
- Create: `src-tauri/src/fs_cmds.rs`
- Modify: `src-tauri/src/lib.rs` (register commands)
- Test: inline `#[cfg(test)]` in `fs_cmds.rs`

**Interfaces:**
- Produces (Tauri commands):
  - `read_dir(path: String) -> Result<Vec<DirEntry>, String>` where `DirEntry { name: String, path: String, is_dir: bool }`, sorted dirs-first then alphabetical (case-insensitive)
  - `read_file(path: String) -> Result<String, String>`
  - `write_file(path: String, content: String) -> Result<(), String>`

- [ ] **Step 1: Write failing tests**

```rust
// src-tauri/src/fs_cmds.rs (tests at bottom)
#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn read_dir_sorts_dirs_first() {
        let tmp = tempfile::tempdir().unwrap();
        std::fs::create_dir(tmp.path().join("zdir")).unwrap();
        std::fs::write(tmp.path().join("afile.txt"), "x").unwrap();
        let entries = read_dir(tmp.path().to_string_lossy().into()).unwrap();
        assert_eq!(entries[0].name, "zdir");
        assert!(entries[0].is_dir);
        assert_eq!(entries[1].name, "afile.txt");
    }
    #[test]
    fn write_then_read_roundtrip() {
        let tmp = tempfile::tempdir().unwrap();
        let p = tmp.path().join("f.txt").to_string_lossy().to_string();
        write_file(p.clone(), "hello".into()).unwrap();
        assert_eq!(read_file(p).unwrap(), "hello");
    }
    #[test]
    fn read_missing_file_is_err() {
        assert!(read_file("/nonexistent/x".into()).is_err());
    }
}
```

Add to `src-tauri/Cargo.toml`: `tempfile = "3"` under `[dev-dependencies]`.

- [ ] **Step 2: Run tests, verify fail**

Run: `cargo test --manifest-path src-tauri/Cargo.toml`
Expected: compile error — functions not defined.

- [ ] **Step 3: Implement**

```rust
// src-tauri/src/fs_cmds.rs
use serde::Serialize;

#[derive(Serialize)]
pub struct DirEntry {
    pub name: String,
    pub path: String,
    pub is_dir: bool,
}

#[tauri::command]
pub fn read_dir(path: String) -> Result<Vec<DirEntry>, String> {
    let mut entries: Vec<DirEntry> = std::fs::read_dir(&path)
        .map_err(|e| e.to_string())?
        .filter_map(|e| e.ok())
        .map(|e| DirEntry {
            name: e.file_name().to_string_lossy().into_owned(),
            path: e.path().to_string_lossy().into_owned(),
            is_dir: e.file_type().map(|t| t.is_dir()).unwrap_or(false),
        })
        .collect();
    entries.sort_by(|a, b| {
        b.is_dir
            .cmp(&a.is_dir)
            .then(a.name.to_lowercase().cmp(&b.name.to_lowercase()))
    });
    Ok(entries)
}

#[tauri::command]
pub fn read_file(path: String) -> Result<String, String> {
    std::fs::read_to_string(&path).map_err(|e| format!("{path}: {e}"))
}

#[tauri::command]
pub fn write_file(path: String, content: String) -> Result<(), String> {
    std::fs::write(&path, content).map_err(|e| format!("{path}: {e}"))
}
```

In `src-tauri/src/lib.rs`, add `mod fs_cmds;` and register:

```rust
.invoke_handler(tauri::generate_handler![
    fs_cmds::read_dir,
    fs_cmds::read_file,
    fs_cmds::write_file
])
```

Add `tauri-plugin-dialog = "2"` to `Cargo.toml` dependencies and `npm install @tauri-apps/plugin-dialog`; register `.plugin(tauri_plugin_dialog::init())` in `lib.rs` (used by Task 4's Open Folder).

- [ ] **Step 4: Run tests, verify pass**

Run: `cargo test --manifest-path src-tauri/Cargo.toml`
Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: fs commands (read_dir, read_file, write_file)"
```

### Task 3: VS Code shell chrome + Dark+ theme variables

**Files:**
- Create: `src/theme/darkplus.css`, `src/components/Shell.tsx`, `src/components/ActivityBar.tsx`, `src/components/StatusBar.tsx`
- Modify: `src/App.tsx`, `src/index.css`

**Interfaces:**
- Produces: `<Shell sidebar={node} editor={node} />` layout component; CSS variables `--vscode-*` available app-wide.

- [ ] **Step 1: Create Dark+ variables**

```css
/* src/theme/darkplus.css — values from VS Code Dark+ (dark_plus.json) */
:root {
  --vscode-editor-background: #1e1e1e;
  --vscode-editor-foreground: #d4d4d4;
  --vscode-sideBar-background: #252526;
  --vscode-sideBar-foreground: #cccccc;
  --vscode-activityBar-background: #333333;
  --vscode-activityBar-foreground: #ffffff;
  --vscode-activityBar-inactiveForeground: #ffffff66;
  --vscode-statusBar-background: #007acc;
  --vscode-statusBar-foreground: #ffffff;
  --vscode-tab-activeBackground: #1e1e1e;
  --vscode-tab-inactiveBackground: #2d2d2d;
  --vscode-tab-activeForeground: #ffffff;
  --vscode-tab-inactiveForeground: #ffffff80;
  --vscode-editorGroupHeader-tabsBackground: #252526;
  --vscode-list-hoverBackground: #2a2d2e;
  --vscode-list-activeSelectionBackground: #094771;
  --vscode-focusBorder: #007fd4;
  --vscode-notifications-background: #252526;
  --vscode-widget-border: #454545;
  --vscode-font-family: -apple-system, "SF Pro Text", "Segoe WPC", sans-serif;
  --vscode-editor-font-family: "SF Mono", Menlo, Monaco, monospace;
}
body {
  font-family: var(--vscode-font-family);
  font-size: 13px;
  color: var(--vscode-editor-foreground);
  background: var(--vscode-editor-background);
  overflow: hidden;
  user-select: none;
}
```

Import in `src/index.css`: `@import "./theme/darkplus.css";`

- [ ] **Step 2: Shell layout**

```tsx
// src/components/Shell.tsx
import { ReactNode } from "react";
import { ActivityBar } from "./ActivityBar";
import { StatusBar } from "./StatusBar";

export function Shell({ sidebar, editor }: { sidebar: ReactNode; editor: ReactNode }) {
  return (
    <div className="flex h-screen flex-col">
      <div className="flex min-h-0 flex-1">
        <ActivityBar />
        <div className="w-[300px] shrink-0 overflow-auto" style={{ background: "var(--vscode-sideBar-background)" }}>
          {sidebar}
        </div>
        <div className="min-w-0 flex-1">{editor}</div>
      </div>
      <StatusBar />
    </div>
  );
}
```

```tsx
// src/components/ActivityBar.tsx
export function ActivityBar() {
  return (
    <div className="flex w-12 shrink-0 flex-col items-center pt-2" style={{ background: "var(--vscode-activityBar-background)" }}>
      <span className="codicon codicon-files" style={{ fontSize: 24, color: "var(--vscode-activityBar-foreground)", padding: 10 }} />
      <span className="codicon codicon-search" style={{ fontSize: 24, color: "var(--vscode-activityBar-inactiveForeground)", padding: 10 }} />
    </div>
  );
}
```

```tsx
// src/components/StatusBar.tsx
import { useEditorStore } from "../stores/editorStore";

export function StatusBar() {
  const { cursorLine, cursorCol } = useEditorStore();
  return (
    <div className="flex h-[22px] shrink-0 items-center justify-end px-2 text-xs" style={{ background: "var(--vscode-statusBar-background)", color: "var(--vscode-statusBar-foreground)" }}>
      <span>Ln {cursorLine}, Col {cursorCol}</span>
    </div>
  );
}
```

Replace `src/App.tsx` body with `<Shell sidebar={<FileTree />} editor={<EditorArea />} />` (components arrive in Tasks 4–5; until then use placeholder `<div />`s so the app compiles).

- [ ] **Step 3: Verify visually**

Run: `npm run tauri dev`. Expected: dark shell with activity bar (48px), empty sidebar (300px), blue status bar (22px). Screenshot and compare against VS Code.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: VS Code shell chrome with Dark+ theme variables"
```

### Task 4: editorStore + file tree with Open Folder

**Files:**
- Create: `src/stores/editorStore.ts`, `src/components/FileTree.tsx`
- Test: `src/stores/editorStore.test.ts` (Vitest; `npm install -D vitest`)

**Interfaces:**
- Consumes: Tauri `read_dir`, `read_file` (Task 2); dialog plugin.
- Produces (Zustand store `useEditorStore`):
  - `projectRoot: string | null`, `openFolder(path: string): void`
  - `tabs: Tab[]` where `Tab { path: string; name: string; content: string; savedContent: string }` (dirty = `content !== savedContent`)
  - `activeTabPath: string | null`
  - `openFile(path: string): Promise<void>` (invokes `read_file`, activates existing tab if open)
  - `updateContent(path: string, content: string): void`
  - `closeTab(path: string): void`
  - `markSaved(path: string): void`
  - `cursorLine: number`, `cursorCol: number`, `setCursor(l: number, c: number): void`

- [ ] **Step 1: Write failing store tests**

```ts
// src/stores/editorStore.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
vi.mock("@tauri-apps/api/core", () => ({ invoke: vi.fn(async () => "file body") }));
import { useEditorStore } from "./editorStore";

beforeEach(() => useEditorStore.setState({ tabs: [], activeTabPath: null }));

describe("editorStore", () => {
  it("openFile adds a tab and activates it", async () => {
    await useEditorStore.getState().openFile("/p/a.ts");
    const s = useEditorStore.getState();
    expect(s.tabs).toHaveLength(1);
    expect(s.activeTabPath).toBe("/p/a.ts");
    expect(s.tabs[0].content).toBe("file body");
  });
  it("openFile twice does not duplicate the tab", async () => {
    await useEditorStore.getState().openFile("/p/a.ts");
    await useEditorStore.getState().openFile("/p/a.ts");
    expect(useEditorStore.getState().tabs).toHaveLength(1);
  });
  it("updateContent marks dirty; markSaved clears it", async () => {
    await useEditorStore.getState().openFile("/p/a.ts");
    useEditorStore.getState().updateContent("/p/a.ts", "changed");
    let t = useEditorStore.getState().tabs[0];
    expect(t.content !== t.savedContent).toBe(true);
    useEditorStore.getState().markSaved("/p/a.ts");
    t = useEditorStore.getState().tabs[0];
    expect(t.content === t.savedContent).toBe(true);
  });
  it("closeTab activates the neighbor", async () => {
    await useEditorStore.getState().openFile("/p/a.ts");
    await useEditorStore.getState().openFile("/p/b.ts");
    useEditorStore.getState().closeTab("/p/b.ts");
    expect(useEditorStore.getState().activeTabPath).toBe("/p/a.ts");
  });
});
```

- [ ] **Step 2: Run, verify fail** — `npx vitest run` → module not found.

- [ ] **Step 3: Implement store**

```ts
// src/stores/editorStore.ts
import { create } from "zustand";
import { invoke } from "@tauri-apps/api/core";

export interface Tab { path: string; name: string; content: string; savedContent: string }

interface EditorState {
  projectRoot: string | null;
  tabs: Tab[];
  activeTabPath: string | null;
  cursorLine: number;
  cursorCol: number;
  openFolder: (path: string) => void;
  openFile: (path: string) => Promise<void>;
  updateContent: (path: string, content: string) => void;
  closeTab: (path: string) => void;
  markSaved: (path: string) => void;
  setCursor: (l: number, c: number) => void;
}

export const useEditorStore = create<EditorState>((set, get) => ({
  projectRoot: null,
  tabs: [],
  activeTabPath: null,
  cursorLine: 1,
  cursorCol: 1,
  openFolder: (path) => set({ projectRoot: path, tabs: [], activeTabPath: null }),
  openFile: async (path) => {
    if (get().tabs.some((t) => t.path === path)) return set({ activeTabPath: path });
    const content = await invoke<string>("read_file", { path });
    const name = path.split("/").pop() ?? path;
    set((s) => ({ tabs: [...s.tabs, { path, name, content, savedContent: content }], activeTabPath: path }));
  },
  updateContent: (path, content) =>
    set((s) => ({ tabs: s.tabs.map((t) => (t.path === path ? { ...t, content } : t)) })),
  closeTab: (path) =>
    set((s) => {
      const idx = s.tabs.findIndex((t) => t.path === path);
      const tabs = s.tabs.filter((t) => t.path !== path);
      const active = s.activeTabPath === path ? tabs[Math.max(0, idx - 1)]?.path ?? null : s.activeTabPath;
      return { tabs, activeTabPath: active };
    }),
  markSaved: (path) =>
    set((s) => ({ tabs: s.tabs.map((t) => (t.path === path ? { ...t, savedContent: t.content } : t)) })),
  setCursor: (l, c) => set({ cursorLine: l, cursorCol: c }),
}));
```

- [ ] **Step 4: Run, verify pass** — `npx vitest run` → 4 passed.

- [ ] **Step 5: FileTree component**

```tsx
// src/components/FileTree.tsx
import { useEffect, useState } from "react";
import { invoke } from "@tauri-apps/api/core";
import { open } from "@tauri-apps/plugin-dialog";
import { useEditorStore } from "../stores/editorStore";

interface Entry { name: string; path: string; is_dir: boolean }

function Node({ entry, depth }: { entry: Entry; depth: number }) {
  const [expanded, setExpanded] = useState(false);
  const [children, setChildren] = useState<Entry[]>([]);
  const openFile = useEditorStore((s) => s.openFile);
  const active = useEditorStore((s) => s.activeTabPath) === entry.path;

  const onClick = async () => {
    if (entry.is_dir) {
      if (!expanded) setChildren(await invoke<Entry[]>("read_dir", { path: entry.path }));
      setExpanded(!expanded);
    } else openFile(entry.path);
  };

  return (
    <>
      <div
        className="flex h-[22px] cursor-pointer items-center gap-1 hover:bg-[var(--vscode-list-hoverBackground)]"
        style={{ paddingLeft: depth * 8 + 4, background: active ? "var(--vscode-list-activeSelectionBackground)" : undefined }}
        onClick={onClick}
      >
        <span className={`codicon codicon-${entry.is_dir ? (expanded ? "chevron-down" : "chevron-right") : "file"}`} style={{ fontSize: 16 }} />
        <span className="truncate">{entry.name}</span>
      </div>
      {expanded && children.map((c) => <Node key={c.path} entry={c} depth={depth + 1} />)}
    </>
  );
}

export function FileTree() {
  const { projectRoot, openFolder } = useEditorStore();
  const [roots, setRoots] = useState<Entry[]>([]);

  useEffect(() => {
    if (projectRoot) invoke<Entry[]>("read_dir", { path: projectRoot }).then(setRoots);
  }, [projectRoot]);

  if (!projectRoot)
    return (
      <button
        className="m-4 rounded bg-[var(--vscode-statusBar-background)] px-3 py-1 text-white"
        onClick={async () => {
          const dir = await open({ directory: true });
          if (typeof dir === "string") openFolder(dir);
        }}
      >
        Open Folder
      </button>
    );
  return <div className="py-1">{roots.map((e) => <Node key={e.path} entry={e} depth={0} />)}</div>;
}
```

- [ ] **Step 6: Verify in app** — `npm run tauri dev`, open a folder, expand dirs. Expected: VS Code-like tree, dirs first.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: editor store and file tree with open-folder"
```

### Task 5: Tabs + Monaco editor + save

**Files:**
- Create: `src/components/EditorArea.tsx`, `src/components/TabBar.tsx`, `src/lang.ts`
- Modify: `src/App.tsx` (wire real components)
- Test: `src/lang.test.ts`

**Interfaces:**
- Consumes: `useEditorStore` (Task 4), `write_file` (Task 2).
- Produces: `detectLang(path: string): string` mapping extension → Monaco language id; complete editor UI.

- [ ] **Step 1: Failing test for detectLang**

```ts
// src/lang.test.ts
import { describe, it, expect } from "vitest";
import { detectLang } from "./lang";

describe("detectLang", () => {
  it.each([
    ["a.ts", "typescript"], ["a.tsx", "typescript"], ["a.rs", "rust"],
    ["a.py", "python"], ["a.json", "json"], ["a.md", "markdown"],
    ["a.css", "css"], ["a.html", "html"], ["a.js", "javascript"],
    ["Makefile", "plaintext"],
  ])("%s -> %s", (file, lang) => expect(detectLang(file)).toBe(lang));
});
```

- [ ] **Step 2: Run, verify fail** — `npx vitest run` → module not found.

- [ ] **Step 3: Implement**

```ts
// src/lang.ts
const MAP: Record<string, string> = {
  ts: "typescript", tsx: "typescript", js: "javascript", jsx: "javascript",
  rs: "rust", py: "python", json: "json", md: "markdown", css: "css",
  html: "html", toml: "ini", yml: "yaml", yaml: "yaml", sh: "shell",
};
export function detectLang(path: string): string {
  const ext = path.includes(".") ? path.split(".").pop()!.toLowerCase() : "";
  return MAP[ext] ?? "plaintext";
}
```

- [ ] **Step 4: Run, verify pass** — `npx vitest run` → all pass.

- [ ] **Step 5: TabBar + EditorArea**

```tsx
// src/components/TabBar.tsx
import { useEditorStore } from "../stores/editorStore";

export function TabBar() {
  const { tabs, activeTabPath, closeTab } = useEditorStore();
  const setActive = (path: string) => useEditorStore.setState({ activeTabPath: path });
  return (
    <div className="flex h-[35px] shrink-0 overflow-x-auto" style={{ background: "var(--vscode-editorGroupHeader-tabsBackground)" }}>
      {tabs.map((t) => {
        const active = t.path === activeTabPath;
        const dirty = t.content !== t.savedContent;
        return (
          <div
            key={t.path}
            className="group flex cursor-pointer items-center gap-1.5 border-r border-black/30 px-3 text-[13px]"
            style={{
              background: active ? "var(--vscode-tab-activeBackground)" : "var(--vscode-tab-inactiveBackground)",
              color: active ? "var(--vscode-tab-activeForeground)" : "var(--vscode-tab-inactiveForeground)",
            }}
            onClick={() => setActive(t.path)}
          >
            <span>{t.name}</span>
            <span
              className={`codicon ${dirty ? "codicon-circle-filled" : "codicon-close opacity-0 group-hover:opacity-100"}`}
              style={{ fontSize: 14 }}
              onClick={(e) => { e.stopPropagation(); closeTab(t.path); }}
            />
          </div>
        );
      })}
    </div>
  );
}
```

```tsx
// src/components/EditorArea.tsx
import { useEffect } from "react";
import MonacoEditor from "@monaco-editor/react";
import { invoke } from "@tauri-apps/api/core";
import { useEditorStore } from "../stores/editorStore";
import { detectLang } from "../lang";
import { TabBar } from "./TabBar";

export function EditorArea() {
  const { tabs, activeTabPath, updateContent, markSaved, setCursor } = useEditorStore();
  const tab = tabs.find((t) => t.path === activeTabPath);

  useEffect(() => {
    const onKey = async (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === "s") {
        e.preventDefault();
        const t = useEditorStore.getState().tabs.find((x) => x.path === useEditorStore.getState().activeTabPath);
        if (!t) return;
        try {
          await invoke("write_file", { path: t.path, content: t.content });
          markSaved(t.path);
        } catch (err) {
          alert(String(err)); // toast component replaces this in a later plan
        }
      }
    };
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [markSaved]);

  return (
    <div className="flex h-full flex-col">
      <TabBar />
      {tab ? (
        <MonacoEditor
          key={tab.path}
          height="100%"
          theme="vs-dark"
          language={detectLang(tab.path)}
          value={tab.content}
          onChange={(v) => updateContent(tab.path, v ?? "")}
          onMount={(editor) =>
            editor.onDidChangeCursorPosition((e) => setCursor(e.position.lineNumber, e.position.column))
          }
          options={{ fontSize: 13, fontFamily: "SF Mono, Menlo, monospace", minimap: { enabled: true }, automaticLayout: true }}
        />
      ) : (
        <div className="flex flex-1 items-center justify-center opacity-40">
          <span className="codicon codicon-files" style={{ fontSize: 64 }} />
        </div>
      )}
    </div>
  );
}
```

Wire `src/App.tsx`:

```tsx
import { Shell } from "./components/Shell";
import { FileTree } from "./components/FileTree";
import { EditorArea } from "./components/EditorArea";
import "./index.css";

export default function App() {
  return <Shell sidebar={<FileTree />} editor={<EditorArea />} />;
}
```

- [ ] **Step 6: Full smoke test**

Run: `npm run tauri dev`. Open folder → open two files → edit one (dirty dot appears) → ⌘S (dot clears; file changed on disk) → close tab (neighbor activates) → status bar shows Ln/Col. Screenshot against VS Code for 1:1 check.

- [ ] **Step 7: Run all tests**

Run: `npx vitest run && cargo test --manifest-path src-tauri/Cargo.toml`
Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: tabs, monaco editor, save"
```

---

## Follow-up plans (separate documents)

1. Terminals: PTY backend (`portable-pty`), xterm.js + WebGL, general terminal panel + Claude Code panel.
2. Git: `git2` status/diff/stage/commit, DiffEditor view, status-bar branch item.
3. Search: bundled ripgrep, results panel, quick-pick (⌘P / ⌘⇧P).
4. Polish: toasts, breadcrumbs, save-conflict dialog, fs watcher, theme loader.
