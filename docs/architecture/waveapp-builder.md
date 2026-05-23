# WaveApp Builder — Architecture Reference

> **Purpose**: On-demand reference for agents/developers working on the WaveApp Builder feature.
> **Status**: Dev-only feature. Gated in production behind `feature:waveappbuilder` config flag.
> **Last updated**: 2026-05-18

---

## 1. What is a WaveApp?

A **WaveApp** is a standalone Go binary that renders a rich UI inside Wave Terminal using a Virtual DOM framework (Tsunami). Think of it as an in-terminal application platform — you write Go components, Wave Terminal hosts the UI in a WebView.

WaveApps live on disk at `~/waveapps/<namespace>/<appname>/` with two namespaces:
- **`draft/`** — working/editing copy (the builder edits this)
- **`local/`** — published/final version

Each app directory contains:
- `app.go` — Go source (the main entry point)
- `manifest.json` — metadata: title, icon, short description, secret declarations, schema
- `secret-bindings.json` — maps declared secret names to secret-store keys

---

## 2. Feature Gating

| Context | Gate |
|---|---|
| Menu item "New WaveApp Builder Window" | `isDev \|\| settings["feature:waveappbuilder"]` |
| Config key | `"feature:waveappbuilder"` (`wconfig.ConfigKey_FeatureWaveAppBuilder`) |
| Settings JSON field | `SettingsType.FeatureWaveAppBuilder` (json: `"feature:waveappbuilder"`) |

**File**: `emain/emain-menu.ts:147` — the menu guard. In dev builds (`isDev`), the menu item always appears.

---

## 3. High-Level Architecture (Layers)

```
┌─────────────────────────────────────────────────────┐
│                 Electron Main Process                │
│  emain/emain-menu.ts   → menu item                  │
│  emain/emain-ipc.ts    → openBuilderWindow()        │
│  emain/emain-builder.ts → createBuilderWindow()     │
└──────────────┬──────────────────────────────────────┘
               │ IPC
┌──────────────▼──────────────────────────────────────┐
│             Builder Window (Renderer)                │
│  frontend/builder/                                   │
│  ├── builder-app.tsx         Root component          │
│  ├── builder-workspace.tsx   Tab layout              │
│  ├── app-selection-modal.tsx App picker / creator    │
│  ├── builder-apppanel.tsx    Code + preview panel    │
│  ├── builder-buildpanel.tsx  Build output panel      │
│  └── tabs/                   Per-tab components      │
│  └── store/                  Jotai state models      │
└──────────────┬──────────────────────────────────────┘
               │ WshRpc (domain socket)
┌──────────────▼──────────────────────────────────────┐
│            Wave Terminal Backend (Go)                │
│  pkg/buildercontroller/  Build + run orchestration  │
│  pkg/waveappstore/       ~/waveapps/ filesystem ops │
│  pkg/waveapp/            VDOM client (in-app SDK)   │
│  pkg/waveapputil/        Scaffolding, Go path utils │
│  pkg/blockcontroller/    WaveApp-as-widget runner   │
└──────────────┬──────────────────────────────────────┘
               │ Build & Exec
┌──────────────▼──────────────────────────────────────┐
│           Tsunami SDK (Compiled app binary)          │
│  tsunami/engine/clientimpl.go  App lifecycle         │
│  tsunami/vdom/                 VDOM types & render   │
│  tsunami/build/build.go        Go build pipeline     │
│  tsunami/ui/                   Built-in UI widgets   │
└─────────────────────────────────────────────────────┘
```

---

## 4. File Map — Every Key File

### 4.1 Electron Main Process

| File | Role |
|---|---|
| `emain/emain-menu.ts:147-153` | Adds "New WaveApp Builder Window" menu item |
| `emain/emain-ipc.ts:35-43` | `openBuilderWindow(appId)` — opens or focuses a builder window |
| `emain/emain-builder.ts` (entire file, ~127 lines) | `createBuilderWindow()`, `BuilderWindowType`, lifecycle (focus/blur/close), list of all builder windows |
| `emain/emain.ts:309-310` | Hides builder windows when main window hidden |
| `emain/emain.ts:370` | `windows-updated` event for builder-aware status |

### 4.2 Builder Window UI (Frontend)

