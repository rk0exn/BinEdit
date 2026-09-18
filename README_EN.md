# BinEdit

A binary editor rendered with Win32, Direct3D 11, Direct2D 1.1, and DirectWrite, requiring no external runtime.

For development architecture, invariants, and the change checklist, see [CODEX.md](CODEX.md). Developer-facing comments in the source code are written entirely in English.

日本語のREADMEは[こちら](README.md)です。

## Build

Open `BinEdit.slnx` in Visual Studio 2026 and build `x64` `Debug` or `Release`. The project uses the `v145` toolset.

```powershell
& 'C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe' BinEdit.slnx /m /p:Configuration=Release /p:Platform=x64
```

### Tests

After building, the following tests verify each area of functionality (all PowerShell scripts take `-Configuration Release`).

| Test | Verifies |
|---|---|
| `x64\Release\BinEdit.Core.Tests.exe` | Core file I/O, partial saves, external conflicts, search, encoding, undo/redo, a 20 KiB random-edit scenario, and the close/detach permission policy for the main window's last remaining tab (unmodified untitled, modified untitled, named file, multiple tabs, detached window, and invalid index — all UI-independent) |
| `tests\AsyncFileIoUiTests.ps1` | Memory-mapped reads/asynchronous saves of a 384 MiB file, UI responsiveness during processing, menu disabling, input rejection, and data integrity after inserts crossing a 4 MiB boundary |
| `tests\EditorUiStress.ps1` | Sending a large volume of hex input and Backspace/Delete/insert/overwrite operations at random positions into a real window, then byte-for-byte verifying the saved result |
| `tests\ScrollBarUiTests.ps1` | Thumb dragging on the D2D scrollbar, track paging, and the absence of native scroll styling |
| `tests\MiddleScrollUiTests.ps1` | Middle-click accelerating up/down auto-scroll |
| `tests\BitToolUiTests.ps1` | Launching the bit-manipulation tool on an empty file and editing the input value/operand |
| `tests\ExternalChangeUiTests.ps1` | Automatic reload after a file is atomically replaced externally, the "Reload / Keep Editing" prompt when there are unsaved changes, and the saved content after each choice |
| `tests\TabDockingUiTests.ps1` | Tab separation across multiple files, menu differences between main and sub windows, focus, re-docking, and cascading exit when the main window closes |
| `tests\TabStripScrollUiTests.ps1` | Shift+wheel horizontal scrolling and hit testing with a large number of tabs |
| `tests\UntitledCloseUiTests.ps1` | The save confirmation for a modified untitled tab, and closing the last named file with Ctrl+W to return to untitled |
| `tests\MemoryReleaseUiTests.ps1` | That a 96 MiB file is not duplicated into a private buffer, and that its mapped region is fully released when the tab is closed |
| `tests\MemoryRegionStressUiTests.ps1` | Repeatedly creating and closing sub-windows while sampling virtual memory regions, private bytes, handles, and GDI/USER resources; also map release for large sub-documents, continued operation of Explorer/DWM, and taskbar UI thread responsiveness |
| `tests\SearchAsyncUiTests.ps1` | Asynchronous search worker threads, UI responsiveness during search, cancellation on exit, and F3 selection position |
| `tests\RecoveryUiTests.ps1` | Force-terminating a real process holding unsaved files in both the main and a sub window, then byte-for-byte verifying that on the next launch the recovered content is consolidated into the main window's tabs |

## Operation

