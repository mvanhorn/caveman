---
generated_by: osc-newfeature
type: feat
bulk_run: true
repo: JuliusBrussee/caveman
feature_complexity: M
Files:
  - src/hooks/caveman-budget.js
  - src/hooks/caveman-config.js
  - src/hooks/caveman-parse.js
  - src/hooks/caveman-mode-tracker.js
  - src/hooks/caveman-activate.js
  - src/hooks/caveman-stats.js
  - src/hooks/caveman-statusline.sh
  - src/hooks/caveman-statusline.ps1
  - src/hooks/checksums.sha256
  - src/hooks/README.md
  - src/hooks/package.json
  - bin/install.js
  - skills/caveman/SKILL.md
  - skills/caveman/README.md
  - skills/caveman-help/SKILL.md
  - skills/caveman-help/README.md
  - skills/caveman-stats/SKILL.md
  - skills/caveman-stats/README.md
  - src/rules/caveman-activate.md
  - src/tools/caveman-init.js
  - src/plugins/opencode/plugin.js
  - src/plugins/opencode/commands/caveman.md
  - src/plugins/opencode/commands/caveman-help.md
  - src/plugins/opencode/commands/caveman-stats.md
  - .claude-plugin/plugin.json
  - plugins/caveman/skills/caveman/SKILL.md
  - plugins/caveman/skills/caveman-stats/SKILL.md
  - README.md
  - INSTALL.md
  - tests/test_caveman_budget.js
  - tests/test_caveman_parse.js
  - tests/test_mode_tracker_stdin.js
  - tests/test_repo_local_config.js
  - tests/test_caveman_stats.js
  - tests/test_hook_missing_sibling.js
  - tests/test_symlink_flag.js
  - tests/test_hook_stdin_lifecycle.js
  - tests/verify_repo.py
---

# Session token-budget intensity ladder

Give a user a numeric output-token ceiling. As the running session (or calendar day) spends that ceiling, caveman steps its own intensity up so later turns stay shorter. An explicit `/caveman <level>` still wins until the user releases the pin. No budget configured means today's behavior is unchanged.

This is a new control surface. It does not add a usage-command helper, a new named intensity, a rolling statusline window, or lifetime-history accounting.

## User-visible behavior

### Configuration

Read a `budget` object from the same files `getDefaultMode()` already walks, in that order, and stop at the first file that contains a valid `budget` key:

1. `CAVEMAN_OUTPUT_BUDGET` (positive integer output tokens) plus optional `CAVEMAN_BUDGET_WINDOW=session|day` (default `session`)
2. Repo-local `.caveman/config.json` or `.caveman.json` from `findRepoConfigPath(startDir)`
3. User config at `getConfigPath()` (`$XDG_CONFIG_HOME/caveman/config.json`, `~/.config/caveman/config.json`, or `%APPDATA%\caveman\config.json`)

Repo example:

```json
{
  "defaultMode": "lite",
  "budget": {
    "window": "session",
    "outputTokens": 20000,
    "ladder": [
      { "remainPct": 100, "mode": "lite" },
      { "remainPct": 50, "mode": "full" },
      { "remainPct": 20, "mode": "ultra" }
    ]
  }
}
```

Validation, fail closed to "no budget":

- `outputTokens` is a positive integer
- `window` is `session` or `day`
- `ladder` is a non-empty array
- each rung has `remainPct` in `(0, 100]` and `mode` in `VALID_MODES` minus `off` and the independent modes `commit` / `review` / `compress`
- duplicate `remainPct` values are rejected
- env-only config synthesizes a default ladder: 100 `lite`, 50 `full`, 20 `ultra` (or the current `defaultMode` at 100 if it is already a prose mode)

### Ladder rule

`used` is assistant `output_tokens` from the Claude Code transcript (`parseSession` in `caveman-stats.js`, extracted so both stats and the ladder call one function).

- `session`: parse `data.transcript_path` when the hook stdin provides it; otherwise `findRecentSession(claudeDir)`
- `day`: sum `output_tokens` from session files under `$CLAUDE_CONFIG_DIR/projects` whose mtime is on the local calendar day, with the same directory walk already used by `findRecentSession` and a hard cap of 200 files / 8 MB each. Skip unreadable files.