| File | Role |
|---|---|
| `frontend/builder/builder-app.tsx` | Root React component. Renders either `BuilderWorkspace` (if draft loaded) or `AppSelectionModal` (picker). Sets `WebkitAppRegion: drag` on title bar. |
| `frontend/builder/app-selection-modal.tsx` | Lists existing WaveApps, "Create New WaveApp" form. Validates names with `AppNameRegex` (max 50 chars). On select, converts `local/` → `draft/` via `MakeDraftFromLocalCommand`. |
| `frontend/builder/builder-workspace.tsx` | Workspace layout with resizable panels |
| `frontend/builder/builder-apppanel.tsx` | The app editing panel (tabs: preview, files, code, secrets, configdata) |
| `frontend/builder/builder-buildpanel.tsx` | Build output terminal view |
| `frontend/builder/tabs/builder-codetab.tsx` | Monaco editor for `app.go` |
| `frontend/builder/tabs/builder-previewtab.tsx` | Live WebView preview of the running app |
| `frontend/builder/tabs/builder-filestab.tsx` | File tree browser |
| `frontend/builder/tabs/builder-secrettab.tsx` | Secret bindings editor |
| `frontend/builder/tabs/builder-configdatatab.tsx` | JSON schema-based config data editor |
| `frontend/builder/store/builder-apppanel-model.ts` | Core state: active tab, code content, env vars, save/load/restart logic, `BuilderAppPanelModel` singleton |
| `frontend/builder/store/builder-buildpanel-model.ts` | Build output state, auto-scroll |
| `frontend/builder/store/builder-focusmanager.ts` | Keyboard focus routing |
| `frontend/app/store/windowtype.ts` | `isBuilderWindow()` — detects `"builder"` window type set by Electron main |

### 4.3 WaveApp Runtime (Go) — The In-App SDK

| File | Role |
|---|---|
| `pkg/waveapp/waveapp.go` | **Primary SDK for WaveApp developers.** `Client` struct, `MakeClient()`, `DefineComponent[P]()`, `RegisterComponent()`, `SetAtomVal()`, `RegisterFileHandler()`, `RegisterUrlPathHandler()`, `RegisterFilePrefixHandler()`, VDOM rendering (full + incremental), RPC connection setup. |
| `pkg/waveapp/waveappserverimpl.go` | RPC server implementation. `VDomRenderCommand()` — receives frontend events, applies them to VDOM root, renders, returns updates. `VDomUrlRequestCommand()` — proxies HTTP requests from WebView to app's URL handlers. |
| `pkg/waveapp/streamingresp.go` | `StreamingResponseWriter` — adapts RPC response channel to `http.ResponseWriter` for URL request handling |

**Key `Client` fields:**
- `Root` / `RootElem` — VDOM root and root element
- `RpcClient` / `RpcContext` — connection to Wave Terminal
- `UrlHandlerMux` — gorilla/mux router for URL path handlers
- `GlobalEventHandler` — optional global keyboard event handler
- `Opts` — `VDomBackendOpts` (CloseOnCtrlC, GlobalKeyboardEvents)

### 4.4 Builder Controller (Go) — Build & Run Orchestration

| File | Role |
|---|---|
| `pkg/buildercontroller/buildercontroller.go` | **Build and run pipeline.** `BuilderController` struct, status FSM (`init→building→running→error→stopped`). `Start()` builds the Go binary via Tsunami SDK and spawns it as a child process. Manages output buffer, status events, env vars. |
| `pkg/waveappstore/waveappstore.go` | **Filesystem operations on `~/waveapps/`.** CRUD for app files, manifest, secret bindings. Namespace management (`draft/` ↔ `local/`). `PublishDraft()`, `RevertDraft()`, `MakeDraftFromLocal()`. Path traversal protection. |
| `pkg/waveapputil/waveapputil.go` | Utility: `GetTsunamiScaffoldPath()`, `ResolveGoFmtPath()`, `FormatGoCode()` |

**Builder status constants** (from `buildercontroller.go:31-36`):
```
BuilderStatus_Init     = "init"
BuilderStatus_Building = "building"
BuilderStatus_Running  = "running"
BuilderStatus_Error    = "error"
BuilderStatus_Stopped  = "stopped"
```

### 4.5 Block Controller — Running WaveApps as Widgets

| File | Role |
|---|---|
| `pkg/blockcontroller/tsunamicontroller.go` | Runs published WaveApps as regular block widgets inside Wave Terminal tabs. `TsunamiController`, `TsunamiAppProc`. Reads manifest metadata, builds + runs the app binary, streams output. Uses the same `build.TsunamiBuildInternal()` pipeline as the builder. |

