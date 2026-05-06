# Live test report — gaps in Corvid v0.1.0 / corvid-cli 0.0.1

Tested on: Windows 11 Pro 26200, PowerShell 5.1, AXON\SBW domain account.

## What I did

1. Installed Corvid via the v0.1.0 prebuilt archive: `irm .../install.ps1 | iex`.
2. Discovered the prebuilt `corvid.exe` won't run on this machine; built locally with `cargo build --release -p corvid-cli` instead.
3. Scaffolded a fresh project: `corvid new triage_bot`.
4. Wrote a non-trivial app exercising effects / dangerous tools / `approve` boundaries / `prompt`+LLM calls / `@budget` / `@trust` / agent-to-agent dispatch / stdlib import.
5. Ran `corvid check`, `corvid build` (Python / native / wasm), `corvid run`, `corvid doctor`, `corvid tour --list`, `corvid abi`, `corvid test/verify/trace/bundle/capsule/approver --help`.

The sample lives at `C:\Users\SBW\AppData\Local\Temp\corvid-playground-5404\triage_bot\src\main.cor` (97 lines, exercises every safety primitive in the tour catalog).

---

## Gaps, by severity

### CRITICAL

#### Gap #1 — `corvid new`'s vendored stdlib is in the wrong place

**Severity:** critical (every fresh project I scaffold breaks at first import).

**Repro:**

```sh
corvid new myproj
cd myproj
# add to src/main.cor:
#     import "./std/effects" use EffectEnvelope
corvid build src/main.cor
```

**Result:** `error: import './std/effects' from '<root>/src/main.cor' could not be found at '<root>/src/std/effects.cor'`.

**Why it breaks:** `corvid new` (after the patch I added) vendors `std/` into `<project_root>/std/`. But Corvid's import resolver is purely relative to the importing file. So `import "./std/effects"` from `src/main.cor` looks at `src/std/effects.cor`, which doesn't exist. To reach the vendored stdlib, the import has to be `import "../std/effects"`.

**Why I missed this in the patch:** I tested `vendor_std_from()` only as a unit test (copies `src` to `dst`); I never wrote an integration test that actually scaffolded a project and tried to import the vendored stdlib. That gap is now visible.

**Suggested fix:** vendor `std/` into `<project>/src/std/` instead of `<project>/std/`. Then `import "./std/foo"` from `src/main.cor` resolves naturally. This is a one-line change in `crates/corvid-driver/src/scaffold.rs`:

```diff
- let dst = project_root.join("std");
+ let dst = project_root.join("src").join("std");
```

The unit test `vendor_std_from_copies_directory_tree` still passes because it's path-agnostic. Add a new integration test that does the full scaffold + import + check round-trip.

#### Gap #2 — `corvid check` doesn't validate imports

**Severity:** critical (the check/build skew is misleading).

**Repro:** the broken-import code from Gap #1 above.

**Result:** `corvid check src/main.cor` returns `ok: src/main.cor — no errors` even though `corvid build` errors with three E0000 missing-import diagnostics on the same file.

**Why this matters:** `corvid check` is advertised as "type-check a Corvid source file" and is the obvious feedback loop in editors / IDEs / pre-commit. If it doesn't surface missing imports, every editor integration will say "looks good" while the actual build is broken. Worse, the user doesn't learn until they hit `corvid run` or `corvid build`.

**Suggested fix:** `cmd_check` should run module resolution before reporting clean. Either (a) run the full lower-and-resolve pipeline and stop before codegen, or (b) at minimum, attempt to read every imported `.cor` file and validate top-level names match the `use` clauses.

---

### HIGH

#### Gap #3 — Prebuilt `corvid.exe` is unsigned, blocked on locked-down Windows

**Severity:** high (silent install failure for all corporate Windows users).

**Repro:**

```powershell
irm https://raw.githubusercontent.com/Micrurus-Ai/Corvid-lang/main/install/install.ps1 | iex
corvid --version
```

**Result on AXON-domain Windows 11:** `Program 'corvid.exe' failed to run: Access is denied`. The PE is valid, the ACL allows execute, the file isn't quarantined by Mark-of-the-Web — but AppLocker / SmartScreen / Defender silently block unsigned executables originating from the internet. Locally-`cargo build`-produced binaries are exempt because they were never on the network.