`remainingPct = max(0, (outputTokens - used) / outputTokens * 100)`.

Sort rungs by `remainPct` descending. Choose the matching rung with the **smallest** `remainPct` among those where `remainingPct <= remainPct`. If none match (numeric glitch), keep the current flag and do not write.

Example with the default ladder: 80% remaining → `lite`; 40% → `full`; 10% → `ultra`; 0% stays `ultra`.

The ladder never writes `off`, `commit`, `review`, or `compress`. Auto-clarity in `skills/caveman/SKILL.md` still drops caveman for security warnings, irreversible confirms, and ambiguous multi-step sequences; the flag may stay at `ultra` while the skill tells the model to speak clearly for that turn.

### Pin vs automatic step

A new session flag file `$CLAUDE_CONFIG_DIR/.caveman-budget-hold` (opencode: `~/.config/opencode/.caveman-budget-hold`) stores a pinned prose mode. All reads and writes go through `safeWriteFlag` / `readFlag` (extend `readFlag` to accept an optional allowlist, or add `readHoldFlag` that only accepts `VALID_MODES` prose values; 64-byte cap, symlink refuse, silent fail).

Pin sources:

- `/caveman hold` — pin the current prose mode
- `/caveman <level>` or a natural-language activation that sets a concrete level — pin that level for the rest of the session
- `/caveman release` or `/caveman auto` — delete the hold file and apply the ladder on this turn
- `/caveman off` / "stop caveman" / "normal mode" — existing clear path; also delete the hold file

Bare `/caveman` (no args) does **not** pin. It activates `getDefaultMode()` and then lets the ladder run.

Independent one-shot modes (`/caveman-commit` and siblings) keep today's `#599` restore. After restore, if hold is set write the held mode; otherwise apply the ladder.

### Commands and surfaces

`caveman-parse.js` gains:

- `/caveman hold` → `{ action: 'hold' }`
- `/caveman release` and `/caveman auto` → `{ action: 'release' }`
- `/caveman budget` → `{ action: 'budget-status' }`
- `/caveman budget <positive-int>` → `{ action: 'budget-override', outputTokens }` written to `$CLAUDE_CONFIG_DIR/.caveman-budget-override` (digits only, 16-byte cap) and treated as the session ceiling
- natural phrases: "hold this caveman level", "keep this caveman level", "resume caveman budget", "release caveman budget" — same quoted-span blanking as existing triggers so pasted help text cannot fire them

`/caveman-stats` prints four extra lines when a budget is configured: ceiling, window, used, remaining percent, active rung, next rung threshold, hold on/off. No new savings percentage. Honest-numbers overhead lines stay as they are.

Statusline: when `CAVEMAN_STATUSLINE_BUDGET` is not `0`, append a second suffix from `$CLAUDE_CONFIG_DIR/.caveman-statusline-budget`. The budget module writes that file via `safeWriteFlag` as a short whitelist string (`[0-9.]+[km]? left`, max 16 chars). Refuse symlinks; strip control bytes the same way as `.caveman-statusline-suffix`. Absent file renders nothing.

SessionStart (`caveman-activate.js`): if a valid budget exists and hold is empty, write the resolved ladder mode instead of raw `defaultMode`. If `defaultMode` is `off`, stay off and do not start a ladder.

opencode: `src/plugins/opencode/plugin.js` must apply hold / release / budget-override through `parseModeChange`, and on `session.created` plus each `chat.message` with no pin, resolve the ladder when a budget is configured. opencode has no Claude transcript; `session` window uses a local counter file `$XDG_CONFIG_HOME/opencode/.caveman-budget-used` that the plugin increments only when the host later exposes usage (until then, document that opencode ladder applies only for `day` if history exists, otherwise no-ops with no flag write). Do not invent usage numbers.

Help and README: add `/caveman hold`, `/caveman release`, `/caveman budget`, and the `budget` config key. Update the What you get table with one row for the budget ladder. Do not invent or round benchmark percentages.

