# openjd-rs-koans

Worked examples and teaching materials for Open Job Description (OpenJD)
features, with a focus on capabilities proposed in in-flight RFCs.

Each subdirectory contains a set of small, self-contained YAML templates
that demonstrate one feature end-to-end, with inline comments explaining
the non-obvious parts. The goal is to be a sandbox — read a few, try a few,
adapt one into your own job templates.

## Contents

| Folder | Topic |
|--------|-------|
| [`rfc-0008/`](./rfc-0008/) | Environment Wrap Actions (RFC 0008) — the three `onWrap*` hooks, `runOnHost: true`, and EXPR-driven safe command composition with `repr_sh()`. 21 worked examples covering Docker, Apptainer, SSH, Wine, profiling, privilege isolation, audit logging, retry, conda, resource limits, and more. |

## Related RFCs

The examples here reference these RFCs from
[openjd-specifications](https://github.com/OpenJobDescription/openjd-specifications):

- [RFC 0002 — Model Extensions](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0002-model-extensions.md) — how extensions are gated
- [RFC 0005 — Expression Language](https://github.com/OpenJobDescription/openjd-specifications/pull/93) (`EXPR`) — Python-subset expression language
- [RFC 0006 — Expression Function Library](https://github.com/OpenJobDescription/openjd-specifications/pull/104) — `repr_sh`, `repr_json`, `flatten`, `fail`, etc.
- [RFC 0008 — Environment Wrap Actions](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0008-environment-wrap-task-run.md) (`WRAP_ACTIONS`)

## Running the examples

These templates require a scheduler that implements the relevant extensions
(`WRAP_ACTIONS`, `EXPR`, etc.). They're intended as study material and
starting points, not drop-in production templates — each one focuses on a
specific pattern and skips error handling / logging you'd want in real use.

## License

Apache-2.0. See [LICENSE](./LICENSE).
