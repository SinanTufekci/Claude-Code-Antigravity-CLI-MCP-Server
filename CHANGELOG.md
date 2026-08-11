# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

GitHub Release notes are auto-generated from commits on each `vX.Y.Z` tag
(see `.github/workflows/release.yml`); this file is the curated, human-facing
summary.

## [Unreleased]

## [0.24.0] - 2026-08-11

agy **1.1.12 killed the `model` argument** by making `agy models` machine-readable, and
**1.1.11/1.1.12 made quota readable for free** — so this release is one break repaired and
one capability adopted. Verified live against agy 1.1.12 end-to-end.

### Fixed

- **`model` was rejected on every antigravity tool (agy 1.1.12).** 1.1.12 turned each `agy
  models` line from a bare slug into a tab-separated `<slug>\t<human label>` record
  (`gemini-3.6-flash-high\tGemini 3.6 Flash (High)`). The bridge validated the requested slug
  against whole lines, so **every valid model was refused** with an error that listed the very
  slug it had just rejected: `unknown agy model 'gemini-3.6-flash-high'; expected one of:
  gemini-3.6-flash-high<TAB>Gemini 3.6 Flash (High), …`. Reproduced end-to-end through
  `antigravity_ask` before the fix; `antigravity_ask`/`antigravity_continue`/`agent_swarm` were
  all affected, while calls that omitted `model` were untouched. `_parse_models_output` now keeps
  the first tab field — which reads the old bare-slug format too — and drops fields containing
  whitespace, so a progress line can never enter the accepted-model list. The whole suite stayed
  green through this break because every model test mocked the old format; the new tests pin both
  formats, and the live canary that *did* catch it now passes against 1.1.12.
  - Worth knowing for the next release: 1.1.12's changelog advertises `--output-format json` for
    the `models` and `agents` subcommands, but the **shipped binary has no such flag** (`agy
    models --output-format json` exits 1 with `flags provided but not defined: -output-format`).
    TSV is the only machine-readable form the subcommand has.

### Added

- **`antigravity_status` reports remaining AI Pro quota, and still spends none.** agy 1.1.11 and
  1.1.12 answer read-only slash commands in print mode themselves — no agent turn, no quota, no
  conversation left behind — so the status tool now runs `agy -p "/usage"` and adds a row per
  model family: `quota: Gemini Models [ok] Weekly 100%, Five Hour 100%`. A family at **0%** is
  reported as a problem, since every call against it will fail until its window resets; the tool
  already existed to tell you what will go wrong before you spend a call, and "you are out of
  quota" is the most common such answer.
  - Gated at agy 1.1.11 as a **safety** gate, not a feature probe: below that the same argv is a
    prompt, so a diagnostic advertised as free would quietly spend a call. On older agy the rows
    are simply absent rather than reported as a failure.
  - The probe is the one bridge call that deliberately **omits** `--disable-slash-commands` — that
    flag is what makes agy treat a leading `/usage` as literal text, which would send the prompt
    to the model. A regression test asserts the argv keeps agy's slash handling (and carries no
    permission opt-out).

### Verified

- **The slash shield still holds on 1.1.12.** 1.1.11 replaced the silent fall-through for
  interactive-only commands with an explicit refusal recommending the exact flag this bridge
  already passes (`agy -p "/clear"` → exit 2, *"pass --disable-slash-commands to send /clear to
  the model as literal text"*). Re-verified end-to-end: `antigravity_ask("/clear Reply with the
  single word BRIDGE…")` returned `BRIDGE`. The new read-only command set is why the shield stays
  load-bearing — unshielded, a prompt opening with `/model` gets agy's table, not an answer.
- **The continue path is unaffected by 1.1.12's diagnostics change.** 1.1.12 stopped swallowing
  startup diagnostics into the log file, including the `--conversation` not-found warning this
  bridge can trigger. They go to **stderr**: verified with a deliberately bogus `--conversation`
  that stdout stays a pure JSON result object (exit 0, warning on stderr, agy silently starting a
  new conversation).
- **`--model` is honored in print mode**, re-confirmed the free way on 1.1.12: `agy --model
  gemini-3.1-pro-high -p "/model"` answers `gemini-3.1-pro-high` where a bare `-p "/model"`
  answers the settings.json default. The live slug list is unchanged from 1.1.6.
- `VERIFIED_AGY_VERSION` → `(1, 1, 12)`. Benign wins needing no change: a **Windows** crash
  resolving the conversation transcript path is fixed (the artifact watch mode's `log_uri` points
  at), headless `-p` now settles a choice itself instead of stalling on an unanswerable question,
  and 1.1.11 made retries honor the server's retry delay and stopped an empty credits response
  reading as "Out of credits". `--effort` stays unadopted with a harder reason than before: it is
  not a universal axis — `--model claude-sonnet-4-6 --effort low` fails with *"--effort is not
  supported for model"* — while the gemini slugs already bake the level in. Everything else in
  1.1.11/1.1.12 (Vim editing mode, artifact-viewer polish, plugin enablement, admin controls and
  MCP progress callbacks) is interactive-TUI or agy-as-MCP-*client* work.

## [0.23.1] - 2026-08-05

Hardens the **watch viewer**, the one part of the bridge that opens a network
listener. No behaviour change: watch mode works exactly as before. Found by reviewing a
downstream fork that had disabled the viewer outright in its own deployment — the underlying
observation was right, but disabling it costs the feature, and a Host check does not.

### Security

- **The watch viewer served prompts, answers, and the commands agents ran with no access
  check.** Both HTTP servers (`server.py`'s single-run viewer and `swarm_watch.py`'s swarm
  dashboard) bind `127.0.0.1` on an ephemeral port, but binding is not an access boundary — and
  the server is started lazily and **never stopped**, so one `watch=true` run leaves the port
  open for the life of the process, not just the run. Every request now passes
  `_watch_authorized`, which applies two independent checks:
  - **Host must be a loopback literal** (`127.0.0.1` / `localhost` / `::1`, port ignored; a
    missing Host is refused). This is what closes **DNS rebinding**: a rebound page reaches the
    port under the *attacker's* hostname, so the check rejects it regardless of anything else.
    Names that merely *resolve* to loopback (`127.0.0.1.nip.io`) are refused too.
  - **A per-process token** (`secrets.token_urlsafe`, compared with `hmac.compare_digest`)
    carried as `k` in the query. The Host check cannot stop another local process — it can send
    any Host it likes — but the token can, which matters on shared or multi-user machines. It
    costs nothing in UX because the bridge opens every one of these URLs itself.

  Verified over real sockets against both servers: loopback+token → 200, missing token → 403,
  wrong token → 403, rebinding Host with a *valid* token → 403. The served pages were checked to
  carry the token through every `fetch` they make, so the viewer still polls normally.
- **`/image` now takes its path as a named `p` parameter** instead of "everything after the `?`",
  which the token sharing the query string would otherwise have corrupted. Its allowlist is
  unchanged and was already tight — `_watch_image_allowed` requires an exact match against a
  registered run's image path, so it was never an arbitrary file read.

## [0.23.0] - 2026-08-05

agy **1.1.9 silently broke print mode** for a whole class of prompt, and a
long-standing decoding bug meant **every non-ASCII answer came back mangled on Windows**. Both are
fixed here. Re-verified against agy 1.1.10 and cursor-agent 2026.07.23; codex 0.144.1 and copilot
1.0.69 are unchanged and still correct.

### Fixed

- **agy 1.1.9 executed slash-prefixed prompts instead of answering them.** 1.1.9 made `-p` expand
  slash commands and skills rather than passing them to the model as literal text. A prompt whose
  **first token** names a registered command was therefore run as that command and never reached the
  model. Verified live on 1.1.10 through the bridge: `antigravity_ask("/help")` returned agy's own
  help page. This was more than wrong output — agy's registered set includes **side-effecting**
  commands (`/goal` launches an autonomous long-running task, `/schedule` creates cron jobs), and
  bridge prompts routinely carry text the caller did not author, so an untrusted string beginning
  `/schedule …` would have run it. `_agy_base_args` now appends `--disable-slash-commands`, which
  covers **all six agy argv paths** at once (ask, continue, both watched runners, both swarm
  workers) since they all build on it. Version-gated at 1.1.9 via `supports_disable_slash_commands()`
  — the flag is absent from older parsers, and so is the expansion — mirroring the existing
  `supports_json_output()` 1.1.8 gate. Prompts starting with a POSIX path (`/etc/hosts …`) were never
  affected, matching no command, but that was luck rather than a boundary.
  **`AGY_BRIDGE_ALLOW_SLASH_COMMANDS=1`** opts back in for callers who *want*
  `-p "/my-skill <args>"` to invoke a skill.
- **Non-ASCII answers were corrupted on any non-UTF-8 locale (Windows).** The backends emit UTF-8,
  but the bridge spawned them with bare `text=True`, which decodes with
  `locale.getpreferredencoding()` — cp1254 on a Turkish Windows, cp1252 elsewhere. Every non-ASCII
  answer came back mojibake: `dosyası` arrived as `dosyasÄ±`, exactly
  `'dosyası'.encode('utf-8').decode('cp1254')`. `server.py` (5 call sites), `codex_bridge.py` (4),
  `copilot_bridge.py` (3) and `swarm.py` (4) now spread a `_TEXT` dict of
  `{"encoding": "utf-8", "errors": "replace"}` — the pattern `cursor_bridge.py` already had and the
  other four modules never picked up. `errors="replace"` keeps a stray byte from raising mid-answer.
  A regression test asserts no module reintroduces bare `text=True` in a subprocess call. ASCII-only
  answers were unaffected, which is why this went unnoticed for so long.
- **A leading or trailing banner made the bridge return a raw JSON blob.** `_parse_json_result`
  required stdout to *start* with `{`, so any surrounding chatter dropped the run onto the plain-text
  fallback and handed the caller the whole serialized result object as their "answer". agy 1.1.10
  started printing exactly such a banner — a non-blocking advisory when the same conversation is
  already open in another CLI instance, precisely what `antigravity_continue` and the swarm can
  trigger. The parser now **locates** the first decodable object carrying the contractual `response`
  key, absorbing leading and trailing text. Plain text still answers `None` and still falls back
  unchanged.

### Changed

- **`VERIFIED_AGY_VERSION` → `(1, 1, 10)`**, so `antigravity_status` stops reporting "newer than
  verified". 1.1.10 fixed `--model`/`--effort` being **silently ignored in headless `-p`** (applied
  after model configuration had already initialized, so the run fell back to the persisted/default
  model) — meaning on **1.1.8–1.1.9 the bridge's `model` argument was a no-op**, even though a typo
  was still correctly rejected up front by `validate_model`. Anyone who pinned a model in that window
  was served the default instead. Re-confirmed working on 1.1.10 through the bridge
  (`model="claude-sonnet-4-6"` returned a Claude answer, not Gemini). No bridge change was required.
  Nothing else in 1.1.9/1.1.10 reaches the bridge — the remainder is interactive-TUI, hooks, auth,
  and MCP-*client* work.