### 4.6 WaveApp-as-Widget Frontend View

| File | Role |
|---|---|
| `frontend/app/view/tsunami/tsunami.tsx` | `TsunamiViewModel` extends `WebViewModel`. Renders a WaveApp block in a WebView. Watches controller status, loads manifest metadata (title, icon, iconcolor) for the block header. |
| `frontend/app/block/blockregistry.ts:34` | `BlockRegistry.set("tsunami", TsunamiViewModel)` |

### 4.7 Tsunami SDK (Go) — The Underlying Framework

| File | Role |
|---|---|
| `tsunami/engine/clientimpl.go` | **Standalone engine**. `ClientImpl` struct, `GetDefaultClient()`, HTTP listener, SSE event channels, modal system, `RunMain()`, `AppInitFn`. This is what a WaveApp binary uses when NOT running inside Wave Terminal builder (standalone mode). |
| `tsunami/vdom/vdom.go` | VDOM rendering engine — diff/patch, component resolution, event dispatch |
| `tsunami/vdom/vdom_types.go` | VDOM type definitions (`VDomElem`, `VDomBackendUpdate`, `VDomFrontendUpdate`, `VDomRenderUpdate`, `VDomEvent`, etc.) |
| `tsunami/build/build.go` | **Build pipeline.** `TsunamiBuildInternal(BuildOpts)` — scaffolds Go project, runs `go build`, generates manifest JSON from Go source annotations. `BuildOpts` includes scaffold path, SDK version, Go path, output file. |
| `tsunami/build/build-ast.go` | Parses `app.go` AST to extract `AppInit` function presence, manifest metadata |
| `tsunami/ui/table.go` | Built-in table UI component |

### 4.8 RPC Types & Interfaces

| File | Role |
|---|---|
| `pkg/wshrpc/wshrpctypes_builder.go` | **All builder RPC types.** `WshRpcBuilderInterface` (20+ commands), `AppInfo`, `AppManifest`, `AppMeta`, `SecretMeta`, `BuilderStatusData`, command request/response types for read/write/delete/rename app files, start/stop/restart builder, publish, make draft. |

**Key types:**
- `AppInfo` = `{appid, modtime, manifest?}`
- `AppManifest` = `{appmeta: AppMeta, configschema?, dataschema?, secrets: map[string]SecretMeta}`
- `AppMeta` = `{title, shortdesc, icon (fontawesome), iconcolor (HTML color)}`
- `SecretMeta` = `{desc, optional}`
- `BuilderStatusData` = `{status, port?, exitcode?, errormsg?, version, manifest?, secretbindings?, secretbindingscomplete}`

### 4.9 Config & Settings

| File | Role |
|---|---|
| `pkg/wconfig/settingsconfig.go:73` | `FeatureWaveAppBuilder bool` field |
| `pkg/wconfig/settingsconfig.go:179-183` | Tsunami settings: `TsunamiScaffoldPath`, `TsunamiSdkReplacePath`, `TsunamiSdkVersion`, `TsunamiGoPath` |
| `pkg/wconfig/metaconsts.go` | `ConfigKey_FeatureWaveAppBuilder = "feature:waveappbuilder"` |

---

## 5. Runtime Data Flow (End-to-End)

### 5.1 Opening the Builder
```
User clicks "New WaveApp Builder Window" (File menu)
  → emain-menu.ts calls openBuilderWindow("")
  → emain-ipc.ts checks for existing builder window with same appId
  → If not found: calls createBuilderWindow(appId)
    → emain-builder.ts creates BrowserWindow
    → Generates a unique builderId (UUID)
    → If appId provided, sets RTInfo "builder:appid"
    → Loads frontend index.html with initOpts: {builderId, clientId, windowId}
  → Frontend detects windowType="builder" (windowtype.ts)
  → builder-app.tsx renders AppSelectionModal (no draft loaded yet)
```

### 5.2 Creating / Editing an App
```
User selects or creates an app
  → app-selection-modal.tsx calls SetRTInfoCommand("builder:appid", appId)
  → If local/: calls MakeDraftFromLocalCommand → draft/
  → globalStore.set(atoms.builderAppId, draftAppId)
  → UI transitions from AppSelectionModal → BuilderWorkspace
  → builder-apppanel-model.ts initializes:
    - Subscribes to "builderstatus" events
    - Loads app.go via ReadAppFileCommand
    - Loads env vars via GetRTInfoCommand
    - Subscribes to "waveapp:appgoupdated" events
```

