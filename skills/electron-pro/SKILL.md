---
name: electron-pro
description: Electron desktop apps: main and renderer architecture, IPC contracts, packaging, auto-update, security hardening, and native integration.
---

# Electron Pro

Build Electron applications that feel like real desktop software without inheriting avoidable browser-security mistakes. This skill prioritizes clear process boundaries, typed IPC, secure preload APIs, predictable packaging, and update strategies that work across macOS, Windows, and Linux.

## When to Use

- Building or refactoring an Electron desktop application
- Designing main-process, preload, and renderer boundaries
- Adding filesystem, tray, notifications, printing, or native desktop integration
- Hardening BrowserWindow security and IPC contracts
- Setting up packaging, signing, notarization, and auto-update flows
- Fixing performance, memory leaks, or renderer crashes

## Workflow

1. Split responsibilities up front.
   Put OS and privileged operations in the main process, expose only a narrow preload API, and keep the renderer focused on UI state.
2. Design IPC as a contract, not a shortcut.
   Define named channels, typed payloads, validation, and explicit error responses instead of ad hoc message passing.
3. Lock down the renderer.
   Enable `contextIsolation`, disable `nodeIntegration`, and expose only minimal vetted APIs through `contextBridge`.
4. Plan local data and updates early.
   Decide where config, caches, logs, and user data live, then choose auto-update and signing flows that match the target platforms.
5. Measure desktop-specific performance.
   Watch renderer memory, preload cost, large bundle size, window startup time, and idle CPU usage.
6. Package like a product.
   Build signing, notarization, installer behavior, crash reporting, and rollback expectations into the release process.

## Security Rules

- `contextIsolation: true` by default
- `nodeIntegration: false` by default
- No direct renderer access to filesystem, shell, child processes, or secrets
- Validate every IPC payload at the boundary
- Restrict `shell.openExternal()` to approved protocols and hosts
- Treat preload as a tiny audited API surface, not a dumping ground

## Architecture Guide

| Layer | Owns |
|---|---|
| Main process | Windows, menus, tray, filesystem, updates, native integrations |
| Preload | Safe bridge between privileged APIs and renderer |
| Renderer | UI, state, presentation logic, optimistic interactions |

## Packaging Checklist

- Platform-specific bundle IDs and code signing
- macOS notarization and hardened runtime
- Windows installer/update path and certificate handling
- Linux package format strategy if supported
- Crash reporting and log collection
- Auto-update channel strategy for stable, beta, and internal builds

## Output Format

When using this skill, produce:

1. **Process architecture** - main, preload, renderer, and IPC boundaries
2. **Security plan** - BrowserWindow flags, IPC validation, and privileged API surface
3. **Implementation plan** - native features, local storage, and update flow
4. **Testing plan** - IPC, packaging, update, crash, and OS-integration checks
5. **Release checklist** - signing, notarization, installers, telemetry, and rollback notes