- **`cursor_bridge.py` dropped its redundant `text=True`.** Every cursor subprocess already spread
  `_TEXT`, so `text=True` alongside it was a no-op (`encoding` implies text mode). Removing it makes
  the "never bare `text=True`" rule uniform across all five modules and therefore testable.

### Docs

- **Re-verified cursor-agent at 2026.07.23** (badge, requirements, and the Cursor bridge section had
  all still said 2026.07.08, even though `cursor_bridge.py`'s own docstring was bumped in 0.22.1).
  The live menu is now **197 ids**; the README's `sonnet-4-thinking` example is retired and replaced
  with the live `claude-4-sonnet-thinking`. New families the docs hadn't caught up with: Opus 5,
  GPT-5.6 Sol/Terra/Luna, Kimi K3, GLM 5.2. The bridge validates against `cursor-agent models` live,
  so this was documentation rot only — but every rotted example is a guaranteed rejected call.
- agy badge → 1.1.10, and the requirements line now records the behaviour re-verification separately
  from the 1.0.15 state-file layout check.

## [0.22.1] - 2026-07-28

Swept the other three CLIs after the agy work. **codex and copilot needed no code
change; cursor did** — and the documented *model names* had rotted for both cursor and copilot,
which is the agy-1.1.5 failure mode all over again: the bridge keeps working while every example a
caller might copy becomes a guaranteed rejected call.

### Fixed

- **cursor: `validate_model` rejected a form cursor itself documents.** `cursor-agent models`
  enumerates **pre-composed** ids — `claude-opus-4-8-high`, `-xhigh`, `-thinking-max`, each with a
  `-fast` twin — and never the bare `claude-opus-4-8`. But cursor's own `--help` advertises exactly
  that bare id as the base for its bracket syntax (`claude-opus-4-8[context=1m,effort=high,fast=false]`),
  so requiring an exact match blocked a valid call **with no workaround**, which is strictly worse than
  letting a typo through (a typo costs one call and earns cursor's authoritative error). Validation now
  accepts an exact id **or a family base** (a prefix ending at a `-` boundary). It stays tight enough to
  matter: `gpt-5` still fails against `gpt-5.2` (no `gpt-5-` id exists), as do retired names. The known,
  deliberate looseness — a partial prefix like `claude-opus` passes — is documented and tested.
- **cursor: two documented model examples no longer exist.** 2026.07.23 reshuffled the list (193 ids):
  `sonnet-4-thinking` and the bare `claude-opus-4-8` that the README and the `cursor_ask`/
  `cursor_continue` docstrings named are **gone**. They now name live ids (`auto`, `gpt-5.2`,
  `claude-opus-4-8-high`, `composer-2.5`, `cursor-grok-4.5-high`), explain that cursor bakes the effort
  and speed axes into the id, and tell you to check `cursor-agent models` rather than trust an example.
- **copilot: both documented model examples are dead.** Verified live on 1.0.69 — `gpt-5.3-codex` and
  `claude-sonnet-4.6` both return `Model "…" is not available`, and so does GitHub's own `--help`
  example `gpt-5.4`. Only `auto` worked. copilot exposes **no non-interactive model list** (no `models`
  subcommand, and the error names no alternatives), so unlike the agy and cursor bridges this one
  *cannot* validate up front — a bad id costs a real call. The docs now name only `auto`, say plainly
  that the working set is account-dependent and unvalidatable, and steer callers toward omitting the
  argument.

### Added

- **`DOCUMENTED_CURSOR_MODELS` guard test**, mirroring `DOCUMENTED_AGY_MODELS`: it checks every model
  id the docs advertise against the **live** `cursor-agent models` output, so this rot fails a test
  instead of a user's first call. (Skips when cursor isn't installed.)

### Changed

- `cursor_bridge` verified version → **2026.07.23** (from 2026.07.08; cursor self-updated twice during
  this session, 07.09 → 07.23). Everything structural still holds — `create-chat` mints an id, `models`
  lists, `status` reports auth, every flag the bridge passes is still in `--help`, and the
  `chats/<id>/meta.json` `cwd` layout the continue-fallback reads is unchanged.
  ⚠️ **A live cursor agent turn could not be re-verified** — the account hit its Cursor usage limit — so
  the run path is confirmed by structure plus the bridge cleanly surfacing cursor's own limit error,
  not by a completed round-trip.
- **codex 0.144.1 and copilot 1.0.69 need nothing.** Both match the version their bridge was verified
  against, both `*_status` checks pass, and both completed a live ask + continue round-trip with the
  thread carried. Codex names no specific model ids in its docs, so it had nothing to rot.

## [0.22.0] - 2026-07-28

### Added

- **agy 1.1.8 structured print-mode output — the bridge now asks for `--output-format json`.** 1.1.8
  gave `-p` an `--output-format` flag (`text` | `json` | `stream-json`) plus `--json-schema`. Nothing
  was broken by it (see *Changed*), but the `json` format is strictly better than reading bare text,
  so `_run_agy` requests it whenever the installed agy is 1.1.8+. Verified live, the result is one
  object — `{conversation_id, status, response, duration_seconds, num_turns, usage{…,
  cache_read_tokens}}` — and it composes with every flag the bridge passes: a `--conversation`
  continue came back `num_turns: 2` with conversation memory intact, `--model` round-tripped, and
  `--dangerously-skip-permissions` is unaffected.
- **Continue now pins to the conversation agy itself reported.** The real prize in that object is
  `conversation_id`: agy telling the bridge *exactly* which conversation a run used. Previously that
  could only be inferred from `last_conversations.json` — shared state agy rewrites for every session,
  including your own interactive TUI work in the same folder — or from brain-dir mtimes. `_run_agy`
  now records the id per workspace and `_build_agy_args` prefers it when pinning, falling back to
  `last_conversations.json` and then `-c`. **Behavior worth knowing:** on agy 1.1.8+,
  `antigravity_continue` resumes *the bridge's own* last thread in that workspace, where before it
  could land on a conversation you had since started interactively in the same folder. That is what
  the pinning always intended; it can only now be done exactly. The map is process-local, so a
  restarted server simply falls back to the previous resolution order.

### Changed

- **⚠️ Behavior change: a multi-step answer now includes the model's narration.** agy's `response` is
  the *whole turn* — every agent-response step concatenated. The old transcript scrape returned only
  the **last** completed planner response. On a single-step ask the two are identical (verified:
  `'0.21.4'` either way), but on a chatty multi-step run they differ — measured on one run, 297 chars
  (`"I will first count the .py files.\nNext, I will print today's date.\n…\nCompleted counting .py
  files (19 found)…"`) versus 128 for the final step alone, with the full answer *ending in* the old
  one. The full turn is what every path now returns. Rationale: `result.response` is agy's own
  contract for "what this turn produced", and this release's whole direction is to trust that
  contract rather than the bridge's reconstruction of it. The last-step rule was never a product
  decision — it was the best a transcript scrape could do — and it silently **drops content** whenever
  the model does the real work and then closes with a short "Done." The cost is narration noise on
  multi-step runs; the watch window exists to show that narration, but dropped content isn't
  recoverable.
- **Re-verified against agy 1.1.7 and 1.1.8 — nothing broke.** Before changing anything, the existing
  text path was confirmed live on 1.1.8: ask, conversation-pinned continue, and `--model` all returned
  clean. The new json path was then verified end-to-end through the bridge itself (answer, captured
  `conversation_id`, same-thread continue, `--model`, a tool-heavy run, and the non-watched
  `antigravity_image` path, which runs through `_run_agy` too). `VERIFIED_AGY_VERSION` → `(1, 1, 8)`
  so `antigravity_status` stops reporting "newer than verified". The watched paths were verified the
  same way — a live watched run, a live watched image generation, and the **pre-1.1.8 fallback**
  exercised by forcing the support probe off against a real agy (no `--output-format` in argv,
  transcript scrape, right answer).
- **The version gate is deliberate.** On 1.1.8 an unrecognised `--output-format` *value* is silently
  ignored and agy prints plain text (verified: `--output-format bogus` exited 0 with a normal answer),
  but a pre-1.1.8 agy has no such *flag* and could fail arg parsing on every call. So
  `supports_json_output()` checks the version and answers `False` when it can't be parsed, and
  `_parse_json_result` returns `None` for anything that isn't the expected object — a
  silently-ignored or removed flag degrades to the text path instead of crashing. An unexpected
  `status` that still carries a `response` returns the answer rather than discarding it.
- Structured output is **opt-in per caller**, not folded into `_agy_base_args`: the watch and image
  runners read agy's stdout as text while polling the transcript for live progress, and the swarm
  workers build their own argv, so only the plain ask/continue path requests it.
- Nothing else in 1.1.7 or 1.1.8 reaches the bridge. 1.1.7's `-p` fix (it sent a prompt before the
  account-eligibility check finished) is a benign win on the exact path the bridge drives; its
  disabled-plugins-still-running-hooks fix, MCP OAuth relaxation (agy as an MCP **client** — the
  opposite direction from this bridge), `/btw` first-action crash, and Windows CJK clipboard fix are
  interactive or client-side. Same for the rest of 1.1.8: `copyOnSelect` is a TUI setting, and the
  compound-command allow-always improvement is an interactive permission-prompt change that print mode
  bypasses anyway.

