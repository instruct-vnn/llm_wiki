# LLM Wiki Phase 1: Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a working Tauri v2 desktop app with file tree navigation, wiki page browsing, and a Milkdown WYSIWYG editor — the skeleton that all subsequent phases build on.

**Architecture:** Tauri v2 desktop shell with React frontend and Rust backend. Rust handles all file system operations exposed via Tauri IPC commands. React renders a VS Code-style layout: icon sidebar, file tree, content area, and a collapsible Chat bar placeholder.

**Tech Stack:** Tauri v2, React 19, TypeScript, Vite, Tailwind CSS v4, shadcn/ui, Milkdown, Zustand

---

## Phase Overview

All 6 phases for reference:

| Phase | Scope | Status |
|-------|-------|--------|
| **Phase 1: Foundation** | Tauri scaffold, file system ops, layout shell, wiki browser, Milkdown editor | **Current** |
| Phase 2: LLM Integration | LLM Router, Chat UI, basic ingest (single source) | Planned |
| Phase 3: Search & Query | tantivy search, query pipeline, multi-format rendering, Save to Wiki | Planned |
| Phase 4: Lint & Graph | Lint engine (structural + semantic), graph view, lint dashboard | Planned |
| Phase 5: Polish | i18n, settings UI, templates system, schema co-evolution | Planned |
| Phase 6: Extension | Browser clipper extension, batch ingest, image processing | Planned |

---

## File Structure

```
llm_wiki/
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── index.html
├── src/
│   ├── main.tsx                          # React entry point
│   ├── App.tsx                           # Root component, layout shell
│   ├── index.css                         # Tailwind import
│   ├── lib/
│   │   └── utils.ts                      # shadcn cn() utility
│   ├── stores/
│   │   └── wiki-store.ts                 # Zustand store: current project, selected file, file tree
│   ├── components/
│   │   ├── ui/                           # shadcn/ui components (auto-generated)
│   │   ├── layout/
│   │   │   ├── app-layout.tsx            # Top-level layout: sidebar + file tree + content + chat bar
│   │   │   ├── icon-sidebar.tsx          # Left icon bar (wiki, sources, search, graph, lint, settings)
│   │   │   ├── file-tree.tsx             # File tree panel showing wiki/ structure
│   │   │   ├── content-area.tsx          # Main content area (renders editor or welcome)
│   │   │   └── chat-bar.tsx              # Collapsible chat bar placeholder at bottom of content
│   │   ├── editor/
│   │   │   └── wiki-editor.tsx           # Milkdown WYSIWYG editor with [[wikilink]] support
│   │   └── project/
│   │       ├── welcome-screen.tsx        # Shown when no project is open
│   │       └── create-project-dialog.tsx # Dialog to create new wiki project
│   ├── commands/
│   │   └── fs.ts                         # TypeScript wrappers for Rust file system commands
│   └── types/
│       └── wiki.ts                       # Shared types: WikiProject, FileNode, WikiPage
├── src-tauri/
│   ├── Cargo.toml
│   ├── build.rs
│   ├── tauri.conf.json
│   ├── capabilities/
│   │   └── default.json
│   └── src/
│       ├── main.rs                       # Desktop entry point
│       ├── lib.rs                        # Tauri builder setup, registers all commands
│       ├── commands/
│       │   ├── mod.rs                    # Re-exports command modules
│       │   ├── project.rs                # create_project, open_project, list_recent_projects
│       │   └── fs.rs                     # read_file, write_file, list_directory, create_directory
│       └── types/
│           ├── mod.rs                    # Re-exports type modules
│           └── wiki.rs                   # FileNode, WikiProject structs
└── tests/
    └── src-tauri/
        └── commands/
            ├── project_test.rs           # Tests for project commands
            └── fs_test.rs                # Tests for file system commands
```

---

### Task 1: Scaffold Tauri v2 + React + TypeScript Project

**Files:**
- Create: entire project scaffold via CLI
- Modify: `package.json` (add dependencies)

- [ ] **Step 1: Create Tauri project**

```bash
cd /Users/nash_su/projects
npm create tauri-app@latest llm_wiki -- --template react-ts --manager npm
```

When prompted:
- Identifier: `com.llmwiki.app`
- Frontend: TypeScript
- Package manager: npm
- UI template: React
- UI flavor: TypeScript

- [ ] **Step 2: Verify scaffold runs**

```bash
cd /Users/nash_su/projects/llm_wiki
npm install
npm run tauri dev
```

Expected: A window opens with the default Tauri + React starter page.

- [ ] **Step 3: Stop the dev server and commit**

```bash
git init
echo ".superpowers/" >> .gitignore
git add -A
git commit -m "chore: scaffold Tauri v2 + React + TypeScript project"
```

---

### Task 2: Add Tailwind CSS v4 + shadcn/ui

**Files:**
- Modify: `vite.config.ts`
- Modify: `tsconfig.app.json`
- Modify: `src/index.css`
- Create: `src/lib/utils.ts`
- Create: `src/components/ui/` (via shadcn CLI)

- [ ] **Step 1: Install Tailwind CSS v4**

```bash
npm install tailwindcss @tailwindcss/vite
```

- [ ] **Step 2: Configure Vite plugin and path alias**

Replace `vite.config.ts`:

```ts
import path from "path"
import { defineConfig } from "vite"
import react from "@vitejs/plugin-react"
import tailwindcss from "@tailwindcss/vite"

const host = process.env.TAURI_DEV_HOST

export default defineConfig(async () => ({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
  clearScreen: false,
  server: {
    port: 1420,
    strictPort: true,
    host: host || false,
    hmr: host ? { protocol: "ws", host, port: 1421 } : undefined,
    watch: { ignored: ["**/src-tauri/**"] },
  },
}))
```

- [ ] **Step 3: Add path alias to tsconfig**

Add to `tsconfig.app.json` inside `compilerOptions`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

- [ ] **Step 4: Replace src/index.css with Tailwind import**

```css
@import "tailwindcss";
```

