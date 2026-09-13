# What a 180B model builds when you ask it for something beautiful

Three HTML pages written by **Qwen3.8-Flash-Next** running locally on a single
**NVIDIA DGX Spark**. No cloud, no API key, no editing of the design.

This repo exists to show the raw material: the exact prompts, the exact output,
the throughput it ran at, and an honest list of what it got wrong.

The headline one is [`pages/marginalia.html`](pages/marginalia.html) — an invented
typography almanac. The prompt never mentioned typography, or almanacs, or a
colour palette. It picked all of that itself.

---

## The setup

| | |
|---|---|
| Model | [`nvidia/Qwen3.8-Flash-Next-NVFP4`](https://huggingface.co/nvidia/Qwen3.8-Flash-Next-NVFP4) |
| Parameters | **180.0 B total, 7.31 B active per token** (512 experts, 10 fire) |
| Architecture | `Qwen4ExpForConditionalGeneration` — 48 layers, 36 linear-attention + 12 full-attention |
| Context | 262,144 tokens native |
| Hardware | 1× DGX Spark, GB10 Blackwell, 121.7 GiB unified memory, 20-core Grace CPU |
| Server | vLLM nightly `8a728663`, tensor-parallel 1 |
| Quantization | NVFP4 (96.3% of weight tensors), FP8 block-scaled MTP head, BF16 elsewhere |
| KV cache | FP8 e4m3 — 838,860 tokens in 12.29 GiB |
| Speculative decoding | MTP depth 3, 47,149-token reduced draft vocabulary |

The whole checkpoint is 123.53 GiB of tensor data on a machine with 121.7 GiB of
memory. It fits because the 47.68 GiB FP8 n-gram embedding table never enters
memory — it stays on NVMe and each token reads 16 rows of it.

---

## The three pages

| Page | Tokens | tok/s | TTFT | Wall clock | Lines |
|---|---|---|---|---|---|
| [MARGINALIA](pages/marginalia.html) | 16,991 | **42.81** | 0.222 s | 6 m 37 s | 863 |
| [Sparse Giant](pages/sparse-giant.html) | 14,383 | **45.41** | 1.61 s | 5 m 18 s | 592 |
| [Spec sheet](pages/spec-sheet.html) | 9,463 | **46.09** | 0.623 s | 3 m 26 s | 398 |

All three: greedy-ish sampling, thinking mode off, single stream, no system prompt.
Full prompts and sampling parameters are in [`prompt.md`](prompt.md).

### MARGINALIA — the open brief

One sentence asking for something beautiful. It returned a fictional biannual
journal about printed type: cover, manifesto, contents with page numbers, five
type specimens, a letterform anatomy diagram, a photo gallery, a colophon and a
subscribe form. It wrote all the copy. *"Before the eye can read, the eye feels."*

Fraunces + Space Grotesk + Spectral, ink-and-gold palette, film-grain overlay,
scramble-decode masthead, Ken Burns cover, line-mask reveals, custom cursor.

The JavaScript is defensively written: a `prefers-reduced-motion` branch that
reveals everything immediately, an `IntersectionObserver`-missing fallback,
rAF-throttled scroll, passive listeners, and a cursor gated behind `pointer:fine`.

### Sparse Giant & Spec sheet — the constrained briefs

Both were handed real measured numbers and asked to visualise them. Both drew
their SVG charts to correct scale — the bar arithmetic checks out. The Sparse
Giant page also kept a deliberately awkward result honest: video input was marked
**"ACCEPTED — inconclusive"** rather than upgraded to a pass.

---

## Viewing them

```bash
git clone https://github.com/<you>/html-experiment-qwen-180b
cd html-experiment-qwen-180b
open pages/marginalia.html        # macOS.  Linux: xdg-open
```

Two directories:

- **`pages/`** — the model's output unedited. `spec-sheet.html` and `sparse-giant.html`
  are byte-for-byte the API response. `marginalia.html` is the response with only the
  surrounding prose and the ```` ```html ```` fence removed — the full response is in `raw/`.
  One of these three renders blank; that is the point, see below.
- **`pages-fixed/`** — minimal repairs, each one listed below. Nothing about the design, copy or layout was touched.
- **`raw/`** — the complete API response for MARGINALIA, including the prose it wrapped around the code.

---

## What it got wrong

This is the interesting half. The model produces genuinely strong design and
correct chart arithmetic. Its failures cluster in one place: **claims about its
own output.** It never renders what it writes, so it cannot check.

**1. Sparse Giant renders completely blank.**

```css
.reveal    { opacity: 0; }        /* every section on the page */
.reveal.in { opacity: 1; }        /* .in is only ever added by JavaScript */
.grow      { transform: scaleY(0); }   /* every chart bar */
```

The brief said no JavaScript. It complied — then wrote reveal animations that
*require* JavaScript to add a `.in` class. Nothing ever adds it. It also left a
note in the file stating *"The reveal animations are driven purely by CSS
transitions"* — but a transition with no state change never fires. It reasoned
about the constraint and still shipped the contradiction.

MARGINALIA, generated one prompt later, uses the same `data-reveal` pattern and
writes the `IntersectionObserver` to drive it. Same model, same session.

**2. MARGINALIA claims to run offline.** Its preamble says *"Everything is
original and runs offline."* The page hot-links 8 images from `picsum.photos` and
three font families from Google Fonts.

**3. Garbled CSS token.** Sparse Giant emitted `background:#3a4considered;`
mid-declaration. Harmless — browsers drop invalid declarations and the next one
wins — but it is a real defect.

**4. Duplicate attributes.** MARGINALIA has `style="--r:2deg" data-reveal style="--d:1"`
on four elements. Browsers keep the first `style` and discard the second, so some
stagger delays silently never apply.

**5. Invented a date.** The spec sheet stamped itself "Rev 2025.10". Nothing in
the brief supplied a date.

---

## Every change made to `pages-fixed/`

Nothing else was altered. No design, copy, colour, layout or chart values.

| File | Change |
|---|---|
| `sparse-giant.html` | `.reveal{opacity:0}` → `1`, `.grow{transform:scaleY(0)}` → `none`, so the page renders |
| `sparse-giant.html` | removed the garbled `background:#3a4considered` |
| `sparse-giant.html` | added two Hugging Face links in the footer |
| `marginalia.html` | the 8 `picsum.photos` images downloaded and inlined as data URIs — the *same* images it chose, embedded rather than fetched |
| all three | stripped the `<html>/<head>/<body>` wrapper so the pages embed cleanly (`<title>`, `<style>`, `<link>` and all body content kept verbatim) |

`pages-fixed/spec-sheet.html` received nothing but the wrapper strip — it needed no repairs.

---

## Reproducing it

Serving recipe: [madeye/qwen38-flash-next-on-dgx-spark](https://github.com/madeye/qwen38-flash-next-on-dgx-spark),
which vendors work from [blazux](https://github.com/blazux/qwen3.8-Flash-DGX) and
[MiaAI-Lab](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Next-Single-DGX-Spark). The
TP1 recipe originates at
[tonyd2wild](https://github.com/tonyd2wild/Qwen3.8-Flash-Next-NVFP4-DGX-Spark).

It is **stock upstream vLLM** — no fork. The NVMe PLE offload works by
bind-mounting patched Python files read-only over the container's `site-packages`
at runtime, switched on with `QWEN4EXP_PLE_MMAP=1 QWEN4EXP_PLE_STAGED=1`.

One trap: `download-weights.sh` sets `HF_HUB_DISABLE_XET=1`, and the 53.7 GB PLE
file exceeds the non-Xet size limit. `hf download` dies at 72% with *"file is too
large to be downloaded using the regular download method."* Fetch that one file
with `curl -C -` instead; the server supports range requests.

---

## Licence

The pages are model output, published as generated. The `picsum.photos` images in
`pages-fixed/marginalia.html` are from [Lorem Picsum](https://picsum.photos/).
The model checkpoint carries the `nvidia-open-model-license`.
