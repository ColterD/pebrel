# Command process isolation

## Status
Proposed in the PR for #B.

## Context
`pebrel -e` / `--command` starts a process for one command; Linux desktop
launchers with `Terminal=true` use it when Pebrel is the default terminal. Since
`2026-10-03-empty-workspace-residency.md`, `keep_session` hides an empty
workspace instead of closing it and saves `TabsClosed`. A command window is an
ordinary Regular window, so the process stays hidden after its command exits,
its checkpoint and `TabsClosed` saves replace the shared `session.json`, and the
first such process publishes `runtime.port`, so a later plain launch is handed
into a process whose windows the user did not start.

## Evidence
In a clean Ubuntu 22.04 container with a throwaway `HOME` (published AppDirs
run from a copied `/tmp/.mount_*` directory, since the container has no FUSE):
on 2.2.2 and on `main`, `-e sleep 2` was still running after 15 s with
`keep_session=1`, and a seeded one-tab `session.json` ended as `"tabs":[]`.
With `keep_session=0`, and on 2.1.1, the process exited but the file was still
overwritten. A plain `pebrel` started beside `-e sleep 40` was handed into the
`-e` process.

## Decision
Isolate a process started with an initial command the way an elevated process
is already isolated. `open_initial_window` calls `SessionPersistence::isolate()`,
so `save_with` skips every save. `keep_session_on_close` reads the same flag,
whose default is still `requires_isolation()`. The runtime server stays private:
`discovery_path()` publishes no `runtime.port`, while panes keep the API through
their endpoint variable. Beside a running owner, a command process previously had
no runtime server at all; it now has a private one. It does not resume the saved
mobile link, and a pending update restore ticket waits for the next plain launch.

## Rejected alternatives
- Handing `-e` to the resident instance: callers that wait for
  `$TERMINAL -e cmd` to exit would return early, and the Windows handover
  already excludes command launches (`try_hand_over_to_resident`).
- A new window role: Regular also gates activation order and runtime window
  listings, which a command window should keep.
- Exempting only residency: checkpoints would still replace the saved session,
  and later launches would still join a process that is not saved.
- A setting: the explicit command already states intent, and the residency note
  rejects new residency settings.

## Consequences
Tabs or windows opened later inside a command process are not saved. Tools
outside the process cannot discover it through `runtime.port`. An in-app update
started from a command process has no durable workspace snapshot and aborts
before commit. Elevated processes and ordinary launches behave as before.

## Validation
`isolated_servers_never_publish_discovery` and
`isolated_windows_never_write_shared_session_storage` cover the two decisions.
The container check above passes for both `keep_session` values and for the
handover case. macOS and Windows were not built for this change.

## Supersedes
None.

## Revisit when
Session persistence becomes per window or per process, or command launches need
to contribute tabs to the saved workspace.
