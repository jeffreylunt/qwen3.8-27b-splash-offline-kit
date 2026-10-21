# Qwen3.8-27B Splash offline kit

A mirror of everything needed to bring up [Splash](https://github.com/incoai/splash)
with **Qwen3.8-27B** on a Mac that cannot reach `huggingface.co`, reassembled
from GitHub release assets.

Upstream sources (versions of record):

| Component | Source | SHA-256 |
| --- | --- | --- |
| DFlash2 draft (BF16) | [`incoai/Qwen3.8-27B-DFlash2`](https://huggingface.co/incoai/Qwen3.8-27B-DFlash2) → `model.safetensors` | `67fc76d68dc5a9415511a4f394ef744d67510cd20e93b37cc2cc7d28e4bab65c` |
| Target, 4-bit GGUF | [`unsloth/Qwen3.8-27B-GGUF`](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF) → `Qwen3.8-27B-UD-Q4_K_M.gguf` | `322e194ff79741c7baa497c240f677f54b201b0efab44ca8e50f122b39123482` |
| Vision projector | same repo → `mmproj-BF16.gguf` | `83ee4f4f205fa514161778c41df1ea14144faa0f713510893b63c2395f5c2d53` |

GitHub caps a release asset at 2 GiB, so the two large files are split into
1,992,294,400-byte parts (last part shorter). `config.json` for both models
lives in this repository at the paths shown below.

## 1. Get the parts

```sh
mkdir -p ~/splash-kit/Qwen3.8-27B-DFlash2 ~/splash-kit/Qwen3.8-27B-GGUF
cd ~/splash-kit
gh release download v1 --repo jeffreylunt/qwen3.8-27b-splash-offline-kit
```

No `gh`? Every file is also a plain URL:

    https://github.com/jeffreylunt/qwen3.8-27b-splash-offline-kit/releases/download/v1/<part-name>

## 2. Reassemble and verify

```sh
cd ~/splash-kit
cat DFlash2-model.safetensors.part-* > Qwen3.8-27B-DFlash2/model.safetensors
cat Qwen3.8-27B-UD-Q4_K_M.gguf.part-*  > Qwen3.8-27B-GGUF/Qwen3.8-27B-UD-Q4_K_M.gguf
mv mmproj-BF16.gguf Qwen3.8-27B-GGUF/

# config.json for each model (from the repo, not the release)
curl -fsSL -o Qwen3.8-27B-DFlash2/config.json \
  https://raw.githubusercontent.com/jeffreylunt/qwen3.8-27b-splash-offline-kit/main/Qwen3.8-27B-DFlash2/config.json
curl -fsSL -o Qwen3.8-27B-GGUF/config.json \
  https://raw.githubusercontent.com/jeffreylunt/qwen3.8-27b-splash-offline-kit/main/Qwen3.8-27B-GGUF/config.json

shasum -a 256 Qwen3.8-27B-DFlash2/model.safetensors \
              Qwen3.8-27B-GGUF/Qwen3.8-27B-UD-Q4_K_M.gguf \
              Qwen3.8-27B-GGUF/mmproj-BF16.gguf
```

The three hashes must match the table above. Do not continue if they do not.

## 3. Install Splash (Homebrew, from GitHub — no Hugging Face involved)

```sh
brew install incoai/tap/splash
```

Apple silicon M3 or newer, macOS 26.4+, 36 GB unified memory (48 GB+ recommended).

## 4. Prepare the model offline

`splash serve --model` takes Hub repo IDs, so local files go through the
installer's public entry point (`splash serve` would otherwise ask the Hub):

```sh
ROOT="$(brew --prefix splash)/libexec"
cd ~/splash-kit
PYTHONPATH="$ROOT" "$ROOT/python/bin/python3" -P -c '
from install.local import prepare_local_model
print(prepare_local_model(
    "Qwen3.8-27B-GGUF/Qwen3.8-27B-UD-Q4_K_M.gguf",
    draft="Qwen3.8-27B-DFlash2",
    mmproj="Qwen3.8-27B-GGUF/mmproj-BF16.gguf",
))'
```

It prints the prepared model directory. For text only, drop `mmproj=` and add
`language_only=True`.

## 5. Serve

```sh
ROOT="$(brew --prefix splash)/libexec"
PYTHONPATH="$ROOT" "$ROOT/python/bin/python3" -P -m server.server \
    "<prepared directory from step 4>" \
    --model unsloth/Qwen3.8-27B-GGUF --binary "$ROOT/engine/splash" --port 8000
```

Open <http://127.0.0.1:8000>, or `splash opencode` / `splash hermes` / `splash claude`.

## Notes

- The draft directory must hold `config.json` and the BF16 `model.safetensors`
  together — that is the layout above.
- Splash refuses whole-BF16/F16/F32 GGUFs; `UD-Q4_K_M` is a quant Splash's own
  examples use.
- If the Mac can reach the Hub even once, none of this is needed:
  `splash serve --model unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M` downloads the same
  files itself.
- Upstream engine docs: <https://github.com/incoai/splash> — DEVELOPMENT.md,
  sections "Local model files" and "Drafts".
