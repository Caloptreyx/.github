# Writing Caloptreyx extensions

Short rules for every Calagopus extension in this organization. Each one exists because an
extension broke it. Paths below are in [calagopus/panel](https://github.com/calagopus/panel)
`release-1.2.10`.

## Permissions: an extension permission never replaces a core permission

An extension's own permission (`gdrive.push`, `eggchanger.change`, ...) lets a server owner opt
a subuser in to the extension. It does not grant what core guards separately. When an action
does something core has a permission for, check **both**: the extension's permission and the
core one, the same way the panel's own route checks it.

| The action... | also requires (core) | Core route that checks it |
|---|---|---|
| wipes files or reinstalls the server | `settings.install` | `backend/src/routes/api/client/servers/_server_/settings/install/mod.rs` |
| starts the server (e.g. "start when done") | `control.start` | `.../_server_/power.rs` |
| sets or keeps a custom startup command | `startup.command` | `.../_server_/startup/command.rs` |
| changes the Docker image | `startup.docker-image` | `.../_server_/startup/docker_image.rs` |
| returns file content (including echoes, diffs, dry runs, parse errors) | `files.read-content` | `.../_server_/files/contents.rs` |
| creates or overwrites a file | `files.create` | `.../_server_/files/write.rs` |
| renames, moves or chmods | `files.update` | `.../_server_/files/rename.rs` |
| deletes | `files.delete` | `.../_server_/files/delete.rs` |
| compresses or extracts | `files.archive` | `.../_server_/files/compress.rs` |
| lists backups | `backups.read` | `.../_server_/backups/mod.rs` |
| copies a backup anywhere (download, off-site push) | `backups.download` | `.../_server_/backups/_backup_/download.rs` |
| restores a backup | `backups.restore` | `.../_server_/backups/_backup_/restore.rs` |
| creates a backup | `backups.create` | `.../_server_/backups/mod.rs` |

```rust
permissions.has_server_permission("backups.download")?;
permissions.has_server_permission("gdrive.push")?;
```

Check permissions before any expensive work (Wings calls, reading the request body), and
check every permission a request *can* use, not only the one the UI usually sends: a `force`
or `dry_run` flag that returns content still needs `files.read-content`.

Two more core checks get lost when an extension does its own lookups:

- Routes that resolve a server themselves (`Server::by_user_identifier`) skip core's server
  middleware (`.../_server_/mod.rs`). Refuse with 409 when `server.unavailable_reason()` is
  set, keeping core's exemption for admins with `servers.update` on suspended servers.
- Routes that change a user's credentials or linked accounts call
  `permissions.deny_while_impersonating()?`, as core's account routes do.

## Files through Wings

**Wings serves gzip files decompressed: re-compress before writing back binary files.**
`/files/contents` detects gzip by its magic bytes and returns the inflated data, so
`level.dat`, `playerdata/*.dat`, `.nbt`, `.schem` and `.litematic` arrive as raw NBT.
Writing those bytes back unchanged leaves a file the game cannot read. Re-compress (gzip)
before the write, or keep the original compression explicitly.

## panel_version

`panel_version` in `Metadata.toml` is `>=1.2.10`: the panel release the shared check
(`.github/workflows/extension-check.yml`) builds against. Don't claim an older floor that CI
never builds.