- [ ] **Step 5: Install shadcn/ui dependencies**

```bash
npm install -D @types/node
npx shadcn@latest init
```

When prompted, accept defaults (New York style, Zinc base color).

- [ ] **Step 6: Add initial shadcn components**

```bash
npx shadcn@latest add button scroll-area separator tooltip dialog input label
```

- [ ] **Step 7: Verify Tailwind + shadcn work**

Replace `src/App.tsx`:

```tsx
import { Button } from "@/components/ui/button"

function App() {
  return (
    <div className="flex h-screen items-center justify-center bg-background">
      <Button>LLM Wiki</Button>
    </div>
  )
}

export default App
```

Run: `npm run tauri dev`
Expected: A centered button with shadcn styling appears.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "chore: add Tailwind CSS v4 + shadcn/ui"
```

---

### Task 3: Define Shared Types

**Files:**
- Create: `src/types/wiki.ts`
- Create: `src-tauri/src/types/mod.rs`
- Create: `src-tauri/src/types/wiki.rs`
- Modify: `src-tauri/src/lib.rs`

- [ ] **Step 1: Create TypeScript types**

Create `src/types/wiki.ts`:

```ts
export interface WikiProject {
  name: string
  path: string
}

export interface FileNode {
  name: string
  path: string
  is_dir: boolean
  children?: FileNode[]
}

export interface WikiPage {
  path: string
  content: string
  frontmatter: Record<string, unknown>
}
```

- [ ] **Step 2: Create Rust types**

Create `src-tauri/src/types/mod.rs`:

```rust
pub mod wiki;
```

Create `src-tauri/src/types/wiki.rs`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WikiProject {
    pub name: String,
    pub path: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FileNode {
    pub name: String,
    pub path: String,
    pub is_dir: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub children: Option<Vec<FileNode>>,
}
```

- [ ] **Step 3: Register types module in lib.rs**

Add to `src-tauri/src/lib.rs`:

```rust
mod types;
```

- [ ] **Step 4: Commit**

```bash
git add src/types/wiki.ts src-tauri/src/types/
git commit -m "feat: define shared WikiProject and FileNode types"
```

---

### Task 4: Rust File System Commands

**Files:**
- Create: `src-tauri/src/commands/mod.rs`
- Create: `src-tauri/src/commands/fs.rs`
- Create: `src-tauri/src/commands/project.rs`
- Modify: `src-tauri/src/lib.rs`

- [ ] **Step 1: Create commands module**

Create `src-tauri/src/commands/mod.rs`:

```rust
pub mod fs;
pub mod project;
```

- [ ] **Step 2: Create file system commands**

Create `src-tauri/src/commands/fs.rs`:

```rust
use crate::types::wiki::FileNode;
use std::fs;
use std::path::Path;

#[tauri::command]
pub fn read_file(path: String) -> Result<String, String> {
    fs::read_to_string(&path).map_err(|e| format!("Failed to read {}: {}", path, e))
}

#[tauri::command]
pub fn write_file(path: String, contents: String) -> Result<(), String> {
    if let Some(parent) = Path::new(&path).parent() {
        fs::create_dir_all(parent)
            .map_err(|e| format!("Failed to create directory: {}", e))?;
    }
    fs::write(&path, contents).map_err(|e| format!("Failed to write {}: {}", path, e))
}

#[tauri::command]
pub fn list_directory(path: String) -> Result<Vec<FileNode>, String> {
    let path = Path::new(&path);
    if !path.is_dir() {
        return Err(format!("{} is not a directory", path.display()));
    }
    list_dir_recursive(path, 0, 3)
}

fn list_dir_recursive(path: &Path, depth: usize, max_depth: usize) -> Result<Vec<FileNode>, String> {
    let mut entries: Vec<FileNode> = Vec::new();
    let read_dir = fs::read_dir(path)
        .map_err(|e| format!("Failed to read directory {}: {}", path.display(), e))?;

    for entry in read_dir {
        let entry = entry.map_err(|e| e.to_string())?;
        let entry_path = entry.path();
        let name = entry
            .file_name()
            .to_string_lossy()
            .to_string();

        if name.starts_with('.') {
            continue;
        }

        let is_dir = entry_path.is_dir();
        let children = if is_dir && depth < max_depth {
            Some(list_dir_recursive(&entry_path, depth + 1, max_depth)?)
        } else if is_dir {
            Some(Vec::new())
        } else {
            None
        };

        entries.push(FileNode {
            name,
            path: entry_path.to_string_lossy().to_string(),
            is_dir,
            children,
        });
    }

    entries.sort_by(|a, b| match (a.is_dir, b.is_dir) {
        (true, false) => std::cmp::Ordering::Less,
        (false, true) => std::cmp::Ordering::Greater,
        _ => a.name.to_lowercase().cmp(&b.name.to_lowercase()),
    });

    Ok(entries)
}

#[tauri::command]
pub fn create_directory(path: String) -> Result<(), String> {
    fs::create_dir_all(&path).map_err(|e| format!("Failed to create directory {}: {}", path, e))
}
```

- [ ] **Step 3: Create project commands**

Create `src-tauri/src/commands/project.rs`:

```rust
use crate::types::wiki::WikiProject;
use std::fs;
use std::path::Path;

const DEFAULT_SCHEMA: &str = r#"# Schema

## Page Types
- source: Source summary (title, origin, date, summary, key points, related pages)
- entity: Entity page (name, type, description, first source, related entities)
- concept: Concept page (definition, background, key arguments, related concepts, cited sources)
- comparison: Comparison page (subjects, dimensions, comparison table, conclusion)
- query: Archived query (original question, answer, cited pages, date)
- overview: Global overview (single page, high-level picture)
- synthesis: Cross-source synthesis (deep analysis, evolving thesis, evidence tracking)

## Naming Conventions
- Lowercase, kebab-case
- Sources: sources/author-year-keyword.md
- Entities: entities/name.md
- Concepts: concepts/name.md

## Frontmatter
Every wiki page has YAML frontmatter:
- type, title, created, updated, sources[], tags[], source_count

## Index Format
Organized by category. Each entry:
- [Page Title](path.md) — one-line summary (N sources, YYYY-MM-DD)

## Log Format
Each entry: ## [YYYY-MM-DD] type | title
Types: ingest, query, lint, update

## Cross-referencing Rules
- Use [[wikilink]] syntax
- First mention of entity/concept must be linked
- After creating a new page, update all related pages' references

## Contradiction Handling
- On ingest, check new information against existing pages
- Mark contradictions with > [!conflict] callouts
- Update synthesis pages to record thesis evolution
- Lint re-reviews unresolved contradictions
"#;

const DEFAULT_PURPOSE: &str = r#"# Purpose

## Goal
(Describe what this wiki is for in one or two sentences)

## Key Questions
- (What are you trying to understand or answer?)

## Scope
- In scope: (What topics belong here)
- Out of scope: (What to exclude)

## Thesis
(Your current working hypothesis — this will evolve as you add sources)
"#;

#[tauri::command]
pub fn create_project(name: String, path: String) -> Result<WikiProject, String> {
    let project_path = Path::new(&path).join(&name);

    if project_path.exists() {
        return Err(format!("Directory already exists: {}", project_path.display()));
    }

    let dirs = [
        "",
        "raw/sources",
        "raw/assets",
        "wiki/entities",
        "wiki/concepts",
        "wiki/sources",
        "wiki/queries",
        "wiki/comparisons",
        "wiki/synthesis",
    ];

    for dir in &dirs {
        fs::create_dir_all(project_path.join(dir))
            .map_err(|e| format!("Failed to create {}: {}", dir, e))?;
    }

    fs::write(project_path.join("schema.md"), DEFAULT_SCHEMA)
        .map_err(|e| format!("Failed to write schema.md: {}", e))?;

    fs::write(project_path.join("purpose.md"), DEFAULT_PURPOSE)
        .map_err(|e| format!("Failed to write purpose.md: {}", e))?;

    let now = chrono::Local::now().format("%Y-%m-%d").to_string();

    let index_content = format!(
        "# Index\n\n*Last updated: {}*\n\n## Entities\n\n(No entities yet)\n\n## Concepts\n\n(No concepts yet)\n\n## Sources\n\n(No sources yet)\n\n## Queries\n\n(No queries yet)\n\n## Comparisons\n\n(No comparisons yet)\n\n## Synthesis\n\n(No synthesis yet)\n",
        now
    );
    fs::write(project_path.join("wiki/index.md"), index_content)
        .map_err(|e| format!("Failed to write index.md: {}", e))?;

    let log_content = format!("# Log\n\n## [{}] init | Project created\n\nInitialized wiki project: {}\n", now, name);
    fs::write(project_path.join("wiki/log.md"), log_content)
        .map_err(|e| format!("Failed to write log.md: {}", e))?;

    let overview_content = format!(
        "---\ntype: overview\ntitle: Overview\ncreated: {}\nupdated: {}\nsources: []\ntags: []\nsource_count: 0\n---\n\n# Overview\n\nThis wiki is new. Add sources to start building knowledge.\n",
        now, now
    );
    fs::write(project_path.join("wiki/overview.md"), overview_content)
        .map_err(|e| format!("Failed to write overview.md: {}", e))?;

    let project = WikiProject {
        name,
        path: project_path.to_string_lossy().to_string(),
    };

    Ok(project)
}

#[tauri::command]
pub fn open_project(path: String) -> Result<WikiProject, String> {
    let project_path = Path::new(&path);

    if !project_path.join("schema.md").exists() {
        return Err("Not a valid LLM Wiki project: schema.md not found".to_string());
    }
    if !project_path.join("wiki").is_dir() {
        return Err("Not a valid LLM Wiki project: wiki/ directory not found".to_string());
    }

    let name = project_path
        .file_name()
        .map(|n| n.to_string_lossy().to_string())
        .unwrap_or_else(|| "Unnamed".to_string());

    Ok(WikiProject {
        name,
        path: project_path.to_string_lossy().to_string(),
    })
}
```

- [ ] **Step 4: Add chrono dependency**

Add to `src-tauri/Cargo.toml` under `[dependencies]`:

```toml
chrono = "0.4"
```

- [ ] **Step 5: Register commands in lib.rs**

Replace `src-tauri/src/lib.rs`:

```rust
mod commands;
mod types;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            commands::fs::read_file,
            commands::fs::write_file,
            commands::fs::list_directory,
            commands::fs::create_directory,
            commands::project::create_project,
            commands::project::open_project,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 6: Verify it compiles**

```bash
cd src-tauri && cargo build
```

Expected: Compiles with no errors.

- [ ] **Step 7: Commit**

```bash
git add src-tauri/src/commands/ src-tauri/Cargo.toml src-tauri/src/lib.rs
git commit -m "feat: add Rust file system and project commands"
```

---

### Task 5: TypeScript Command Wrappers

**Files:**
- Create: `src/commands/fs.ts`

- [ ] **Step 1: Create TypeScript wrappers for Rust commands**

Create `src/commands/fs.ts`:

```ts
import { invoke } from "@tauri-apps/api/core"
import type { WikiProject, FileNode } from "@/types/wiki"

export async function readFile(path: string): Promise<string> {
  return invoke<string>("read_file", { path })
}

export async function writeFile(path: string, contents: string): Promise<void> {
  return invoke("write_file", { path, contents })
}

export async function listDirectory(path: string): Promise<FileNode[]> {
  return invoke<FileNode[]>("list_directory", { path })
}

export async function createDirectory(path: string): Promise<void> {
  return invoke("create_directory", { path })
}

export async function createProject(
  name: string,
  path: string
): Promise<WikiProject> {
  return invoke<WikiProject>("create_project", { name, path })
}