- **Watch mode now reads agy's event stream instead of scraping its transcript.** The watched runners
  (`antigravity_ask/continue` with `watch=true`, and watched image generation) request
  `--output-format stream-json` and consume agy's typed NDJSON events — `init` / `step_update` /
  `result` — live off stdout, via the new `_StreamWatch`. The premise was verified *before* the
  rewrite: events arrive **incrementally** (a 17 s run spread its 18 events over 12.4 s), and a live
  watched run through the bridge grew its step count 0 → 3 → 6 → 7 while agy worked. What the stream
  gives that the scrape could not: the real `tool_info.parameters.CommandLine` as a nested object
  (the transcript stores tool args JSON-encoded *inside a string* — that's what `_clean_tool_arg`
  unwraps), `text_delta` fragments per response step, explicit `ACTIVE`→`DONE` transitions, and a
  `conversation_id`. Consequence: a **watched** run now records its conversation for later pinning
  exactly like the plain path, closing an asymmetry where watch still leaned on
  `last_conversations.json`. This retires the timer-based transcript polling on agy 1.1.8+ — notable
  because agy has announced JSONL is going away in favour of SQLite, and watch was the last path that
  would have broken when it does.
- **`watch=true` and `watch=false` can no longer disagree about the answer.** Both now read the same
  `response` field from agy's result. Previously the watched path returned the transcript's last
  planner response while the plain path returned agy's `response`.

### Not adopted

- **`--json-schema`** — works (verified: it adds a validated `structured_output` object alongside the
  prose `response`), but no bridge tool currently needs a caller-supplied schema.

## [0.21.4] - 2026-07-24

### Changed

- **Re-verified against agy 1.1.6 — no code change needed.** agy self-updated 1.1.5 → 1.1.6, which
  added the `gemini-3.6-flash` family to `agy models` and moved the `settings.json` default to
  **Gemini 3.6 Flash (High)**. Verified live through the bridge: the default path and
  `--model gemini-3.6-flash-high` both round-tripped clean, and both read paths still work on 1.1.6's
  **unchanged** conversation schema — the JSONL transcript and the SQLite `steps` fallback each
  extracted the same answer from a real `-p` run. `validate_model` needed nothing: it matches whatever
  `agy models` prints, so the new slugs pass and typos still fail fast. `VERIFIED_AGY_VERSION` →
  `(1, 1, 6)` so `antigravity_status` stops reporting "newer than verified".
- **Docs refreshed for the new model list.** The module docstring, the `antigravity_ask` param docs,
  and the README's model table + FAQ now enumerate the 1.1.6 list (`gemini-3.6-flash-{low,medium,high}`
  ahead of the 3.5 flash, pro, Claude, and gpt-oss entries) and name `gemini-3.6-flash-high` as the
  default, so no documented example steers a caller at a stale default. `DOCUMENTED_AGY_MODELS` gains
  `gemini-3.6-flash-high` so the live-list guard test covers the new default.
- Nothing else in 1.1.6 reaches the bridge. Its one adjacent fix — print mode surfacing the real
  conversation-creation failure instead of a misleading "no active conversation" — only sharpens the
  error the bridge already reads on a failed run. The rest (Markdown custom agents, `/copy <n>` and
  streaming `/codesearch`, the temp-dir read grant, background-task double-spawn and print-mode
  fixes) is interactive-TUI or client-side work that doesn't touch the print-mode + transcript path
  this bridge drives.

## [0.21.3] - 2026-07-21

### Changed

- **agy 1.1.5 renamed every model — the docs, not the code, were what broke.** 1.1.5 introduced
  "stable, user-facing model slugs" accepted by `--model`, and `agy models` now reports *only* those:
  `gemini-3.5-flash-{low,medium,high}`, `gemini-3.1-pro-{low,high}`, `claude-sonnet-4-6`,
  `claude-opus-4-6-thinking`, `gpt-oss-120b-medium`. The old human labels (`"Gemini 3.5 Flash (High)"`,
  `"Claude Sonnet 4.6 (Thinking)"`) are gone from the list, so `validate_model` rejects them. Verified
  live on 1.1.5: the old label raised `unknown agy model ... expected one of: <slugs>` **without
  spending an agy call**, while `--model gemini-3.5-flash-high` round-tripped clean. No plumbing
  changed — validation matches whatever `agy models` prints — but every docstring and README example
  named a now-invalid label, i.e. the docs were steering callers into a guaranteed failed first call.
  All of them now name slugs, in `antigravity_ask` / `antigravity_continue` / `agent_swarm`, the
  module docstring, and the README's model table and FAQ.
