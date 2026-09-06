# Locally patched Qwen3.8 Flash Next MTP build

## Provenance

- Upstream project: [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp), MIT license.
- Experimental model support: [PR #28243](https://github.com/ggml-org/llama.cpp/pull/28243), locally based on `2c967293c2632bd0d09096406628a7e4baf97b88`.
- Local fork baseline: `8d457bfa859315be5493433a5baf3ad7fba6cc05`, which added the earlier benchmark documentation.
- This fork includes our local cache-handling patch and synthetic regression test. An equivalent production correction was already committed upstream as [`d1a92352`](https://github.com/ggml-org/llama.cpp/pull/28243/commits/d1a92352cbd417fd840b4e765c0b82f5fe3d1d89) on September 5, 2026. The upstream updated build was not tested here. This is not a claim of a novel upstream fix.
- Development, diagnosis and testing were AI-assisted with Codex. The public source preserves the upstream license and history. No upstream PR was submitted for this local publication.

## What changes

`common/speculative.cpp` previously treated `ctx_other == ctx_tgt` as proof that a draft shared the target's KV cache. A Qwen4exp shared draft can use that pointer to borrow embedding/output weights while retaining its own cache.

That mismatch skipped draft catch-up and reused a sequence position for later draft tokens, causing repeated `inconsistent sequence positions` and `llama_decode[1] returned -1` messages. The patch restricts the shared-KV path to `gemma4-assistant`, the actual cross-context KV-sharing architecture in this source revision. Qwen drafts retain independent catch-up and rollback.

No model weights, quantization files or trained MTP heads were changed. The tested target was Qwen3.8 Flash Next `UD-Q4_K_XL`; this is a runner fix, not a new model.

## Build on Apple Silicon

Requires the Apple command-line toolchain and CMake.

```sh
git clone --single-branch --branch codex/qwen38-mtp-pr28243 https://github.com/gelubodrug/llama.cpp-qwen38-mtp.git
cd llama.cpp-qwen38-mtp
cmake -S . -B build-metal \
  -DCMAKE_BUILD_TYPE=Release \
  -DGGML_METAL=ON \
  -DGGML_METAL_EMBED_LIBRARY=ON \
  -DLLAMA_BUILD_TESTS=ON
cmake --build build-metal --target llama-server test-llama-archs test-batch-alloc -j4
```

## Run

Replace the model paths with your own. Keep all four target shards together. Model files are not distributed in this repository.

```sh
caffeinate -i -s ./build-metal/bin/llama-server \
  -m '/path/to/Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf' \
  --model-draft '/path/to/mtp-Qwen3.8-Flash-Next-shared-Q4_K_M.gguf' \
  -ngl 99 -ngld 99 -c 131072 -fa on -np 1 \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  --reasoning on --reasoning-format deepseek \
  --host 127.0.0.1 --port 8081
```

`--spec-draft-n-max 2` is the draft-token limit, not two independent draft models. The real coding-agent run used this value. A newly compiled binary may show a different build identifier than the local binary used for the measurements, because the tested source changes were committed after the run.

## Validation

```sh
./build-metal/bin/test-llama-archs --test-mtp-shared -s 123
./build-metal/bin/test-batch-alloc
```

- The new CPU fixture in `tests/test-llama-archs.cpp` constructs a small Qwen4exp target and a stripped draft that borrows weights. It checks two prompt batches, two and three draft tokens, independent target/draft cache progression, and rollback/catch-up.
- Red/green validation: the fixture fails at draft-cache catch-up with the old condition; it passes with the local patch for both draft lengths.
- `test-batch-alloc`: 30 tests, 198 assertions, zero failures in local validation.
- A short real-model smoke generated 256 tokens with 141/228 draft tokens accepted and approximately 49.07 tok/s reported by the server. This capped probe is not the coding-agent run or a task-completion result.
- A separate read-only Pi audit completed 35 generation requests without the old cache/decode errors before it was interrupted by the operator. It produced no completed audit result and is excluded from the task-time comparison below.

The new fixture is an explicit test option, not an automatically registered CTest case. It tests cache progression, not bitwise output equivalence or broad backend correctness. No full upstream CI, CUDA validation or Gemma model runtime regression is claimed.

## CO_DE task observations

System: Apple M5 Max, 128 GB unified memory. Target: `UD-Q4_K_XL`. Context: 131072 tokens. MTP draft: shared `Q4_K_M`. Parallel slots: 1. Same prompt: "Make an entry point audit on this repo and report back."

These numbers were recorded from the operator's CO_DE screenshots. They describe two completed task runs, not the separate interrupted Pi run.

| Measurement | Stock llama.cpp, MTP off | Our locally patched build, MTP on |
| --- | ---: | ---: |
| Task stopwatch | 8:52.40 | 4:47.08 |
| UI median generation speed | 27.5 tok/s | 33.2 tok/s |
| Requests | 17 | 10 |

The observed duration is 46.1% lower and the displayed median is 20.7% higher. The runs followed different tool-call paths and the repository was under active development, not a frozen benchmark snapshot. They did not produce identical output workloads. These measurements do not isolate the causal contribution of MTP or establish a general speedup.

In the last visible MTP request, the server reported 32.41 tok/s and draft acceptance of 1490/2110 (70.6%). Those are last-request measurements, not averages for the whole task. The Settings card's separate 57 tok/s reading is not the run median and is not used here.

## Known limitations

- The startup memory-fitting probe may still emit `qwen4exp requires ctx_other to be set` and say it cannot measure the extra model. Actual joint loading can succeed afterward. This patch does not fix draft-memory estimation; memory headroom still matters.
- Performance and memory use depend on context, hardware, draft acceptance and workload. Two and three draft tokens passing the CPU cache fixture does not prove that every configuration preserves identical generated text.
- Private audit transcripts, repository source excerpts, local model files and raw local logs are intentionally not published.
- The earlier [IQ3 microbenchmark](QWEN38_MTP_BENCHMARK.md) used a different target quantization, prompt length and context. It must not be mixed with this Q4 task comparison.
