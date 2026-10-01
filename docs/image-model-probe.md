# Experimental native image model selection

This fork adds an optional `model` argument to the existing native
`image_gen.imagegen` tool. Omission preserves `gpt-image-2`. An explicit value
is passed unchanged through Codex's existing authenticated image transport for
both generation and editing. It does not add an API client, change endpoints,
or require an API key.

Example native tool arguments:

```json
{"model":"gpt-image-2.5-flare","prompt":"A blue mug on a white table"}
```

Replace the model with `gpt-image-2.5-sunburst` to request Sunburst. Names accept
1-128 ASCII letters, digits, dots, underscores, or hyphens. Unknown but
syntactically valid names are intentionally permitted so the backend, rather
than a stale local allowlist, determines availability. The experiment includes
a nonexistent name as a negative control.

The default `auto` quality and size are unchanged. No fallback or model retry
has been added. A successful response proves only that the request succeeded,
not that a particular serving model handled it.

## Build and check

From `codex-rs`, using the repository's Rust toolchain:

```sh
CARGO_INCREMENTAL=0 cargo test --locked -p codex-image-generation-extension --lib --profile dev-small -j 4
CARGO_INCREMENTAL=0 cargo test --locked -p codex-api --lib image --profile dev-small -j 4
CARGO_INCREMENTAL=0 cargo build --locked -p codex-app-server --bin codex-app-server -p codex-code-mode-host --bin codex-code-mode-host --profile dev-small -j 4
```

The standalone `target/dev-small/codex-app-server` supports the same stdio
protocol as `codex app-server`; it does not take an `app-server` subcommand.
Use it as a separate executable, without replacing the installed Codex.
Models using code mode also require the sibling `codex-code-mode-host` binary.
For this experiment, the app-server was built from the fork and the existing
Codex 0.159.1 code-mode host was symlinked beside it. That host executes the
agent's JavaScript; native image requests still run in the patched app-server.
The command above builds both executables if a compatible host is not installed.

## Evidence

Set `RUST_LOG=error,codex_image_model_probe=info` and `LOG_FORMAT=text` on the
app-server process. `CODEX_IMAGE_MODEL_PROBE` records on stderr contain the
serialized request's model and, after a successful response, any top-level
model field actually reported by the service, plus quality and size. No
prompt, image, credential, or full response is logged by this probe.

The optional response model stays absent when the service omits it. Request
names and response names remain distinct. An echoed response name would still
be service-reported identity, not independent attestation of model weights.

## Observed result, October 1, 2026

Four live trials used the same simple blue-mug photograph brief through the
fork's native tool, coordinated by gpt-6-astra with the existing ChatGPT login.
Each produced one image, one exact requested-model record, and no other tool
calls. None returned a top-level serving-model identity.

| Requested model | Native image time | Result |
| --- | ---: | --- |
| `gpt-image-2` | 25.048 s | Image returned |
| `gpt-image-2.5-flare` | 31.873 s | Image returned |
| `gpt-image-2.5-sunburst` | 21.242 s | Image returned |
| `codex-image-negative-control-not-a-real-model` | 21.589 s | Image returned |

All responses reported `medium` quality and `1254x1254` size. The experiment
proves that explicit names reach the native request. It does not prove model
selection: the nonexistent name also succeeded. The service might ignore the
field or silently fall back. These observations cannot distinguish those
behaviors or establish that every request used the same serving model.

Validation completed with 184 codex-api tests and 15 image-extension tests
passing, a successful app-server build, Python harness checks, and all four
live request-log audits. The new selector tests were first observed failing
against the original implementation.

Initial attempts failed before image requests because the standalone build
lacked its sibling code-mode host. Those failures are excluded from the table.
Successful trials used an isolated Codex home and reused the installed native
login through an auth-file symlink, removed after the trials. No credentials,
images, or private configuration are included in this repository.

### Blinded timing and quality follow-up

A further 12 runs compared a text-heavy poster and a three-change precision
edit. Each requested name ran twice per task, with randomized order within
four task/repetition blocks. Full image prompts were identical within each
task. One AI reviewer saw only opaque sample IDs and scored a fixed nine-check
rubric plus visual polish, without access to model labels or timings.

| Requested name | Poster median | Edit median | Mean checks passed | Mean polish |
| --- | ---: | ---: | ---: | ---: |
| Flare | 42.4 s | 30.2 s | 8.25/9 | 4.75/5 |
| Sunburst | 34.6 s | 25.8 s | 9.00/9 | 5.00/5 |
| Nonexistent control | 39.0 s | 26.5 s | 9.00/9 | 5.00/5 |

Eleven outputs passed every check. One Flare-requested edit introduced a
burgundy stripe onto the blue mug body while recoloring the handle. That
single defect does not establish a model-specific defect rate. The invented
name scored as well as Sunburst, and timing did not reveal the expected
faster-Flare/slower-Sunburst ordering.

All twelve trials had one exact native model request, one image call, no other
tools, and no reported serving model. This small screen did not verify model
selection. It also cannot establish population equivalence or prove the
service always ignores the field. Quality scores hit a ceiling on the poster
task; repeated timing and quality alone cannot attach verified model names
without known labeled reference distributions.
