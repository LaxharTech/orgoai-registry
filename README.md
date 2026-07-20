# OrgoAI model registry

The signed catalogs of on-device AI models for the Laxhar apps. One catalog per app,
each signed with the same publisher key but bound to its app id (`--app`), so an app
refuses every other app's manifest:

| App      | Catalog              | Fetched from                                                              |
|----------|----------------------|---------------------------------------------------------------------------|
| OrgoChat | `packs.json` (root)  | `https://raw.githubusercontent.com/LaxharTech/orgoai-registry/main/manifest` |
| Swasth   | `swasth/packs.json`  | `.../main/swasth/manifest`                                                |
| Kridax   | `kridax/packs.json`  | `.../main/kridax/manifest`                                                |
| Sentorg  | `sentorg/packs.json` | `.../main/sentorg/manifest` (OL.6.2: core-text 77M web pack, browser tier) |                                                |

Subdirectory catalogs reference the shared root `weights/` via `path: '../weights/...'`.

It is an Ed25519-signed list of the models users are allowed to download. The signature is
verified in the browser against `NEXT_PUBLIC_AI_SIGNING_PUBLIC_KEY`, fail-closed — a manifest
that doesn't verify yields no models, and there is no unsigned fallback.

Nothing here is secret. The weights themselves are public bytes hosted by Hugging Face; this
repo only says which ones are blessed, and proves the list came from us.

## Re-signing

The manifest carries a TTL. Once it lapses the catalog is treated as stale and quietly empties,
so it must be re-signed before then. Current expiries: root (OrgoChat) **2027-07-19** (19 packs, OL.4: adds the `text-desktop-7b` rung);
`swasth/manifest` **2027-07-17** (18 packs; the 7B rung is deliberately NOT here — see below); `kridax/manifest` **2027-07-19** (14 packs, KX.4; MT + classify minRamMb lowered to 2048 in this catalog only). Re-sign each
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

## The desktop ladder (OL.4)

The root catalogue carries two native rungs, and the app offers whichever the machine can
actually run:

| Pack              | Model             | Size    | Declares                                     |
|-------------------|-------------------|---------|----------------------------------------------|
| `text-desktop`    | Qwen2.5-1.5B Q4_K_M | 1.1 GB | `minRamMb: 6144`                             |
| `text-desktop-7b` | Qwen2.5-7B Q4_K_M | 4.7 GB  | `minRamMb: 8192`, `minCpuCores: 6`, `deviceClass: desktop` |

`minRamMb: 8192` is measured, not guessed: `scorePackFit` read the 7B's GGUF header straight
from its URL and resolved a 6,602 MB effective cost at a 32k context, so 8 GB is that with
headroom.

**`minVramMb` is deliberately absent.** The published device-class table puts a 7B at 8 GB of
video memory, but the model scored a perfect 1.0 on a test machine with *no dedicated video
memory at all* — an integrated AMD part with plenty of system RAM. Declaring `minVramMb` would
hard-block hardware measured to run it, so the declaration follows the measurement.

The 7B is a **single-file** build (bartowski's) rather than Qwen's own, which ships Q4_K_M as a
two-part split. The desktop host resolves one `modelFile` per record, so a split GGUF would
need multi-file records and per-part hash verification — work this rung does not require.

**Swasth does not carry the 7B rung yet.** It sits on olai `0.13.0` and still selects its native
pack with `catalog.find((p) => p.runtime === 'native')` and pins generation to the id
`text-desktop`, so a second rung would be shown arbitrarily and, if installed, would read as no
desktop model at all. Tracked as OL.4.4: bump olai, port the gating fixes, then add and re-sign.
