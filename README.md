# VeilCC agent skills

An [agent skill](https://skills.sh) that teaches a coding agent (Claude Code, Codex,
Cursor, …) to protect a compiled binary with **[VeilCC](https://veilcc.com)** — sealing
or obfuscating the functions that matter so they can't be reverse-engineered out of the
shipped artifact.

## Skills

| Skill | What it does |
|---|---|
| [`veil-protect`](./veil-protect/SKILL.md) | Install the `veil` CLI, pick the crown-jewel functions, mark them (C/C++/Rust/Swift/IL2CPP), insert the server-side protect step between compile and link, wire CI, and verify the signed build receipt. |

## Install

```sh
npx skills add almata-ai/veilcc-skill@veil-protect
```

Then just ask your agent to "protect my license check with VeilCC" (or similar) and
it will follow the skill.

## Links

- Product & docs: <https://veilcc.com> · <https://veilcc.com/docs>
- CLI: `brew install almata-ai/tap/veil` ([almata-ai/veil-cli](https://github.com/almata-ai/veil-cli))
- GitHub Action: [almata-ai/veilcc-protect](https://github.com/almata-ai/veilcc-protect)

## License

MIT — see [LICENSE](./LICENSE).
