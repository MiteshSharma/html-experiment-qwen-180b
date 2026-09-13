# The prompts

Every prompt sent to the model, verbatim. Nothing is paraphrased or trimmed.

There was **no system prompt** in any run. Thinking mode was **off** in all three.

---

## 1 — MARGINALIA

The open-ended one. The prompt says nothing about typography, almanacs, colour or
layout. Every creative decision in the resulting page is the model's own.

```text
Create the most beautiful HTML file you could think of. Something that when people see it, all they can say is WOW!!! It must be visually impressive, beautiful, and just music to the eyes. Be thorough, but creative. Don't make mistakes.
```

```json
{
  "model": "qwen3.8-flash-next",
  "chat_template_kwargs": { "enable_thinking": false },
  "max_tokens": 32000,
  "temperature": 0.85,
  "top_p": 0.95,
  "stream": true
}
```

→ [`pages/marginalia.html`](pages/marginalia.html) · [raw response](raw/marginalia.response.txt)

---

## 2 — Spec sheet

A constrained brief with measured data supplied. Tests whether the model can hold
real numbers and draw charts to scale.

```text
Create a single self-contained HTML file: a visually striking one-page "spec sheet" for an AI inference deployment.

Subject: Qwen3.8-Flash-Next (NVFP4) running on a single NVIDIA DGX Spark.

Real measured data to feature:
- Model: 512 experts, 10 active per token, 48 layers (36 linear-attention + 12 full-attention), 262,144 token context
- Checkpoint: 123.53 GiB total, of which 47.68 GiB is an FP8 n-gram PLE table served from NVMe, not resident
- Hardware: GB10 Blackwell, 121.7 GiB unified memory, 20-core Grace CPU
- Weights + non-torch resident: 78.16 GiB
- GPU KV cache: 838,860 tokens = 3.20x concurrency at full context
- Speculative decoding: MTP depth 3, 47,149-token reduced draft vocabulary
- Measured decode: 45.0 tok/s code, 33.5 tok/s prose, TTFT 0.23-0.34 s
- Prefill: 1,139 tok/s on a 10,665-token cold prompt
- Load time: 14.5 minutes to healthy

HARD REQUIREMENTS:
- ONE file. No external requests, no CDN, no web fonts. All CSS inline in one <style> tag.
- NO JavaScript at all. Pure HTML + CSS + inline SVG.
- Dark theme, refined typography using system font stacks, strong visual hierarchy.
- Include a stat grid and at least two inline SVG charts drawn with static coordinates from the numbers above (e.g. a memory-breakdown bar and a throughput comparison).
- Keep it under 500 lines. Finish the document completely and close every tag.
- Output ONLY the HTML starting with <!DOCTYPE html>. No markdown fences, no commentary.
```

```json
{
  "model": "qwen3.8-flash-next",
  "chat_template_kwargs": { "enable_thinking": false },
  "max_tokens": 24000,
  "temperature": 0.6,
  "stream": true
}
```

→ [`pages/spec-sheet.html`](pages/spec-sheet.html)

**Note on the first attempt.** An earlier run of a near-identical prompt used
`max_tokens: 12000` and did *not* forbid JavaScript. It ran out of budget at
exactly 12,000 tokens and truncated mid-function. Adding "NO JavaScript" and
raising the cap produced a complete document. Both facts are worth knowing before
you judge a long-form generation.

---

## 3 — Sparse Giant

The longest brief. Same idea as #2 but with the full architecture and a capability
matrix, including one deliberately inconclusive result to see whether the model
would overclaim it. (It did not — see the README.)