- `Ctrl+O` / `Ctrl+S`: Open / Save
- `Ctrl+N`: Create a new untitled tab
- `Ctrl+W`: Close the current tab (prompts to save if unsaved. After closing the main window's last named file or a modified untitled tab, it returns to a new untitled tab. Closing the last tab of a sub-window closes that sub-window)
- `Ctrl+Shift+S`: Save the current tab as
- `Alt+F4`: Exit the current window
- `Ctrl+Tab` / `Ctrl+Shift+Tab`: Switch to the next/previous tab
- When tabs don't fit the available width, `Shift`+mouse wheel scrolls the tab strip left/right
- File names that don't fit the tab width are truncated with an ellipsis at the end; hovering over a tab name shows the untruncated file name in a D3D11/D2D1 tooltip
- Dragging a tab left/right: reorders tabs within the same window
- Dragging a tab outside the tab strip: detaches it into a new window while preserving edit state and undo/redo history (only when the source window has two or more tabs)
- Dragging a detached tab onto another BinEdit window's tab strip: re-docks it at that position
- Passing multiple files on the command line, or running "Open" while editing an existing file, creates a tab per file
- A normal duplicate launch is suppressed via the named mutex `MX_BinEdit_F80F`, forwarding command-line files to the existing process to open in a new tab
- `Ctrl+F` / `F3`: Binary or text search / find next. The search type can be switched with the mouse, Space/arrow keys, or Alt+B/Alt+T
- `Ctrl+Z` / `Ctrl+Y`: Undo / Redo
- "Tools" → "Bit Manipulation Tool": computes AND, OR, XOR, NOT, left/right shift/rotate, and add/subtract/multiply/divide on an input value and operand in 8-bit units
- Click, Shift+arrow: selection. In the hex pane, type hex digits; in the character pane, type characters to edit
- `Insert`: toggles insert/overwrite mode. Starts in insert mode; the current mode is shown in the status bar
- `Tab`: switches between the hex and character input panes. The active side is indicated by an accent line in the header and "Input" in the status bar
- Typing past the end advances to the next virtual EOF cell, and the file is extended only if input continues from there
- `Backspace`: deletes the previous input unit. One byte in the hex pane; one character, based on the selected character format, in the character pane
- `Delete`: in insert mode, deletes forward from the current position. In overwrite mode, replaces the target byte with `00`
- Dragging the D3D11/D2D1 scrollbar on the right edge, clicking the track, mouse wheel, or Page Up/Page Down: scrolls
- Middle-clicking the data area: starts auto-scroll. The dead zone, upward, and downward regions are each shown with a dedicated cursor, accelerating the farther you move from the start point above or below it. Ends with another middle-click or with a normal click/key/wheel action
- "Edit" → "Mark Selection" marks a range. Disabled when the caret is on the virtual EOF cell
- The "Color" menu sets the mark color and the color for the byte value at the current position. Setting/clearing the current byte value's color is disabled when the caret is on the virtual EOF cell

### Color scheme and language

Color scheme, theme, and character format are saved to `%LOCALAPPDATA%\BinEdit\profile.json`. The key colors for selection, primary buttons, focus, and the caret follow the UI accent color set by the current Windows user. When high contrast is enabled, the accent color is not used; Windows' high-contrast system colors take precedence instead.

Color selection does not use the standard Windows color dialog; it uses BinEdit's own D3D11/D2D1 color picker. It supports a saturation/value plane, hue bar, current-vs-new color comparison, direct RGB (0–255) and HEX (`#RRGGBB`) entry, mouse dragging, clipboard, keyboard fine-tuning, and Per-Monitor V2 DPI changes.

The UI supports Japanese and English. "Auto" in the "言語 / Language" menu follows the Windows user UI language; an explicit selection is saved to `language` in `profile.json`.

### Window architecture

The main window created at startup is the sole instance of the `BinEdit.MainWindow` class within the process. A top-level window detached from a tab is treated as a sub-window of the `BinEdit.DetachedTabWindow` class, and does not show "Theme," "Language," or "File" → "Exit." Since sub-windows use the main window as their Win32 owner, they maintain an independent normal frame, movement, resizing, and input focus, while repeated detaching/closing does not accumulate taskbar thumbnail resources. Setting changes propagate from the main window to all sub-windows. Closing the main window prompts for unsaved changes across all windows, and on success closes all sub-windows in cascade.

The search screen, bit-manipulation tool, and "About BinEdit" screen are also rendered with D3D11/D2D1/DirectWrite rather than using native child controls. They support Light, Dark, and High Contrast palettes, DPI changes, and keyboard operation. The bit-manipulation tool edits an input value and operand and displays the result. Arithmetic overflow wraps to the lower 8 bits; shift/rotate accepts 0–7, and division accepts any nonzero operand.

### Asynchronous search

Search runs asynchronously against a shared, immutable snapshot created from the editing data. The candidate range is split across up to 16 workers to match hardware parallelism, while preserving overlapping matches, up to 10,000 highlighted results, and F3 wraparound. The pattern is split at wildcard bytes into literal fragments; an all-wildcard pattern enumerates candidate positions directly. Otherwise, the byte position with the lowest sampled document frequency becomes the anchor, scanned with a two-byte SSE2 SIMD seed in 16-byte blocks, and only those candidates are verified against the complete pattern. A worker stops early once the highlight cap and the F3 position are settled, so scans of frequently matching patterns avoid scanning every position. The UI thread and R2PR rendering are never blocked during search, and outdated search generations are cooperatively canceled on edit, tab switch, file switch, or window close. Rapid consecutive input coalesces re-searches into a 150 ms window. The snapshot's lifetime is retained until the coordinator and all workers finish, so background processing never holds a borrowed reference to the caller's byte buffer.

### File I/O

File loading, normal save, save-as, reload on external change, and temporary evacuation for administrator restart all run on an owned file I/O worker. During processing, the UI thread continues to respond to rendering, DPI changes, resizing, and system theme changes, showing a D2D indicator and Japanese/English progress text. Meanwhile, editing, menus, tab operations, and exit are disabled, and the UI never references document data currently borrowed by the worker. Additional single-instance IPC or drag-and-drop paths that arrive in the meantime are queued into an owned queue of up to 256 entries and opened sequentially once the current I/O completes.

Non-empty files are mapped read-only with `CreateFileMappingW(PAGE_READONLY)` and `MapViewOfFile(FILE_MAP_READ)`, relying on the OS's demand paging. The entire file is never duplicated into a private `std::vector`. Edits are represented as a persistent piece table combining unmodified mapped ranges with small owned blocks, so a single-byte edit or insertion never expands an entire large file into RAM. Rendering only references the currently displayed bytes; search streams in fixed chunks of up to 1 MiB per worker, and save/recovery logging streams in fixed 4 MiB chunks. When a file or tab is closed, the final owner of the shared snapshot calls `UnmapViewOfFile` to release the mapped region.

### Dialog rendering

Save confirmations, warnings, and errors do not use `MessageBox`; they are shown in an independent, GDI-based modal dialog. Since this dialog does not depend on D3D11/D2D1, it remains usable even if GPU rendering initialization fails. The info, warning, error, and question glyphs are not drawn from font glyphs but rendered as smoothly downsampled 4x-resolution GDI paths using theme colors, sized modestly within a colored badge with ample padding. Body text and button labels are drawn with transparency onto the theme background, and `Tab`/`Shift+Tab` moves button focus forward/backward. Buttons use the same 14 DIP Segoe UI Variable Text as the D3D-based dialogs, with the primary button in Semibold and the secondary in Normal. 96 DPI (100%) is the logical layout baseline; on a Per-Monitor V2 DPI change, the body text is re-measured at the destination DPI, simultaneously rescaling the window, fonts, padding, buttons, and hit-test regions.

### Appearance

The client area follows a WinUI 3 / Fluent-inspired appearance: a Segoe UI Variable type hierarchy, a single-layer editing surface, rounded selection highlights and controls, subtle borders, and the Windows user's accent color. At high-DPI medium widths, density adjusts automatically, and at narrower widths the character pane is hidden entirely to prioritize the hex pane. This is a visual design built on the existing Win32/D3D11/D2D1 implementation, not a migration to the Windows App SDK or XAML.

The vertical scrollbar, too, is not a non-client-area Win32 control but is rendered with D3D11/D2D1. Thumb and track dimensions follow Per-Monitor V2 DPI; Light/Dark modes use a subtle WinUI-style appearance, while High Contrast uses Windows system colors. Row position is not truncated to `SCROLLINFO`'s 32-bit range; it is computed directly from the file's total row count as a `size_t`.

At the start point of middle-click auto-scroll, D2D up/down direction markers are shown. A dead zone is established around the marker, switching between Windows' auto-pan cursors for stopped/up/down, with quadratic acceleration based on distance for rapidly navigating large files. A dedicated timer is only active during the operation, and updates the data area and scrollbar via R2PR only when the row position actually changes.

The top-level menu bar in Dark theme uses UAH drawing messages to distinguish background, hover, selection, and disabled states. Menu items within dropdowns also update Windows' native menu theme in sync with the selected System/Light/Dark setting. Since UAH and the `uxtheme.dll` exports it uses are not part of Windows' public API, the implementation falls back to standard Windows rendering on environments where they are unavailable, and during High Contrast.

### Restarting with administrator privileges

If access is denied when saving a protected file, the current edits are evacuated to a CNG-generated temporary file, and BinEdit restarts with administrator privileges after UAC consent, carrying the content forward. The internal command line's temporary file, destination path, and recovery ID are quoted following the inverse of `CommandLineToArgvW`'s parsing rules, safely preserving empty strings, whitespace, and backslashes immediately before quotes or at the end. After elevation, the destination's absolute path is reconfirmed via the custom GDI dialog, and the temporary file's name, real path, reparse attributes, and destination type are validated before the atomic save.

### Shutdown protection

While there are unsaved edits, a reason is registered via `ShutdownBlockReasonCreate`. During file I/O, the operation in progress is registered as a shutdown-blocking reason, and `WM_QUERYENDSESSION` refuses to exit without showing modal UI. While unsaved, asynchronous updates to the latest recovery generation begin, and the registered state is synchronized after saving or upon cancellation/completion of session end.

## Safety and crash recovery

### Partial saves and atomic replacement

A normal fixed-length overwrite edit writes only the modified contiguous range back to the original file. For edits that change length or offset structure, such as insertion or deletion, calling `SetEndOfFile` on a mapped stream is rejected by Windows with `ERROR_USER_MAPPED_FILE`, and rewriting the mapped source forward could corrupt not-yet-copied subsequent data; so instead, the content is streamed in fixed 4 MiB buffers to a temporary file in the same directory with a CNG random name, then atomically swapped in. The entire content is never expanded into RAM.

Before beginning a non-atomic partial save, a complete recovery snapshot of the same revision as the save target is committed via flush + atomic replace. If the recovery record cannot be committed, the original file is not partially updated; the operation instead falls back to a full atomic save. If the number of discrete changed ranges exceeds 4,096, rather than merging them into one huge range, the operation also falls back to a full atomic save. This ensures that if a force-termination occurs mid partial-save, the full content just before the save can be restored on the next launch.

### Save-time validation

Each time a save occurs, the current `FILE_ATTRIBUTE_READONLY` of the target is re-checked; if read-only, the save is refused without suggesting an administrator restart. Attributes are not retained from load time, so if attributes are set or cleared after loading, the state at save time is what applies. During a save transaction, writes/replacements from other processes are refused, and `FlushFileBuffers` is called before completion. After Save As, after crash recovery data, or after choosing "Keep Editing" on an external conflict, the assumptions behind incremental changed ranges no longer hold, so the operation falls back to a full snapshot. This full save writes all bytes to a CNG-random-named temporary file in the same directory, then publishes it atomically — via `ReplaceFileW` for an existing file, or `MoveFileExW` for a new one. The profile is also capped at 1 MiB and saved the same way, via flush + atomic replace.

### Monitoring external changes

The parent directory of an open file is monitored via Win32 change notifications; on notification, the volume/file ID, last modified time, and size are compared. If there are no unsaved changes, the external version is automatically reloaded. If there are unsaved changes, a custom GDI dialog (in Japanese/English) lets the user choose "Reload" or "Keep Editing." Choosing to continue records the current external version as the new conflict baseline, and switches the next save to a full snapshot. If the file is externally deleted or moved, the in-memory content is retained as unsaved, so accidentally closing does not lose the content. The same file identity is re-checked immediately before saving, so updates that occur between a change notification and the save operation are never silently overwritten.

### Recovery records

Unsaved tabs create recovery records under `%LOCALAPPDATA%\BinEdit\Recovery`. These records use a protected DACL limited to the user and SYSTEM, SHA-256 integrity checking, a CNG-generated 128-bit ID, and flush + atomic replace, and enable EFS as well on volumes that support it. The first change updates immediately; after input stops, updates occur at 500 ms, and during continuous input, at intervals of up to 2 seconds. An interrupted temporary record is validated for a safe name and removed on the next launch; incomplete or tampered records are not restored. Even if unsaved files after a crash are split across multiple windows, approved recovery records are all consolidated into the tabs of the main window on the next launch; sub-windows are not automatically recreated.

### Resource release

When closing a file or replacing it with another, the byte buffer, undo/redo history, changed-range tracking, search results, and markup's reserved capacity are all swapped out for empty containers rather than being retained by the long-lived main window. Reuse capacity is not retained via a simple `clear()`. When closing a sub-window along with its last tab, D3D/DXGI device resources are released immediately in `WM_DESTROY`, and the sub-controller is destroyed on the main window's message queue after the window procedure returns. This ensures the closed sub-window's document, history, search state, and renderer are not left behind in the owning array.

### Handling forced termination

`taskkill /F` and `TerminateProcess` cannot run cleanup code inside the target process, but since Windows terminates all threads of that process, no orphaned threads remain outside the process. On a normal exit, all `std::jthread`s are sent a stop request and joined; on a force-termination, the atomic recovery record left outside the process is used on the next launch. `RegisterApplicationRestart` is also registered for crash/hang cases (force-termination itself is not subject to automatic restart).

### Security mitigations

The executable enables `/GS`, `/sdl`, Spectre mitigations, CFG, EH continuation metadata, CET compatibility, DEP, ASLR, and High Entropy VA. At startup it configures safe DLL search, heap corruption termination, strict HANDLE checking, prohibition of legacy extension points, and prohibition of remote/low-integrity image loads. C++ exceptions are never allowed to escape Win32 callbacks; they fail fast and are left to the next recovery. The single-instance IPC verifies the session, user SID, and integrity level of both the destination and source HWND, as well as the transfer size, UTF-16 boundaries, and full paths, before any side effects. It also captures the HWND, PID, and process creation time as a tuple, re-verifying the same tuple and trust conditions immediately before sending or immediately before opening a file, rejecting any window reuse that occurred after the check. If the mutex cannot be created safely, startup is aborted rather than falling back to allowing multiple instances.

## R2PR

This implementation treats Require-To-Partial-Re-Render (R2PR) as damage tracking. Actions like editing, caret movement, search, and scrolling `Require` an update region, and requests within the same message cycle are coalesced into one. At render time, the same rectangle is passed both as the Direct2D clip and as the dirty rect for `IDXGISwapChain1::Present1`. If nothing has changed, neither rendering nor Present occurs, and since display is vsynced, it also supports 240 Hz displays.

The data area only displays rows that fit entirely between the header and the status bar. The leftover fractional area below the last row is excluded from both rendering and hit-testing, so depending on DPI or window size, the next offset row is never left isolated and orphaned. The row count used for display is also shared consistently across scroll range and page navigation.

## Source layout

| File | Contents |
|---|---|
| `src/App.*` / `src/TabPolicy.h` | Win32 message handling, editor state, UI-independent tab command eligibility checks |
| `src/Document.*` | Byte data, partial/atomic saves, external change monitoring, parallel search, history |
| `src/Renderer.*` | D3D11/D2D1 rendering of the main screen and R2PR |
| `src/DialogSurface.*` | Shared GPU rendering foundation for custom dialogs |
| `src/ColorPickerDialog.*` / `src/ColorMath.*` | The custom color picker and HSV/RGB conversion |
| `src/MessageDialog.*` | GDI-based confirmation/warning/error dialog independent of D3D |
| `src/SearchDialog.*` / `src/BitToolDialog.*` / `src/AboutDialog.*` | Fully themed screens |
| `src/BitOperations.*` | UI-independent 8-bit arithmetic and input constraints |
| `src/Profile.*` / `src/Localization.*` | Settings, color scheme, Japanese/English |
| `src/RecoveryStore.*` | Protection, validation, atomic saving, and restoration of unsaved snapshots |
| `src/Security.*` | Process mitigations, IPC sender verification, fail-fast at the Win32 exception boundary |
| `src/CommandLineEscaping.h` | Strict quoting and construction of Win32 command-line arguments |
| `src/ThemeMenu.*` | Theme synchronization for the Win32 menu bar and dropdown items |
| `tests/CoreTests.cpp` | Real-file tests for the UI-independent core |
| `tests/*.ps1` (various) | Tests via real HWNDs for asynchronous file I/O, editing, scrollbar, middle-click accelerated scrolling, the bit-manipulation tool, external changes, memory/GUI resource release and taskbar responsiveness on file/sub-window close, asynchronous search, main/sub windows, tab-strip horizontal scrolling, closing the last tab, and recovery after force-terminating multiple windows (see the Tests table above for individual script names) |