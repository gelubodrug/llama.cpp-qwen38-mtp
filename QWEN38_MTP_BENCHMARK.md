# Historical IQ3 microbenchmark - September 4, 2026

This document records an earlier test of upstream llama.cpp [PR #28243](https://github.com/ggml-org/llama.cpp/pull/28243) at `2c967293c2632bd0d09096406628a7e4baf97b88`. It does not describe the latest head of that PR or the locally patched Q4 audit published on September 6. See [the current patched-build notes](QWEN38_MTP_CACHE_FIX.md) for that separate test. This fork is experimental and is not an official llama.cpp release.

## Result

| Configuration | Generation runs | Mean |
| --- | --- | ---: |
| llama.cpp + MTP | 63.9, 63.8, 63.8 tok/s | **63.8 tok/s** |
| llama.cpp without MTP | 44.6, 44.2, 44.3 tok/s | **44.4 tok/s** |

The rounded headline comparison is **+43.7%**. Both configurations used the same target GGUF, prompt, sampling parameters, Metal offload, context, and output length. An instrumented short MTP run accepted 10 of 10 drafted tokens; acceptance varies with workload and should not be generalized from that one prompt.

## Test system and model

- Apple Silicon M5 Max
- 128 GB unified memory
- macOS arm64
- Target: `Qwen3.8-Flash-Next-UD-IQ3_XXS`
- Draft head: `mtp-Qwen3.8-Flash-Next-Q4_K_M.gguf`
- Context: 4096 tokens
- Output limit: 96 tokens
- Sampling: temperature 0, top-k 1
- Full Metal offload for target and draft
- PR commit tested: `2c967293c2632bd0d09096406628a7e4baf97b88`

## Build

```bash
cmake -S . -B build-metal \
  -DGGML_METAL=ON \
  -DGGML_METAL_EMBED_LIBRARY=ON \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-metal --config Release -j
```

## Run the server with MTP

This is a server launch example, not an exact replay of the microbenchmark: it uses 32768 context, while the recorded benchmark context above is 4096. The exact benchmark prompt is not included in these historical notes.

```bash
./build-metal/bin/llama-server \
  -m /path/to/Qwen3.8-Flash-Next-UD-IQ3_XXS-00001-of-00003.gguf \
  -md /path/to/mtp-Qwen3.8-Flash-Next-Q4_K_M.gguf \
  -ngl 999 \
  -ngld 999 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  -fa on \
  -c 32768 \
  --host 127.0.0.1 \
  --port 8082
```

The benchmark used `--spec-draft-n-max 2`. Larger values were not validated here. Re-test on your own prompts before treating the headline result as representative of production traffic.

## Provenance

- Upstream project: [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)
- Experimental implementation: [PR #28243](https://github.com/ggml-org/llama.cpp/pull/28243)
- This fork keeps the upstream license and commit history.