### 5.3 Build & Run Pipeline
```
User saves app.go (or auto-save triggers)
  → builder-apppanel-model.ts.saveAppFile()
    → WriteAppGoFileCommand (formats code server-side)
    → debouncedRestart() → startBuilder()
  → StartBuilderCommand(builderId)
  → buildercontroller.go.Start(ctx, appId, builderEnv)
    → Status: "building"
    → build.TsunamiBuildInternal(BuildOpts{
        AppPath, AppNS, ScaffoldPath, SdkVersion,
        NodePath, GoPath, OutputFile, OutputCapture
      })
    → Creates temp Go project, writes go.mod, scaffolds
    → go build → binary at appPath/bin/app
    → If success: spawns binary as child process
      → Binary connects back to Wave Terminal via domain socket (JWT auth)
      → Binary calls VDomCreateContextCommand to establish VDOM context
    → Status: "running" with port
    → If error: Status: "error" with errormsg
```

### 5.4 VDOM Render Cycle (App ↔ Wave Terminal)
```
1. App binary calls client.fullRender()
   → Serializes VDOM tree → VDomBackendUpdate
   → Sends to Wave Terminal via RPC

2. Wave Terminal frontend renders VDOM in WebView
   User interacts (clicks, types, scrolls)

3. Frontend sends VDomFrontendUpdate (events, state sync)
   → waveappserverimpl.go VDomRenderCommand()
   → Applies atom updates to Root
   → Dispatches events to components
   → Re-renders (full or incremental)
   → Returns VDomBackendUpdate

4. App binary registers URL handlers
   → WebView makes HTTP requests
   → Frontend proxies via VDomUrlRequestCommand
   → App serves content (static files, dynamic endpoints)
```

### 5.5 Publishing
```
User clicks "Publish"
  → PublishAppCommand({appid: "draft/myapp"})
  → waveappstore.go.PublishDraft(draftAppId)
    → Copies draft/ → local/
  → Returns localAppId
```

---

## 6. Key RPC Commands (Builder)

All defined in `pkg/wshrpc/wshrpctypes_builder.go`:

| Command | Direction | Purpose |
|---|---|---|
| `ListAllAppsCommand` | Frontend→Backend | List all WaveApps in `~/waveapps/` |
| `ListAllEditableAppsCommand` | Frontend→Backend | List apps with draft prioritized over local |
| `ReadAppFileCommand` | Frontend→Backend | Read any file from app directory (base64) |
| `WriteAppGoFileCommand` | Frontend→Backend | Write + format `app.go`, returns formatted code |
| `StartBuilderCommand` | Frontend→Backend | Build + run the app |
| `StopBuilderCommand` | Frontend→Backend | Kill the running app process |
| `RestartBuilderAndWaitCommand` | Frontend→Backend | Restart and wait for build result |
| `GetBuilderStatusCommand` | Frontend→Backend | Get current status, manifest, port, etc. |
| `GetBuilderOutputCommand` | Frontend→Backend | Get build output lines |
| `DeleteBuilderCommand` | Frontend→Backend | Stop builder + remove controller |
| `PublishAppCommand` | Frontend→Backend | Publish draft→local |
| `MakeDraftFromLocalCommand` | Frontend→Backend | Create draft from local |
| `WriteAppSecretBindingsCommand` | Frontend→Backend | Save secret bindings |

---

## 7. Events

| Event Type | Scope | Payload | When |
|---|---|---|---|
| `"builderstatus"` | `builder:<id>` | `BuilderStatusData` | Builder status changes (build/run/error/stop) |
| `"waveapp:appgoupdated"` | appId string | — | `app.go` updated externally |
| `"controllerstatus"` | `block:<id>` | BlockControllerRuntimeStatus | Tsunami block widget status changes |
| `"blockclose"` | block ORef | — | App block closed → triggers `doShutdown()` |

---

## 8. State Management (Frontend)

Uses **Jotai** atoms via `globalStore`. Key atoms:

| Atom | Location | Value |
|---|---|---|
| `atoms.builderId` | `global-atoms.ts` | Current builder UUID |
| `atoms.builderAppId` | `global-atoms.ts` | Current `draft/appname` |
| `builderStatusAtom` | `builder-apppanel-model.ts` | `BuilderStatusData` |
| `codeContentAtom` | `builder-apppanel-model.ts` | Current `app.go` text in editor |
| `originalContentAtom` | `builder-apppanel-model.ts` | Saved `app.go` text (for dirty detection) |
| `saveNeededAtom` | `builder-apppanel-model.ts` | `codeContent !== originalContent` |
| `envVarsArrayAtom` | `builder-apppanel-model.ts` | Builder environment variables |
| `activeTab` | `builder-apppanel-model.ts` | `"preview" \| "files" \| "code" \| "secrets" \| "configdata"` |

