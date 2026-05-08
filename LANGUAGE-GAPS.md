# Corvid language gaps — detailed report

Tested against [Micrurus-Ai/Corvid-lang](https://github.com/Micrurus-Ai/Corvid-lang) at commit `259dd59` (corvid-cli 0.0.1 / language v0.1.0). Discovered while building a non-trivial sample app (a ticket-triage agent exercising effects, dangerous tools, approve boundaries, prompts, budgets, and stdlib imports). All eight gaps were re-verified with isolated test cases on the same toolchain after the initial write-up; this version reflects the corrected findings.

Scope: this file documents only **language / compiler / CLI gaps** — issues that require an upstream change to Corvid-lang itself. Install-pipeline and release-engineering issues are tracked separately in [LIVE-TEST-GAPS.md](./LIVE-TEST-GAPS.md).

## Verification status (2026-05-06 re-test)

| # | Gap | Verified | Notes after re-test |
| --- | --- | --- | --- |
| L-1 | `corvid check` doesn't resolve imports | ✓ | Reproduced with a bogus import: `check` exits 0 saying "ok"; `build` emits 2 errors on the same file. |
| L-2 | Python codegen drops struct types | ✓ | Confirmed with isolated nested-struct test. Sibling primitive lowered fine; nested struct → `object`. |
| L-3 | Native rejects struct returns | ✓ | **Broader than originally claimed** — affects both prompt returns *and* entry-agent returns at the command-line boundary. Two error sites, one underlying gap. |
| L-4 | WASM rejects String params | ✓ | Same code with Int param compiles to wasm+js+d.ts cleanly. |
| L-5 | `corvid run` auto picks native unsafely | ✓ | **Reframed.** Auto correctly picks native per its documented signature-based rule. The actual gap: when native fails for *environmental* reasons (missing runtime staticlib), auto does **not** fall back to interpreter even though the interpreter target produces correct output. |
| L-6 | ANSI color leaks to non-TTY | ✓ | **Worse than originally claimed.** I said `NO_COLOR=1` cleans it up — it does not. The renderer ignores `NO_COLOR` entirely and emits ariadne's `Color::Red` unconditionally. Both env-var honoring and TTY detection are missing. |
| L-7 | No `\` line continuation in strings | ✓ | Reproduced. |
| L-8 | `approve` must match tool name | ✓ | **Less restrictive than originally claimed.** I said "must be PascalCase of tool name". Reality: both `approve send_to_pagerduty(...)` (snake) and `approve SendToPagerduty(...)` (Pascal) are accepted. Only mismatched names are rejected. |

## Upstream fix status (re-tested 2026-05-06 against `Corvid-lang@ae8e3dd`)

After this report was published, the maintainer landed phases 20l (first-impression gap repair) and 20m (verifier-driven corrections). I re-pulled, rebuilt the corvid binary, and re-ran every isolated repro from this file. Five of eight gaps are now closed.

| # | Gap | Status | Upstream commit | Re-test result |
| --- | --- | --- | --- | --- |
| L-1 | `corvid check` doesn't resolve imports | **FIXED** | (in 20l-A — also lands `crates/corvid-cli/tests/check_validates_imports.rs`) | `check` on a bogus import now exits 1 with three E0000 diagnostics including "could not be found" |
| L-2 | Python codegen drops struct types | **FIXED** | `11230d4 fix(codegen-py): preserve struct, list, and option types in Python annotations` | Nested `inner: Inner` now correctly typed (was `inner: object`) |
| L-3 | Native rejects struct returns | open (roadmap) | — | Same error verbatim; honest "not yet implemented" |
| L-4 | WASM rejects String params | open (roadmap) | — | Same error verbatim; honest "not yet implemented" |
| L-5 | `corvid run` no fallback when native fails | **FIXED** | `3fb577e fix(driver): auto-fall-back to interpreter when native staticlib missing` | `corvid run arith.cor` now emits `↻ running via interpreter: native staticlib unavailable`, prints `47`, exits 0 |
| L-6 | ANSI color leaks to non-TTY | **FIXED** | `c822dd5 fix(driver): suppress ANSI escapes in diagnostics when stderr is not a TTY` | Non-TTY output now has 0 ESC sequences (was 27) |
| L-7 | No `\` line continuation in strings | open (design) | — | Still rejected; design choice flagged as such in the original report |
| L-8 | `approve` naming undocumented | **DOCS UPDATED** | `68f8dca` then `e1b1728` — first names the PascalCase rule, then corrects the spec to accept both forms | Both `approve send_to_pagerduty(...)` and `approve SendToPagerduty(...)` accepted; mismatched name rejected |

**Score:** 5 of 8 fixed in patch-sized commits; 2 remain as roadmap items with honest "not yet implemented" errors; 1 is a deliberate design choice. The two cheapest, highest-impact gaps from the priority table at the bottom of this file (L-1 and L-2) shipped first, exactly as suggested.

The upstream regression test for L-1 (`crates/corvid-cli/tests/check_validates_imports.rs`) follows the shape sketched in this report and credits the case as "the reporter's exact case."

The remaining open gaps (L-3, L-4, L-7) were correctly tagged as roadmap or design-decision items in the original report; none of them are blocking everyday Corvid use today.

## Severity legend

- **Critical:** breaks the first-time user experience or quietly emits incorrect feedback. Must fix before next release.
- **High:** real correctness bug or major usability regression; not blocking but needs a near-term fix.
- **Medium:** known incomplete codegen path with honest "not yet implemented" error. Roadmap territory.
- **Low:** polish, ergonomics, documentation gaps.

## Summary

| # | Title | Severity | Component | Lines to fix |
| --- | --- | --- | --- | --- |
| L-1 | `corvid check` doesn't resolve imports — false-OK on broken code | Critical | `corvid-cli` | 1 |
| L-2 | Python codegen drops nested struct types as `object` | High | `corvid-codegen-py` | ~10 |
| L-3 | Native codegen rejects struct returns from prompts | Medium | `corvid-codegen-cl` | unbounded (real feature) |
| L-4 | WASM target rejects `String` parameters | Medium | `corvid-codegen-wasm` | unbounded (real feature) |
| L-5 | `corvid run` auto-dispatch picks native when interpreter would suffice | Low | `corvid-cli` | ~20 |
| L-6 | ANSI color escapes leak into non-TTY stderr | Low | `corvid-driver` | ~5 |
| L-7 | String literals don't support `\` line continuation | Low | `corvid-syntax` | ~10 |
| L-8 | `approve` identifier must match tool name (undocumented) | Low | docs only | docs |

---

## L-1 — `corvid check` doesn't resolve imports

**Severity:** Critical
**Component:** `crates/corvid-cli` (the check command itself, not the resolver)
**Status:** active bug

### Reproduction

`src/main.cor`:

```corvid
import "./std/effects" use EffectEnvelope, effect_envelope

agent main() -> Bool:
    return true
```

…where `std/effects` either does not exist on disk or doesn't export `EffectEnvelope` / `effect_envelope`.

```sh
corvid check src/main.cor
# ok: src/main.cor — no errors        <-- LIES

corvid build src/main.cor
# error: import './std/effects' from '<path>/src/main.cor' could not be found
# error: the module imported as './std/effects' has no declaration named 'EffectEnvelope'
# error: the module imported as './std/effects' has no declaration named 'effect_envelope'
# 3 error(s) found.
```

### Why it's critical

`corvid check` is the obvious feedback loop for editor / LSP / pre-commit integrations. If it returns "ok" on code that won't build, every downstream consumer reports false positives. The user only learns the build is broken when they try to ship — exactly the wrong moment.

### Root cause

The driver crate already has the correct entry point. There are two functions:

```rust
// crates/corvid-driver/src/pipeline/compile.rs:36
pub fn compile(source: &str) -> CompileResult { … }

// crates/corvid-driver/src/pipeline/compile.rs:44
//
// "Run the full frontend on `source`. Stops collecting output when
//  errors before codegen would make it misleading."
//
pub fn compile_with_config(source: &str, config: Option<&CorvidConfig>) -> CompileResult { … }

// crates/corvid-driver/src/pipeline/compile.rs:91
//
// "Compile a source string that came from `source_path`. Unlike
//  `compile_with_config`, THIS PATH CAN RESOLVE SIBLING `.cor`
//  IMPORTS because the driver still has a filesystem anchor."
//
pub fn compile_with_config_at_path(
    source: &str,
    source_path: &Path,
    config: Option<&CorvidConfig>,
) -> CompileResult { … }
```

The driver's own doc comment explicitly says the path-less variant can't resolve imports. But `cmd_check` calls the path-less variant despite having a path:

```rust
// crates/corvid-cli/src/commands/misc.rs:41-45
pub(crate) fn cmd_check(file: &Path) -> Result<u8> {
    let source = std::fs::read_to_string(file)
        .with_context(|| format!("cannot read `{}`", file.display()))?;
    let config = load_corvid_config_for(file);
    let result = compile_with_config(&source, config.as_ref());   // <-- bug
    …
}
```

It already has `file: &Path`. It just isn't passing it.

### Fix

```diff
--- a/crates/corvid-cli/src/commands/misc.rs
+++ b/crates/corvid-cli/src/commands/misc.rs
@@ -15,7 +15,7 @@ use anyhow::{Context, Result};
 use corvid_driver::{
-    compile_with_config, diff_snapshots, load_corvid_config_for, render_all_pretty,
+    compile_with_config_at_path, diff_snapshots, load_corvid_config_for, render_all_pretty,
     render_effect_diff, scaffold_new, snapshot_revision, vendor_std,
 };
@@ -42,7 +42,7 @@ pub(crate) fn cmd_check(file: &Path) -> Result<u8> {
     let source = std::fs::read_to_string(file)
         .with_context(|| format!("cannot read `{}`", file.display()))?;
     let config = load_corvid_config_for(file);
-    let result = compile_with_config(&source, config.as_ref());
+    let result = compile_with_config_at_path(&source, file, config.as_ref());
     if result.ok() {
         println!("ok: {} — no errors", file.display());
```

Three lines changed. No new logic. Both functions return the same `CompileResult` type.

### Regression test to add

`crates/corvid-cli/tests/check_validates_imports.rs`:

```rust
use std::process::Command;
use tempfile::tempdir;

#[test]
fn check_rejects_missing_import() {
    let tmp = tempdir().unwrap();
    let main = tmp.path().join("main.cor");
    std::fs::write(&main, r#"
        import "./does_not_exist" use Anything
        agent main() -> Bool: return true
    "#).unwrap();

    let out = Command::new(env!("CARGO_BIN_EXE_corvid"))
        .args(["check"]).arg(&main).output().unwrap();
    assert!(!out.status.success(), "check should reject missing import");
    let err = String::from_utf8(out.stderr).unwrap() + &String::from_utf8(out.stdout).unwrap();
    assert!(err.contains("could not be found"), "expected import-not-found diagnostic, got: {err}");
}
```

---

## L-2 — Python codegen drops nested struct types as `object`

**Severity:** High
**Component:** `crates/corvid-codegen-py`
**Status:** active bug

### Reproduction

```corvid
type Severity:
    label: String
    confidence: Float

type Triage:
    severity: Severity        # <-- nested record
    summary: String
```

`corvid build src/main.cor` emits `target/py/main.py`:

```python
@dataclass
class Severity:
    label: str
    confidence: float

@dataclass
class Triage:
    severity: object          # <-- should be: Severity (or "Severity")
    summary: str
```

Primitive fields lower correctly (`String → str`, `Float → float`). Nested records — and several other typed fields like `List[T]`, `Stream[T]`, `Option[T]` — collapse to `object`.

### Why this matters

The Python target is the documented integration story for users embedding Corvid in existing Python codebases. `severity: object` defeats every IDE autocomplete, every `mypy` / `pyright` check, and every static refactor downstream. The runtime behavior is fine; the type fidelity is gone.

### Root cause

`crates/corvid-codegen-py/src/codegen.rs:495-519`:

```rust
fn python_type_hint_of(ty: &corvid_types::Type) -> String {
    use corvid_types::Type as T;
    match ty {
        T::Int => "int".into(),
        T::Float => "float".into(),
        T::String => "str".into(),
        T::Bool => "bool".into(),
        T::Nothing => "None".into(),
        T::Struct(_)
        | T::ImportedStruct(_)
        | T::Function { .. }
        | T::List(_)
        | T::Stream(_)
        | T::Partial(_)
        | T::ResumeToken(_)
        | T::RouteParams(_)
        | T::Unknown => "object".into(),                  // <-- the gap
        T::Result(_, _) | T::Option(_) | T::Weak(_, _) => "object".into(),
        T::Grounded(inner) => python_type_hint_of(inner),
        T::TraceId => "str".into(),
        …
```

The comment two lines below acknowledges it: *"Emitting 'object' here is a safe approximation until the Python backend decides on its representation."* It's an acknowledged TODO, not a hidden bug.

### Fix

Extract the struct name from `T::Struct` and emit as a string forward-reference (PEP 563 / `from __future__ import annotations`, which the generated code already enables at line 2):

```diff
-        T::Struct(_)
-        | T::ImportedStruct(_)
-        | T::Function { .. }
+        T::Struct(name) => format!("\"{}\"", name).into(),
+        T::ImportedStruct(qname) => format!("\"{}\"", qname.local_name()).into(),
+        T::Function { .. }
         | T::List(_)
         | T::Stream(_)
         | T::Partial(_)
         | T::ResumeToken(_)
         | T::RouteParams(_)
         | T::Unknown => "object".into(),
```

`List(inner)` should become `list[<python_type_hint_of(inner)>]`. `Option(inner)` should become `<python_type_hint_of(inner)> | None`. Both are mechanical extensions of the same pattern.

### Regression test to add

`crates/corvid-codegen-py/tests/struct_field_types.rs`:

```rust
#[test]
fn nested_struct_field_emits_class_name() {
    let py = corvid_codegen_py::compile_to_python(r#"
        type Inner:
            x: Int
        type Outer:
            inner: Inner
        agent main() -> Outer:
            return Outer(Inner(42))
    "#).unwrap();
    assert!(py.contains("inner: \"Inner\""), "expected forward-ref, got: {py}");
}
```

---

## L-3 — Native codegen rejects struct returns

**Severity:** Medium
**Component:** `crates/corvid-codegen-cl`
**Status:** known incomplete (good error messages, two distinct sites)

### Reproduction — site 1: prompt returning a struct

```corvid
type Triage:
    severity: String
    summary: String

prompt classify(ticket: String) -> Triage:
    """ … """

agent main(t: String) -> Triage:
    return classify(t)
```

```sh
corvid build src/main.cor --target=native
# error: native codegen does not yet support: prompt 'classify'
# returns 'struct' — the native prompt bridge currently supports
# only Int / Bool / Float / String returns; structured prompt
# returns are not implemented yet
```

### Reproduction — site 2: entry agent returning a struct

```corvid
type Decision:
    label: String
    score: Float

prompt classify(text: String) -> Decision:
    """ … """

agent main(input: String) -> Decision:
    return classify(input)
```

```sh
corvid build src/main.cor --target=native
# error: native codegen does not yet support: entry agent 'main'
# returns 'struct' — the native command-line boundary currently
# supports only Int/Bool/Float/String returns; structured output
# needs a dedicated serialization layer
```

### Verdict

Two distinct error paths in native codegen, both honest "not yet implemented" messages. The underlying limitation is the same: the native target lacks a struct serialization layer for the prompt-bridge boundary AND for the command-line entry boundary. Python and interpreter targets get this for free because of dynamic typing.

The original write-up of this gap covered only site 1 (prompt returns). Re-testing surfaced site 2 (entry-agent returns); they share fix scope but live in different lowering paths.

### Where it lives

`crates/corvid-codegen-cl/src/lowering/prompt.rs` (prompt bridge) and the entry-agent lowering path in the same crate. Search the codegen-cl crate for the literal error strings to find both.

### Suggested approach

Extend the native runtime to allocate structs on the runtime heap, deserialize from JSON via the existing `corvid-runtime` deserializer, and return pointers. Same shape as the existing `Grounded<T>` handling for primitives. Both sites can share the implementation once it exists.

### Workaround for users today

Return a flat tuple of primitives from the prompt and re-pack into the struct in the calling agent. Or use `--target=python` until the native path catches up.

---

## L-4 — WASM target rejects `String` parameters

**Severity:** Medium
**Component:** `crates/corvid-codegen-wasm`
**Status:** known incomplete (good error message)

### Reproduction

```corvid
agent greet(name: String) -> String:
    return name
```

```sh
corvid build src/main.cor --target=wasm
# error: failed to build 'src/main.cor' (wasm): wasm codegen failed:
# wasm target currently supports only Int, Float, Bool, and Nothing
# scalar parameters; agent 'auto_respond' parameter 'ticket' has 'String'
```

### Verdict

WASM ABI doesn't have a native string type — strings cross the boundary as `(ptr, len)` pairs with allocator coordination. Implementing this requires picking a string ABI (UTF-8 with explicit length is common; WASM Component Model has a richer story) and threading it through codegen + the JS loader. Real engineering work, well-flagged as not-yet-shipped.

### Where it lives

`crates/corvid-codegen-wasm/src/lib.rs`. Search for the literal error string.

### Workaround for users today

Encode strings as `Int` indices into a side table (impractical) or stick to scalar parameters in WASM-targeted agents until the String ABI lands. Note that for the common "browser embedding" use case, JS callers can already drive a Python-targeted runtime over Pyodide; WASM is one of three deployment paths, not the only one.

---

## L-5 — `corvid run` auto-dispatch has no fallback when native build fails

**Severity:** Low
**Component:** `crates/corvid-cli`
**Status:** confirmed; original framing corrected

### Reproduction

```corvid
# arith.cor — no tools, no prompts, pure compute
agent compute(x: Int, y: Int) -> Int:
    return x * y + 5

agent main() -> Int:
    return compute(6, 7)
```

```sh
corvid run src/arith.cor                  # exits 2
# error: failed to run 'src/arith.cor': native codegen failed for 'arith':
# linker error: corvid-runtime staticlib missing at
# '<exe-dir>/corvid_runtime.lib' …

corvid run src/arith.cor --target=interpreter   # exits 0
# 47

corvid run src/arith.cor --target=native        # exits 2 (same staticlib error)
```

The same program runs cleanly under the interpreter; auto refuses to fall back.

### Corrected framing

The original write-up claimed auto picks native "when interpreter would suffice." That's not quite right — the auto heuristic is doing exactly what its docs describe: an `Int → Int → Int` agent stays within the native command-line boundary, so auto picks native. Per spec.

The actual gap: when native build fails for *environmental* reasons (missing runtime staticlib, missing toolchain, link error, etc.), auto exits with the failure instead of falling back to the interpreter. The interpreter would have run the program correctly, but auto never tries it.

This matters because the prebuilt release archive doesn't ship the runtime staticlib (LIVE-TEST-GAPS issue #4). End users who installed via the prebuilt path will hit this every time on tool-free programs — which is the simplest possible first program.

### Where it lives

`crates/corvid-cli/src/run_cmd.rs` — the `cmd_run` function and `RunTarget::Auto` dispatch. Look for where native build failures propagate up: they likely don't catch the link error and retry with the interpreter.

### Suggested fix

Two viable approaches:

1. **Eager probe.** Before committing to native, check whether `corvid_runtime.{lib,a}` is locatable. If not, transparently use the interpreter. Pro: no double-work on failure. Con: probe logic has to mirror the linker's actual search rules.
2. **Lazy fallback.** Try native; on any link or codegen error, retry with the interpreter and emit `↻ falling back to interpreter: <reason>` to stderr (mirroring the existing `↻ running via interpreter` notice for tool-using code). Pro: no probe drift. Con: pays one failed compile before falling back.

Approach 2 mirrors the pattern the dispatcher already uses for tool-using code without `--with-tools-lib`, so it's consistent with existing UX.

### Note on related issues

Once LIVE-TEST-GAPS issue #4 (runtime staticlib in release archive) is fixed, this issue becomes much less visible — most users will never hit the failing native path. But the fallback gap is still a robustness improvement worth shipping.

---

## L-6 — ANSI color escapes emitted unconditionally

**Severity:** Low
**Component:** `crates/corvid-driver` (diagnostic renderer)
**Status:** confirmed; original framing corrected

### Reproduction

PowerShell on Windows (which doesn't render ANSI by default in classic conhost):

```powershell
corvid check src/main.cor 2>&1
# corvid.exe : [31m[E0003] error:[0m unexpected character `\`
#  [38;5;246m╭[0m[38;5;246m─[0m[38;5;246m[[0msrc/main.cor:77:67[38;5;246m][0m
#  …
```

Or in bash, redirecting to a file (definitely not a TTY):

```sh
corvid build src/bogus.cor > /tmp/out.txt 2>&1
grep -c $'\x1b\[' /tmp/out.txt
# 27        ← 27 ANSI escape sequences in non-TTY output
```

### Corrected framing

The original write-up claimed *"setting `$env:NO_COLOR=1` cleans it up — so the renderer respects `NO_COLOR`, but doesn't auto-detect."* That was wrong. Re-testing with `NO_COLOR=1` (both as a per-command prefix and via `export`) produces identical output: 27 escape sequences either way. The renderer ignores `NO_COLOR` entirely. **Both env-var honoring and TTY detection are missing**, not just one.

The PowerShell behavior I previously attributed to NO_COLOR was probably an artifact of how `Out-String -Stream | ForEach-Object { $_ -replace '\[…' }` was filtering my own captured output, not the renderer responding to env state.

### Root cause

The diagnostic renderer uses [ariadne](https://crates.io/crates/ariadne) (workspace dep, version `0.4`). At `crates/corvid-driver/src/render.rs:28`:

```rust
.with_color(Color::Red)
```

There's no `Config::with_color(false)` toggle anywhere in the file, no env-var check, no `IsTerminal` probe. The `Color::Red` is hard-wired and ariadne emits the corresponding escapes whenever it serializes the report.

### Fix

Build an ariadne `Config` once, with color disabled when either `NO_COLOR` is set per the [no-color.org spec](https://no-color.org) OR when stderr is not a TTY:

```diff
+use std::io::IsTerminal;
+
 pub fn render_all_pretty(diags: &[Diagnostic], file: &Path, source: &str) -> String {
+    let with_color = std::env::var_os("NO_COLOR").is_none()
+        && std::io::stderr().is_terminal();
+    let cfg = ariadne::Config::default().with_color(with_color);
     …
-    Report::build(…)
+    Report::build(…).with_config(cfg)
         …
```

`std::io::IsTerminal` is in std since Rust 1.70 — no new dependency. ariadne's `Config::with_color(bool)` is the standard knob.

### Regression test

Snapshot test: invoke `cargo run --bin corvid -- check` with stderr redirected to a captured pipe, assert the captured output contains no `\x1b[` escape sequences.

---

## L-7 — String literals don't support `\` line continuation

**Severity:** Low
**Component:** `crates/corvid-syntax` (lexer)
**Status:** language design choice

### Reproduction

```corvid
agent main() -> String:
    return "first part " \
           "second part"
```

```sh
corvid check src/main.cor
# [E0003] error: unexpected character `\`
```

### Discussion

Python supports `\` at end-of-line to continue a logical line. Corvid doesn't — `\` is treated as an unrecognized character. This is a design choice, not a bug per se, but it surprises Pythonistas (the most likely first-time Corvid users given the AI-agent positioning).

Triple-quoted `"""…"""` works for multi-line strings, and `+` works for concatenation, so there's always a workaround — but they cost an extra newline or paren level.

### Workaround for users today

Use `+`:

```corvid
return "first part " + "second part"
```

Or triple-quote:

```corvid
return """first part second part"""
```

### Suggested fix (optional)

Extend the lexer to consume a `\` followed by `\n` and skip it. ~10 lines in the string-literal lexing path (probably in `crates/corvid-syntax/src/lexer/`). Worth doing if Python familiarity is a deliberate language goal; safe to defer otherwise.

---

## L-8 — `approve` identifier rule is undocumented (and more permissive than expected)

**Severity:** Low (docs)
**Component:** docs only
**Status:** confirmed; original claim corrected

### Reproduction — three variants

```corvid
tool send_to_pagerduty(s: String, b: String) -> Nothing dangerous

# Variant A: mismatched name → REJECTED
agent a(s: String, b: String) -> Nothing:
    approve PageOnCall(s, b)
    send_to_pagerduty(s, b)
# [E0101] error: dangerous tool 'send_to_pagerduty' called without a prior 'approve'
# Help: add `approve SendToPagerduty(arg1, arg2)` on the line before this call

# Variant B: matching PascalCase → ACCEPTED
agent b(s: String, b: String) -> Nothing:
    approve SendToPagerduty(s, b)
    send_to_pagerduty(s, b)
# ok: src/rightname.cor — no errors

# Variant C: matching snake_case → ALSO ACCEPTED
agent c(s: String, b: String) -> Nothing:
    approve send_to_pagerduty(s, b)
    send_to_pagerduty(s, b)
# ok: src/lower.cor — no errors
```

### Corrected framing

The original write-up claimed *"the identifier following `approve` must be the PascalCase form of the dangerous tool's snake_case name."* That's too strict. Re-testing surfaced that the compiler accepts **either** the PascalCase form (`SendToPagerduty`) **or** the snake_case form matching the tool name verbatim (`send_to_pagerduty`).

Only mismatched names (anything that doesn't normalize to the tool name) are rejected. The compiler's help message *suggests* PascalCase, which is probably the recommended convention, but it's not enforced.

### Verdict

The rule is real — pinning approve identifiers to the tool name lets reviewers grep approval sites per-tool, which is a real security property. But:

- It's not documented anywhere I can find.
- It's more permissive than the help message suggests, which can confuse a careful reader.
- New users learn it from compiler errors, not from the spec.

### Fix

Add a section to `docs/effects-spec/03-typing-rules.md` (or wherever approve gates are formally specified) titled "approve identifier naming" that says:

> The identifier following `approve` must match the corresponding `dangerous` tool's name in one of two normalized forms: the tool's exact `snake_case` name, or its `PascalCase` equivalent. The compiler rejects any other identifier with diagnostic E0101. The PascalCase form is recommended convention because it's visually distinct from regular function calls in approval-review settings, but both forms are accepted.

Also extend the `corvid tour --topic approve-gates` blurb to mention the rule once.

### Possible follow-on tightening

If the maintainers decide that *only* one form should be accepted (probably PascalCase, given the help-message convention), that's a backwards-incompatible language change — current code that uses `approve <snake_case>` would break. Worth a brief language-team discussion before changing.

---

## What's NOT in scope for this file

These were also discovered in the live test but are not language-side issues — they live in the install pipeline / release engineering and are tracked in [LIVE-TEST-GAPS.md](./LIVE-TEST-GAPS.md):

- The `vendor_std` path bug in `corvid new` (in our own scaffold patch — fixable with a one-line `scaffold.rs` change).
- Prebuilt Windows binary blocked by AppLocker / SmartScreen (release isn't code-signed).
- `corvid_runtime.lib` missing from the prebuilt release archive (`release.yml` doesn't ship it).

Together those three plus L-1 are the "first-impression failure" set — they're what new users hit before they can run a single `.cor` file. L-1 is the one in this file; the other three are install-side.

## Suggested rollout

The two truly cheap and high-impact fixes are:

1. **L-1 (3-line change in `cmd_check`)** — restores correctness for every editor / pre-commit / IDE integration.
2. **L-2 (Python codegen forward-refs)** — restores type fidelity for the documented Python integration target.

Both could land in a v0.1.1 patch release the same week. L-6 is also nearly free if you want to ship a third small fix.

L-3, L-4, L-5 are larger codegen / dispatch work — file as roadmap issues and don't block on them.

L-7 and L-8 are nice-to-haves; pick them up when convenient.
