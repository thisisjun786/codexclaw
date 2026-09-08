# Windows short-path source bindings

The Windows jobs for lidge-jun/codexclaw#84 reject valid linked worktrees when
their temporary parent uses an 8.3 alias. Node's JavaScript `realpathSync`
can preserve that alias while Git reports the long spelling, so the exact
worktree-root comparison fails before the integration scenario starts.

Use `realpathSync.native` for both input paths and Git identity paths, including
the native-cwd check when reading a binding. This preserves the existing exact
root, common-directory, worktree-directory and immutable-binding checks.
The existing `worktree-guard.canonicalize` helper was considered but deliberately
not reused: its fallback accepts missing paths, whereas source binding must fail
when a worktree is gone.

The integration fixture now resolves its temporary root with the same native
API so expected paths agree with Git and child-process cwd on Windows. A new
Windows-only test asks cmd.exe for a real 8.3 alias, binds through it, reads and
repeats the binding through both spellings, checks byte-for-byte immutability,
and advances B/C before validating a receipt executed in the bound worktree.
It skips explicitly if the temporary volume does not provide 8.3 names.

## Verification

- The new regression failed on the parent implementation with the original
  linked-worktree-root error, then passed after the fix on Windows / Node 24.15.0
  with no skip.
- `npm run build`: 160 files compiled; only session-source.js changed in dist.
- `npm run gate` and `git diff --check`: passed.
- WSL Ubuntu / Node 24.15.0: the integration file passed 17 tests with zero
  failures; the Windows-only alias test skipped. This includes the damaged
  symlink binding check and the shipped CLI flow.
- Full Windows suite: 2,669 tests, 2,578 passed, 81 skipped, 10 failed. Nine
  failures are the existing session-binding symlink fixtures and one is the
  damaged-symlink integration fixture; this host cannot create those symlinks.
  The new short-path regression and the other 16 integration cases passed.
  This is not a green full-suite result. No permission checks or tests were
  weakened to work around the host limitation.
- Updated the three published test-count badges with
  `inventory.mjs --write --tests 2669`, using the measured suite total.

This follow-up targets the author's `fix/worktree-source-binding` branch at
57e63fdcc5757ae4a2446e305d317e72ae26b3d7. It does not integrate unrelated dev
changes or resolve the parent PR's changelog conflict with dev.
