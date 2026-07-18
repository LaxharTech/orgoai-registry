# OrgoAI model registry

The signed catalogs of on-device AI models for the Laxhar apps. One catalog per app,
each signed with the same publisher key but bound to its app id (`--app`), so an app
refuses every other app's manifest:

| App      | Catalog              | Fetched from                                                              |
|----------|----------------------|---------------------------------------------------------------------------|
| OrgoChat | `packs.json` (root)  | `https://raw.githubusercontent.com/LaxharTech/orgoai-registry/main/manifest` |
| Swasth   | `swasth/packs.json`  | `.../main/swasth/manifest`                                                |
| Kridax   | `kridax/packs.json`  | `.../main/kridax/manifest`                                                |

Subdirectory catalogs reference the shared root `weights/` via `path: '../weights/...'`.

It is an Ed25519-signed list of the models users are allowed to download. The signature is
verified in the browser against `NEXT_PUBLIC_AI_SIGNING_PUBLIC_KEY`, fail-closed — a manifest
that doesn't verify yields no models, and there is no unsigned fallback.

Nothing here is secret. The weights themselves are public bytes hosted by Hugging Face; this
repo only says which ones are blessed, and proves the list came from us.

## Re-signing

The manifest carries a TTL. Once it lapses the catalog is treated as stale and quietly empties,
so it must be re-signed before then. Current expiries: root (OrgoChat) **2027-07-17**;
`swasth/manifest` **2027-07-17**; `kridax/manifest` **2027-07-18** (1 pack, KX.1). Re-sign each
with its own `--app` id (`orgochat` / `swasth` / `kridax`).

OCR language packs (`ocr-*`) are served from this repo's `tessdata/` directory (ungzipped
`.traineddata`, ~2-5MB each; source: the tesseract.js data packages, `4.0.0_best_int`
variant, gunzipped). Adding a language = drop the ungzipped file in `tessdata/`, add a
pack entry, re-sign. All other weights stay on Hugging Face (fetched locally only to
fingerprint).

```bash
# Fetch the weights so they can be fingerprinted (not stored in this repo)
mkdir -p weights
BASE=https://huggingface.co/Xenova/LaMini-Flan-T5-77M/resolve/main/onnx
curl -sL -o weights/encoder_model_quantized.onnx        $BASE/encoder_model_quantized.onnx
curl -sL -o weights/decoder_model_merged_quantized.onnx $BASE/decoder_model_merged_quantized.onnx

# Sign (KEYS.txt holds the keypair; keep it out of this repo)
TOOL=<orgochat>/node_modules/@laxhartech/olai/scripts/olai-registry.mjs
OLAI_SIGNING_KEY=$(grep -v '^#' KEYS.txt | tail -1) \
  node "$TOOL" sign packs.json --app orgochat --key-id orgoai-2026 --ttl 31536000 > manifest

git commit -am "re-sign manifest" && git push
```

`key-id` must keep matching `NEXT_PUBLIC_AI_SIGNING_KEY_ID` in OrgoChat's environment.

## Note on fingerprints

`sha256` in the manifest is **not** `sha256sum`. It is a chunked hash-list: the file is split
into 8 MiB chunks, each hashed, and the value is the hash of those hashes concatenated. Use the
tool — a hand-computed digest downloads fine and is then refused at verification.
