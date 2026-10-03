---
generated_by: osc-newfeature
type: feat
bulk_run: true
repo: JuliusBrussee/caveman
feature_complexity: M
Files:
  - bin/lib/install-receipt.js
  - bin/install.js
  - bin/lib/owned-install.js
  - bin/lib/settings.js
  - bin/lib/openclaw.js
  - src/tools/caveman-init.js
  - INSTALL.md
  - README.md
  - tests/installer/install-receipt.test.mjs
  - tests/installer/e2e.freshinstall.test.mjs
  - tests/installer/uninstall-safety.test.mjs
  - tests/installer/owned-install.test.mjs
  - tests/installer/skills-global-install.test.mjs
  - tests/verify_repo.py
---

# Install transaction receipt and last-run revert

Record every file the unified installer actually writes, then let the user verify or undo **that run** without a full-machine uninstall.

`--uninstall` stays the wipe of every known caveman hook, plugin, and fence. `--revert-last` only walks the last receipt. External package commands (`npx skills add`, `claude plugin install`) are listed on the receipt and printed as manual follow-ups; the revert path never shells them.

This is a new installer transaction log for `bin/install.js`. It is not a change to the CLI wrap journal in `bin/lib/owned-install.js`, and it does not replace `--dry-run`.

## User-visible behavior

After any non-dry-run `node bin/install.js` / `npx caveman` / shim install that writes at least one owned path, settings merge, or fence, write:

`$XDG_CONFIG_HOME/caveman/last-install.json`

with the same directory fallbacks already used for user config (`~/.config/caveman/config.json`, `%APPDATA%\caveman\config.json`). Create the directory with `0700`. Write the receipt with `0600`, atomic temp + rename, and refuse a receipt path or parent that is a symlink.

Print one line at the end of a successful or partial install:

`receipt: <absolute path>  (verify: node bin/install.js --verify-last; undo this run: node bin/install.js --revert-last)`

Add the same two flags to `printHelp()` and to the FLAGS table in `INSTALL.md`. Add one short row to the README What you get table.

`--dry-run` never writes or rotates a receipt.

A mid-run failure still flushes a partial receipt covering entries that already landed, so `--revert-last` can clean a crashed install.

### Receipt schema (version 1)

```json
{
  "version": 1,
  "id": "<utc-compact>-<8 hex>",
  "createdAt": "<RFC3339>",
  "argv": ["--only", "claude"],
  "pinnedRef": "v2.2.0",
  "configDir": "/home/user/.claude",
  "entries": [
    {
      "op": "write",
      "path": "/home/user/.claude/hooks/caveman-activate.js",
      "sha256": "<hex>",
      "created": true
    },
    {
      "op": "merge",
      "path": "/home/user/.claude/settings.json",
      "marker": "caveman",
      "keys": ["hooks.SessionStart", "hooks.UserPromptSubmit", "statusLine"]
    },
    {
      "op": "fence",
      "path": "/home/user/.openclaw/workspace/SOUL.md",
      "begin": "<!-- caveman-begin -->",
      "end": "<!-- caveman-end -->"
    },
    {
      "op": "command",
      "argv": ["npx", "-y", "skills", "add", "JuliusBrussee/caveman", "-a", "cursor"],
      "revert": "manual"
    }
  ]
}
```

Rules:

- `path` values are absolute, resolved, and never stored if they contain a NUL
- `write.created` is true only when the file did not exist before this run
- `write` on a file we overwrote stores `created: false` plus `previousSha256` when the previous bytes were readable and not a symlink
- `command.revert` is always `"manual"` in v1
- unknown `op` values on read are skipped with a warning, never executed

Keep one generation: `last-install.json`. Optionally copy the previous file to `last-install.prev.json` before replace so a revert that fails halfway can still be retried from `last-install.json` (do not chain `--revert-last` into the previous generation automatically).

### `--verify-last`

Exit codes: `0` all owned entries match, `1` drift or missing expected write, `2` no receipt / unreadable receipt.

For each `write`: `lstat` must be a regular file, not a symlink, and `sha256` must match. For each `merge`: `bin/lib/settings.js` `readSettings` must still contain the caveman marker on the listed keys (or the statusline command still contains `caveman-statusline` if that key was recorded). For each `fence`: both markers exist and the begin marker appears before the end marker. `command` entries print `manual` and do not affect the exit code.

Print one line per entry: `ok`, `missing`, `hash-mismatch`, `symlink`, or `manual`.

### `--revert-last`

Walk `entries` in reverse.

- `write` + `created: true`: delete only if current sha256 matches the receipt. If the file is missing, skip. If hash differs or the path is a symlink, warn and leave it. Never delete `hooks/package.json` unless `hooksManifestIsOurs` is true (existing helper in `bin/install.js`).
- `write` + `created: false`: restore `previousSha256` bytes only when the current hash still equals the receipt `sha256` (we still own the last write). If the user edited the file, warn and leave it. If `previousSha256` is absent, do not delete; warn that restore is unavailable.
- `merge`: call `SETTINGS.removeCavemanHooks` and drop a statusline command that still points at `caveman-statusline`, then `validateHookFields` + `writeSettings`. Same hard stop as today's uninstall if settings cannot be parsed: leave hook files in place and exit 1.
- `fence`: strip the exact begin/end block using the existing OpenClaw / init fence helpers. If markers are absent, skip. If begin exists without end, warn and do not splice.
- `command`: print the argv and `revert: manual`; do not spawn.

`--dry-run --revert-last` prints the reverse plan and writes nothing.

