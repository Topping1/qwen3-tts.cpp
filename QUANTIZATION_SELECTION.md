# Quantization Selection Guide

This document describes all of the mechanisms provided by the
`qwen3-tts.cpp` repository for selecting which quantized model artifacts are
used during setup and runtime.  The goal is to make it trivial to generate or
load *any* supported quantization type, and to understand how the system
chooses a file when multiple versions are present.

---

## Supported Quantization Types

- **TTS model** (`qwen3-tts-0.6b` stem):
  - `f16`  (no quantization)
  - `f32`  (float32; larger than f16)
  - `q8_0` (8-bit GGUF quantization)
  - `q4_k` (4-bit GGUF quantization; recommended for smaller size)

- **Tokenizer/vocoder model** (`qwen3-tts-tokenizer` stem):
  - `f16`
  - `f32`
  - `q8_0`  (*q4_k is not supported by the converter*)

These types correspond to the `--type` argument of the conversion scripts
`convert_tts_to_gguf.py` and `convert_tokenizer_to_gguf.py`.


## Setup Script (`scripts/setup_pipeline_models.py`)

The one‑shot model setup script can now produce any of the above quantized
outputs.  Two new CLI options were added:

```bash
--tts-type {f16,f32,q8_0,q4_k}          # default: q4_k
--tokenizer-type {f16,f32,q8_0}          # default: q8_0
```

When you run the script, it names the resulting GGUF files accordingly:

- `models/qwen3-tts-0.6b-<tts-type>.gguf`
- `models/qwen3-tts-tokenizer-<tokenizer-type>.gguf`

This file naming is the convention that the runtime loader uses to pick models.

Example:

```bash
python scripts/setup_pipeline_models.py \
    --tts-type q4_k --tokenizer-type q8_0
```

`--skip-download` and `--force` still work exactly as before.  The readme
documentation has been updated with the new flags and examples.


## Runtime Model Discovery and Explicit Selection

`src/qwen3_tts.cpp` contains the logic for locating GGUF files when the CLI is
called with `-m <model_dir>`. Earlier versions hard‑coded
`qwen3-tts-0.6b-f16.gguf` and
`qwen3-tts-tokenizer-f16.gguf`; this has been replaced with
`find_model_file()`:

```cpp
static std::string find_model_file(const std::string &model_dir,
                                   const std::string &stem) {
    static const char *preferred[] = {"f16", "q8_0", "q4_k", "f32"};
    // look for stem-type.gguf in preferred order, otherwise return first match
}
```

This function returns the first existing file matching `stem-*.gguf` using a
preferred type ordering.  By default `f16` wins, so if *only* a `q4_k` file is
present it will still be selected, but if both `f16` and `q4_k` exist the
script keeps picking `f16` unless instructed otherwise.

To give callers complete control, the model‑loading API was extended:

```cpp
bool Qwen3TTS::load_models(const std::string &model_dir,
                            const std::string &tts_model_path = {},
                            const std::string &tokenizer_model_path = {});
```

If either override argument is non‑empty it is used verbatim (absolute paths are
accepted; relative paths are joined with `model_dir`).  These parameters are
exposed via two new CLI options:

```
--tts-model <file>        # path to a GGUF, overriding discovery
--tokenizer-model <file>  # same for tokenizer/vocoder
```

Examples:

```bash
./build/qwen3-tts-cli -m models --tts-model qwen3-tts-0.6b-q4_k.gguf \
    --tokenizer-model qwen3-tts-tokenizer-q8_0.gguf -t "Hello" -o out.wav
```

The help text and README explain both the automatic and explicit modes.


## Discoverability Rules Summary

1. If explicit paths are given on the CLI, they are used.
2. Otherwise, `find_model_file()` searches `model_dir` for
   `stem-*.gguf`:
   - Preferred order is `f16`, `q8_0`, `q4_k`, `f32`.
   - If none of the preferred types exist but any file matches `stem-*.gguf`,
     the lexicographically first match is returned.
   - If no file is found an error is reported and loading fails.

This allows mixed directories such as:

```
models/
  qwen3-tts-0.6b-f16.gguf
  qwen3-tts-0.6b-q4_k.gguf
  qwen3-tts-tokenizer-q8_0.gguf
```

…to behave predictably while still enabling explicit selection when desired.


## Practical Advice

- **For size-conscious deployments**, run the setup script with `--tts-type
  q4_k` and `--tokenizer-type q8_0`.  The resulting files are about 1.8 GB and
  273 MB respectively (after mixed-precision fallbacks).
- **For development or debugging**, using `f16` models avoids quantization
  variability and is the default when running `./build/qwen3-tts-cli` without
  any tuning.
- **When multiple quantizations coexist** (e.g. you’re evaluating trade‑offs),
  add the `--tts-model`/`--tokenizer-model` flags to pick the desired pair
  explicitly.
- The `find_model_file()` preferred type order can be reversed or modified in
  code if you decide another default preference makes more sense in the future.


## Notes on Conversion Failures

During quantization the converter may warn that certain tensors could not be
quantized (e.g. "Can't quantize tensor with shape … to Q8_0, falling back to
F16").  These warnings are informational; the converter still writes a valid
GGUF file but the particular tensor stays in higher precision.  This behavior
is orthogonal to model selection and is a property of the converter itself.

If you need a stricter conversion policy, you could modify the converter to
error out on fallback or to print a summary of how many tensors were actually
quantized versus kept in F16.


## Repository Notes

The symbolic rules and CLI options are now documented in this file so future
contributors understand the design.  The details are also mentioned in
`/memories/repo/model-artifacts.md` to assist long-lived memory-based tools.

---

Any changes to supported quantization types or file naming should be reflected
both here and in the C++ discovery logic.
