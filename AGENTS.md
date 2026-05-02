# AGENTS.md

## Project Snapshot

Ace3 is a standalone bundle of WoW addon development libraries, not a normal gameplay addon. Treat every exported method and error string as public API unless the surrounding code clearly marks it private. This checkout's root TOC currently targets `## Interface: 30300`, so preserve Lua 5.1 and older WoW API compatibility unless a task explicitly asks for a broader compatibility update.

## Structure

- `Ace3.toc` is the standalone bundle manifest. It controls addon load order: `LibStub`, `CallbackHandler`, the Ace libraries, then `Ace3.lua`.
- `Ace3.lua` and `Bindings.xml` are standalone developer helpers for the Ace3 bundle, including `/ace3`, `/rl`, and `/print`.
- `LibStub/` and `CallbackHandler-1.0/` are foundational libraries used by most other modules.
- Each `Ace*-3.0/` directory usually contains one `*.lua` implementation and one `*.xml` loader.
- `AceConfig-3.0/` contains nested libraries: `AceConfigRegistry-3.0`, `AceConfigCmd-3.0`, and `AceConfigDialog-3.0`. Its XML includes those before loading `AceConfig-3.0.lua`.
- `AceGUI-3.0/` contains the core GUI library and `widgets/`. `AceGUI-3.0.xml` defines widget/container load order.
- `AceComm-3.0/ChatThrottleLib.lua` is loaded before `AceComm-3.0.lua` and is a required dependency.
- `tests/` contains Lua 5.1 test scripts plus a stubbed WoW API in `tests/wow_api.lua`.

## Coding Conventions

- Keep code Lua 5.1 compatible. Avoid Lua 5.2+ features such as `_ENV`, `goto`, `table.pack`, `table.unpack`, and bitwise operators.
- Prefer local aliases for Lua and WoW APIs near the top of files, matching the existing style.
- Avoid new globals. If a global is intentional, document it with a `-- GLOBALS:` comment where appropriate and update `.luacheckrc`.
- Preserve exact library major names such as `"AceAddon-3.0"` and `"AceGUI-3.0"`.
- Keep public error message wording stable when tests or downstream callers may depend on it.
- Do not add per-library `.toc` files unless the task is explicitly about packaging; they are ignored by `.gitignore` in this repository.

## LibStub And Upgrade Rules

- Most libraries use `local MAJOR, MINOR = "...", n` followed by `LibStub:NewLibrary(MAJOR, MINOR)`.
- If `NewLibrary` returns nil, the file must return immediately because a newer or equal copy is already loaded.
- Bump the minor version when changing runtime behavior that must replace older loaded copies. Never lower or randomize minor versions.
- Preserve upgrade-safe state patterns such as `lib.table = lib.table or {}` and explicit cleanup of obsolete structures.
- Re-embed upgraded mixins after defining them. Existing libraries usually finish with a loop over `lib.embeds`.

## XML And Load Order

- XML files are part of the runtime contract. If a Lua file is added, removed, or reordered, update the relevant XML and audit `Ace3.toc`.
- Keep dependency order explicit. For example, `AceComm-3.0.xml` loads `ChatThrottleLib.lua` before `AceComm-3.0.lua`.
- For nested libraries, use `<Include>` for child XML files and `<Script>` for Lua implementation files, following the existing pattern.

## AceGUI Notes

- Widgets declare `local Type, Version = "...", n`, fetch AceGUI with `LibStub("AceGUI-3.0", true)`, and return early if the registered widget version is newer or equal.
- Constructors should create frames, attach scripts, copy methods, and return `AceGUI:RegisterAsWidget(widget)` or `AceGUI:RegisterAsContainer(widget)`.
- Register widgets with `AceGUI:RegisterWidgetType(Type, Constructor, Version)`.
- `OnAcquire` must restore default visible state. `OnRelease` should clear transient data, detach scripts or callbacks when needed, hide frames, and leave objects safe for recycling.
- If adding a widget/container, add it to `AceGUI-3.0/AceGUI-3.0.xml` in a sensible order.

## Testing And Verification

- Lint with:
    ```sh
    luacheck . --no-color -q
    ```
- Run tests from inside `tests/`; the scripts use relative globs and `dofile` paths:
    ```bat
    cd tests
    runall.bat
    ```
    On Windows, `runall.bat` defaults to `lua5.1.exe`. In PowerShell you can override it with `$env:lua = "path\\to\\lua.exe"` before running the batch file.
- On Unix-like shells:
    ```sh
    cd tests
    lua=lua5.1 ./runall.sh
    ```
- Individual tests can be run from `tests/`, for example `lua5.1 AceDB-3.0.lua`.
- `tests/check_globals.sh` uses `luac -p -l` to find accidental global writes in non-test Lua files.
- When changing FrameXML behavior, widget layout, events, saved variables, comms, or hooks, also verify in-game where possible. The Lua tests do not fully emulate the WoW client.

## Test Naming

- `tests/runall.bat` and `tests/runall.sh` discover most tests with the `*-?.*.lua` pattern. If a new test does not match that pattern, update both scripts or document how to run it.
- Tests normally load `wow_api.lua`, `LibStub.lua`, any required helper libraries, then the real library file from `..`.
- Keep test assertions simple and Lua 5.1 compatible. Several tests assert error substrings, so be deliberate when editing validation messages.

## Working Safely

- Check `git status --short` before edits. This addon may live inside an active WoW installation, and unrelated local changes should be preserved.
- Keep changes narrowly scoped to the affected library and its tests.
- Avoid broad formatting churn in legacy files. Many files use longstanding style and comments from upstream Ace3.
- Update `.luacheckrc` only when a new WoW API/global is intentionally referenced.
- Packaging metadata lives in `.pkgmeta`; it currently packages as `Ace3` and ignores `tests/`.