export async function openProject(path: string): Promise<WikiProject> {
  return invoke<WikiProject>("open_project", { path })
}
```

- [ ] **Step 2: Commit**

```bash
git add src/commands/fs.ts
git commit -m "feat: add TypeScript wrappers for Rust commands"
```

---

### Task 6: Zustand Store

**Files:**
- Create: `src/stores/wiki-store.ts`

- [ ] **Step 1: Install Zustand**

```bash
npm install zustand
```

- [ ] **Step 2: Create wiki store**

Create `src/stores/wiki-store.ts`:

```ts
import { create } from "zustand"
import type { WikiProject, FileNode } from "@/types/wiki"

interface WikiState {
  project: WikiProject | null
  fileTree: FileNode[]
  selectedFile: string | null
  fileContent: string
  chatExpanded: boolean
  activeView: "wiki" | "sources" | "search" | "graph" | "lint" | "settings"

  setProject: (project: WikiProject | null) => void
  setFileTree: (tree: FileNode[]) => void
  setSelectedFile: (path: string | null) => void
  setFileContent: (content: string) => void
  setChatExpanded: (expanded: boolean) => void
  setActiveView: (view: WikiState["activeView"]) => void
}

export const useWikiStore = create<WikiState>((set) => ({
  project: null,
  fileTree: [],
  selectedFile: null,
  fileContent: "",
  chatExpanded: false,
  activeView: "wiki",

  setProject: (project) => set({ project }),
  setFileTree: (fileTree) => set({ fileTree }),
  setSelectedFile: (selectedFile) => set({ selectedFile }),
  setFileContent: (fileContent) => set({ fileContent }),
  setChatExpanded: (chatExpanded) => set({ chatExpanded }),
  setActiveView: (activeView) => set({ activeView }),
}))
```

- [ ] **Step 3: Commit**

```bash
git add src/stores/wiki-store.ts package.json package-lock.json
git commit -m "feat: add Zustand wiki store"
```

---

### Task 7: Layout Shell — Icon Sidebar

**Files:**
- Create: `src/components/layout/icon-sidebar.tsx`

- [ ] **Step 1: Create icon sidebar component**

Create `src/components/layout/icon-sidebar.tsx`:

```tsx
import {
  FileText,
  FolderOpen,
  Search,
  Network,
  ClipboardCheck,
  Settings,
} from "lucide-react"
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from "@/components/ui/tooltip"
import { useWikiStore } from "@/stores/wiki-store"
import type { WikiState } from "@/stores/wiki-store"

const navItems: { view: WikiState["activeView"]; icon: typeof FileText; label: string }[] = [
  { view: "wiki", icon: FileText, label: "Wiki" },
  { view: "sources", icon: FolderOpen, label: "Sources" },
  { view: "search", icon: Search, label: "Search" },
  { view: "graph", icon: Network, label: "Graph" },
  { view: "lint", icon: ClipboardCheck, label: "Lint" },
  { view: "settings", icon: Settings, label: "Settings" },
]

export function IconSidebar() {
  const activeView = useWikiStore((s) => s.activeView)
  const setActiveView = useWikiStore((s) => s.setActiveView)

  return (
    <TooltipProvider delayDuration={300}>
      <div className="flex h-full w-12 flex-col items-center gap-1 border-r bg-muted/50 py-2">
        {navItems.map(({ view, icon: Icon, label }) => (
          <Tooltip key={view}>
            <TooltipTrigger asChild>
              <button
                onClick={() => setActiveView(view)}
                className={`flex h-10 w-10 items-center justify-center rounded-md transition-colors ${
                  activeView === view
                    ? "bg-accent text-accent-foreground"
                    : "text-muted-foreground hover:bg-accent/50 hover:text-accent-foreground"
                }`}
              >
                <Icon className="h-5 w-5" />
              </button>
            </TooltipTrigger>
            <TooltipContent side="right">{label}</TooltipContent>
          </Tooltip>
        ))}
      </div>
    </TooltipProvider>
  )
}
```

- [ ] **Step 2: Install lucide-react**

```bash
npm install lucide-react
```

- [ ] **Step 3: Commit**

```bash
git add src/components/layout/icon-sidebar.tsx package.json package-lock.json
git commit -m "feat: add icon sidebar component"
```

---

### Task 8: Layout Shell — File Tree

**Files:**
- Create: `src/components/layout/file-tree.tsx`

- [ ] **Step 1: Create file tree component**

Create `src/components/layout/file-tree.tsx`:

```tsx
import { useState } from "react"
import { ChevronRight, ChevronDown, File, Folder } from "lucide-react"
import { ScrollArea } from "@/components/ui/scroll-area"
import { useWikiStore } from "@/stores/wiki-store"
import type { FileNode } from "@/types/wiki"

function TreeNode({ node, depth }: { node: FileNode; depth: number }) {
  const [expanded, setExpanded] = useState(depth < 1)
  const selectedFile = useWikiStore((s) => s.selectedFile)
  const setSelectedFile = useWikiStore((s) => s.setSelectedFile)

  const isSelected = selectedFile === node.path
  const paddingLeft = 12 + depth * 16

  if (node.is_dir) {
    return (
      <div>
        <button
          onClick={() => setExpanded(!expanded)}
          className="flex w-full items-center gap-1 py-1 text-sm text-muted-foreground hover:bg-accent/50 hover:text-accent-foreground"
          style={{ paddingLeft }}
        >
          {expanded ? (
            <ChevronDown className="h-3.5 w-3.5 shrink-0" />
          ) : (
            <ChevronRight className="h-3.5 w-3.5 shrink-0" />
          )}
          <Folder className="h-3.5 w-3.5 shrink-0 text-blue-400" />
          <span className="truncate">{node.name}</span>
        </button>
        {expanded && node.children?.map((child) => (
          <TreeNode key={child.path} node={child} depth={depth + 1} />
        ))}
      </div>
    )
  }

  return (
    <button
      onClick={() => setSelectedFile(node.path)}
      className={`flex w-full items-center gap-1 py-1 text-sm ${
        isSelected
          ? "bg-accent text-accent-foreground"
          : "text-muted-foreground hover:bg-accent/50 hover:text-accent-foreground"
      }`}
      style={{ paddingLeft: paddingLeft + 14 }}
    >
      <File className="h-3.5 w-3.5 shrink-0" />
      <span className="truncate">{node.name}</span>
    </button>
  )
}