---

## 9. Directory Structure Summary

```
~/waveapps/
├── draft/
│   └── myapp/
│       ├── app.go
│       ├── manifest.json
│       ├── secret-bindings.json
│       └── bin/
│           └── app (or app.exe)    ← compiled binary (cached)
└── local/
    └── myapp/                       ← published copy
        ├── app.go
        ├── manifest.json
        └── secret-bindings.json
```

---

## 10. Tsunami Build Settings

These settings control the build pipeline (stored in `settings.json`):

| Setting Key | Default | Purpose |
|---|---|---|
| `tsunami:scaffoldpath` | `<resources>/tsunamiscaffold` | Path to Go scaffold templates |
| `tsunami:sdkreplacepath` | (none) | Local SDK path for development |
| `tsunami:sdkversion` | `v0.12.4` | SDK version to use in go.mod |
| `tsunami:gopath` | System `go` binary | Path to Go executable |

---

## 11. WaveApp SDK Quick Reference

A minimal WaveApp (`app.go`):

```go
package main

import "github.com/wavetermdev/waveterm/pkg/waveapp"

func main() {
    client := waveapp.MakeClient(waveapp.AppOpts{})
    client.AddSetupFn(func() {
        // Register components, state, URL handlers
        waveapp.DefineComponent(client, "App", func(ctx context.Context, props struct{}) any {
            // Return VDOM elements using vdom.H(), vdom.E(), etc.
            return nil
        })
    })
    client.RunMain()
}
```

**Key SDK APIs** (see `pkg/waveapp/waveapp.go` for full reference):
- `MakeClient(AppOpts)` — create client (opts: CloseOnCtrlC, GlobalKeyboardEvents, GlobalStyles, RootComponentName, NewBlockFlag, TargetNewBlock, TargetToolbar)
- `DefineComponent[P](client, name, renderFn)` — typed component registration
- `client.SetAtomVal(name, val)` / `client.GetAtomVal(name)` — state
- `client.RegisterFileHandler(path, FileHandlerOption)` — static file serving
- `client.RegisterUrlPathHandler(path, http.Handler)` — dynamic endpoints
- `client.RegisterFilePrefixHandler(prefix, optionProvider)` — directory serving
- `client.SetGlobalEventHandler(handler)` — global keyboard events
- `client.RunMain()` — blocks until shutdown

---

## 12. Two Runtime Paths

WaveApps have two execution paths:

| Path | Trigger | Runner | Use Case |
|---|---|---|---|
| **Builder** | User edits in Builder window | `buildercontroller.go` → child process | Development, hot-reload, preview |
| **Block Widget** | User adds WaveApp as a block in a terminal tab | `tsunamicontroller.go` → child process | Production usage as a widget |

Both paths use the same `build.TsunamiBuildInternal()` pipeline and spawn the same binary. The difference is lifecycle management and metadata handling.

---

## 13. Common Developer Tasks

### Adding a new builder tab
1. Create tab component in `frontend/builder/tabs/`
2. Add tab type to `TabType` union in `builder-apppanel-model.ts`
3. Wire up in `builder-apppanel.tsx`

### Adding a new RPC command
1. Define request/response types in `pkg/wshrpc/wshrpctypes_builder.go`
2. Add method to `WshRpcBuilderInterface`
3. Implement in `pkg/wshrpc/wshserver/wshserver.go`
4. Auto-generate frontend client via `cmd/wsh`

### Changing the build pipeline
1. `tsunami/build/build.go` — `TsunamiBuildInternal()` is the main entry
2. `tsunami/build/build-ast.go` — AST parsing for `app.go`
3. Build output flows through `OutputCapture` → `buildercontroller.outputBuffer` → frontend via `GetBuilderOutputCommand`

### Debugging the VDOM render cycle
1. App side: `pkg/waveapp/waveapp.go` `fullRender()` / `incrementalRender()`
2. Transport: `pkg/waveapp/waveappserverimpl.go` `VDomRenderCommand()`
3. Frontend: `tsunami/frontend/src/vdom.tsx` (VDOM reconciliation in React)