- `VERIFIED_AGY_VERSION` → `(1, 1, 5)` so `antigravity_status` stops reporting "newer than verified".
  Nothing else in 1.1.5 reaches the bridge: the `/effort` command and `--effort` flag are a second
  model axis we deliberately don't pass (the slug already pins the effort variant), the agent-frontmatter
  `model` option and `subagent: false` fix are interactive-subagent features, the MCP fixes (embedded
  resources, auth crash, call timeouts) are agy acting as an MCP **client** — the opposite direction
  from this bridge — and the rest is TUI, `/diff`, `/btw`, and background-task lifecycle work. The
  headless-`settings.json` note that 1.1.5 repeats was already assessed in 0.21.2:
  `--dangerously-skip-permissions` still overrides those policies.

### Added

- **A test that would have caught this.** `test_documented_model_slugs_still_accepted_by_live_agy`
  checks every model name the docs hand to callers against the live `agy models` output, failing with
  the current list when agy renames things. The existing model tests all mock `list_agy_models`, and
  the plumbing is format-agnostic, so all 347 stayed green through a break that made every documented
  `model` value invalid — a green suite was not evidence the docs were right. Spends no AI Pro quota
  (`agy models` is a local subcommand) and skips where agy isn't installed.

## [0.21.2] - 2026-07-20

### Changed

- **Re-verified against agy 1.1.4 — no code change needed.** 1.1.4 walked back part of the 1.1.3
  headless gate: `-p` now honors your persisted `settings.json` policies (permissions, file access,
  sandbox mode, auto-execution, artifact review) rather than blanket-denying. The bridge's
  `--dangerously-skip-permissions` still overrides those policies, so it stays load-bearing and stays
  where it is. Verified live against a workspace deliberately **absent** from `trustedWorkspaces`,
  with a `permissions.allow` list naming neither file nor command access: a workspace file read
  returned the right contents, and a terminal command and a file write both executed.
  `VERIFIED_AGY_VERSION` is bumped to `(1, 1, 4)` so `antigravity_status` stops reporting
  "newer than verified", and the module docstring records the 1.1.4 review.
- Two 1.1.4 notes carried into the docs: the flag is now the only thing between a bridge call and
  the user's own `settings.json` policy (dropping it yields that policy, not 1.1.3's deny-all), and
  1.1.4's `/btw` fix — side-questions no longer leak into the conversation list as duplicates
  carrying the parent's title — removes one way for `_read_last_conv_id` to pin the wrong thread.

## [0.21.1] - 2026-07-16

### Fixed

- **agy 1.1.3 broke every tool-using Antigravity call; the bridge now passes
  `--dangerously-skip-permissions`.** 1.1.3 stopped headless `-p` from auto-approving tool calls:
  a tool needing a permission confirmation is **soft-denied** (print mode cannot prompt), agy exits
  **0** having done nothing, and the reason appears only on stderr. This denied even a plain file
  read, so `antigravity_ask` / `antigravity_continue` / `antigravity_image` and the antigravity
  `agent_swarm` workers could no longer read files or run commands. All six agy argv paths now go
  through a shared `_agy_base_args()` that passes the flag — agy's own remedy — restoring file
  writes, terminal commands and workspace reads (verified live end-to-end).
  - **The flag must precede `-p`.** agy's `-p`/`--print` takes the prompt as its *value*, so
    `-p --dangerously-skip-permissions <task>` parses the flag *as* the prompt and silently drops
    the task. `_agy_base_args()` documents this; every caller appends `-p` last.
  - **No opt-out knob**, deliberately. 1.1.3's gate is not a usable safety mode for this bridge (it
    soft-denies read-only reads and cannot prompt), so a "gated" call would be a sub-agent that can
    do nothing. The security posture is unchanged: `-p` still runs arbitrary code with your
    privileges — trusted prompts only.
- **A failed agy run no longer reports a misleading transcript error.** When agy exits 0 but writes
  neither stdout nor a readable transcript, every agy runner now folds agy's stderr into the raised
  error instead of surfacing only the scrape failure, which read as a bridge/schema bug and hid agy's
  actionable notice (e.g. the allow-rule 1.1.3 asks for). Covers all four paths: `_run_agy`, the
  watched `_run_agy_watched`, and both swarm text workers. (The image workers key off the produced
  file, not a transcript scrape, so their "no image produced" error was already meaningful.)

### Changed

- **Docs re-verified against agy 1.1.3** (badge, SECURITY notes, compat log, and the
  `VERIFIED_AGY_VERSION` constant → `(1, 1, 3)` so the startup staleness warning no longer fires on a
  current agy). Two long-standing claims are now retired: `--dangerously-skip-permissions` is no
  longer a no-op for `-p`, and an unknown `--model` no longer falls back silently — **1.1.2** made it
  hard-fail in print mode with a non-zero exit. `validate_model` stays: it fails fast without spending
  an agy call, and still matters on pre-1.1.2 agy.
- **All four backends re-verified against their current CLIs; only agy needed a code change.** The
  other three bumped without breaking anything the bridge depends on (flags and on-disk session
  layouts intact): **codex-cli 0.141.0 → 0.144.1** (live `codex_ask` round-trip) and **copilot 1.0.68
  → 1.0.69** (live set-then-resume round-trip — 1.0.69 adds a `--resume` convenience flag the bridge
  doesn't use, and `--session-id` still both sets and resumes an id) have their verified badges/notes
  bumped. **cursor-agent 2026.07.08 → 2026.07.09** is status-verified (found, logged in, flags/layout
  intact) but its live round-trip was deferred (Cursor usage limit), so its verified claim stays at
  2026.07.08.

## [0.21.0] - 2026-07-10

### Added

- **Fourth backend: Cursor (`cursor-agent`).** The bridge now drives Cursor's agent CLI alongside
  Antigravity, Codex, and Copilot, with the same shape as the others: `cursor_ask`,
  `cursor_continue`, and `cursor_status`, plus a `cursor` backend option in `agent_swarm`. Fifteen
  tools total.
  - **Stdout-native.** `cursor-agent -p --output-format text --trust` returns the clean final answer
    on stdout — no scraping, like Codex/Copilot.
  - **Deterministic, race-free continue.** The bridge mints a chat id with `cursor-agent create-chat`,
    runs the ask with `-p --resume <id>`, and pins it to the workspace; `cursor_continue` resumes that
    exact chat. Restart-proof fallback: the newest on-disk chat under
    `~/.cursor/chats/<md5(workspace)>/<chat-id>/` whose `meta.json` cwd matches.
  - **Wide model menu.** `model` maps to `--model` (GPT / Claude / Grok / Composer, and parameterized
    ids like `claude-opus-4-8[context=1m]`), validated against `cursor-agent models` and rejected on a
    typo — like agy.
  - **Sandbox knob** maps to cursor's mode/force flags for cross-backend uniformity: `read-only` =
    `--mode ask` (agent-enforced — write/shell tools unavailable; best-effort, not an OS sandbox, so
    for a hard boundary use Codex), `workspace-write` = `--force`, `danger-full-access` =
    `--force --sandbox disabled`.
  - **Watch mode** streams cursor's steps from its `--output-format stream-json` event stream (with a
    `cursor` swarm badge). Note: cursor stores its transcript in an opaque SQLite blob store, so a
    watched `cursor_continue` opens without prior-turn history (best-effort).
  - **Auth** uses your Cursor login (`cursor-agent login`) or a `CURSOR_API_KEY` env var; check with
    `cursor_status`. On Windows the `cursor-agent.CMD` shim is resolved via PATH; set `CURSOR_BIN` to
    override (mirrors `AGY_BIN` / `CODEX_BIN` / `COPILOT_BIN`).
  - Covered by a new `test_cursor.py` (54 offline tests) plus server/swarm wiring tests.