`--revert-last` and `--uninstall` together is an error. `--verify-last` with `--uninstall` is an error. `--verify-last` with a normal install is an error.

After a successful revert, rename `last-install.json` to `last-install.reverted.json` so a second `--revert-last` does not delete twice.

## Implementation notes

New module `bin/lib/install-receipt.js` (CommonJS, stdlib only):

- `userConfigDir()` — copy the XDG / APPDATA / `~/.config/caveman` resolution from `src/hooks/caveman-config.js` `getConfigDir()`. Do not `require` the hook file from the installer (hooks are not on the install path for a curl|bash run that has not copied them yet).
- `receiptPath()`, `readReceipt()`, `writeReceipt(receipt)`, `sha256File(absPath)`, `beginReceipt({ argv, pinnedRef, configDir })`, `recordWrite(receipt, { path, created, previousSha256 })`, `recordMerge(...)`, `recordFence(...)`, `recordCommand(...)`, `verifyReceipt(receipt)`, `revertReceipt(receipt, { dryRun, hooksManifestIsOurs, settings, openclaw })`
- All filesystem errors on the receipt itself are reported to stderr and must not abort an install that already succeeded. A failed receipt write prints `receipt: unavailable` and is not an install failure.

`bin/install.js` changes:

- `parseArgs`: `--verify-last`, `--revert-last`. Mutual exclusion as above.
- `main`: if verify or revert, load the receipt and exit; do not run provider installs.
- Thread a `receipt` object on `ctx` for the normal install path. Every owned write already performed by `installClaude`, `installGemini`, `installOpencode`, `installOpenclaw`, `installHermes`, `installViaSkills`, and `runInit` must call `recordWrite` / `recordMerge` / `recordFence` / `recordCommand`.
- Settings merges already go through `bin/lib/settings.js`; record `merge` immediately after a successful `writeSettings`.
- OpenClaw SOUL.md and init `<!-- caveman-begin -->` blocks record `fence`.
- `npx skills add`, `claude plugin install`, `gemini extensions install`, and `claude mcp` record `command` only.
- `HOOK_FILES` copies in the Claude hooks directory are individual `write` entries (needed so revert can leave a foreign `package.json` alone).

`src/tools/caveman-init.js`: return the list of paths it created or updated (and whether each was new) so `runInit` can record them. Keep the public CLI of init unchanged; add an optional `--json-paths` stdout line or a programmatic export if init is required as a library. Prefer exporting `runInitTo(target, opts) → { files: [{ path, created }] }` and having `bin/install.js` call that when running from a clone. The curl-piped init path does not write an installer receipt (no `bin/install.js` parent).

Do not add flags to `install.sh` / `install.ps1`. Those shims already forward argv to `bin/install.js`.

`tests/verify_repo.py` already `node --check`s `bin/install.js`; add `bin/lib/install-receipt.js`.

## Tests

`tests/installer/install-receipt.test.mjs` (new):

- `beginReceipt` + three `write`s + `writeReceipt` produces version 1 JSON with mode 0600
- symlink receipt path is refused
- `verifyReceipt` ok / missing / hash-mismatch / symlink
- revert deletes a `created: true` write whose hash matches
- revert refuses to delete a `created: true` write whose bytes changed
- revert restores `previousSha256` when current hash still matches
- revert of `merge` uses `removeCavemanHooks` and does not wipe unrelated hooks
- revert of `fence` strips only the marked block
- `command` entries never spawn
- `--verify-last` with no file exits 2
- `--revert-last` then a second `--revert-last` exits 2 (receipt renamed)

Extend:

- `tests/installer/e2e.freshinstall.test.mjs` — after `--only claude --with-hooks --non-interactive`, `last-install.json` exists and `--verify-last` exits 0; `--revert-last` removes the hook files this run created and leaves a foreign `hooks/package.json` in place
- `tests/installer/uninstall-safety.test.mjs` — `--revert-last` uses the same settings-parse hard stop as `--uninstall` (do not delete hooks if settings cannot be rewritten)
- `tests/installer/owned-install.test.mjs` — unchanged behavior; add one assertion that `bin/lib/install-receipt.js` does not import CLI journal helpers (keeps the two journals independent). If a shared sha256 helper is extracted, keep it in `install-receipt.js` or a new tiny `bin/lib/sha256-file.js` — do not route skill-installer receipts through `owned-install.js`
- `tests/installer/skills-global-install.test.mjs` — `npx skills add` is recorded as `command` / `manual`
- `tests/verify_repo.py` — syntax-check the new module

## Risks

- Revert that deletes a user-edited hook or a shared `settings.json` is the failure mode that matters. Hash match is mandatory; settings revert must stay marker-scoped.
- `hooks/package.json` is shared. Reuse `hooksManifestIsOurs`. A receipt `write` for that file with `created: false` must not restore a foreign manifest from `previousSha256` if the current file is no longer ours.
- Windows rename and path separators: store receipt paths as `path.resolve` output; compare with `path.resolve` on read.
- Concurrent installs can clobber `last-install.json`. Document last-writer-wins; do not take a lock across network `npx` calls.
- Partial receipts after a crash are useful only if every successful write recorded **before** the next write. Record immediately, not at process end (except the final `writeReceipt` flush, which should also run in a `finally` if any entries exist).
- `--revert-last` is easy to confuse with `--uninstall`. Help text must say "undo the last install transaction only" and point at `--uninstall` for a full wipe.
- Do not teach revert to run `npx skills` removal. Skill installs are untracked by this repo's installer today; pretending otherwise would delete the user's other skills.
