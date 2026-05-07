# RETIRED 2026-05-07

**This branch is retired.** Upstream MemPalace shipped equivalent concurrency-hardening fixes via PRs [#1162](https://github.com/MemPalace/mempalace/pull/1162) (lock at `ChromaCollection.add/upsert/update/delete` seam) + [#1339](https://github.com/MemPalace/mempalace/pull/1339) + [#1342](https://github.com/MemPalace/mempalace/pull/1342) (HNSW segment quarantine helpers) + [#1310](https://github.com/MemPalace/mempalace/pull/1310) (`mempalace repair --mode from-sqlite` recovery flag), targeted for v3.3.5. Verified on develop@03ed4c45 against M4 Air macOS 26.3.1: single-writer 100k-vector ingest baseline passes, sustained 2-process concurrent writers fail-fast with `MineAlreadyRunning` exception, `mempalace repair --mode from-sqlite` recovered a real 139,644-row palace cleanly. Of the 4 originally-planned fork-side patches, **zero were applied** — upstream beat us to all four. This branch is preserved as an audit-trail of the diagnostic-loop with upstream issue [#1401](https://github.com/MemPalace/mempalace/issues/1401). **Do not consume this branch as a dependency.** Atlas's `tier-3-verbatim.sh:93` pin updated to develop@03ed4c45 in atlas-claude-code-stack, retiring this fork's role.

---

# concurrency-hardening branch

Patches against MemPalace v3.3.4 that serialize concurrent ChromaDB writers on
a shared palace path. Tracks upstream `MemPalace/mempalace` and retires when
the corresponding PR merges.

## Why

A diagnostic on macOS 26 ARM64 (2026-05-07) confirmed ChromaDB itself is stable
under single-writer load on Tahoe — 100k single-writer ingest plus re-open plus
queries pass cleanly with the v3.3.4 binding. Sustained two-process concurrent
writers against the same palace, however, corrupt HNSW segment files. At larger
scale this surfaces as the SIGSEGV signature in chroma-core/chroma#6979 and
#6984; at the diagnostic's scale it surfaces as a hung post-stress reader.

Static analysis identified the structural cause:

- `mempalace.mcp_server` holds a long-lived `chromadb.PersistentClient` for an
  entire Claude Code session.
- `mempalace.hooks_cli._spawn_mine` launches `mempalace mine` as a child
  subprocess; the child opens its own `PersistentClient` on the same palace.
- `mempalace.palace.mine_palace_lock` exists but is acquired only by
  `miner.mine()`. `mcp_server`, `sweep`, `repair`, `migrate`, `dedup`, and the
  `cli` shim writers bypass it.
- No `atexit` or `SIGTERM` handler in the package; no explicit
  `client.persist()` on shutdown.

Two `PersistentClient` instances writing the same `chroma.sqlite3` plus HNSW
segment files concurrently, with no shutdown flush, is the corruption
mechanism.

## Patches landing here

1. Extend `palace.mine_palace_lock` to shared/exclusive flock semantics on a
   `~/.mempalace/locks/palace_<sha256>.lock` sentinel. MCP server takes
   `LOCK_SH`; every writer takes `LOCK_EX`.
2. Add `atexit` and `SIGTERM` handlers in `mcp_server.main()` and the CLI
   entry points, calling `ChromaBackend.close()` and releasing the client
   cache.
3. Replace the non-atomic `_MINE_PID_FILE` check in `hooks_cli._spawn_mine`
   with a non-blocking `fcntl.flock` on `~/.mempalace/locks/spawn_mine.lock`
   before `subprocess.Popen`.
4. Wrap `sweeper.sweep`, `repair.*`, `migrate.*`, `dedup.*`, and the `cli`
   shim writers in `with mine_palace_lock(palace_path):`.

## Upstream references

- chroma-core/chroma#6852, #6979, #6984, #6963 — open SIGSEGV reports
- MemPalace/mempalace#1376 — chromadb cap workaround (author retracted)
- MemPalace/mempalace#1386 — sqlite-vec backend swap alternative

## Retirement

When the patch set merges upstream, consumers switch their pin back to the
upstream tag and this branch retires.

## Consumer

`atlas-claude-code-stack` Phase 9.5.0 pins
`installer/platforms/macos/tier-3-verbatim.sh` to
`git+https://github.com/Seph396/mempalace.git@<sha>` while patches are in
flight.
