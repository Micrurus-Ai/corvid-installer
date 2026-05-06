# Corvid language gaps — detailed report

Tested against [Micrurus-Ai/Corvid-lang](https://github.com/Micrurus-Ai/Corvid-lang) at commit `259dd59` (corvid-cli 0.0.1 / language v0.1.0). Discovered while building a non-trivial sample app (a ticket-triage agent exercising effects, dangerous tools, approve boundaries, prompts, budgets, and stdlib imports).

Scope: this file documents only **language / compiler / CLI gaps** — issues that require an upstream change to Corvid-lang itself. Install-pipeline and release-engineering issues are tracked separately in [LIVE-TEST-GAPS.md](./LIVE-TEST-GAPS.md).

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

## L-3 — Native codegen rejects struct returns from prompts

**Severity:** Medium
**Component:** `crates/corvid-codegen-cl`
**Status:** known incomplete (good error message)

### Reproduction

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
# error: failed to build 'src/main.cor' (native): native codegen failed:
# [1679..1695] native codegen does not yet support: prompt 'classify'
# returns 'struct' — the native prompt bridge currently supports only
# Int / Bool / Float / String returns; structured prompt returns are
# not implemented yet
```

### Verdict

Honest "not yet implemented" message, with a useful source span (`[1679..1695]` indexes into the source). The native prompt bridge needs to learn how to marshal struct returns from the LLM into Cranelift-codegen-friendly memory layouts. Non-trivial — the Python and interpreter targets get this for free because of dynamic typing.

### Where it lives

`crates/corvid-codegen-cl/src/lowering/prompt.rs`. The error is emitted from there based on the prompt's return type kind.

### Suggested approach

Extend the native prompt bridge to allocate a struct on the runtime heap, deserialize the LLM response into it via the existing JSON deserializer in `corvid-runtime`, and return a pointer. Roughly the same shape as how `corvid-runtime` already handles `Grounded<T>` for primitives.

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

## L-5 — `corvid run` auto-dispatch picks native when interpreter would suffice

**Severity:** Low
**Component:** `crates/corvid-cli`
**Status:** logic bug in target heuristic

### Reproduction

```corvid
# arith.cor — no tools, no prompts, pure compute
agent compute(x: Int, y: Int) -> Int:
    return x * y + 5

agent main() -> Int:
    return compute(6, 7)
```

```sh
corvid run src/arith.cor
# error: failed to run 'src/arith.cor': native codegen failed for 'arith':
# linker error: corvid-runtime staticlib missing at
# '<exe-dir>/corvid_runtime.lib' and no release fallback was found.
```

Compare to `corvid run` on a program that uses `prompt`, which correctly emits `↻ running via interpreter` and proceeds.

### Why it's a bug

`run_cmd.rs`'s module-level doc says auto mode picks "the native AOT tier when the program stays within the current native command-line boundary; falls back to the interpreter otherwise." A pure-arithmetic program with `Int → Int → Int` signatures *clearly* stays within the boundary — Int is the simplest scalar. Yet auto mode sends it to native, which then fails on the missing staticlib (and would fail anyway for users without cargo set up).

The auto heuristic seems to be inverted: it picks native when the program *uses tools*, then falls back to interpreter. It should pick interpreter for programs that don't need native specifically.

### Where it lives

`crates/corvid-cli/src/run_cmd.rs` — the `cmd_run` function and the `RunTarget::Auto` resolution path.

### Suggested fix

Re-read the eligibility check. Today's logic appears to be "use native unless tools are present without a tools-lib" — should instead be "use interpreter unless the program explicitly needs native (compiled libraries, FFI, etc.)". For most agents the interpreter is the correct default — it's the lowest-friction path and produces identical observable behavior.

### Note

Once L-5 is fixed, `corvid run` on tool-free programs will Just Work without users needing to know about the native runtime staticlib. That removes the second-most-common first-impression failure (after L-1).

---

## L-6 — ANSI color escapes leak into non-TTY stderr

**Severity:** Low
**Component:** `crates/corvid-driver` (diagnostic renderer)
**Status:** TTY detection missing

### Reproduction

PowerShell on Windows (which doesn't render ANSI by default in classic conhost):

```powershell
corvid check src/main.cor 2>&1
# corvid.exe : [31m[E0003] error:[0m unexpected character `\`
#  [38;5;246m╭[0m[38;5;246m─[0m[38;5;246m[[0msrc/main.cor:77:67[38;5;246m][0m
#  …
```

The diagnostic is unreadable in PowerShell because escape codes aren't being interpreted. Setting `$env:NO_COLOR=1` cleans it up — so the renderer respects `NO_COLOR`, but doesn't auto-detect.

### Where it lives

`crates/corvid-driver/src/render.rs` — diagnostic rendering. Currently emits color unconditionally when `NO_COLOR` is unset.

### Fix

Use the [`is-terminal`](https://crates.io/crates/is-terminal) crate (or `std::io::IsTerminal` in Rust 1.70+) to detect whether stderr is a TTY. Skip color when not. Approximate diff:

```diff
+use std::io::IsTerminal;

 pub fn render_all_pretty(diags: &[Diagnostic], file: &Path, source: &str) -> String {
-    let with_color = std::env::var_os("NO_COLOR").is_none();
+    let with_color = std::env::var_os("NO_COLOR").is_none()
+        && std::io::stderr().is_terminal();
     …
```

Five lines, no new dependencies (`IsTerminal` is std).

### Regression test

Snapshot test: invoke `cargo run --bin corvid -- check` with stderr piped to a non-TTY, assert the captured output contains no `\x1b[` escape sequences.

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

## L-8 — `approve` identifier must match dangerous-tool name (undocumented)

**Severity:** Low (docs)
**Component:** docs only
**Status:** language behavior is correct; documentation gap

### Reproduction

```corvid
tool send_to_pagerduty(s: String, b: String) -> Nothing dangerous

agent escalate(s: String, b: String) -> Nothing:
    approve PageOnCall(s, b)              # rejected
    send_to_pagerduty(s, b)
```

```sh
corvid check src/main.cor
# [E0101] error: dangerous tool 'send_to_pagerduty' called without a prior 'approve'
# Help: add `approve SendToPagerduty(arg1, arg2)` on the line before this call
```

The `approve` identifier must be the PascalCase form of the tool's `snake_case` name. `PageOnCall` is rejected; `SendToPagerduty` is required.

### Verdict

This is by design — pinning the approve name to the tool name lets reviewers grep `^\s*approve SendToPagerduty\b` and find every approval site for a given tool. Good security property.

But:

- The `approve-gates` tour topic shows the syntax without naming the rule.
- The language reference under `docs/` doesn't state the PascalCase convention explicitly.
- New users hit this on their first `dangerous` tool and learn it from the compiler error rather than the docs.

### Fix

Add a section to `docs/effects-spec/03-typing-rules.md` (or wherever approve gates are specified) titled "approve identifier naming" that says: *"The identifier following `approve` must be the PascalCase form of the dangerous tool's snake_case name. The compiler rejects mismatches at typecheck time (E0101). This makes approval sites greppable per-tool."*

Also extend the `corvid tour --topic approve-gates` blurb to mention the rule once.

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
