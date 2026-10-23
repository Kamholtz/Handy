# Plan: expose Handy transcription status to Talon

## Objective

Make Handy expose its current speech pipeline state in a form that Talon can
reliably inspect, even when the Handy settings window is hidden or minimised.
The primary integration should be the native main-window title. A JSON status
file should also be investigated and, if practical, implemented as a more
structured alternative for Talon configurations that can watch files.

The status must describe the backend operation rather than a frontend visual
state. In particular, an always-on microphone stream must not be reported as a
user recording session: Handy should report `recording` only while it is
capturing audio for a transcription request, and `transcribing` after capture
has stopped while speech recognition is still running.

## Findings from the current implementation

- The main Tauri window is created in `src-tauri/src/lib.rs` with the initial
  title `Handy`.
- `src-tauri/src/actions.rs` is the central operation path. It sets the tray
  state to `Recording` when capture begins, `Transcribing` immediately when
  capture stops, optionally shows `Processing` during post-processing, and
  returns to `Idle` on success, cancellation, empty input, errors, or failed
  startup.
- `src-tauri/src/managers/audio.rs` owns the actual recording lifecycle and
  already maintains a lock-free `is_recording()` indicator. Its internal
  states are `Idle`, `Recording`, and `Stopping`.
- `src-tauri/src/transcription_coordinator.rs` serialises shortcut, signal,
  and CLI lifecycle transitions, including a busy `Processing` stage and
  remembered starts.
- `src-tauri/src/utils.rs` centralises cancellation and already resets the tray
  and recording overlay to idle.
- The existing tray state is useful for confirming the intended state model,
  but it is not a good Talon interface: it is platform-specific, has no stable
  textual value, and does not expose a file path.
- Handy already uses a portable-aware application-data directory through
  `src-tauri/src/portable.rs`, so a status file can use the same location on
  installed and portable builds.

## Recommended state contract

Use a small, fixed, non-localised state vocabulary:

| State | `recording` | `transcribing` | Meaning |
| --- | ---: | ---: | --- |
| `idle` | `false` | `false` | No active transcription operation |
| `recording` | `true` | `false` | Handy is capturing audio for the current request |
| `transcribing` | `false` | `true` | Capture has stopped and speech recognition is running |
| `processing` | `false` | `false` | Optional post-processing/output work is running |
| `error` | `false` | `false` | The operation failed and Handy is returning to idle |

`processing` should remain distinct from `transcribing` so Talon does not
mistake an LLM post-processing delay for speech recognition. The implementation
may briefly publish `error` before settling on `idle`, but it must never leave a
stale busy state after cancellation or failure.

## Window-title implementation

### Machine-readable format

Keep a human-readable prefix and append a fixed ASCII marker with no translated
content:

```text
Handy | handy_state=idle
Handy | handy_state=recording
Handy | handy_state=transcribing
Handy | handy_state=processing
Handy | handy_state=error
```

The `handy_state=` token should be the only contract Talon parses. This makes
the extraction insensitive to the window's visibility, locale, and any future
human-facing title text. A Talon matcher can use an anchored expression such
as `^.*\\| handy_state=(idle|recording|transcribing|processing|error)$` and then
test for `recording` or `transcribing`.

### Where to update it

1. Add a small backend status module or helper, for example
   `src-tauri/src/integration_status.rs`, containing the state enum, title
   formatter, and a single `publish_status(app, state)` function.
2. Have the helper locate the `main` webview window and call Tauri's native
   title setter. Treat a missing window as a non-fatal condition because Handy
   can start hidden and can run headless CLI operations.
3. Publish `idle` during normal startup after the main window is created, so
   Talon never sees an old title from a previous run.
4. Publish `recording` at the same successful transition that starts capture.
   Do not publish it merely when the shortcut is pressed or while model loading
   is still failing; use the actual recording-start result.
5. Publish `transcribing` immediately when `TranscribeAction::stop` begins the
   asynchronous transcription path, before the potentially slow audio drain.
6. Publish `processing` when post-processing starts, if post-processing is
   enabled.
7. Publish `idle` on every existing completion and cancellation path, including
   empty audio, failed microphone startup, transcription errors, paste failures,
   and explicit cancellation. Consider an RAII finish guard or one shared
   `finish_status()` helper to prevent a newly added return path from leaving a
   stale title.
8. Keep title updates on the appropriate Tauri/main-thread boundary if the
   selected Tauri API requires it. Status publication must be non-blocking and
   must never hold audio, model, or transcription locks while updating the UI.

### Title tests

- Unit-test the formatter for every state and assert that the machine-readable
  suffix is ASCII, stable, and not translated.
