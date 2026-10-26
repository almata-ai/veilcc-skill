---
name: veil-protect
description: "Use when the user wants to protect a compiled binary against reverse-engineering (including by AI/LLMs) — hiding a license check, attestation routine, key-derivation, or a proprietary algorithm from being read out of the shipped artifact. Covers the VeilCC veil CLI and GitHub Action — installing it, choosing which functions to seal, marking them in C/C++/Rust/Swift (or by symbol), inserting the server-side protect step between compile and link, wiring CI, and verifying the signed build receipt. Use this when someone asks to 'protect/obfuscate/seal my binary', 'stop people reverse-engineering my license check', 'harden a native function', or mentions VeilCC / the veil CLI / the almata-ai/veilcc-protect action. Not for source-level secrets management or general code signing."
---

# VeilCC — protect a binary against (AI) reverse-engineering

VeilCC seals or obfuscates the **functions that matter** inside a compiled artifact.
The pass runs **server-side** against the user's API key, so the protection algorithm
never ships inside the binary. The output is a **drop-in replacement object** — the
link step does not otherwise change.

Your job as the agent: identify the crown-jewel functions, get the user to confirm
them, mark them, insert the protect step into the build and CI, and verify the
receipt. **Only protect functions the user confirms.** Everything else stays native
(no runtime cost).

Docs: <https://veilcc.com/docs> · CLI & API: <https://veilcc.com/docs/cli>

## What it does and does not do

- **Does** stop static recovery and raise a *determined local* attacker from minutes
  to measured hours/days, with a per-build, verifiable cost-to-break number.
- **Does not** make locally-run code mathematically unrecoverable: someone who runs
  the binary and dumps RAM during a sealed call can still recover that function. No
  pure-software scheme escapes that — for near-zero recovery against a copy, use
  **device-bound** keys (runtime key unwrap via enclave/TPM).

Be honest about this with the user; don't oversell "unbreakable."

## Workflow

### 1. Install the CLI

```sh
brew install almata-ai/tap/veil
# or:
go install github.com/almata-ai/veil-cli@latest
```

Authenticate: `veil login` (browser, stores a token in `~/.config/veilcc`) for a
person, or an API key for CI (`export VEILCC_KEY=veil_sk_live_…`, or `--api-key`).
Auth resolves in order: `--api-key` → `VEILCC_KEY` → stored login.

### 2. Identify the crown jewels — then STOP and confirm

Scan the project for the 1–3 functions whose logic must not be readable: license /
entitlement checks, attestation, key derivation, a proprietary kernel/algorithm.
List them as `file:line` and **ask the user to confirm before editing anything.**
Don't seal broadly — sealing hot, high-frequency functions is wasteful; seal the
few that carry secret logic.

### 3. Mark the confirmed functions

| Language | How |
|---|---|
| C / C++ | `__attribute__((annotate("veil:seal")))` above the function |
| Rust | export with the C ABI: `#[no_mangle] pub extern "C" fn …`, seal by symbol |
| Swift | `@_cdecl("symbol_name")`, seal by symbol |
| Anything compiled (IL2CPP, etc.) | no annotation needed — pass `--symbols` at build |

The sealer operates on the compiled **object**, so it is language-agnostic. If the
source can't be annotated, plan to pass `--symbols name1,name2`.

### 4. Insert the protect step: after compile, before link

Compile to an object (`.o`) or LLVM IR (`.ll`) as usual, then:

```sh
veil protect --level seal app.o -o app.sealed.o --receipt build-receipt.json
```

Link `app.sealed.o` in place of `app.o`. **Keep every other flag and the link order
identical.** Show the user the diff before applying it.

Levels (`--level`):
- `obfuscate` — operates on IR; moving-target seed (rotate per build, or `--seed 0x…` for reproducible).
- `seal` — encrypt-at-rest; decrypt per call then scrub (default). `--persistent` caches for very-hot functions; `--anti-debug` folds a debugger check into the key.
- `device-bound` — key unwrapped at runtime via `--key-provider SYM`; strongest against a copied binary.

See `references/languages.md` for per-language build snippets.

### 5. Wire CI (GitHub Action)

```yaml
- uses: almata-ai/veilcc-protect@v1
  with:
    level: seal            # obfuscate | seal | device-bound
    api-key: ${{ secrets.VEILCC_KEY }}
```

Tell the user to add the `VEILCC_KEY` repository secret (veilcc.com → console → API
keys; shown once). **Never commit the key.**

### 6. Verify — every build carries a signed receipt

```sh
veil verify build-receipt.json --artifact app.sealed.o
# → receipt hash matches · sealed: check_license · level seal · ok
```

Run this in CI so each build proves what was sealed, with what seed, and its
cost-to-break estimate.

## Doing it over HTTP instead of the CLI

The CLI and Action are thin clients of one endpoint — `POST https://veilcc.com/api/protect`
(Bearer key, multipart upload, base64 artifact + receipt back). On-prem / air-gapped
teams run the identical API inside their network and set `--api-url`. Full request/
response shapes and error codes are in `references/api.md`.

## Guardrails

- Only scan or protect binaries the user **owns or is authorized to test**.
- Confirm the function list before editing code; show diffs before applying.
- Don't change the user's toolchain, compiler flags, or link order beyond swapping
  in the sealed object.
- Keep the API key out of source and logs.
