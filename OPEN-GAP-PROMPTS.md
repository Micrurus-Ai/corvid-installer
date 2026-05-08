# Open-gap implementation prompts

Three gaps from [LANGUAGE-GAPS.md](./LANGUAGE-GAPS.md) are still open: **L-3**, **L-4**, and **L-7**. Each section below is a self-contained prompt you can paste into a fresh AI-agent session (or hand to a contributor) to get to a working upstream PR. Prompts assume the agent has a clone of `Micrurus-Ai/Corvid-lang` at HEAD and a working Rust toolchain.

Each prompt is independent — none refer to the others. Pick whichever you want to ship next.

- [L-3 — Native codegen: struct returns](#l-3-prompt)
- [L-4 — WASM target: String parameters](#l-4-prompt)
- [L-7 — Lexer: `\` line continuation](#l-7-prompt)

---

## L-3 prompt

> **Title:** Implement native codegen support for struct returns (prompt-bridge + entry-agent boundaries)

```
You are working in Micrurus-Ai/Corvid-lang at HEAD. Corvid is a
programming language for AI-native software with a Cranelift-based
native code generator at `crates/corvid-codegen-cl/`.

The native target currently rejects struct returns at two distinct
sites in the codegen. Your job is to implement the missing
serialization layer so both sites accept structs.

## Two failing reproductions

### Site 1 — prompt returning a struct

```corvid
type Decision:
    label: String
    score: Float

prompt classify(text: String) -> Decision:
    """Classify {text} as a Decision."""

agent main(input: String) -> Decision:
    return classify(input)
```

```sh
corvid build src/main.cor --target=native
# error: native codegen does not yet support: prompt 'classify'
# returns 'struct' — the native prompt bridge currently supports only
# Int / Bool / Float / String returns; structured prompt returns are
# not implemented yet
```

### Site 2 — entry agent returning a struct

The same program above also fails at a *different* error site:

```sh
# error: native codegen does not yet support: entry agent 'main' returns
# 'struct' — the native command-line boundary currently supports only
# Int/Bool/Float/String returns; structured output needs a dedicated
# serialization layer
```

Find both error sites by grepping the codegen-cl crate for the literal
error strings.

## Goal

Both sample programs compile under `--target=native` and produce a
working binary that:

1. Accepts the `input` argument from argv.
2. Calls the LLM via the prompt bridge with `CORVID_MODEL` configured.
3. Marshals the JSON LLM response back into a heap-allocated `Decision`.
4. Serializes the resulting struct (JSON to stdout is fine) before
   exit.

## Suggested approach

Mirror how `Grounded<T>` is already handled for primitives.

1. **Heap allocation.** Use the existing `corvid-runtime` allocator
   to reserve space for the struct based on its type descriptor.
2. **Prompt-bridge return.** Extend the prompt bridge in
   `crates/corvid-codegen-cl/src/lowering/prompt.rs` (and any helpers
   it calls in `corvid-runtime`) to use the existing JSON deserializer
   with the destination struct's type descriptor and populate the
   heap slot. Return a pointer to that slot.
3. **Entry-agent return.** In the entry-agent lowering path, after
   the agent returns, walk the struct's type descriptor and serialize
   to stdout via the same JSON encoder used for `Grounded<T>`.
4. Share `serialize_struct` / `deserialize_struct` between the two
   sites — same underlying type descriptor, same JSON shape.

## Files you'll likely touch

- `crates/corvid-codegen-cl/src/lowering/prompt.rs` (prompt bridge)
- The entry-agent lowering file in the same crate (find via grep on
  the entry-boundary error string)
- `crates/corvid-runtime/src/` — extend the JSON
  serializer/deserializer if it doesn't yet handle structs
- `crates/corvid-codegen-cl/src/errors.rs` — remove or narrow the two
  "not yet supported" error variants

## Acceptance criteria

1. Both reproductions above compile with `--target=native`.
2. `cargo test --workspace` still passes; new tests in
   `crates/corvid-codegen-cl/tests/` cover both struct-return sites
   (prompt-return and entry-return) end to end.
3. The interpreter and Python targets continue to handle the same
   code unchanged. Add a cross-target verify test that runs the same
   `.cor` source through interpreter, Python, and native, and checks
   that all three produce the same struct values.
4. Documentation under `docs/` (the native-target spec, if one
   exists) is updated to remove the "structured returns not
   supported" caveat.

## Out of scope

- Streaming struct returns (`Stream<Decision>`) — separate gap.
- Recovery from malformed LLM JSON beyond what the existing
  `Grounded<T>` path already does.
- Removing the `Grounded<T>` primitive-only restriction (separate
  gap, parallel work).
- Cross-FFI struct passing for tools (different boundary).

## Why this matters

`prompt foo(...) -> SomeStruct` is the most common shape in real
Corvid programs. Forcing users to flatten to primitives or fall back
to the Python target defeats the native target's single-binary
deployment pitch. This is the largest remaining usability gap on
native.

## Reference

The `LANGUAGE-GAPS.md` entry for L-3 in
https://github.com/Micrurus-Ai/corvid-installer/blob/main/LANGUAGE-GAPS.md
documents the gap. The `crates/corvid-codegen-cl/src/lowering/prompt.rs`
file contains the rejection logic; search the crate for the literal
error strings above to find both sites.
```

---

## L-4 prompt

> **Title:** Implement String parameter and return support in the WASM target

```
You are working in Micrurus-Ai/Corvid-lang at HEAD. Corvid's WASM
code generator at `crates/corvid-codegen-wasm/` rejects any agent or
tool whose signature uses `String` for parameters or returns. Today
only `Int`, `Float`, `Bool`, and `Nothing` cross the WASM boundary.

Your job is to implement a UTF-8 string ABI across codegen, the JS
loader, and the TypeScript types.

## Failing reproduction

```corvid
agent shout(msg: String) -> String:
    return msg
```

```sh
corvid build src/main.cor --target=wasm
# error: wasm codegen failed: wasm target currently supports only Int,
# Float, Bool, and Nothing scalar parameters; agent 'shout' parameter
# 'msg' has 'String'
```

## Goal

`String` parameters and return values work on the WASM boundary. From
JS:

```js
import { shout } from './main.js';
await shout('hello'); // returns 'hello'
```

…and the generated `main.d.ts` correctly types `shout` as
`(msg: string) => Promise<string>`.

## Suggested approach

Use the bare `(ptr: i32, len: i32)` ABI — UTF-8 with explicit length.
WASM Component Model adapters can come later; the bare ABI is the
correct first step.

### 1. Codegen

In `crates/corvid-codegen-wasm/`:

- For each `String` parameter, replace it with two `i32` args:
  `<name>_ptr`, `<name>_len`. Inside the function body, read those
  bytes from linear memory into a UTF-8 string before further
  lowering.
- For a `String` return, change the export signature to take a
  caller-provided write slot (`out_ptr_and_len: i32`) and have the
  module write `(ptr, len)` into that slot. Or use the simpler
  multi-value return convention if Cranelift's WASM backend supports
  it on the configured profile.
- Export `corvid_alloc(size: i32) -> i32` and
  `corvid_free(ptr: i32, size: i32)` so the loader can manage the
  memory lifecycle.

### 2. JS loader

In the loader-emission code (find via grep on `.js` template
strings or `loader.rs`):

For each function that takes a `String`:

```js
async function shout(msg) {
  const enc = new TextEncoder();
  const bytes = enc.encode(msg);
  const ptr = wasm.exports.corvid_alloc(bytes.length);
  new Uint8Array(wasm.exports.memory.buffer, ptr, bytes.length).set(bytes);
  try {
    const retSlot = wasm.exports.corvid_alloc(8);
    wasm.exports._shout(ptr, bytes.length, retSlot);
    const view = new DataView(wasm.exports.memory.buffer);
    const retPtr = view.getInt32(retSlot, true);
    const retLen = view.getInt32(retSlot + 4, true);
    const out = new TextDecoder().decode(
      new Uint8Array(wasm.exports.memory.buffer, retPtr, retLen)
    );
    wasm.exports.corvid_free(retPtr, retLen);
    wasm.exports.corvid_free(retSlot, 8);
    return out;
  } finally {
    wasm.exports.corvid_free(ptr, bytes.length);
  }
}
```

Wrap that pattern in a helper so each export gets compact glue.

### 3. TypeScript types

Update the `.d.ts` emitter so `Corvid::String` lowers to TypeScript
`string`. Today it almost certainly lowers to `number` (because the
underlying WASM signature uses `i32`). Map at the Corvid type level,
not the lowered WASM level.

### 4. Manifest

Update `<file>.corvid-wasm.json` so each function's parameter
descriptor lists `kind: "string"` for String params (instead of
`"i32"`). Lets downstream tooling tell the difference.

## Files you'll likely touch

- `crates/corvid-codegen-wasm/src/lib.rs` — the parameter-type check
  that currently rejects `String`
- `crates/corvid-codegen-wasm/src/codegen.rs` (or similar) — WASM
  function emission
- `crates/corvid-codegen-wasm/src/loader.rs` (or similar) — JS
  glue emission
- `crates/corvid-codegen-wasm/src/dts.rs` (or similar) — `.d.ts`
  emission
- `crates/corvid-runtime-wasm/src/` (if it exists) — the
  `corvid_alloc` / `corvid_free` exports

## Acceptance criteria

1. The reproduction above compiles with `--target=wasm` and emits
   `main.wasm`, `main.js`, `main.d.ts`, `main.corvid-wasm.json`.
2. `main.d.ts` declares
   `export function shout(msg: string): Promise<string>;`. No
   `number` types for string fields.
3. A round-trip integration test under
   `crates/corvid-codegen-wasm/tests/`: load `main.wasm` in Node,
   call `shout('hello')`, assert the result is `'hello'`.
4. Multi-byte UTF-8 round-trip test: `shout('héllo 🦀')` returns the
   same string.
5. The current Int-parameter happy path
   (`agent dbl(x: Int) -> Int: return x * 2`) still compiles and
   runs unchanged.
6. `cargo test --workspace` passes.

## Out of scope

- WASM Component Model adapters (use the bare ABI first).
- UTF-16 strings or DOM string interop — UTF-8 is enough.
- `Stream<String>` — that's a streaming-channel feature, separate
  gap.
- Multi-string return tuples — single-String returns are sufficient
  for v1.

## Why this matters

WASM is one of three deployment paths Corvid advertises (interpreter,
native, wasm). Without String support, the WASM path can only ship
arithmetic / boolean APIs — useless for the AI-agent positioning.
Real-world WASM-targeted Corvid programs need at minimum strings and
prompts.

## Reference

The `LANGUAGE-GAPS.md` entry for L-4 in
https://github.com/Micrurus-Ai/corvid-installer/blob/main/LANGUAGE-GAPS.md
documents the gap. The literal error string above lives in
`crates/corvid-codegen-wasm/src/lib.rs`; grep for it.
```

---

## L-7 prompt

> **Title:** Decide on `\` line continuation, then either implement it or document the absence

```
You are working in Micrurus-Ai/Corvid-lang at HEAD. The lexer
(`crates/corvid-syntax/src/lexer/`) rejects `\` (backslash) at end of
line with diagnostic E0003 "unexpected character `\`". Python and
JavaScript developers (the most likely first-time Corvid users)
reflexively reach for `\` to continue long lines.

Your job is BOTH:

1. Determine whether the language design committee wants `\` line
   continuation. Check `docs/effects-spec/`, `ROADMAP.md`, any
   open RFCs, or commit history for prior discussion. If unclear,
   open a brief proposal under `docs/` for review and stop here
   pending a decision.
2. Once the decision is made, do EITHER (A) "implement it" OR (B)
   "document the absence" — never both.

## Failing reproduction (current state)

```corvid
agent main() -> String:
    return "first part " \
           "second part"
```

```sh
corvid check src/main.cor
# [E0003] error: unexpected character `\`
```

## Decision: A — implement `\` line continuation

If the design decision is "yes, support it", implement these
behaviors:

- `\` followed immediately by a newline (and any leading whitespace
  on the next line) is consumed silently, both inside `"..."` string
  literals and outside any string.
- Inside triple-quoted `"""..."""` blocks, the existing rules apply
  unchanged — don't add `\` continuation there because triple-quoted
  blocks already span lines.
- A `\` not at end-of-line (e.g., before a non-newline character)
  remains an E0003 error.

### Approach (A)

In the string-literal lexing function (likely
`crates/corvid-syntax/src/lexer/string.rs` or `mod.rs`):

1. After consuming a normal character inside a `"..."` literal, check
   for the `\` + `\n` sequence and skip both characters.
2. In the top-level lexer dispatch, before emitting a `Newline`
   token, check whether the preceding non-whitespace character was a
   `\` outside a string — if so, suppress the newline and the `\`.

Tests to add (in `crates/corvid-syntax/tests/` or wherever the lexer
tests live):

- `\` at end of line inside a `"..."` string compiles.
- `\` at end of line outside any string compiles (e.g., split a long
  decorator across two lines).
- `\` followed by a non-newline character is still rejected.
- `\\\n` inside a triple-quoted block behaves as before (i.e., is
  a backslash followed by a literal newline, not a continuation).

### Acceptance criteria (A)

1. The reproduction above checks and builds cleanly.
2. New tests in `crates/corvid-syntax/tests/` cover the four cases
   above.
3. Existing tests continue to pass.
4. `docs/syntax.md` (create if needed) gets a paragraph titled
   "Continuation rules" describing the behavior.
5. The `corvid tour` topic for AI-native keywords (or whichever tour
   topic covers syntax) gets a one-line mention.

## Decision: B — document the absence and improve the error

If the design decision is "no, don't support it":

1. Update the lexer's E0003 diagnostic for `\` at end of line to be
   helpful:
   ```
   [E0003] error: unexpected character `\`
   help: Corvid does not support backslash line continuation. For
         long strings, use `+` concatenation or triple-quoted
         strings. For long expressions, use parentheses.
   ```
2. Add a paragraph to `docs/syntax.md` (create if needed) titled
   "Continuation rules" stating that the language deliberately omits
   `\` line continuation, with a one-sentence rationale and pointers
   to the recommended alternatives.
3. Add a regression test that verifies the new diagnostic includes
   the `help:` line so the message stays helpful in future
   refactors.

### Acceptance criteria (B)

1. The reproduction above produces the new helpful error message.
2. `docs/syntax.md` has the "Continuation rules" paragraph.
3. New diagnostic-help regression test passes.
4. No semantic change to lexer behavior.

## Files you'll likely touch (for either decision)

- `crates/corvid-syntax/src/lexer/` (exact file depends on layout)
- `crates/corvid-syntax/src/errors.rs` (for the diagnostic)
- `docs/syntax.md` or `docs/effects-spec/` (rationale paragraph)
- `crates/corvid-syntax/tests/` (regression test)

## Out of scope

- Implicit line continuation inside parentheses (a separate
  language-design decision; don't conflate).
- Heredoc-style multi-line strings beyond the existing triple-quote.
- Changing how `\\` escape sequences inside strings work.

## Why this matters

Low priority — there's always a workaround (`+` concatenation,
triple-quoted strings, parens). But the surprise factor for incoming
Pythonistas is real, and either implementing the feature OR
prominently documenting its absence resolves the surprise. The bug
isn't the absence of the feature; it's that there's no signal in the
docs or the error message that it's a deliberate choice.

## Reference

The `LANGUAGE-GAPS.md` entry for L-7 in
https://github.com/Micrurus-Ai/corvid-installer/blob/main/LANGUAGE-GAPS.md
documents the gap and lists the workarounds new users currently have
to discover by trial and error.
```

---

## How to use these prompts

Each prompt is designed to drop into a fresh AI-agent session (or a
GitHub issue body) with no prior context. The agent should be able to
read the prompt, clone Corvid-lang, and produce a working PR
from those instructions alone.

Recommended order if you want to ship more language fixes:

1. **L-7 first.** Smallest scope, fastest decision-and-ship cycle.
   Either way it concludes (implement or document), the change is
   bounded to ~30 lines + a docs paragraph.
2. **L-4 next.** Bounded scope (one ABI choice + glue), high impact
   for the WASM-deployment story, doesn't block on language-design
   discussions.
3. **L-3 last.** Largest scope (touches runtime serialization +
   codegen at two sites), but also the most impactful single fix —
   makes `prompt foo(...) -> SomeStruct` work natively, which is
   probably the most common real-world pattern.

If you'd rather pick by impact alone, L-3 is the biggest unlock; if
by speed-to-ship, L-7 is the smallest. L-4 is the best middle
ground.