- Exercise the integration lifecycle with a fake/status sink: startup → idle,
  successful start → recording, stop → transcribing, post-process → processing,
  success/error/cancel → idle.
- Verify that model-load failure and microphone permission failure do not leave
  the title at `recording`.
- Verify that rapid toggle/hold transitions still produce the same sequence as
  the coordinator's state machine.
- Manually inspect the native title on Windows, macOS, and Linux, including
  start-hidden mode and when the settings window is reopened.

## JSON status-file investigation and implementation plan

### Feasibility

The JSON approach is feasible and is likely easier for Talon to consume when a
file watcher or a small Talon Python action is available. Rust already depends
on `serde` and `serde_json`, and the portable-aware app-data path gives Handy a
stable per-install location. The main risks are partial reads, stale files
after a crash, and multiple Handy instances. Handy's single-instance design
reduces the last risk, but the reader still needs defensive handling.

### Proposed location and schema

Write `handy-status.json` in the portable-aware Handy application-data
directory. Document the resolved path for each platform and expose it through
the existing app-data-directory command or a small diagnostic log entry so a
Talon configuration does not need to guess platform paths.

Example:

```json
{
  "schema_version": 1,
  "state": "recording",
  "recording": true,
  "transcribing": false,
  "processing": false,
  "sequence": 42,
  "updated_at": "2026-10-23T12:34:56.789Z",
  "pid": 12345
}
```

The boolean fields make simple Talon readers possible, while `state` preserves
the full lifecycle. `sequence` lets a watcher reject an older event, and
`updated_at`/`pid` help diagnose stale data. Talon should treat a missing,
malformed, or old file as `idle`/unknown rather than assuming that Handy is
actively recording.

### Safe-write protocol

1. Create the parent directory during status initialisation.
2. Serialize the complete document in memory.
3. Write it to a sibling temporary file in the same directory.
4. Flush and close the temporary file, then atomically replace/rename it to
   `handy-status.json`.
5. Log write failures without interrupting recording or transcription.
6. Publish `idle` on clean shutdown. Do not delete the file by default: leaving
   the last status plus `pid`/timestamp allows a reader to detect a crashed or
   stale process. Optionally add a short freshness rule to the Talon reader.

Use one serialised publisher or a mutex around sequence allocation and writes;
status updates can originate from shortcut, recording, async transcription,
and main-thread callbacks.

### JSON tests

- Verify schema serialisation for every state.
- Read the file repeatedly during rapid state changes and assert every observed
  document is valid JSON and internally consistent.
- Test replacement when the target file already exists.
- Test unwritable-directory errors as non-fatal.
- Test restart and stale-file handling using `pid`/`updated_at`.
- Test portable mode and the normal per-user application-data mode.

## Relationship between the two interfaces

Implement one status publisher that updates both the title and JSON document
from the same call. This avoids maintaining two independent state machines and
ensures Talon receives equivalent transitions. The title should be the
zero-configuration baseline because Talon can usually inspect the active
window title without filesystem permissions. The JSON file should be an
optional, richer integration for watchers and automation that need booleans,
timestamps, or a reliable event sequence.

If implementation scope must be reduced, ship the title contract first and
keep the publisher's JSON output behind a small feature/settings flag. The
state contract, transition points, and tests should remain shared so enabling
the file later does not require reworking the lifecycle.

## Delivery sequence

1. Add the shared status enum, title formatter, JSON schema, and publisher.
2. Initialise the publisher and publish `idle` during application startup.
3. Wire all recording, transcription, processing, cancellation, error, and
   completion paths in `src-tauri/src/actions.rs` and
   `src-tauri/src/utils.rs`.
4. Add focused Rust tests for formatting, serialisation, atomic writes, and
   lifecycle coverage; add a small Talon-side parser test if the Talon config
   is updated in a follow-up.
5. Run `cargo fmt`, `cargo test`, `bun run lint`, and `bun run format:check`.
6. Validate manually on each supported desktop platform, including hidden
   startup and an interrupted/crashed transcription.
7. Document the exact title grammar and JSON path/schema for Talon users in the
   Handy integration documentation or README.

## Acceptance criteria

- Talon can extract one of the documented state tokens from the native Handy
  window title without locale-specific parsing.
- The title reports `recording` only during active capture and
  `transcribing` only while speech recognition is running.
- Cancellation, errors, empty input, and paste failures always converge to
  `idle`.
- When enabled, `handy-status.json` is always atomically readable and matches
  the title state.
- A stale or unreadable JSON file cannot make Talon believe Handy is active.
- The integration does not block the audio callback, transcription worker, or
  shortcut coordinator.
