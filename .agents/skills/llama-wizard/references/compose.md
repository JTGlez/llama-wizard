# Compose

## Goal

Bring up the `llm-server` container with a GGUF model loaded, validate it via the OpenAI-compatible HTTP API, and present a final report. This is the last step of the flow: the image is built (or pulled), the model is downloaded (or pre-existing), the container runs, and a real chat completion round-trips successfully.

## Pre-conditions

- `references/detect.md` has run with a verdict of `ready` or `ready with warnings`.
- One of the following is true:
  - The compile flow (`references/compile/llama-cpp.md`) has produced the image `llama-wizard-llama-cpp:gguf-v0.19.0`, **or**
  - The pre-built path has pulled `ghcr.io/ggml-org/llama.cpp:server-cuda` (forthcoming; for now, this flow assumes the compile path).
- A GGUF model is available in `models/<dir>/` (from `references/models/download.md` or pre-existing).
- The cwd of the agent is the repo root, where `docker-compose.example.yml` and `.env.example` are tracked.

## A. Validate the image exists

```bash
if ! docker image inspect llama-wizard-llama-cpp:gguf-v0.19.0 >/dev/null 2>&1; then
  echo "Image llama-wizard-llama-cpp:gguf-v0.19.0 not found locally."
  echo "Run references/compile/llama-cpp.md first, or replace the image"
  echo "reference in docker-compose.yml with a pre-built one."
  exit 1
fi
```

## B. Resolve the model filename

If the user just came from `references/models/download.md`, the file is the most recent `.gguf` under `models/`. Otherwise, the user has to tell you the filename.

List the candidates:

```bash
find models -maxdepth 2 -type f -name '*.gguf'
```

If exactly one file is found, use it. If multiple files are found, ask the user which one to mount (or `ls -lS` to show sizes). If none are found, abort with: "No GGUF found under `models/`. Run `references/models/download.md` or place an existing GGUF there."

## C. Stage the local compose and env files

Skip if they already exist (the user may have edited them):

```bash
[ -f docker-compose.yml ] || cp docker-compose.example.yml docker-compose.yml
[ -f .env ]              || cp .env.example              .env
```

If `docker-compose.yml` already exists and contains the literal placeholder `<MODEL_FILE>`, replace it with the resolved filename from step B:

```bash
sed -i "s|<MODEL_FILE>|${MODEL_FILENAME}|" docker-compose.yml
```

After replacement, verify:

```bash
grep -q -- "<MODEL_FILE>" docker-compose.yml && \
  { echo "Placeholder <MODEL_FILE> still present; replace it manually."; exit 1; } || true
```

## D. Bring the container up

```bash
docker compose up -d
```

Wait for `/health` to return OK. Poll every 5 seconds, timeout 90 seconds.

```bash
for i in $(seq 1 18); do
  if curl -sf http://localhost:8080/health >/dev/null; then
    echo "healthy after ${i} polls"
    break
  fi
  sleep 5
done
curl -sf http://localhost:8080/health || { echo "Server not healthy after 90s"; docker compose logs --tail=50; exit 1; }
```

The expected body is `{"status":"ok"}` (curl swallows it; that's fine — the exit code is what matters).

## E. Validate the model is registered

```bash
curl -sf http://localhost:8080/v1/models | jq .
```

The response must include the loaded model id and `format: "gguf"`. If the response is empty or the model id is not in the list, the `--model` argument was wrong: abort, show the compose file's `command:` section, and tell the user to fix it.

## F. Send a chat completion

Use a small prompt that exercises the chat template. The exact wording is not important; the goal is to confirm that token generation works end-to-end.

```bash
curl -sf -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "default",
    "messages": [{"role": "user", "content": "Reply with the single word: ready"}],
    "max_tokens": 16,
    "temperature": 0
  }' | jq .
```

Two things must be true:

- The response has HTTP 200 (the `curl -sf` would have exited non-zero otherwise).
- `choices[0].finish_reason` is `"stop"` (not `"length"`). If it is `"length"`, the model is generating but truncating — that's a model/template issue, not a setup issue; abort and tell the user to check the model and `--jinja` / chat template config.
- `choices[0].message.content` is non-empty.

If both pass, the setup is end-to-end working.

## G. Report

Present:

```
Server report
-------------
Container:     llm-server  (llama-wizard-llama-cpp:gguf-v0.19.0)
Compose:       docker-compose.yml  (copied from docker-compose.example.yml)
Env file:      .env                (copied from .env.example)
Model:         <path under models/ as mounted at /models/...>
Endpoint:      http://localhost:8080
Health:        ok  (after <N> polls)
Model list:    /v1/models returns the loaded model with format=gguf
Chat smoke:    POST /v1/chat/completions returned HTTP 200, finish_reason=stop

Verdict: ready
```

## H. Teardown (optional)

The user can stop the container with:

```bash
docker compose down
```

To also remove the image:

```bash
docker compose down --rmi local
```

Neither of these is run automatically — leaving the server up lets the user interact with it. Mention both options in the final summary so the user knows they exist.

## What this file does NOT do

- It does not download a model. That is `references/models/download.md`.
- It does not build the image. That is `references/compile/llama-cpp.md`.
- It does not modify `docker-compose.example.yml` or `.env.example`; it only writes to the gitignored copies (`docker-compose.yml`, `.env`).
- It does not auto-teardown.