## Implementation notes

- New module `src/hooks/caveman-budget.js` owns `readBudgetConfig(startDir)`, `usedOutputTokens({ window, transcriptPath, claudeDir })`, `resolveLadderMode(budget, used)`, `writeBudgetBadge(claudeDir, remainingTokens)`, `readHold`, `writeHold`, `clearHold`. Export those names.
- Extract `parseSession` and `findRecentSession` from `caveman-stats.js` into a tiny shared helper **inside** `caveman-budget.js`, and have stats require that helper. Do not create a fourth sibling if a require-cycle appears; if stats cannot require budget, duplicate the 30-line parser in budget and add a test that both functions agree on the same fixture transcript.
- `caveman-mode-tracker.js` loads the new sibling with the existing `requireSibling` guard. Missing or bad exports: skip ladder, keep current flag, exit 0.
- Add `caveman-budget.js` to `HOOK_FILES` in `bin/install.js` and to `src/hooks/checksums.sha256` (regenerate the whole manifest in the same change).
- `tests/verify_repo.py` must `node --check src/hooks/caveman-budget.js`.
- Plugin path already runs `src/hooks/caveman-activate.js` and `caveman-mode-tracker.js` from `.claude-plugin/plugin.json`; no new hook event. The new file is a sibling require only.
- CI still copies `skills/caveman/SKILL.md` into `plugins/caveman/skills/caveman/SKILL.md`. Edit the source skill; include the plugin copy in the same change if the author is not waiting on the sync workflow. `plugins/caveman/skills/caveman-stats/SKILL.md` is a hand copy — update it in the same commit as `skills/caveman-stats/SKILL.md`.
- `src/tools/caveman-init.js` `RULE_BODY` must stay byte-equivalent to `src/rules/caveman-activate.md`. Add one line: budget ladder steps intensity as the configured output ceiling is spent; `/caveman hold` pins.

## Tests

`tests/test_caveman_budget.js` (new):

- default ladder thresholds at 80 / 40 / 10 / 0 percent remaining
- invalid budget object → `null` (no throw)
- env integer wins over repo config; non-integer env is ignored
- `session` uses the provided transcript fixture; missing file → used 0
- `day` walk ignores files older than local midnight and caps at 200 files
- hold file symlink is refused
- badge writer rejects leftover control characters

Extend existing suites:

- `tests/test_caveman_parse.js` — hold, release, budget, budget 15000, quoted "hold this caveman level" does not pin
- `tests/test_mode_tracker_stdin.js` — transcript with 12k used of 20k writes `full`; with hold file leaves `lite`; `/caveman ultra` writes ultra and hold; `/caveman release` clears hold and re-ladders; independent mode restore then ladders
- `tests/test_repo_local_config.js` — `budget` in `.caveman/config.json` is visible to `readBudgetConfig`
- `tests/test_caveman_stats.js` — budget lines appear only when configured; no fake savings
- `tests/test_hook_missing_sibling.js` — omit `caveman-budget.js`; mode-tracker still exits 0 and still honors `/caveman ultra`
- `tests/test_symlink_flag.js` — hold and override paths use `safeWriteFlag`
- `tests/test_hook_stdin_lifecycle.js` — budget resolution does not wait on stdin EOF
- `tests/verify_repo.py` — syntax check the new hook file

## Risks

- The UserPromptSubmit hook already has a 5s host budget. Transcript parse must use `transcript_path`, a size cap, and no recursive walk on the `session` path. Fail open (leave flag) if parse exceeds ~50ms of extra work; do not spawn another Node process.
- An explicit level must pin, or the next prompt will climb the ladder and look like a mode bug.
- Statusline must keep exit 0 and a control-byte whitelist or Claude Code hides the bar.
- Day-window walks of `projects/` can be huge; the file cap is mandatory.
- opencode has no Claude JSONL. Do not fake used-token counts. Document the no-op.
- Do not publish a new savings percentage. Stats may only show used vs ceiling.
- Adding a hook sibling without `HOOK_FILES` + checksums leaves curl|bash installs degraded. Treat that as a release blocker for this feature.
