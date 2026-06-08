# Model download

## Goal

Optionally download a GGUF model from Hugging Face into `models/<model>/`, ready to be mounted by `references/compose.md`. Skip this flow if you already have a GGUF you want to serve; the compose flow accepts a placeholder for the filename.

## Pre-conditions

- `references/detect.md` has run with a verdict of `ready` or `ready with warnings`.
- The cwd of the agent is the repo root.
- `models/` exists (create it if not: `mkdir -p models`).
- Either `wget` is on PATH (preferred) or `curl` is on PATH (fallback; already a required tool in detect.md). If neither is present, abort.

## A. Ask the user

Ask the user two questions before downloading anything:

1. **Do you want to download a model now?**
   - If no: abort this flow. The user will fill in `<MODEL_FILE>` in `docker-compose.yml` manually. Continue to `references/compose.md` (the placeholder branch).
   - If yes: continue.

2. **Which quantization?**
   - The default recommendation is **`Q4_K_M`** for a first run (good quality-to-size ratio, fits in a single 24 GiB GPU with headroom for KV cache).
   - Available quantizations for `Jackrong/Qwopus3.6-27B-v2-MTP-GGUF` (subject to upstream changes; verify against the repo's file listing if in doubt):
     - `Q4_K_M` — recommended
     - `Q4_K_S`
     - `Q5_K_M`
     - `Q5_K_S`
     - `Q6_K`
     - `Q8_0`
   - Each file is named `Qwopus3.6-27B-v2-MTP-<QUANT>.gguf`.

## B. Compute the download URL and expected SHA256

The Hugging Face LFS API returns metadata for each file in the repo, including the expected SHA256 of the LFS object under `.lfs.sha256`. Some repos do not expose this field; in that case the SHA256 is still available from the `x-linked-etag` header of a HEAD request to the resolve URL.

```bash
QUANT="Q4_K_M"
REPO="Jackrong/Qwopus3.6-27B-v2-MTP-GGUF"
FILE="Qwopus3.6-27B-v2-MTP-${QUANT}.gguf"
HF_API="https://huggingface.co/api/models/${REPO}"
URL="https://huggingface.co/${REPO}/resolve/main/${FILE}"
```

Try the API first:

```bash
curl -sL "${HF_API}" -o /tmp/hf_model.json
SHA256=$(jq -r --arg f "${FILE}" '.siblings[] | select(.rfilename == $f) | .lfs.sha256' /tmp/hf_model.json)
```

If the API does not list the file at all (no matching `rfilename`), abort with: "Quantization `${QUANT}` not found in `${REPO}`. Verify the file name with https://huggingface.co/${REPO}/tree/main and re-run."

If the file is listed but `.lfs.sha256` is empty or null, fall back to the HEAD request. The `x-linked-etag` header on the resolve URL is the canonical SHA256 of the LFS object (the file content itself, not the LFS pointer). `curl -I` follows the redirect from the resolve URL to the LFS storage backend; the etag on the final response is the one that matters.

```bash
if [ -z "${SHA256}" ] || [ "${SHA256}" = "null" ]; then
  SHA256=$(curl -sILo /dev/null -D - "${URL}" \
    | awk 'tolower($1) == "x-linked-etag:" {gsub(/"/,"",$2); print $2; exit}')
fi
```

If after both attempts `SHA256` is still empty, the file is not reachable on HF. Abort with: "Could not determine SHA256 of `${FILE}` in `${REPO}`. The repo may be private, deleted, or the API may be rate-limiting. Check https://huggingface.co/${REPO}/tree/main manually."

The download URL is the resolve URL, not the API URL:

```bash
echo "URL=${URL}"
echo "SHA256=${SHA256}"
```

## C. Stage the model directory

```bash
MODEL_DIR="models/Qwopus3.6-27B-v2-MTP"
DEST="${MODEL_DIR}/${FILE}"
mkdir -p "${MODEL_DIR}"
```

If `${DEST}` already exists, compare its current SHA256 to the expected one. If they match, skip the download. If they differ, abort with: "`${DEST}` exists with a different SHA256. Remove it manually or pick a different quantization, then re-run."

```bash
if [ -f "${DEST}" ]; then
  CURRENT_SHA=$(sha256sum "${DEST}" | awk '{print $1}')
  if [ "${CURRENT_SHA}" = "${SHA256}" ]; then
    echo "Already downloaded: ${DEST} (SHA256 matches)"
  else
    echo "SHA256 mismatch for ${DEST}:"
    echo "  current:  ${CURRENT_SHA}"
    echo "  expected: ${SHA256}"
    exit 1
  fi
fi
```

## D. Download

Use `wget` if available, fall back to `curl -L -O`.

```bash
if command -v wget >/dev/null 2>&1; then
  wget -O "${DEST}" "${URL}"
else
  curl -L --fail -o "${DEST}" "${URL}"
fi
```

Set the bash tool timeout long enough for the file to land (a 22 GB Q4_K_M over a typical residential link can take 10–30 minutes).

## E. Verify

Re-checksum after download. The number must match the expected SHA256 from step B.

```bash
ACTUAL_SHA=$(sha256sum "${DEST}" | awk '{print $1}')
[ "${ACTUAL_SHA}" = "${SHA256}" ] && echo "SHA256 OK" || { echo "SHA256 MISMATCH"; exit 1; }
ls -lh "${DEST}"
```

## F. Report

Present:

```
Model report
------------
Model:        Qwopus3.6-27B-v2-MTP  (Jackrong/Qwopus3.6-27B-v2-MTP-GGUF)
Quantization: Q4_K_M
Path:         models/Qwopus3.6-27B-v2-MTP/Qwopus3.6-27B-v2-MTP-Q4_K_M.gguf
Size:         <output of `ls -lh` human-readable size>
SHA256:       <full 64-char hex>

Verdict: ready
```

The path is what the user (or `references/compose.md`) will plug into the `docker-compose.yml` `--model` argument, mounted at `/models/<FILENAME>` inside the container.

## What this file does NOT do

- It does not delete or replace existing models.
- It does not modify the compose file. The compose flow consumes the path produced here.