```text
Create a single self-contained HTML page that explains a specific AI model to a technically literate reader. Everything below is REAL MEASURED DATA from this exact deployment. Use it faithfully; invent nothing, add no dates or version numbers.

MODEL: Qwen3.8-Flash-Next, NVFP4 quantization, arch Qwen4ExpForConditionalGeneration (model_type qwen4_exp), running on one NVIDIA DGX Spark (GB10 Blackwell, 121.7 GiB unified memory, 20-core Grace CPU, NVMe).

ARCHITECTURE
- 512 experts, 10 active per token (1.95% fire), plus a shared expert; moe_intermediate_size 640
- 48 layers: 36 linear-attention + 12 full-attention (every 4th layer)
- hidden 2560, 24 attention heads, head_dim 256, 2 KV heads, vocab 248,320
- Context 262,144 tokens native
- PLE (per-layer n-gram embeddings): ngram_size 3, base n-gram vocab 20,000,000, 8 heads per n-gram, stored in 128 shards. Each token reads only 16 rows.
- Vision tower: 27 layers, patch 16, output hidden 2560
- MTP draft head: 1 hybrid layer with its OWN 512-expert MoE

DISK (123.53 GiB of tensor data)
- 10 NVFP4 body shards: 73.56 GiB
- model-fp8-mtp-ple.safetensors: 50.03 GiB, which is PLE 47.68 GiB + MTP 2.51 GiB
- tokenizer, index and configs: about 0.05 GiB

MEMORY (121.7 GiB unified; GPU budget 91.69 GiB at utilization 0.7535)
- Weights + non-torch RESIDENT: 78.16 GiB
- KV cache (FP8 e4m3): 12.29 GiB = 838,860 tokens = 3.20x concurrency at full context
- Peak activation: 1.24 GiB
- CUDA graphs: about 0
- Host reserve: 30 GiB
- NOT RESIDENT: the 47.68 GiB FP8 PLE table stays on NVMe and is read 16 rows at a time per token

QUANTIZATION (NVIDIA ModelOpt, MIXED_PRECISION, group size 16) - counted from the checkpoint index
- 73,728 weight tensors NVFP4 4-bit = 96.3%  (that is 512 experts x 48 layers x 3 projections)
- 1,536 weight tensors FP8 block-scaled = 2.0%, ALL of them in the MTP draft head (512 experts x 3 projections)
- 1,319 weight tensors BF16 unquantized = 1.7%: attention, norms, embeddings, vision tower
- The PLE table is FP8, not NVFP4
- KV cache FP8 e4m3; the recurrent (Mamba/GDN) state stays BF16

CAPABILITIES - each one tested live on this box today
- Text generation: YES
- Code generation: YES, 43.9 tok/s
- Tool / function calling: YES, verified - returned a proper tool_calls array with finish_reason tool_calls and no prose leakage
- Image input: YES, verified - correctly described a synthetic 96x96 PNG, 92 prompt tokens
- Video input: ACCEPTED, 52 prompt tokens - but the test was inconclusive because the test clip was malformed
- Reasoning / thinking mode: YES, toggled per request, 62 reasoning tokens on a trick question, answered correctly
- Image generation: NO. Video generation: NO. Output is text only - this is an image-text-to-text model.

MEASURED THROUGHPUT (single stream, greedy, thinking off, 320 tokens)
- math 49.9, json 47.4, code 43.9, sql 38.3, prose 31.2 tok/s. Median 43.9 tok/s.
- Time to first token 0.215 s to 0.887 s
- Concurrency: x1 46.2 per-stream; x2 85.7 aggregate; x4 158.9 aggregate; x6 171.1 aggregate
- Prefill: 1,139 tok/s on a cold 10,665-token prompt
- Sustained long generation: 46.1 tok/s over 9,463 tokens
- Speculative decoding: MTP depth 3, reduced draft vocabulary of 47,149 tokens

THE SUPERPOWER - make this the thesis of the page
It fits a 512-expert mixture-of-experts model with a 262,144-token context onto ONE 121.7 GiB machine, by refusing to hold its largest tensor in memory. The 47.68 GiB n-gram embedding table lives on NVMe and is touched 16 rows at a time. Only 10 of 512 experts fire per token. 36 of 48 layers use linear attention, so 838,860 KV tokens compress into just 12.29 GiB. Sparsity in three dimensions at once - experts, attention, and embeddings - is what makes a model this large behave like a small one.

HARD REQUIREMENTS
- ONE file. No external requests, no CDN, no web fonts. All CSS in one <style> tag.
- NO JavaScript at all. Pure HTML + CSS + inline SVG.
- Dark theme, system font stacks, strong typographic hierarchy.
- Include: a memory-vs-disk visual that makes the resident/not-resident split obvious; a capability matrix that clearly separates what it CAN do from what it CANNOT; a throughput chart; a quantization breakdown.
- Draw SVG charts to a consistent scale with static coordinates. Every bar must be proportional to its real value.
- Finish the document completely and close every tag. Aim for about 450 lines.
- Output ONLY the HTML starting with <!DOCTYPE html>. No markdown fences, no commentary.
```

```json
{
  "model": "qwen3.8-flash-next",
  "chat_template_kwargs": { "enable_thinking": false },
  "max_tokens": 28000,
  "temperature": 0.6,
  "stream": true
}
```

→ [`pages/sparse-giant.html`](pages/sparse-giant.html) — **renders blank as written**, see the README.
