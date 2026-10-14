# Readout

Readout is a macOS menu bar app that shows how much of your coding subscription you've used:
- **Limits:** session (5-hour) and weekly limits for your default provider.
- **Today:** tokens used today.
- **Last 7 days:** tokens per day as horizontal bars.
- **Details:** a per-model breakdown.

**Conventions**
- YAGNI, DRY, SOLID. No code comments. Don't refactor outside the task.
- **Commits:**
  - One plain imperative sentence with no prefix and no body.
  - No co-author trailer and no mention of AI.
  - The privacy hooks in `.git/hooks` must pass.
  - Never commit real transcripts, tokens, usernames or hostnames. Test fixtures are synthetic.
- **Stack:**
  - SwiftPM only: `swift-tools-version: 6.2`, `.macOS(.v26)`, Swift 6 language mode, Swift Testing (`@Test` / `#expect`).
  - No third-party dependencies.
  - Ad-hoc signed. No notarization.
- No `foundry.yaml`; this is not a web app.

**Package layout**
```
Package.swift            library ReadoutKit + executable Readout + test target ReadoutKitTests
Sources/ReadoutKit/      everything testable + the public root view
Sources/Readout/         App.swift (AppDelegate, NSStatusItem, right-click menu), Panel.swift
Resources/Info.plist     LSUIElement, bundle id dev.readout.Readout, LSMinimumSystemVersion 26.0
scripts/build-app.sh     swift build -c release → build/Readout.app → ad-hoc sign → ~/Applications → open
```

**App shell (`Sources/Readout`)**

Copy the shape of the sibling Uplink app's `app/Sources/Uplink/App.swift` and `Panel.swift` (the kickoff prompt gives its local path):
- A `@main` AppDelegate with `.accessory` activation policy.
- An `NSStatusItem`. Left click toggles the panel. Right click opens a menu with Settings…, Launch at Login (`SMAppService.mainApp`) and Quit.
- A non-activating floating `NSPanel` with an `NSGlassEffectView` that hosts the SwiftUI root view, sizes itself to the content and positions under the status button.

The status item shows an SF Symbol plus the default provider's session percentage, e.g. `42%`. It shows `—` when there's no data.

## Data layer (`Sources/ReadoutKit`)

**1. Token ledger**

Sources:
- `~/.claude/projects/**/*.jsonl` (recursive, includes the `*/subagents/*.jsonl` files).
- `~/.codex/sessions/**/rollout-*.jsonl`.

Only files whose mtime is inside the window (today plus 6 days, so 8 days with margin) are read. Keep a per-file `(size, mtime, byteOffset)` cache so a refresh only tails new bytes. Rescan every 30 s, and again when the panel opens.

Claude transcript rules (these are exact; earlier scripts got them wrong):
- Only `type == "assistant"` lines that have `message.usage`.
- One API call is split across several lines, one per content block, and each line repeats `usage`.
  - **Dedupe globally by `message.id` + `requestId`, keeping the line with the maximum `output_tokens`.**
  - Dedupe globally, not per file, because resumed sessions can repeat history.
- Tokens for a call = `input_tokens + output_tokens + cache_creation_input_tokens + cache_read_input_tokens`, **plus** the input and output of `usage.iterations[]` entries whose `type == "advisor_message"`.
  - Advisor calls are only counted there.
  - Add **only** those entries. Other iteration types would double count.
- Skip `model == "<synthetic>"`.
- **Provider comes from the model string, not the directory.** Claude Code also runs on Codex accounts through a local proxy, and those calls land in `~/.claude/projects` with model `gpt-*` or `acctN/gpt-*`.
  - `claude-*` and bare `opus`/`sonnet`/`haiku` → Claude.
  - `gpt-*` → Codex.
  - Strip an `acctN/` prefix for display.
- Timestamps are UTC ISO-8601. Bucket by **local** calendar day.

Codex session rules:
- Use `payload.type == "token_count"` events. Take per-session deltas of `info.total_token_usage.total_tokens`, so duplicate events don't double count.
- `input_tokens` already **includes** `cached_input_tokens`; never add both.
- The model comes from the session's `turn_context` event. Verify this in Phase 0.

Output: `[Day: [ModelKey: TokenSplit]]`, where `TokenSplit` is input, output, cacheWrite and cacheRead.
- The headline number is the **sum of all four** across all models.
- The detail view shows the split.

**2. Limits**

`protocol LimitSource { func fetch() async throws -> LimitSnapshot }`, where `LimitSnapshot` has `session` and `weekly`, each with `usedPercent`, `resetsAt` and `asOf`.

