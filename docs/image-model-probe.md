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
CARGO_INCREMENTAL=0 cargo build --locked -p codex-app-server --bin codex-app-server --profile dev-small -j 4
```

The standalone `target/dev-small/codex-app-server` supports the same stdio
protocol as `codex app-server`; it does not take an `app-server` subcommand.
Use it as a separate executable, without replacing the installed Codex.

## Evidence

Set `RUST_LOG=error,codex_image_model_probe=info` and `LOG_FORMAT=text` on the
app-server process. `CODEX_IMAGE_MODEL_PROBE` records on stderr contain the
serialized request's model and, after a successful response, any top-level
model field actually reported by the service, plus quality and size. No
prompt, image, credential, or full response is logged by this probe.

The optional response model stays absent when the service omits it. Request
names and response names remain distinct. An echoed response name would still
be service-reported identity, not independent attestation of model weights.