## [0.20.1] - 2026-07-10

### Fixed

- **The "newer bridge available" notice now shows on any single-backend install, not just
  Antigravity.** `codex_status` and `copilot_status` reported only their own CLI's version and auth,
  so a user who installed just Codex or just Copilot (each backend is independent) never saw the
  bridge's own update notice from a status tool — it lived only in `antigravity_status` and the
  startup stderr warning (which lands in host logs, not the chat). Both now prepend the same
  `bridge version` row (bridge version + best-effort GitHub update check, honoring
  `AGY_BRIDGE_NO_UPDATE_CHECK`) that `antigravity_status` already showed. Guarded by two regression
  tests.

## [0.20.0] - 2026-07-10

### Added

- **The bridge now teaches your MCP client how to use it.** The server ships a set of server-level
  `instructions` (FastMCP's `instructions=`, sent to the client in the `initialize` response), which
  a client like Claude Code injects into its model's context as an "MCP Server Instructions" block.
  So every user who installs the bridge gets their host model taught — with no README reading —
  *when* to reach for these tools (image generation → `antigravity_image`, independent sub-tasks →
  `agent_swarm`, a second/heavier model's opinion, or offloading grunt work off its own token
  budget), *which* backend to pick (Antigravity / Codex / Copilot and their sandbox differences),
  and the most common footgun (pass `workspace` = the project dir, or the sub-agent answers with no
  repo context). Kept deliberately tight — a routing guide, not a manual — since it costs context
  tokens every session; per-tool detail still lives in each tool's own description/annotations.
  Guarded by three regression tests (`test_server.py`).

## [0.19.5] - 2026-07-09

### Changed

- **Docs brought in line with the chat redesign.** The watch-mode README section still named the
  expand/collapse toggle by its old Turkish label ("daha fazla / daha az" → "show more / show less"),
  the GIF table header omitted Copilot ("agy or codex" → "agy, codex, or copilot"), and the GIF
  alt-text/caption described the old streaming view — reworded to the chat-conversation framing
  (a **CLAUDE** prompt bubble → collapsible step trace → Markdown answer card tagged by backend).
  Also refreshed `swarm_watch.py`'s module docstring, which still called the single-worker viewer a
  "singleton (`_WATCH_STATE`)" and mentioned a "typewriter" — neither true after the id-keyed
  multi-window state (`_WATCH_RUNS`) and the chat redesign.

## [0.19.4] - 2026-07-09

### Changed

- **Re-captured the watch GIFs with the English UI.** `assets/watch-ask.gif` and
  `assets/watch-image.gif` were regenerated after the 0.19.3 string fix, so the on-screen labels in
  the GIFs ("working…", "N steps", "show more", "close") now match the app instead of the earlier
  Turkish captures. Frames inspected again for privacy before commit — no paths/usernames leaked.

## [0.19.3] - 2026-07-09

### Changed

- **Watch window UI text is now all English.** A handful of labels in the chat viewers (single-worker
  and swarm per-worker) had slipped into Turkish during the redesign — they now read "working…",
  "show more" / "show less", "N steps", "jump to latest", "close", and "queued" (from "çalışıyor…",
  "daha fazla" / "daha az", "N adım", "en alta in", "kapat", "sırada").

## [0.19.2] - 2026-07-09

### Changed

- **Refreshed the watch-mode GIFs for the chat redesign.** `assets/watch-ask.gif` and
  `assets/watch-image.gif` now show the current chat UI (a **CLAUDE** prompt bubble → a live
  collapsible step trace → a Markdown answer card / inline image) instead of the prior terminal-style
  view. The capture tool (`tools/capture_watch_gif.py`) was made safe for a public asset: the `ask`
  capture uses a path-free prompt (a knowledge question + `git --version`) so agy touches no files
  and no absolute paths or usernames appear on screen, and the `image` capture saves under a neutral
  `C:\Users\Public` dir so the "Saved to …" caption carries no personal username. (The swarm
  dashboard GIF is unchanged — the chat redesign only touched the per-worker detail window, which
  that GIF doesn't capture.)

## [0.19.1] - 2026-07-09

### Changed

- **Codex/Copilot watch finishes ~2s sooner after the answer.** The streaming runners join their
  stdout/stderr reader threads after the process exits; when a lingering child holds the pipe open
  those joins hit their timeout, and two 2s joins made the window sit on "working" ~4s past the
  answer. Trimmed each grace to 1s (~2s total), keeping the "done" transition snappy. The answer is
  unaffected — it comes from codex's `-o` file / copilot's stream, not the trailing pipe.

### Tests

- **Regression guard for the 0.18.1 streaming-exit fix.** New unit tests in `test_codex.py` /
  `test_copilot.py` launch a fake CLI that emits its JSON events, spawns a child that keeps the
  stdout pipe open, then exits — asserting `run_codex_streaming` / `run_copilot_streaming` return
  promptly (on process exit) with the right answer instead of blocking on the lingering child. No
  codex/copilot quota (the fake is a local Python script).

## [0.19.0] - 2026-07-09

### Added

- **Concurrent single-worker watch runs no longer clobber each other's window.** The single-worker
  viewer's state was a process-wide singleton, so two watched runs that overlapped (e.g. a
  `codex_ask` and a `copilot_ask` at the same time — they don't share the agy lock) wrote into one
  shared window and garbled it. The viewer state is now keyed by a per-run id (`_WATCH_RUNS`): the
  common **sequential** case still reuses the `"main"` slot and its already-open window, but a run
  that begins while another is **still working** is given its own id and its **own window**. The
  browser page reads its id from the URL (`/?id=…`) and polls `/events?id=…`; `/image` validates the
  requested path against all live runs. Finished, unwatched runs are evicted so the map can't grow
  without bound. Verified: two concurrent runs render into two isolated windows (each shows only its
  own prompt/answer/backend); a unit test asserts the second run gets a distinct id and independent
  state.

## [0.18.1] - 2026-07-08

### Fixed

- **Codex/Copilot watch mode no longer hangs until timeout after the answer is ready.** The streaming
  runners (`run_codex_streaming` / `run_copilot_streaming`) ended their live loop on **stdout EOF**
  (`for line in proc.stdout`), but codex (and copilot — both node CLIs) can leave a child process
  holding the stdout pipe open after the turn completes, so the loop blocked past completion and the
  watched run only returned when the watchdog killed it at `timeout+30s` — the window showed the
  answer but kept spinning. Completion is now driven by **process exit** (`proc.wait(timeout=…)`) with
  stdout/stderr read on daemon threads, so the run finishes as soon as the CLI exits regardless of a
  lingering child (verified live: a codex watched ask returned in ~4.6s instead of timing out). This
  also drains stderr concurrently, closing the same pipe-buffer deadlock class fixed for agy in 0.17.2.
- **Fresh `workspace` dirs no longer crash the single-worker tools with `WinError 267`.** `antigravity_ask`
  / `antigravity_continue` / the image tool and `codex_*` / `copilot_*` set the child's cwd to
  `workspace` but didn't create it, so a not-yet-existing workspace failed with a cryptic "directory
  name is invalid". Each runner now `os.makedirs(workspace, exist_ok=True)` before spawning, matching
  what the swarm workers already did.

## [0.18.0] - 2026-07-08

### Added

- **Watch mode is now a chat conversation.** Both the single-worker viewer (`antigravity_ask` /
  `antigravity_continue` / the image tool / `codex_*` / `copilot_*` with `watch=true`) and the swarm's
  per-worker detail window were redesigned as a chat UI: the prompt shows as a **chat bubble** (long
  ones clamp with a **daha fazla / daha az** expand-collapse toggle), the agent's live steps stream in
  a collapsible "thinking" trace, and the answer arrives as a Markdown card tagged with the backend.
  Prompt bubbles are labelled **CLAUDE** (the MCP client authors the prompt), answers by backend
  (**AGY** / **CODEX** / **COPILOT**).
- **`*_continue` watch shows the conversation history.** A continued run now opens seeded with the
  **prior turns** of the conversation instead of a blank new window, so it reads as one ongoing
  thread. History is reconstructed from each backend's own session store — agy's JSONL transcript
  (user turns unwrapped from their `<USER_REQUEST>` envelope; the final planner response per turn),
  codex's rollout (`event_msg` `user_message` / `agent_message`), and copilot's `events.jsonl`
  (`user.message` / `assistant.message`). Best-effort: a fresh ask, an unresolved session, or an
  unreadable store yields no history and the run proceeds normally. (agy caps very long older turns in
  its transcript, so their tails can be clipped; the live/last turn is complete.)

## [0.17.2] - 2026-07-08

### Fixed

- **Watch mode no longer deadlocks on long answers (false timeouts + truncated output).** All four
  *watched* runners (`_run_agy_watched`, `_run_agy_image_watched`, and the swarm's
  `_run_text_worker_watched` / `_run_image_worker_watched`) opened agy with `stdout`/`stderr` as
  pipes but never drained them while polling — the classic `Popen` deadlock. Since agy 1.0.15+ writes
  its full final answer to stdout, a large answer filled the fixed OS pipe buffer, blocked agy's
  write, and hung it until the hard deadline fired: a **spurious timeout with a half-written
  transcript answer**. Each pipe is now drained on a background thread (a new `_drain_pipe` helper),
  matching what `subprocess.run` already does for the non-watched paths. Verified with a real 3 MB
  child: the old loop never exits (deadlock) while the drained loop finishes in 0.2 s with the full
  output captured. (The non-watched and Codex/Copilot streaming paths were never affected.)
- **Swarm detail window shows the whole prompt when expanded.** In the swarm watch dashboard's
  per-worker log window, a long prompt expanded past the bottom of the window with no scrollbar, so
  its tail was unreachable. The expanded prompt panel is now capped at `46vh` and scrolls internally,
  mirroring the single-worker viewer in `server.py`.

## [0.17.1] - 2026-07-08

### Changed

- **Verified against agy 1.1.0** (`VERIFIED_AGY_VERSION`), silencing the spurious "newer than
  verified" startup warning that fired for every 1.1.0 user. Full bridge round-trip re-confirmed live:
  `antigravity_ask` + conversation-pinned `antigravity_continue` return clean over stdout, and the
  state-file layout (`last_conversations.json`, JSONL-primary transcript, SQLite dual-write) is intact.
- **agy 1.1.0's new agent execution modes do not affect the bridge.** 1.1.0 added a `--mode` launch
  flag (`accept-edits` | `plan`) and a new interactive **request-review** default that pauses before
  file writes for a diff preview. Because the bridge spawns `-p` with **DEVNULL stdin**, the
  request-review approval gate never engages (it needs an interactive stdin) — print mode still
  auto-executes every tool call. Verified: a file-writing task completed in ~36 s (exit 0),
  identically with and without `--mode accept-edits`, so the bridge keeps passing neither `--mode`
  nor `--sandbox`.
- **Sandbox behavior unchanged on 1.1.0.** Re-verified that `--sandbox` still blocks terminal
  commands (a sandboxed terminal run timed out with no response, exit 1) but still does **not** gate
  `write_to_file` (a sandboxed write succeeded outside the workspace, exit 0). `--mode` and
  `--sandbox` coexist without error; neither makes `-p` safe. The SECURITY note stands.

## [0.17.0] - 2026-07-03

### Added

- **Model selection for Antigravity.** `antigravity_ask`, `antigravity_continue`, and the
  `antigravity` swarm backend now take an optional `model` — agy's `--model` (e.g.
  `"Gemini 3.1 Pro (High)"`, `"Claude Sonnet 4.6 (Thinking)"`). Through ~1.0.14 switching model in
  `-p` **hung** the call, so the bridge stayed single-model; **re-verified on agy 1.0.16 that the
  hang is fixed** — a Claude label answers as Anthropic Claude, a Gemini label as Gemini, each
  returning in seconds. Omit `model` to use agy's `settings.json` default (Gemini 3.5 Flash (High)).
  All three backends now expose the same `model` knob.
- **`agy models` validation.** agy **silently ignores** an unknown `--model` label (falls back to
  the `settings.json` default with no error), so a typo would quietly run the wrong model. The bridge
  now validates a requested label against `agy models` (`validate_model`) and rejects an unknown one
  up front — matching codex/copilot's fail-fast. If the label list can't be read (agy missing, models
  call failed), validation is skipped and the label passes through unchecked.

### Changed

- **Verified against agy 1.0.16** (`VERIFIED_AGY_VERSION`). State-file paths, transcript schema, and
  the 1.0.15 stdout path re-confirmed with a live round-trip; the `--model` print-mode fix is the
  notable change this release builds on.

## [0.16.0] - 2026-07-02

### Added

- **Third backend: the GitHub Copilot CLI (`copilot`).** Three new MCP tools —
  `copilot_ask`, `copilot_continue`, `copilot_status` — plus a `copilot` backend for `agent_swarm`
  (aliases `gh`/`github`) and full watch-mode support. Like Codex it's stdout-native: `copilot -p
  "…" -s` writes the clean answer straight to stdout (no scraping). Highlights, verified live against
  **copilot 1.0.68**:
  - **Deterministic, race-free continue.** `copilot`'s `--session-id <uuid>` both *sets* a new
    session's id and *resumes* an existing one, so the bridge generates the id itself, pins it to the
    workspace, and resumes that exact session — no rollout-scraping. Restart-proof fallback reads the
    newest `~/.copilot/session-state/<id>/workspace.yaml` whose `cwd` matches the workspace.
  - **Model selection** via `--model` (a first-class knob, like Codex's `-m`).
  - **`sandbox` knob** mapped to copilot's tool/path permissions for a uniform cross-backend field:
    `read-only` (default — best-effort: denies the local `write`/`shell` tools; **not** an OS
    sandbox, so unlike Codex it isn't a hard boundary), `workspace-write` (writes confined to the
    workspace), `danger-full-access` (`--allow-all`).
  - **Fast, predictable latency.** Runs headless with `--allow-all-tools --no-ask-user
    --no-auto-update`, and disables copilot's builtin GitHub-API MCP by default
    (`--disable-builtin-mcps`) because its flaky HTTP connect could stall a call up to ~60 s. Set
    `COPILOT_GITHUB_MCP=1` to keep it. `COPILOT_BIN` overrides the executable path (mirrors
    `AGY_BIN`/`CODEX_BIN`), useful since the winget install may be off a stale `PATH`.

### Changed

- **Verified against agy 1.0.15 — and now prefers agy's stdout.** agy 1.0.15 fixes the long-standing
  print-mode stdout bug on Windows: `agy -p` now writes its clean final answer straight to stdout in
  a non-TTY subprocess (verified empirically — stdout carries only the answer, no tool-calling
  narration). `_run_agy` now **prefers stdout when present** and falls back to the transcript/`.db`
  scrape only when it's empty (older agy, non-Windows per the changelog, or `--sandbox` runs). This
  removes the bridge's dependence on agy's *undocumented transcript schema* on the happy path — its
  single biggest fragility — and drops the flush-poll latency. Fully backward-compatible: the
  transcript path stays exercised everywhere stdout is empty. Bumped `VERIFIED_AGY_VERSION` to
  `(1, 0, 15)`. The other 1.0.15 changes don't touch this bridge (the "MCP connection timeout → 60 s"
  is agy acting as an MCP *client*, the opposite direction; the rest are interactive-TUI / paste /
  permissions-panel fixes).

## [0.15.3] - 2026-06-30

### Changed

- **Verified against agy 1.0.14.** Bumped `VERIFIED_AGY_VERSION` to `(1, 0, 14)`, silencing the
  startup compat warning and the status tool's "newer than verified" row. Confirmed with a live
  round-trip (`antigravity_status` green on every row; ask round-trip returns clean text): state-file
  paths, `last_conversations.json` (still keyed by workspace path), and the transcript schema (JSONL
  still primary, SQLite fallback intact) are unchanged across 1.0.13 **and** 1.0.14. Both releases are
  interactive-TUI / plugin / skill / browser-task work plus permission-rule tweaks — clipboard image
  paste in tmux, the lifted `/goal` max limit, subagent "always proceeds" artifact approval, the
  plugin-import directory-copy fix, TUI layout/viewport/rewind fixes, skill slash-prefix rendering,
  the prompt-editor undo/redo fix, the restored browser prompt sections, and the permission changes
  (strict-by-default "Always Approve" matching, a `regex:` opt-in, relaxed redirection checks) — none
  of which the headless `agy -p` path uses. Two items looked worth a closer check and both come up
  clean: 1.0.14's "MCP configuration path mismatch" fix is about agy loading custom MCP servers *as a
  client* (the opposite direction from this bridge, which drives agy via its CLI), and 1.0.13's
  removed "Resume in the same project" exit-hint line never mattered because the bridge reads the
  transcript, not agy's stdout (stdout is surfaced only on a non-zero exit). Permissions still do not
  gate `-p`, so the security posture is unchanged.

## [0.15.2] - 2026-06-25

### Changed

- **Verified against agy 1.0.12.** Bumped `VERIFIED_AGY_VERSION` to `(1, 0, 12)`, silencing the
  startup compat warning and the status tool's "newer than verified" row. Confirmed with a live
  round-trip (`antigravity_status` green on every row, ask round-trip returns clean text): state-file
  paths, `last_conversations.json` (still keyed by workspace path), and the transcript schema (JSONL
  still primary, SQLite fallback intact) are unchanged. 1.0.12 is interactive-TUI / rendering /
  keybinding / network-layer work — `--project`/`--new-project` flags and the "default project
  regardless of active workspace" resolution change, Esc-confirm in comment mode, OSC8 hyperlinks,
  reverse diff cycling, ctrl+o scrollback, Makefile/LaTeX code-block rendering, the AES-NI/DPI TLS
  fix, and backtab/pgdown key-string fixes — none of which the headless `agy -p` path uses. The new
  permission-config precedence (per-project files under `~/.gemini/config/projects/` now outrank
  `~/.gemini/antigravity-cli/settings.json`) is config the bridge never reads; the `model` field
  still lives in `settings.json`, and permissions still do not gate `-p`, so the security posture is
  unchanged. The bridge passes no `--project` flag — it relies on `cwd=workspace` plus conversation
  pinning, both verified intact.

## [0.15.1] - 2026-06-24

### Changed

- **Verified against agy 1.0.11.** Bumped `VERIFIED_AGY_VERSION` to `(1, 0, 11)`, silencing the
  startup compat warning and the status tool's "newer than verified" row. 1.0.11 is entirely
  interactive-TUI / keybindings / additive-env-var work (ctrl+c/ctrl+d handling, `/resume`, the
  ctrl+g AltScreen tool-confirmation view, keybinding validation and lazy `keybindings.json`
  creation, `USE_ADC` ADC auth, `AGY_CLI_CMD_OUTPUT_PERCENTAGE`, command-output/ANSI/VCS-tree
  rendering) — none of which the headless `agy -p` path uses. Its one auth change (return an empty
  config when unsigned in) touches agy's own config read; the bridge never reads agy's config, only
  state-file paths and the transcript. State-file layout, `last_conversations.json`, and the
  transcript schema (JSONL + SQLite fallback) are unchanged — re-confirmed via the status tool.

## [0.15.0] - 2026-06-23

### Added

- **SQLite (`.db`) transcript fallback.** When agy's JSONL transcript is missing or empty,
  `_read_response` now reads the answer from agy's SQLite conversation store
  (`conversations/<id>.db`) instead of failing — walking the `steps` table's protobuf `step_payload`
  (the planner response is the sub-message at field 20, its text at field 1) for the last completed
  planner step. This already covers `--sandbox` runs (which write no JSONL) and future-proofs the
  bridge against agy's announced switch to SQLite as the default conversation format. The reader was
  reverse-engineered and verified to match the JSONL answer across 115 local conversations (104
  byte-identical, 11 a superset, **0 wrong**).

## [0.14.1] - 2026-06-23

### Added

- **Unified `agent_swarm`** — one swarm tool that runs a heterogeneous list of tasks across **both**
  backends at once. Each task names its `backend` (`antigravity` or `codex`) plus a `prompt` (and,
  for Codex, `sandbox`/`model`); workers run concurrently in one pool and, with `watch=true`, share
  one **Agent Swarm** dashboard that now shows a per-worker backend badge.

### Changed

- **BREAKING** (despite the patch bump): removed `antigravity_swarm` and `codex_swarm` — use
  `agent_swarm` instead, passing `tasks=[{"backend": "antigravity"|"codex", "prompt": ...}, ...]`.
  `antigravity_image_swarm` is unchanged (Codex has no image model). Tool count 10 → 9.

### Fixed

- CI now also runs `test_codex.py` (it previously ran only `test_server.py` + `test_swarm.py`),
  which surfaced two latent POSIX bugs: a cwd test recursed infinitely (a monkeypatched `os.getcwd`
  called `os.path.abspath`, which re-enters `getcwd` when the path isn't absolute off-Windows), and
  `codex_bridge.format_swarm_results` used `os.path.basename` (wrong for backslash paths off-Windows).
  The recursion is fixed; the basename bug is moot — `swarm_codex` / `format_swarm_results` /
  `_broadcast_workspaces` became dead code when `agent_swarm` replaced the per-backend swarms, so they
  were removed.

## [0.14.0] - 2026-06-23

### Changed

- **BREAKING:** folded Codex's live watch view into a `watch=true` flag on `codex_ask` and
  `codex_continue` (and **removed the separate `codex_ask_watch` tool**), matching how the
  Antigravity tools have worked since v0.11.0. `codex_continue` gains watch mode (it had none
  before). Codex tool count drops 5 → 4; total 11 → 10. Update any client that called
  `codex_ask_watch` to pass `watch=true` to `codex_ask` instead.

## [0.13.0] - 2026-06-23

### Added

- **OpenAI Codex bridge** — drive `codex exec` as a sub-agent alongside Antigravity. Five new tools
  (in a new `codex_bridge.py` module): `codex_ask`, `codex_continue`, `codex_ask_watch`,
  `codex_swarm`, `codex_status`. Unlike `agy -p`, `codex exec` writes its final message to a file the
  bridge requests (`-o/--output-last-message`), so answers are read cleanly with **no
  transcript-scraping**; continue resumes the exact session via codex's own rollout files
  (`codex exec resume <id>`, with a cwd-matched on-disk fallback after a restart). Codex's
  `-s/--sandbox` is a **real** boundary (default `read-only`) and model selection (`-m`) works — both
  exposed as tool parameters. Verified on **codex-cli 0.141.0**.

### Changed

- **Renamed `antigravity-intern` → `agent-intern`** to reflect that the bridge now drives multiple
  agent CLIs (Antigravity + Codex), not just Antigravity. The MCP server name — and therefore the
  tool prefix — becomes `mcp__agent-intern__*`, so **update your client config**. The live viewer is
  now "Agent Intern" / "Agent Swarm". Backend-specific names are unchanged on purpose: the
  `antigravity_*` tools, the `AGY_BIN` / `AGY_BRIDGE_*` env vars, and "Antigravity" itself all keep
  their names.
- README documents both backends; the header animation now shows two Codex agents (cloud + `>_`)
  alongside two Antigravity agents.

## [0.12.1] - 2026-06-19

### Changed

- Re-verified state-file paths, `last_conversations.json`, and the `-p` JSONL transcript
  schema against **agy 1.0.10** (live `antigravity_status` + ask round-trip), and bumped
  `VERIFIED_AGY_VERSION` so the startup compat check no longer warns on 1.0.10. No functional
  change: nothing in agy 1.0.10 — bash-mode stdout escaping, PowerShell default shell, the new
  alert message type, the permission/`settings.json` fixes, or rundll32 browser sign-in — touches
  the paths, schema, or print-mode TTY-leak the bridge depends on.

## [0.12.0] - 2026-06-18

### Added

- Watch panels overhaul (aesthetics + animation + usability), terminal look kept:
  per-panel time progress bars (elapsed / timeout); the swarm dashboard also gains an
  overall done/total bar, per-row time bars, and keyboard navigation (↑/↓ select, ↵
  open); the worker detail window gains the Markdown + typewriter rendering and a copy
  button from the single-worker viewer; a "jump to latest" follow affordance; and
  status-glow / completion-pop animations.

### Changed

- Watch windows are now **reused** across repeated runs instead of stacking a new
  browser window each time watch mode is used — the bridge detects an already-open
  viewer (via its `/events` polling) and lets it pick up the new run (the page resets
  itself; the swarm dashboard rebuilds for the new fan-out). Set
  `AGY_WATCH_ALWAYS_NEW=1` to force a fresh window per run.

## [0.11.0] - 2026-06-18

### Changed

- **BREAKING:** folded the live "watch" view into the single-prompt tools as a
  `watch` flag instead of separate tools. `antigravity_ask`, `antigravity_continue`
  and `antigravity_image` now take **`watch=true`** to open the Antigravity Intern
  browser window — matching `antigravity_swarm`'s existing `watch` flag. This also
  means **`antigravity_continue` gains watch mode** (it had none before). Tool count
  drops from eight to six.
- Swarm dashboard now shows the **full prompt**: dashboard rows wrap to 3 lines
  (were single-line, ellipsis-clipped), and each worker's detail window shows the
  complete, untruncated prompt in an **expandable** PROMPT pane (click to expand /
  collapse). The truncated row caption is unchanged.

### Removed

- **BREAKING:** `antigravity_ask_watch` and `antigravity_image_watch` — superseded
  by `watch=true` on `antigravity_ask` / `antigravity_image`.

## [0.10.4] - 2026-06-18

### Changed

- Docs: refreshed the `bridge version` example in the README so the illustrative
  versions don't imply a stale release is current.

## [0.10.3] - 2026-06-18

### Changed

- Docs: document the in-chat update notice in the README — both surfaces (the
  `bridge version` row in `antigravity_status`, visible in the client's chat,
  and the startup stderr warning in host logs). Refreshes the PyPI
  long-description.

## [0.10.2] - 2026-06-18

### Added

- `antigravity_status` now reports the bridge's own version and whether a newer
  release is available (e.g. `v0.10.1 -> v0.10.2 available; upgrade: uvx
  antigravity-intern@latest`). This surfaces the update notice **in the MCP
  client's chat** — the startup stderr warning only reaches the host's logs.
  Best-effort GitHub check; honors `AGY_BRIDGE_NO_UPDATE_CHECK` and never flips
  the overall status to PROBLEMS FOUND (an available update is informational).

## [0.10.1] - 2026-06-18

### Changed

- Docs: reworked install guidance now that the package is live on PyPI.
  `uvx antigravity-intern` is the recommended install, with **opt-in** updates —
  uvx caches and does not auto-upgrade, so the bridge only runs a release you
  chose to install (it runs unsandboxed code, so this is deliberate). Upgrade
  with `uvx antigravity-intern@latest`; `@latest` in the config opts into
  hands-off auto-updates. Corrected an earlier inaccurate "latest on every
  launch" claim, and swapped the GitHub-release badge for a PyPI version badge.

## [0.10.0] - 2026-06-17

### Added

- **PyPI packaging + `uvx` install.** `antigravity-intern` is now an installable
  package with an `antigravity-intern` console entry point, so it can be launched
  with `uvx antigravity-intern` (isolated) instead of a hardcoded path to
  `server.py`.
- **MCP tool annotations.** All eight tools now carry MCP annotations
  (`readOnlyHint` / `idempotentHint` / `openWorldHint` / `title`) so clients can
  reason about which tools are safe — `antigravity_status` is read-only and
  idempotent; the agy-invoking tools are flagged open-world and non-read-only.
- **Native MCP progress notifications.** `antigravity_ask`, `antigravity_continue`
  and `antigravity_image` now emit MCP progress (a coarse elapsed/timeout bar)
  while agy works, for clients that send a progress token. The browser "watch"
  tools are unchanged.
- **CI** (`.github/workflows/ci.yml`): ruff + offline tests on Windows / macOS /
  Linux across Python 3.10–3.13.
- **Release automation**: tagging `vX.Y.Z` cuts a GitHub Release with generated
  notes (`release.yml`) and publishes to PyPI via Trusted Publishing
  (`publish.yml`).

## [0.9.0] - 2026-06-17

### Added

- **Startup update check.** The server polls the GitHub tags API once at launch
  and logs a one-line warning if a newer release is tagged than the running
  `__version__`. Best-effort: silent when offline/rate-limited, never blocks
  startup. Opt out with `AGY_BRIDGE_NO_UPDATE_CHECK=1`; point at a fork with
  `AGY_BRIDGE_REPO`.

## [0.8.0] - 2026-06-17

### Added

- `antigravity_swarm` and `antigravity_image_swarm`: run several agy workers in
  parallel, each in an isolated state dir, with error isolation.
- Live browser "watch" mode (`antigravity_ask_watch`, `antigravity_image_watch`)
  that streams agy's steps and shows generated images inline.

### Changed

- **BREAKING:** rebranded to "Antigravity Intern"; tools renamed `agy_*` →
  `antigravity_*`.

### Removed

- **BREAKING:** `antigravity_ask_stream` (superseded by watch mode).

[Unreleased]: https://github.com/SinanTufekci/agent-intern/compare/v0.24.0...HEAD
[0.24.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.23.1...v0.24.0
[0.23.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.23.0...v0.23.1
[0.23.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.22.1...v0.23.0
[0.22.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.22.0...v0.22.1
[0.22.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.21.4...v0.22.0
[0.21.4]: https://github.com/SinanTufekci/agent-intern/compare/v0.21.3...v0.21.4
[0.21.3]: https://github.com/SinanTufekci/agent-intern/compare/v0.21.2...v0.21.3
[0.21.2]: https://github.com/SinanTufekci/agent-intern/compare/v0.21.1...v0.21.2
[0.21.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.21.0...v0.21.1
[0.21.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.20.1...v0.21.0
[0.20.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.20.0...v0.20.1
[0.20.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.19.5...v0.20.0
[0.19.5]: https://github.com/SinanTufekci/agent-intern/compare/v0.19.4...v0.19.5
[0.19.4]: https://github.com/SinanTufekci/agent-intern/compare/v0.19.3...v0.19.4
[0.19.3]: https://github.com/SinanTufekci/agent-intern/compare/v0.19.2...v0.19.3
[0.19.2]: https://github.com/SinanTufekci/agent-intern/compare/v0.19.1...v0.19.2
[0.19.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.19.0...v0.19.1
[0.19.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.18.1...v0.19.0
[0.18.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.18.0...v0.18.1
[0.18.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.17.2...v0.18.0
[0.17.2]: https://github.com/SinanTufekci/agent-intern/compare/v0.17.1...v0.17.2
[0.17.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.17.0...v0.17.1
[0.17.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.16.0...v0.17.0
[0.16.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.15.3...v0.16.0
[0.15.3]: https://github.com/SinanTufekci/agent-intern/compare/v0.15.2...v0.15.3
[0.15.2]: https://github.com/SinanTufekci/agent-intern/compare/v0.15.1...v0.15.2
[0.15.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.15.0...v0.15.1
[0.15.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.14.1...v0.15.0
[0.14.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.14.0...v0.14.1
[0.14.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.13.0...v0.14.0
[0.13.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.12.1...v0.13.0
[0.12.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.12.0...v0.12.1
[0.12.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.11.0...v0.12.0
[0.11.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.10.4...v0.11.0
[0.10.4]: https://github.com/SinanTufekci/agent-intern/compare/v0.10.3...v0.10.4
[0.10.3]: https://github.com/SinanTufekci/agent-intern/compare/v0.10.2...v0.10.3
[0.10.2]: https://github.com/SinanTufekci/agent-intern/compare/v0.10.1...v0.10.2
[0.10.1]: https://github.com/SinanTufekci/agent-intern/compare/v0.10.0...v0.10.1
[0.10.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.9.0...v0.10.0
[0.9.0]: https://github.com/SinanTufekci/agent-intern/compare/v0.8.0...v0.9.0
[0.8.0]: https://github.com/SinanTufekci/agent-intern/releases/tag/v0.8.0