**Why install.ps1 hides it:** the installer's final `corvid doctor` invocation is wrapped in `try { & $exe doctor } catch { Warn2 "doctor reported issues; see output above" }`. The catch swallows the access-denied error and prints a misleading "doctor reported issues" message, so the user thinks doctor surfaced *something*, not that the binary is fundamentally unrunnable.

**Suggested fix (two parts):**

1. Code-sign the Windows release archive in `release.yml` using a real publisher cert (DigiCert ~$300/yr, or Azure Code Signing). Without signing, every corporate user hits this. This is the actual blocker for adoption.
2. In the meantime, make `install.ps1` distinguish "doctor reported environmental issues" from "binary cannot execute". After download, run `& $exe --version` first; if that returns nonzero or throws, surface a clear error with workaround instructions ("your machine blocks unsigned downloaded binaries; build from source with `cargo install`").

#### Gap #4 — `corvid run` and `corvid build --target=native` need `corvid_runtime.lib` next to the binary

**Severity:** high (every prebuilt install will hit this for any program that lowers to native).

**Repro:** any program that's not LLM-bound; e.g. `agent main() -> Int: return 6 * 7`. Then `corvid run program.cor`.

**Result:** `error: failed to run 'program.cor': native codegen failed: linker error: corvid-runtime staticlib missing at '<exe-dir>/corvid_runtime.lib' and no release fallback was found. Build 'corvid-runtime' for the active profile or run 'cargo build -p corvid-runtime --release'.`

**Why:** the native codegen path links against `corvid_runtime.lib`/`.a`, which is built by `cargo build -p corvid-runtime`. The release matrix in `release.yml` only ships `bin/corvid(.exe)` — it doesn't ship the runtime staticlib. So the prebuilt is missing the link target.

**Suggested fix:** extend `release.yml`'s "Stage artifact" step to also copy `target/<target>/release/{libcorvid_runtime.a,corvid_runtime.lib}` (and the equivalent dylib if Linux uses `.so`) into the staged dir. Then update the loader logic in the CLI to look in `<exe-dir>/runtime/` or just `<exe-dir>/` rather than the cargo `target/` path. End users never set up cargo, so the cargo-target search has to be the *fallback*, not the default.

---

### MEDIUM

#### Gap #5 — Approve identifier must match tool name (E0101)

**Severity:** medium (compile-time error has a great help message; just worth documenting).

**Repro:**

```corvid
tool send_to_pagerduty(summary: String, body: String) -> Nothing dangerous
agent escalate(s: String, b: String) -> Nothing:
    approve PageOnCall(s, b)         # rejected
    send_to_pagerduty(s, b)
```

**Result:** `[E0101] dangerous tool 'send_to_pagerduty' called without a prior 'approve'`. Compiler suggests: "add `approve SendToPagerduty(arg1, arg2)` on the line before this call".

**Verdict:** this is by design — approve identifier mirrors the tool name in PascalCase so reviewers can grep. But the docs and tour examples should say so explicitly. Right now the tour topic `approve-gates` shows the approve form but doesn't state the naming rule. Document it.

#### Gap #6 — String literals don't support `\` line continuation

**Severity:** medium (just an unfamiliar lexer choice; affects readability of long strings).

**Repro:**

```corvid
ticket = "long string part 1 " \
         "long string part 2"
```

**Result:** `[E0003] unexpected character '\'`.

**Verdict:** Python supports this; Corvid doesn't. Triple-quoted `"""..."""` works for multi-line, and string literals can be broken with `+` concatenation. Just a lexer limit. Worth one sentence in the language docs.

#### Gap #7 — WASM target rejects `String` parameters

**Severity:** medium (limits what you can wasm-compile; well-communicated).

**Repro:** `corvid build my_program.cor --target=wasm` where any agent or tool takes a `String` parameter.

**Result:** `error: wasm target currently supports only Int, Float, Bool, and Nothing scalar parameters; agent 'auto_respond' parameter 'ticket' has 'String'`.

**Verdict:** honest "not yet implemented" message. Implementing String marshalling across the wasm boundary is non-trivial (bytes + length pair, allocator coordination, encoding). Roadmap item, not a bug.

#### Gap #8 — Native target rejects struct returns from prompts