export function FileTree() {
  const fileTree = useWikiStore((s) => s.fileTree)
  const project = useWikiStore((s) => s.project)

  if (!project) {
    return (
      <div className="flex h-full items-center justify-center p-4 text-sm text-muted-foreground">
        No project open
      </div>
    )
  }

  return (
    <ScrollArea className="h-full">
      <div className="p-2">
        <div className="mb-2 px-2 text-xs font-semibold uppercase text-muted-foreground">
          {project.name}
        </div>
        {fileTree.map((node) => (
          <TreeNode key={node.path} node={node} depth={0} />
        ))}
      </div>
    </ScrollArea>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add src/components/layout/file-tree.tsx
git commit -m "feat: add file tree component"
```

---

### Task 9: Layout Shell — Content Area + Chat Bar

**Files:**
- Create: `src/components/layout/content-area.tsx`
- Create: `src/components/layout/chat-bar.tsx`

- [ ] **Step 1: Create chat bar placeholder**

Create `src/components/layout/chat-bar.tsx`:

```tsx
import { MessageSquare, ChevronUp, ChevronDown } from "lucide-react"
import { useWikiStore } from "@/stores/wiki-store"

export function ChatBar() {
  const chatExpanded = useWikiStore((s) => s.chatExpanded)
  const setChatExpanded = useWikiStore((s) => s.setChatExpanded)

  return (
    <div className="border-t">
      <button
        onClick={() => setChatExpanded(!chatExpanded)}
        className="flex w-full items-center justify-between px-4 py-2 text-sm text-muted-foreground hover:bg-accent/50"
      >
        <span className="flex items-center gap-2">
          <MessageSquare className="h-4 w-4" />
          CHAT
        </span>
        {chatExpanded ? (
          <ChevronDown className="h-4 w-4" />
        ) : (
          <ChevronUp className="h-4 w-4" />
        )}
      </button>
      {chatExpanded && (
        <div className="flex h-64 flex-col border-t">
          <div className="flex-1 overflow-auto p-4">
            <p className="text-sm text-muted-foreground">
              Chat will be available in Phase 2 (LLM Integration).
            </p>
          </div>
          <div className="border-t p-3">
            <input
              type="text"
              placeholder="Type a message..."
              disabled
              className="w-full rounded-md border bg-muted/50 px-3 py-2 text-sm"
            />
          </div>
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Create content area**

Create `src/components/layout/content-area.tsx`:

```tsx
import { useEffect } from "react"
import { useWikiStore } from "@/stores/wiki-store"
import { readFile } from "@/commands/fs"
import { ChatBar } from "./chat-bar"

export function ContentArea() {
  const selectedFile = useWikiStore((s) => s.selectedFile)
  const fileContent = useWikiStore((s) => s.fileContent)
  const setFileContent = useWikiStore((s) => s.setFileContent)

  useEffect(() => {
    if (!selectedFile) {
      setFileContent("")
      return
    }
    readFile(selectedFile)
      .then(setFileContent)
      .catch((err) => setFileContent(`Error loading file: ${err}`))
  }, [selectedFile, setFileContent])

  return (
    <div className="flex h-full flex-col">
      <div className="flex-1 overflow-auto">
        {selectedFile ? (
          <div className="p-6">
            <div className="mb-4 text-xs text-muted-foreground">
              {selectedFile}
            </div>
            <pre className="whitespace-pre-wrap font-mono text-sm">
              {fileContent}
            </pre>
          </div>
        ) : (
          <div className="flex h-full items-center justify-center text-muted-foreground">
            Select a file from the tree to view
          </div>
        )}
      </div>
      <ChatBar />
    </div>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add src/components/layout/content-area.tsx src/components/layout/chat-bar.tsx
git commit -m "feat: add content area and collapsible chat bar"
```

---

### Task 10: Layout Shell — App Layout + Wiring

**Files:**
- Create: `src/components/layout/app-layout.tsx`
- Create: `src/components/project/welcome-screen.tsx`
- Create: `src/components/project/create-project-dialog.tsx`
- Modify: `src/App.tsx`
- Modify: `src/stores/wiki-store.ts` (export type)

- [ ] **Step 1: Export WikiState type from store**

Add to the end of `src/stores/wiki-store.ts`:

```ts
export type { WikiState }
```

- [ ] **Step 2: Create welcome screen**

Create `src/components/project/welcome-screen.tsx`:

```tsx
import { FolderOpen, Plus } from "lucide-react"
import { Button } from "@/components/ui/button"

interface WelcomeScreenProps {
  onCreateProject: () => void
  onOpenProject: () => void
}

export function WelcomeScreen({ onCreateProject, onOpenProject }: WelcomeScreenProps) {
  return (
    <div className="flex h-full items-center justify-center bg-background">
      <div className="flex flex-col items-center gap-6">
        <h1 className="text-3xl font-bold">LLM Wiki</h1>
        <p className="text-muted-foreground">
          Build and maintain your personal knowledge base with LLMs
        </p>
        <div className="flex gap-3">
          <Button onClick={onCreateProject}>
            <Plus className="mr-2 h-4 w-4" />
            New Project
          </Button>
          <Button variant="outline" onClick={onOpenProject}>
            <FolderOpen className="mr-2 h-4 w-4" />
            Open Project
          </Button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create project dialog**

Create `src/components/project/create-project-dialog.tsx`:

```tsx
import { useState } from "react"
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogFooter,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"
import { createProject } from "@/commands/fs"
import type { WikiProject } from "@/types/wiki"

interface CreateProjectDialogProps {
  open: boolean
  onOpenChange: (open: boolean) => void
  onCreated: (project: WikiProject) => void
}

export function CreateProjectDialog({
  open,
  onOpenChange,
  onCreated,
}: CreateProjectDialogProps) {
  const [name, setName] = useState("")
  const [path, setPath] = useState("")
  const [error, setError] = useState("")
  const [creating, setCreating] = useState(false)

  async function handleCreate() {
    if (!name.trim() || !path.trim()) {
      setError("Name and path are required")
      return
    }

    setCreating(true)
    setError("")

    try {
      const project = await createProject(name.trim(), path.trim())
      onCreated(project)
      onOpenChange(false)
      setName("")
      setPath("")
    } catch (err) {
      setError(String(err))
    } finally {
      setCreating(false)
    }
  }

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Create New Wiki Project</DialogTitle>
        </DialogHeader>
        <div className="flex flex-col gap-4 py-4">
          <div className="flex flex-col gap-2">
            <Label htmlFor="name">Project Name</Label>
            <Input
              id="name"
              value={name}
              onChange={(e) => setName(e.target.value)}
              placeholder="my-research-wiki"
            />
          </div>
          <div className="flex flex-col gap-2">
            <Label htmlFor="path">Parent Directory</Label>
            <Input
              id="path"
              value={path}
              onChange={(e) => setPath(e.target.value)}
              placeholder="/Users/you/projects"
            />
          </div>
          {error && <p className="text-sm text-destructive">{error}</p>}
        </div>
        <DialogFooter>
          <Button variant="outline" onClick={() => onOpenChange(false)}>
            Cancel
          </Button>
          <Button onClick={handleCreate} disabled={creating}>
            {creating ? "Creating..." : "Create"}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

- [ ] **Step 4: Create app layout**

Create `src/components/layout/app-layout.tsx`:

```tsx
import { useCallback, useEffect } from "react"
import { useWikiStore } from "@/stores/wiki-store"
import { listDirectory } from "@/commands/fs"
import { IconSidebar } from "./icon-sidebar"
import { FileTree } from "./file-tree"
import { ContentArea } from "./content-area"

export function AppLayout() {
  const project = useWikiStore((s) => s.project)
  const setFileTree = useWikiStore((s) => s.setFileTree)

  const loadFileTree = useCallback(async () => {
    if (!project) return
    try {
      const tree = await listDirectory(project.path)
      setFileTree(tree)
    } catch (err) {
      console.error("Failed to load file tree:", err)
    }
  }, [project, setFileTree])

  useEffect(() => {
    loadFileTree()
  }, [loadFileTree])

  return (
    <div className="flex h-screen bg-background text-foreground">
      <IconSidebar />
      <div className="flex flex-1 overflow-hidden">
        <div className="w-60 border-r">
          <FileTree />
        </div>
        <div className="flex-1">
          <ContentArea />
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Wire up App.tsx**

Replace `src/App.tsx`:

```tsx
import { useState } from "react"
import { useWikiStore } from "@/stores/wiki-store"
import { listDirectory, openProject } from "@/commands/fs"
import { AppLayout } from "@/components/layout/app-layout"
import { WelcomeScreen } from "@/components/project/welcome-screen"
import { CreateProjectDialog } from "@/components/project/create-project-dialog"
import type { WikiProject } from "@/types/wiki"

function App() {
  const project = useWikiStore((s) => s.project)
  const setProject = useWikiStore((s) => s.setProject)
  const setFileTree = useWikiStore((s) => s.setFileTree)
  const [showCreateDialog, setShowCreateDialog] = useState(false)

  async function handleProjectOpened(proj: WikiProject) {
    setProject(proj)
    try {
      const tree = await listDirectory(proj.path)
      setFileTree(tree)
    } catch (err) {
      console.error("Failed to load file tree:", err)
    }
  }

  async function handleOpenProject() {
    const path = window.prompt("Enter the path to an existing wiki project:")
    if (!path) return
    try {
      const proj = await openProject(path)
      await handleProjectOpened(proj)
    } catch (err) {
      window.alert(`Failed to open project: ${err}`)
    }
  }

  if (!project) {
    return (
      <>
        <WelcomeScreen
          onCreateProject={() => setShowCreateDialog(true)}
          onOpenProject={handleOpenProject}
        />
        <CreateProjectDialog
          open={showCreateDialog}
          onOpenChange={setShowCreateDialog}
          onCreated={handleProjectOpened}
        />
      </>
    )
  }

  return <AppLayout />
}

export default App
```

- [ ] **Step 6: Verify the full layout**

```bash
npm run tauri dev
```

Expected: App opens with welcome screen. "New Project" opens dialog. After creating a project, the three-panel layout appears: icon sidebar on left, file tree showing the wiki directory structure, content area on right with collapsible Chat bar at bottom.

- [ ] **Step 7: Commit**

```bash
git add src/App.tsx src/components/ src/stores/wiki-store.ts
git commit -m "feat: wire up app layout with welcome screen, file tree, and content area"
```

---

### Task 11: Milkdown WYSIWYG Editor

**Files:**
- Create: `src/components/editor/wiki-editor.tsx`
- Modify: `src/components/layout/content-area.tsx`

- [ ] **Step 1: Install Milkdown dependencies**

```bash
npm install @milkdown/kit @milkdown/theme-nord @milkdown/react
```

- [ ] **Step 2: Create wiki editor component**

Create `src/components/editor/wiki-editor.tsx`:

```tsx
import { useEffect } from "react"
import { Editor, rootCtx, defaultValueCtx } from "@milkdown/kit/core"
import { commonmark } from "@milkdown/kit/preset/commonmark"
import { history } from "@milkdown/kit/plugin/history"
import { listener, listenerCtx } from "@milkdown/kit/plugin/listener"
import { nord } from "@milkdown/theme-nord"
import { Milkdown, MilkdownProvider, useEditor } from "@milkdown/react"
import "@milkdown/theme-nord/style.css"

interface WikiEditorInnerProps {
  content: string
  onSave: (markdown: string) => void
}

function WikiEditorInner({ content, onSave }: WikiEditorInnerProps) {
  const { get } = useEditor((root) =>
    Editor.make()
      .config(nord)
      .config((ctx) => {
        ctx.set(rootCtx, root)
        ctx.set(defaultValueCtx, content)
        ctx.get(listenerCtx).markdownUpdated((_ctx, markdown) => {
          onSave(markdown)
        })
      })
      .use(commonmark)
      .use(history)
      .use(listener),
    [content]
  )

  useEffect(() => {
    // Editor recreated when content (dependency) changes
  }, [get])

  return <Milkdown />
}

interface WikiEditorProps {
  content: string
  onSave: (markdown: string) => void
}

export function WikiEditor({ content, onSave }: WikiEditorProps) {
  return (
    <MilkdownProvider>
      <div className="prose prose-invert max-w-none p-6">
        <WikiEditorInner content={content} onSave={onSave} />
      </div>
    </MilkdownProvider>
  )
}
```

- [ ] **Step 3: Update content area to use WikiEditor for .md files**

Replace `src/components/layout/content-area.tsx`:

```tsx
import { useEffect, useCallback, useRef } from "react"
import { useWikiStore } from "@/stores/wiki-store"
import { readFile, writeFile } from "@/commands/fs"
import { WikiEditor } from "@/components/editor/wiki-editor"
import { ChatBar } from "./chat-bar"

export function ContentArea() {
  const selectedFile = useWikiStore((s) => s.selectedFile)
  const fileContent = useWikiStore((s) => s.fileContent)
  const setFileContent = useWikiStore((s) => s.setFileContent)
  const saveTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  useEffect(() => {
    if (!selectedFile) {
      setFileContent("")
      return
    }
    readFile(selectedFile)
      .then(setFileContent)
      .catch((err) => setFileContent(`Error loading file: ${err}`))
  }, [selectedFile, setFileContent])

  const handleSave = useCallback(
    (markdown: string) => {
      if (!selectedFile) return
      if (saveTimerRef.current) {
        clearTimeout(saveTimerRef.current)
      }
      saveTimerRef.current = setTimeout(() => {
        writeFile(selectedFile, markdown).catch((err) =>
          console.error("Failed to save:", err)
        )
      }, 1000)
    },
    [selectedFile]
  )

  useEffect(() => {
    return () => {
      if (saveTimerRef.current) {
        clearTimeout(saveTimerRef.current)
      }
    }
  }, [])

  const isMarkdown = selectedFile?.endsWith(".md")

  return (
    <div className="flex h-full flex-col">
      <div className="flex-1 overflow-auto">
        {selectedFile ? (
          isMarkdown ? (
            <WikiEditor
              key={selectedFile}
              content={fileContent}
              onSave={handleSave}
            />
          ) : (
            <div className="p-6">
              <div className="mb-4 text-xs text-muted-foreground">
                {selectedFile}
              </div>
              <pre className="whitespace-pre-wrap font-mono text-sm">
                {fileContent}
              </pre>
            </div>
          )
        ) : (
          <div className="flex h-full items-center justify-center text-muted-foreground">
            Select a file from the tree to view
          </div>
        )}
      </div>
      <ChatBar />
    </div>
  )
}
```

- [ ] **Step 4: Verify editor works**

```bash
npm run tauri dev
```

Expected: Open a project, click on a `.md` file in the file tree. The Milkdown WYSIWYG editor renders the markdown with formatting. Edits auto-save after 1 second of inactivity.

- [ ] **Step 5: Commit**

```bash
git add src/components/editor/ src/components/layout/content-area.tsx package.json package-lock.json
git commit -m "feat: add Milkdown WYSIWYG markdown editor with auto-save"
```

---

### Task 12: Resizable Panels

**Files:**
- Modify: `src/components/layout/app-layout.tsx`
- Modify: `src/components/layout/content-area.tsx`

- [ ] **Step 1: Add resizable panel component from shadcn**

```bash
npx shadcn@latest add resizable
```

- [ ] **Step 2: Update app layout with resizable file tree**

Replace `src/components/layout/app-layout.tsx`:

```tsx
import { useCallback, useEffect } from "react"
import { useWikiStore } from "@/stores/wiki-store"
import { listDirectory } from "@/commands/fs"
import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"
import { IconSidebar } from "./icon-sidebar"
import { FileTree } from "./file-tree"
import { ContentArea } from "./content-area"

export function AppLayout() {
  const project = useWikiStore((s) => s.project)
  const setFileTree = useWikiStore((s) => s.setFileTree)

  const loadFileTree = useCallback(async () => {
    if (!project) return
    try {
      const tree = await listDirectory(project.path)
      setFileTree(tree)
    } catch (err) {
      console.error("Failed to load file tree:", err)
    }
  }, [project, setFileTree])

  useEffect(() => {
    loadFileTree()
  }, [loadFileTree])

  return (
    <div className="flex h-screen bg-background text-foreground">
      <IconSidebar />
      <ResizablePanelGroup direction="horizontal" className="flex-1">
        <ResizablePanel defaultSize={20} minSize={15} maxSize={40}>
          <FileTree />
        </ResizablePanel>
        <ResizableHandle withHandle />
        <ResizablePanel defaultSize={80}>
          <ContentArea />
        </ResizablePanel>
      </ResizablePanelGroup>
    </div>
  )
}
```

- [ ] **Step 3: Update content area with resizable chat panel**

Replace `src/components/layout/content-area.tsx`:

```tsx
import { useEffect, useCallback, useRef } from "react"
import { useWikiStore } from "@/stores/wiki-store"
import { readFile, writeFile } from "@/commands/fs"
import { WikiEditor } from "@/components/editor/wiki-editor"
import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"
import { ChatBar } from "./chat-bar"

export function ContentArea() {
  const selectedFile = useWikiStore((s) => s.selectedFile)
  const fileContent = useWikiStore((s) => s.fileContent)
  const setFileContent = useWikiStore((s) => s.setFileContent)
  const chatExpanded = useWikiStore((s) => s.chatExpanded)
  const saveTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  useEffect(() => {
    if (!selectedFile) {
      setFileContent("")
      return
    }
    readFile(selectedFile)
      .then(setFileContent)
      .catch((err) => setFileContent(`Error loading file: ${err}`))
  }, [selectedFile, setFileContent])

  const handleSave = useCallback(
    (markdown: string) => {
      if (!selectedFile) return
      if (saveTimerRef.current) {
        clearTimeout(saveTimerRef.current)
      }
      saveTimerRef.current = setTimeout(() => {
        writeFile(selectedFile, markdown).catch((err) =>
          console.error("Failed to save:", err)
        )
      }, 1000)
    },
    [selectedFile]
  )

  useEffect(() => {
    return () => {
      if (saveTimerRef.current) {
        clearTimeout(saveTimerRef.current)
      }
    }
  }, [])

  const isMarkdown = selectedFile?.endsWith(".md")

  const editorContent = (
    <div className="h-full overflow-auto">
      {selectedFile ? (
        isMarkdown ? (
          <WikiEditor
            key={selectedFile}
            content={fileContent}
            onSave={handleSave}
          />
        ) : (
          <div className="p-6">
            <div className="mb-4 text-xs text-muted-foreground">
              {selectedFile}
            </div>
            <pre className="whitespace-pre-wrap font-mono text-sm">
              {fileContent}
            </pre>
          </div>
        )
      ) : (
        <div className="flex h-full items-center justify-center text-muted-foreground">
          Select a file from the tree to view
        </div>
      )}
    </div>
  )

  if (!chatExpanded) {
    return (
      <div className="flex h-full flex-col">
        <div className="flex-1 overflow-hidden">{editorContent}</div>
        <ChatBar />
      </div>
    )
  }

  return (
    <div className="flex h-full flex-col">
      <ResizablePanelGroup direction="vertical" className="flex-1">
        <ResizablePanel defaultSize={60} minSize={30}>
          {editorContent}
        </ResizablePanel>
        <ResizableHandle withHandle />
        <ResizablePanel defaultSize={40} minSize={20}>
          <ChatBar />
        </ResizablePanel>
      </ResizablePanelGroup>
    </div>
  )
}
```

- [ ] **Step 4: Update ChatBar to not self-manage expanded state**

Replace `src/components/layout/chat-bar.tsx`:

```tsx
import { MessageSquare, ChevronUp, ChevronDown } from "lucide-react"
import { useWikiStore } from "@/stores/wiki-store"

export function ChatBar() {
  const chatExpanded = useWikiStore((s) => s.chatExpanded)
  const setChatExpanded = useWikiStore((s) => s.setChatExpanded)

  if (!chatExpanded) {
    return (
      <button
        onClick={() => setChatExpanded(true)}
        className="flex w-full items-center justify-between border-t px-4 py-2 text-sm text-muted-foreground hover:bg-accent/50"
      >
        <span className="flex items-center gap-2">
          <MessageSquare className="h-4 w-4" />
          CHAT
        </span>
        <ChevronUp className="h-4 w-4" />
      </button>
    )
  }

  return (
    <div className="flex h-full flex-col">
      <button
        onClick={() => setChatExpanded(false)}
        className="flex w-full items-center justify-between border-b px-4 py-2 text-sm text-muted-foreground hover:bg-accent/50"
      >
        <span className="flex items-center gap-2">
          <MessageSquare className="h-4 w-4" />
          CHAT
        </span>
        <ChevronDown className="h-4 w-4" />
      </button>
      <div className="flex-1 overflow-auto p-4">
        <p className="text-sm text-muted-foreground">
          Chat will be available in Phase 2 (LLM Integration).
        </p>
      </div>
      <div className="border-t p-3">
        <input
          type="text"
          placeholder="Type a message..."
          disabled
          className="w-full rounded-md border bg-muted/50 px-3 py-2 text-sm"
        />
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Verify resizable panels**

```bash
npm run tauri dev
```

Expected: File tree panel is resizable by dragging the handle. Chat panel expands with a resizable divider between content and chat. Collapsing chat returns to the thin bar.

- [ ] **Step 6: Commit**

```bash
git add src/components/layout/ src/components/ui/ package.json package-lock.json
git commit -m "feat: add resizable panels for file tree and chat"
```

---

### Task 13: Final Verification + Cleanup

**Files:**
- Modify: `src/index.css` (if needed)
- Modify: `index.html` (window title)

- [ ] **Step 1: Set window title**

In `src-tauri/tauri.conf.json`, find `"title"` and set to `"LLM Wiki"`.

- [ ] **Step 2: Update index.html title**

Change the `<title>` tag in `index.html` to `LLM Wiki`.

- [ ] **Step 3: Full end-to-end test**

```bash
npm run tauri dev
```

Verify the following manually:
1. App opens with welcome screen showing "LLM Wiki" title
2. Click "New Project" → dialog appears → enter name and path → project is created
3. File tree shows: `purpose.md`, `schema.md`, `raw/`, `wiki/` with subdirectories
4. Click on `wiki/index.md` → Milkdown editor renders the markdown
5. Edit content → wait 1 second → file is auto-saved (verify by re-opening)
6. Click `schema.md` → content switches to schema
7. File tree is resizable by dragging the handle
8. Chat bar at bottom → click to expand → resizable → click to collapse
9. Icon sidebar highlights the active view when clicked

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: complete Phase 1 foundation — layout, file ops, editor"
```

---

## Phase 1 Deliverables

After completing all tasks, you will have:

- ✅ Tauri v2 desktop app running on Mac/Linux/Windows
- ✅ VS Code-style layout: icon sidebar + resizable file tree + content area + collapsible chat
- ✅ Rust backend: file read/write/list, project create/open
- ✅ Wiki project creation with full directory structure (raw/, wiki/, schema.md, purpose.md)
- ✅ Milkdown WYSIWYG Markdown editor with auto-save
- ✅ File tree navigation with folder expand/collapse

## What's Next

**Phase 2: LLM Integration** will add:
- Rust-side LLM Router (OpenAI, Anthropic, Google, Ollama)
- Chat UI in the collapsible panel
- Basic single-source interactive ingest flow
- Schema and purpose.md reading for LLM context
