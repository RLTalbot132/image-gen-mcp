# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm install` — install deps.
- `npm run build` — `tsc` then `chmod 755 build/index.js` (the chmod is required because `index.js` is invoked directly via its `#!/usr/bin/env node` shebang when wired into an MCP host).
- `npm run watch` — `tsc -w` for incremental rebuilds during development.
- `npm start` — runs `node build/index.js`. The server speaks MCP over stdio, so running it standalone is only useful for verifying it starts; real usage is through an MCP host that spawns it as a subprocess (see `README.md` for the host JSON config).

There is no test suite, linter, or formatter configured.

## Architecture

Single-file server: all logic lives in `src/index.ts` and compiles to `build/index.js`. One class, `ImageGenServer`, wires an `@modelcontextprotocol/sdk` `Server` (stdio transport) to a preconfigured `axios` instance pointed at the Stable Diffusion WebUI.

Five tools, all thin wrappers over the WebUI HTTP API:

| Tool | WebUI endpoint |
| --- | --- |
| `generate_image` | `POST /sdapi/v1/txt2img` (+ `POST /sdapi/v1/png-info` per result) |
| `get_sd_models` | `GET /sdapi/v1/sd-models` |
| `set_sd_model` | `POST /sdapi/v1/options` (sets `sd_model_checkpoint`) |
| `get_sd_upscalers` | `GET /sdapi/v1/upscalers` |
| `upscale_images` | `POST /sdapi/v1/extra-batch-images` |

`generate_image` post-processes each returned base64 image with `sharp`, writing it to disk as PNG with the WebUI's PNG-info string embedded as `EXIF IFD0.ImageDescription`. Output filenames are `sd_<uuid>.png`; upscaled outputs are `upscaled_<original-basename>`. The output directory comes from `output_path` (per-call) or `SD_OUTPUT_DIR` (env), and is auto-created.

### Configuration is entirely environment variables

There is no config file. All knobs are env vars read once at process start (see top of `src/index.ts`): `SD_WEBUI_URL`, `SD_AUTH_USER`/`SD_AUTH_PASS` (HTTP basic auth, both must be set), `SD_OUTPUT_DIR`, `REQUEST_TIMEOUT` (ms, default 300000), and the upscale defaults `SD_RESIZE_MODE`, `SD_UPSCALE_MULTIPLIER`, `SD_UPSCALE_WIDTH`, `SD_UPSCALE_HEIGHT`, `SD_UPSCALER_1`, `SD_UPSCALER_2`. Changes to these require restarting the MCP host.

### Two non-obvious API quirks (both fixed in git history — don't reintroduce)

- The txt2img payload field is `scheduler`, **not** `scheduler_name`. The MCP-facing parameter is named `scheduler_name` for clarity, but it's mapped to `scheduler` when sent to the WebUI (see `SDAPIPayload`).
- `upscale_images.resize_mode` is exposed to the MCP client as the **string** `'0'` or `'1'` (enum) and coerced to a number before sending. This is intentional — exposing it as a number caused validation errors with some MCP clients.

### Argument validation

Each tool has a hand-written type guard (`isGenerateImageArgs`, `isSetModelArgs`, `isUpscaleImagesArgs`) that runs before the call is dispatched. The JSON schema in `ListToolsRequestSchema` is *only* for the LLM-facing tool description — it does not validate inputs at runtime. When adding or changing a parameter, update **both** the schema (for the model) and the guard (for safety), and remember the guards mutate the input object (e.g. coercing `steps` from string to number) before the handler reads it.

### Error handling

The single try/catch in the `CallToolRequestSchema` handler funnels every failure into an `McpError`. Axios errors are unwrapped to surface the WebUI's `error` field when present; everything else becomes `InternalError` with the original message. Process-level `unhandledRejection` is logged but non-fatal; `uncaughtException` logs and exits after 500ms to give the transport time to flush.
