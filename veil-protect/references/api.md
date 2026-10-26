# Protection API

The `veil` CLI and the `almata-ai/veilcc-protect` Action are clients of one endpoint.
Everything the CLI does, you can do over HTTP.

```
POST https://veilcc.com/api/protect
Authorization: Bearer veil_sk_live_…
Content-Type: multipart/form-data
```

On-prem / air-gapped: run the identical API inside your network and set `--api-url`
(CLI) or the Action's `api-url`; the object never leaves.

## Form fields

| Field | Value |
|---|---|
| `binary` **(required)** | The object (`.o`) to seal, or the IR (`.ll`) to obfuscate. Max 24 MB. |
| `level` | `obfuscate` · `seal` · `device-bound` (default `seal`) |
| `proto` | Seal: the C signature of the sealed function, e.g. `uint32_t check_license(const char*)`. Repeatable. Omit for a C source object with `veil:seal` annotations. |
| `symbols` | Comma-separated symbol names for unannotated sources (Rust, IL2CPP, Swift). |
| `target` | Output triple (see below). Default `auto-detect`. |
| `persistent` | `1` to cache the decrypted function (seal). |
| `anti_debug` | `1` to fold a debugger-present check into the key (seal). |
| `key_provider` | Runtime key-unwrap symbol (device-bound). |
| `seed` | `0x…` fixed build seed (obfuscate). |

## Response (200, application/json)

```json
{
  "ok": true,
  "artifact_b64": "f0VMRgIBAQ…",
  "receipt": {
    "sealed_symbol": "check_license",
    "level": "seal",
    "target": "arm64-macos",
    "plaintext_in_file": "none",
    "key_cache": "ephemeral (scrubbed per call)",
    "anti_debug": "on",
    "seed": "0x9f3ac1…",
    "receipt_hash": "b7a4…e39c"
  },
  "quota": { "used": 42, "limit": 500 }
}
```

Decode `artifact_b64` to the output path and link it as normal.

### curl example

```sh
curl https://veilcc.com/api/protect \
  -H "Authorization: Bearer $VEILCC_KEY" \
  -F "binary=@license.o" \
  -F "level=seal" \
  -F "proto=uint32_t check_license(const char*)" \
  -F "anti_debug=1" \
  -o response.json
```

## Targets

`auto-detect` (default) · `arm64-macos` · `x86_64-macos` · `x86_64-linux` ·
`aarch64-linux` · `x86_64-windows`

## Errors

| Status | Meaning |
|---|---|
| `400` | Bad request (e.g. sealing without a `proto`, or an unknown target) |
| `401` | Missing/invalid/revoked API key |
| `413` | Object exceeds the 24 MB cap |
| `422` | Toolchain rejected the input (e.g. named symbol not in the object) |
| `429` | Monthly quota reached for the workspace's plan |
| `502` | Protection runner unreachable |