**Severity:** medium (affects any prompt that returns a typed record; same root cause as #7).

**Repro:** my triage_bot has `prompt classify(...) -> Triage` where `Triage` is a record.

**Result:** `error: native codegen failed: native codegen does not yet support: prompt 'classify' returns 'struct' — the native prompt bridge currently supports only Int / Bool / Float / String returns; structured prompt returns are not implemented yet`.

**Verdict:** another honest roadmap item. Documented, message tells you exactly what's missing.

#### Gap #9 — Generated Python loses type fidelity on nested records

**Severity:** medium (works at runtime, but type hints are weaker than they should be).

**Repro:** see `target/py/main.py` line 33-36 in the test playground:

```python
@dataclass
class Triage:
    severity: object        # <-- should be: Severity
    summary: str
    suggested_reply: str
```

**Result:** the field declared as `severity: Severity` in `.cor` is lowered to `severity: object` in Python. Other primitive fields (`summary: String → str`, `confidence: Float → float`) are correctly typed. The nested record reference is dropped.

**Suggested fix:** Python codegen needs to emit `from __future__ import annotations` (already does) and use the bare class name as a string-resolved forward reference. Trivial fix in the codegen.

---

### LOW

#### Gap #10 — `corvid run` doesn't always fall back to interpreter for tool-free programs

**Severity:** low (an inconsistency, not a blocker).

**Repro:** `corvid run` on a program with `prompt` calls falls back to interpreter with a `↻ running via interpreter` notice. `corvid run` on a pure-arithmetic program tries native codegen and fails because of the runtime-staticlib issue (Gap #4).

**Verdict:** the dispatcher claims to fall back to interpreter "when the program stays within the current native command-line boundary." A pure-arithmetic program clearly stays within that boundary, so it shouldn't be triggering native codegen at all. Either the boundary check is wrong, or the fallback decision is.

#### Gap #11 — Lots of stderr ANSI escape codes

**Severity:** low (cosmetic).

**Repro:** any error output without `NO_COLOR=1` set, run from PowerShell.

**Result:** PowerShell doesn't render ANSI escapes, so error messages come back as `[31m[E0003] error:[0m unexpected character '\'` with explicit escape codes interleaved into the diagnostic. Setting `$env:NO_COLOR=1` cleans it up.

**Suggested fix:** detect the `WT_SESSION` environment variable (Windows Terminal sets it; classic console doesn't) and disable color when not on a terminal that supports it. `is-terminal` crate has the right TTY detection.

---

## What works really well

For balance, the things that genuinely impressed me:

- **Diagnostic quality.** Every error message I hit had a `Help:` line with a concrete suggestion, source span with column markers, and a sane error code (E0000–E0101 range). Better than most languages of this maturity.
- **`corvid doctor` output is structured and non-judgmental.** It distinguishes `OK` (toolchain present) from `..` (optional / not configured). No alarming red anywhere unless something is actually wrong.
- **`corvid tour --list` discovers 18 invention topics.** The runnable demos give a real sense of the language's ambition.
- **`corvid build --target=python` Just Works.** Generated `main.py` is clean, idiomatic async Python with `@dataclass`, registered via a runtime SDK. Genuinely usable as the integration story for Python codebases.
- **`corvid build --target=wasm` (for Int/Float/Bool params) emits four artifacts:** the `.wasm`, a JS loader, TypeScript `.d.ts` types, and a manifest JSON. Browser-deployment story is real, not aspirational.
- **The compile-time approval check (E0101) actually works.** I removed the `approve` line and the build correctly refused. That's the central language pitch — and it ships.

---

## Suggested fix priority

| Order | Gap | Effort | Impact |
| --- | --- | --- | --- |
| 1 | #1 (vendor `std/` into `src/`) | 1-line change | Unblocks every fresh project |
| 2 | #2 (`check` validates imports) | small refactor | Editor / pre-commit feedback finally accurate |
| 3 | #4 (ship runtime staticlib in release archive) | release.yml change | Makes `corvid run` work for prebuilt installs |
| 4 | #3 (Windows code signing + better install error) | $300 + script tweak | Unblocks corporate Windows |
| 5 | #11 (TTY-aware color) | ~5 lines | PowerShell users see clean errors |
| 6 | #9 (Python codegen forward-refs) | small codegen fix | Better IDE support |
| 7 | #5 (document approve-name rule) | docs only | Onboarding clarity |
| 8 | #6 (string `\` continuation) | optional lexer addition | Familiarity |
| 9 | #7, #8, #10 (codegen completeness) | larger work | Roadmap items, well-flagged today |

Items 1–2 are the kind of gap that breaks first-impression installs. Worth a follow-up commit before the next release; both would fit comfortably into a v0.1.1.