- **Claude:**
  - Read the access token by shelling out to `/usr/bin/security find-generic-password -s "Claude Code-credentials" -w`. Reading through the `security` tool avoids a keychain ACL prompt on every ad-hoc rebuild; Phase 0 confirms this. Parse it as JSON: `claudeAiOauth.accessToken` and `expiresAt`.
  - Call `GET https://api.anthropic.com/api/oauth/usage` with `Authorization: Bearer …` and `anthropic-beta: oauth-2025-04-20`. Map `five_hour` → session and `seven_day` → weekly.
  - **Never refresh the token.** Rotating it logs Claude Code out.
  - If the token is expired or the call returns 401, keep the last snapshot and mark it **stale**, with a hint that it updates after the next Claude Code use.
  - Poll every 5 min, plus on panel open if the data is more than 60 s old.
  - Persist the last snapshot to `~/Library/Application Support/Readout/limits.json`.
- **Codex:**
  - Read the newest `token_count` event that has `rate_limits` in the newest rollout file: `primary` (window 300 min) → session, `secondary` (10080 min) → weekly, `used_percent`, `resets_at` (epoch s).
  - This is passive. Show the "as of" time. If `resets_at` has passed, show 0% with "reset".

**3. Settings** (UserDefaults, the way Uplink's `Store.swift` does it)
- `defaultProvider` (Claude | Codex).
- `showPercentInMenuBar` (default on).

## UI (`Sources/ReadoutKit`, public `ReadoutView`)

Main panel, about 360 pt wide:
- **Limit bars:** two horizontal capsule bars for the default provider, labelled "Session" and "Weekly". Each shows the percentage on the right and "resets in 2h 14m" / "resets Thu 9:00" below. Colour moves from neutral to amber at 75% and red at 90%. A stale badge appears when needed.
- **Today:** one large number, formatted like "18.4M tokens".
- **Last 7 days:** seven rows, oldest at the top and today at the bottom. Each row has a day label (Mon 13), a horizontal bar whose length is proportional to the busiest day, and a value label. Use plain SwiftUI shapes, no Charts dependency.
- **Details toggle:** per-model rows for today and for 7 days. Each shows the model name, the total and a small split of input / output / cache write / cache read. The bars can be stacked by model.
- **Footer:** the last-updated time, a refresh button and a gear that opens settings inline.

## Phases (each ends green: `swift build`, `swift test`, and a commit)

- **Phase 0: spike (gate; build no UI until this passes).** Write findings to `docs/phase0.md` as **field names and shapes only, no values**.
  1. Read the keychain item from an ad-hoc-signed test binary through `/usr/bin/security`. Record whether a prompt appears, and whether it appears again after a rebuild.
  2. curl the usage endpoint and record the real response shape.
  3. Confirm the Codex `rate_limits` and the model location in rollout files.
  4. Name a fallback if the endpoint shape differs or 4xx's: Codex-style passive data isn't available for Claude, so the fallback is "limits unavailable" plus the token ledger.
  5. Compute one past full day with the ledger rules and compare it against `~/.claude/stats-cache.json` `dailyModelTokens` for that day. Explain any gap.
- **Phase 1: shell.** Package, status item, empty panel, `scripts/build-app.sh`. The app launches from `~/Applications` with no Dock icon.
- **Phase 2: ledger.** Scanner, dedupe and aggregation, with tests on synthetic fixtures covering:
  - multi-line calls (max output wins)
  - advisor iterations
  - `acctN/` prefixes
  - local-day bucketing across midnight UTC
  - Codex cached-inclusive input
- **Phase 3: limits.** Claude and Codex sources, the stale state and persistence. Tests decode fixture JSON shaped like the Phase 0 findings.
- **Phase 4: UI.** Main view, details, settings, the menu bar percentage and Launch at Login.
- **Phase 5: README.** Requirements, install (`scripts/build-app.sh`), what it reads (local files plus one Anthropic endpoint, nothing sent anywhere else), and settings.

## Verification
- `swift test` passes.
- `scripts/build-app.sh` installs and opens the app.
- Session and weekly percentages match Claude Code's `/usage` within a refresh.
- Today's total moves after a Claude Code turn within 30 s.
- A past day's total matches the Phase 0 cross-check.
- Switching the default to Codex swaps the bars and the menu bar percentage, while today's total and the chart stay the same.
- Running `claudex` adds to Codex rows in Details, not Claude.
