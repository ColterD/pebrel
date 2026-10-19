# Durable hook helper for AppImage builds

## Status
Proposed in the PR for #A.

## Context
Every provider installer (Claude Code and Codex hooks, Codex `notify`, Kimi,
Copilot, Grok and Cursor files) and the pane environment use `helper_path()`.
It resolves `pebrel-hook` beside `current_exe()`. In an AppImage that directory
is `$APPDIR`, a per-launch mount such as
`/tmp/.mount_PebrelXXXX` that disappears when the process exits. While Pebrel is
closed, every configured hook therefore runs a missing executable, and each
launch writes a new random path. The login item already avoids the mount and
points at `$APPIMAGE` (`platform/startup/login.rs`, `linux_launcher`).

## Evidence
In a clean Ubuntu 22.04 container without FUSE, the AppDirs of the published
2.1.1 and 2.2.2 AppImages were copied to a `/tmp/.mount_*` directory, run from
there with `APPIMAGE` and `APPDIR` set, and deleted on exit, which reproduces
the mount's lifetime. After exit, the configured Claude `Stop` hook exited 127
and the Codex `notify` executable did not exist. With this change the hook command
points at the copy, exits 0 after the mount is gone, and the `notify` target
exists. `main` built the same way still fails.

## Decision
When the executable lives inside the AppImage mount
(`platform::startup::in_appimage_mount`, now shared with the login item), copy
the bundled helper to `data_dir()/runtime/` once per process and give installers
that path. Copy only when the bytes differ or the copy is not executable. Stage
under a process-unique name with `fs::copy`, which keeps the mode bits, and
replace atomically. Keep the `pebrel-hook` file name, so existing ownership
matching and the config guard repair entries that still name the mount.
Non-AppImage installs keep using the bundled path. If the copy fails, log a
warning and use a runnable copy left by an earlier launch, or else keep the
previous behavior for that process. The copy is read and compared once per
process; a size mismatch skips the byte comparison.

## Rejected alternatives
- Pointing hooks at `$APPIMAGE`: every hook event would start the AppImage
  runtime and a FUSE mount, while the helper must stay a fast no-op outside a
  Pebrel pane.
- A symlink to the bundled helper: it dangles as soon as the mount disappears.
- A stable wrapper with another file name: ownership matching does not
  recognize it, so installers add their own entries beside it.
- Skipping hook installation for AppImage builds: it removes the integration
  instead of making it durable.

## Consequences
One executable file appears in the config directory. Encrypted backups use
explicit file lists and do not include it. Two different AppImage versions
running at once each refresh the copy to their own helper; the hook envelope is
versioned. `setup-ai --remove` removes the hook entries but leaves the copy,
which is a no-op outside a Pebrel pane.

## Validation
`appimage_helper_copy_is_runnable_and_follows_updates` covers the executable
copy, refresh after the bundled helper changes, no leftover staging file, and
the error for an unusable directory. `portable_login_uses_a_persistent_launcher`
still covers the shared mount test. The container check above passes; macOS and
Windows were not built for this change.

## Supersedes
None.

## Revisit when
AppImage builds stop running from a per-launch mount, or `pebrel-hook` gains
runtime dependencies that a copy outside `$APPDIR` cannot satisfy.
