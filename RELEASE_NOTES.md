# Leviathan 2.0.0

Leviathan 2.0.0 is mostly the engine. Integrated graphics work now. A mixture-of-experts model whose experts don't fit your memory streams them from an NVMe drive, and quantised mixture-of-experts models that fit the card write about eight times as fast as in 1.1.0. On a 7B model that fits the card, Leviathan now writes as fast as llama.cpp at a short context, and Studio → Convert makes Stage 10, a new lossy `.shn5` level that makes big models much smaller. Faster replies no longer need a helper model: Leviathan can guess ahead from the conversation itself, and a new switch makes streamed mixture-of-experts models write about a quarter faster, for slightly different answers. Install it over 1.1.0: your models, memories and knowledge banks stay where they are.

Every figure below was measured for this release on one PC: an RTX 4080 (with the NVIDIA app's Automatic Tuning on: +61 MHz core, +200 MHz memory), a Ryzen 9 7950X with 128 GB of DDR5 running at 3,600 MT/s, and a PCIe 5.0 NVMe drive. Where 2.0.0 is compared with 1.1.0, 1.1.0 ran from its released build and 2.0.0 from the code this release ships, timed the same way; every comparison with llama.cpp (build 8952) ran on the same PC in the same session.

## Integrated graphics work

- **Leviathan runs models on integrated graphics now.** 1.1.0 left them out because the AMD Radeon integrated GPU we test on gave wrong results after as little as a millisecond of idle. The Vulkan engine now wakes an integrated GPU with a short warm-up whenever it has been idle, and the fault is gone: 0 wrong runs in 84 after pauses from 1 ms to 1.5 s (52 of 84 were wrong without it), for under 1 % of the speed (Llama 3.2 1B wrote at 49.0–49.3 ms a token with the wake, 48.9 without).
- **Leviathan picks it by itself** when there's no graphics card it can use. `LEVIATHAN_VK_IGPU=0` never uses integrated graphics; `=1` always allows them.
- Tested on the integrated Radeon of a Ryzen 9 7950X. Intel's integrated graphics haven't been tested yet.

## Models bigger than your memory: experts from an NVMe drive

**Mixture-of-experts models no longer need the RAM for every expert.** When the experts don't fit your free system memory, Leviathan reads them from the `.shn5` on an NVMe drive as they're needed, keeps the most-used ones in a RAM cache, and reads the next layers' likely experts ahead. The answers are exactly the same as with every expert in RAM: we check that bit for bit.

Qwen3-30B-A3B in BF16 on an RTX 4080 with a PCIe 5.0 NVMe drive, writing a 128-token reply (from the median time a token), with the same answer bit for bit at every setting:

| Experts held in RAM | Writing a reply | Loading |
|---|---|---|
| All of them (36 GB) | 16.4 tokens/s | 16.3 s |
| A 24 GB cache | 15.2 tokens/s | 5.8 s |
| A 16 GB cache | 14.2 tokens/s | 3.8 s |
| A 12 GB cache | 13.3 tokens/s | 3.4 s |
| An 8 GB cache | 10.4 tokens/s | 3.0 s |

Right after loading it is slower while the cache fills: in our chat test (three prompts, a short reply to each) a 12 GB cache wrote 7.5 tokens a second against 15.4 with every expert in RAM, and read a 700-token prompt at 252 tokens a second against 691 (llama.cpp with its experts in RAM, at the settings our test runs it with: 129). At these sizes it reads 0.1–0.7 GB from the drive for every token, so a slower drive gives less, and a SATA drive is refused (Leviathan says why). `LEVIATHAN_EXPERT_RAM_GB` sets the cache size yourself.

- **It remembers each model's most-used experts** and reads them in the background as the model loads. `LEVIATHAN_EXPERT_PROFILE=0` turns that off.

## A switch for speed: fewer experts a word

**New in Settings → Mixture-of-experts models: Fewer experts a word.** When a mixture-of-experts model streams its experts, each word it writes uses the best three-quarters of the experts it would normally pick, their weights rescaled, so less has to cross to the card for every word. It changes the answers a little, so it's off by default. Qwen3-30B-A3B in BF16 (6 of its 8 experts a word, 16 experts a layer on the card), Leviathan's engine on its own over the same 985 positions:

| | Writing | Sent to the card a word |
|---|---|---|
| All 8 experts | 18.8 tokens/s | 1.09 GB |
| **Fewer experts a word** | **23.6 tokens/s (+25 %)** | **0.71 GB** |

With a 12 GB NVMe cache it went from 11.1 to 13.9 tokens/s (+25 %). The cost: the next-token probabilities sit a KL of 0.02 from the full model's on average (0.65 at the worst position), and the top token is the same 95.8 % of the time. It takes effect from the next reply. A model that fits the card whole runs every expert as before, and a model that uses only two experts a word keeps both. `LEVIATHAN_MOE_TOPK` sets the number of experts yourself (`LEVIATHAN_MOE_TOPK_RESCALE=l1` for plain renormalising, which measured further from the full model: KL 0.03).

## Quantised mixture-of-experts models, about eight times as fast

When a quantised mixture-of-experts model fits on the card, its experts now run in grouped kernels, the router runs on the card, and the whole step of writing a token runs as one CUDA graph. Qwen3-30B-A3B in Q3_K_S on an RTX 4080, each version through its own runner on the same chat (three 160-token replies, three 511-token prompts):

| | Writing a reply | Reading a 511-token prompt |
|---|---|---|
| **Leviathan 2.0.0** | **177 tokens/s** | **604–617 tokens/s** |
| Leviathan 1.1.0 | 22.0 tokens/s | 58 tokens/s |

The answers stay the same: both versions wrote the same replies, and fed the same 49 tokens, the new path's next-token probabilities agree with the old one's to within 1.7e-11 (KL), with the same top token every time. llama.cpp is still ahead on this model, and closing that gap is the next job: in the same session its own benchmark (llama-bench, `-ngl 99 -fa 1`) wrote 214 tokens/s and read a 512-token prompt at 5,726–6,173 tokens/s, where Leviathan's engine on its own, without the app around it, wrote 181 and read 805. The memory planner also counts a model's real overhead now, so a quantised mixture-of-experts model that fits loads whole instead of streaming one expert slot short: on this model, about 180 tokens/s instead of 50.

## Models that fit your card

Qwen 2.5 Coder 7B Q8_0 on an RTX 4080, both programs in the same session (llama.cpp's own llama-bench, `-ngl 99 -fa 1`):

| Writing a reply, after a context of | 128 tokens | 2,048 tokens | 4,096 tokens |
|---|---|---|---|
| **Leviathan 2.0.0** | **83.1 tokens/s** | **78.4 tokens/s** | **75.9 tokens/s** |
| llama.cpp | 82.9 tokens/s | 81.0 tokens/s | 80.0 tokens/s |

- **Writing is level with llama.cpp at a short context**, while Leviathan keeps its activations in full fp32. At 4,096 tokens it is about 5 % behind, because Leviathan keeps the attention cache in fp32 too: twice llama.cpp's bytes to read.
- **Reading a prompt** is about half llama.cpp's speed by default (0.41–0.51× at 512 to 4,096 tokens: 4,175 / 3,888 / 3,546 tokens a second against 8,154 / 8,840 / 8,611), because Leviathan feeds the tensor cores fp16 where llama.cpp feeds int8. Two opt-in switches trade precision for speed, and with both on, prompts read at 1.04–1.21× llama.cpp's speed (9,894 / 9,516 / 8,932), at llama.cpp's own precision class:
  - `LEVIATHAN_INT8_GEMM=1`: int8 matrices for a Q8_0 model's prompt (NVIDIA RTX 30 series or newer);
  - `LEVIATHAN_ATTN_TC=1`: the prompt's attention on the tensor cores (RTX 30 series or newer).

  They cost accuracy: int8 matrices take every Q8_0 model we test past our reference bar (all five), and they change Gemma 4's top token, so don't use them with Gemma 4. On this model, at the end of a 2,048-token prompt, the next-token probabilities sit a KL of 1.7e-7 from the exact path by default, 4.5e-7 with the tensor-core attention and 8.4e-5 with both switches on. That's why both are off by default.
- **What changed:** a prompt's attention runs as one tiled kernel for every model, writing runs as fewer, fused kernels replayed as one CUDA graph, and the attention while writing keeps more of the card busy. Against 1.1.0, each through its own runner on the same chat, this model reads a 2,094-token prompt about twice as fast (2,851–2,857 tokens/s against 1,407) and writes 8–11 % faster (79–82 tokens/s against 74), with the same replies.

## Faster replies, with a helper model or without

- **New: no helper model needed.** Settings → Faster replies → **From the conversation itself (no model)** guesses the next words from text already in the chat, so it needs no download and no extra graphics memory, and works with any model family. Your model still checks every guess, so replies don't change: at temperature 0, every reply in our tests was identical to the one written without it. It pays off when a reply repeats what's already there. With Qwen 2.5 Coder 7B Q8_0, over six requests to edit code or text (add a docstring, rename a variable, fix a typo, …), replies came 1.62× as fast at temperature 0 (77.8 → 126.4 tokens/s) and 1.54× at the default temperature (74.2 → 114.6). On six everyday requests, where there's less to repeat, the gain is small (1.09× and 1.11×), and a helper model does more.
- **The helper model's check pass now runs on the same kernels as writing.** With Qwen 2.5 Coder 7B Q8_0 and the 0.5B helper, over those six everyday requests, replies came 1.87× as fast at temperature 0 (79.9 → 149.1 tokens/s, every reply identical to the one written without the helper) and 1.73× at the default temperature (78.5 → 135.7). The README has the figures for six file types.

## `.shn5` Stage 10: a lossy level

- **Stage 10 is a lossy `.shn5`.** Each weight matrix is quantised around a per-row and per-column scale grid, then entropy-coded; the norms, the router, the embeddings and the output head stay lossless.
- **Qwen3-30B-A3B:** 23.7 GB at about 6.1 bits a weight on its layers, against 40.5 GB for the lossless `.shn5` and 61.1 GB for the BF16 GGUF. With its experts streamed, the smaller file writes about 12 % faster (20.7 tokens/s against 18.5, in the same session), reads a 2,048-token prompt at 1,276 tokens/s against 1,111, and holds 20.6 GB of RAM for the experts instead of 35.8.
- **q3, the smallest level:** Qwen3-30B-A3B at q3 is a 12.2 GB file, against 13.3 GB for a Q3_K_S download of the same model, and the two land equally close to a float64 reference of the original: the same top token at 35 of the 36 positions checked, and a mean KL of 7.3e-3 against the Q3_K_S's 7.0e-3. With its experts streamed it writes about 20 % faster than the lossless file (22.8 tokens/s against 18.9, in the same session) and holds 10.3 GB of RAM for the experts instead of 35.8. The Q3_K_S still writes faster: Leviathan computes on its blocks as they are, while Stage 10 unpacks each expert to 16 bits on the card first. With both streaming their experts through 16 slots, the Q3_K_S writes 42.0 tokens/s against q3's 21.1 and uses 2.7 GB of VRAM against 10.4; q3 reads a 700-token prompt faster (959 tokens/s against 594).
- **Make one in Studio → Convert:** choose **→ .shn5 Stage 10** and a level (q8, q6, q5, q4 or q3), and each model's row shows the file it will make. It writes `<model>.stage10-<level>.shn5` beside any lossless `.shn5`, and it only shrinks F16, BF16 or F32 models: quantised ones are left out, with the reason. Add the file with **Models → 🗜 Add compressed model**.
- **Leviathan 2.0.0 runs Stage 10 files** on CUDA and on Vulkan, whole or with streamed experts, unpacking them on the card and refusing a damaged file.

**The levels, each beside the ggml format its error is set to.** TinyLlama 1.1B from its F16 (2.20 GB), every file run by Leviathan, teacher-forced over three prompts (312 positions):

| File | Size | Bits a weight on the layers | KL from the F16 | Same top token |
|---|---|---|---|---|
| **Stage 10 q8** | 1.13 GB | **7.82** | **1.5e-4** | 99.7 % |
| Q8_0 | 1.17 GB | 8.50 | 2.3e-4 | 99.7 % |
| **Stage 10 q6** | 0.92 GB | **6.10** | **1.7e-3** | 98.7 % |
| Q6_K | 0.90 GB | 6.56 | 2.7e-3 | 97.1 % |
| **Stage 10 q5** | 0.80 GB | **5.15** | **6.9e-3** | 96.5 % |
| Q5_K_M | 0.78 GB | 5.63 | 7.9e-3 | 96.2 % |
| **Stage 10 q4** | 0.69 GB | **4.19** | **2.6e-2** | 92.6 % |
| Q4_K_M | 0.67 GB | 4.76 | 3.1e-2 | 91.3 % |
| **Stage 10 q3** | 0.55 GB | **3.03** | **1.1e-1** | 86.9 % |
| Q3_K, every layer | 0.68 GB | 3.44 | 1.4e-1 | 82.4 % |

On average Stage 10 lands closer to the original at every level, with 7–12 % fewer bits on the layers. From q6 to q4 the whole file is a little bigger, because Stage 10 keeps the embeddings and the output head lossless. The Q3_K row keeps them at F16 too (it is llama.cpp's Q3_K on every layer, the error q3 is set to), so there the Stage 10 file is 20 % smaller. From q6 down, Stage 10's single worst position is worse than the ggml format's (at q4, a KL of 2.0 against Q4_K_M's 0.32). Three bits cost a small model far more than a big one, in either format, so q3 is for big models. On Qwen3-30B-A3B the q6 level lands further from the original than Q6_K (KL 2.6e-4 against 1.6e-4, with the same top token at every position checked) at 6.8 % fewer bits: each level's error is set on TinyLlama, and setting it per model is the next step. At q3 it matched the Q3_K_S download (above).

## Less memory, faster low-bit formats

- **Less VRAM for the same model.** A model's weights now sit in a few shared allocations instead of one per tensor: 380 MiB back on Qwen 2.5 Coder 7B Q8_0 and 113 MiB on Qwen3-30B-A3B Q3_K_S, with the same answers bit for bit, on CUDA and Vulkan. `LEVIATHAN_PACK_WEIGHTS=0` turns it off.
- **Low-bit GGUF formats write faster on NVIDIA cards.** The kernels that read quantised blocks while writing now run two rows a warp: on TinyLlama 1.1B a token takes 20–33 % less time in Q2_K (−20 %), Q4_0 (−26 %), IQ4_XS (−27 %) and IQ1_M (−33 %), with the same answers (the next-token probabilities agree with the old kernels to within 4.2e-12). Q5_K, Q5_0 and BF16 keep the old kernel, which is faster for them.

## Fixes

- **The first reply after loading a model starts as fast as the rest.** The first message used to wait while the reasoning layer loaded its examples (about 3.5 s on our PC) before the model even began. Leviathan now loads it, and ANI's embedder, in the background as soon as the model is ready: in our chat test the first reply landed about 4 s after Enter instead of about 9 (the model's own time: 3.1 s).
- **Unloading a streamed mixture-of-experts model now gives its memory back.** In 1.1.0 it kept the experts' pinned RAM (about 36 GB for Qwen3-30B-A3B in BF16) until Leviathan closed, and loading it again could fail. In our test, unloading now gives back 12.6 GB of available RAM for a 12.5 GB store of experts.

## Also new

- **Graph:** two models in one graph can be wired both ways with ▼ wires. They then run in lockstep, a token each in turn, each reading the other's latest value.
- **ANI Mind:** Ask ANI's choice for unproven answers gains **user**: ANI delivers what she can prove; what she can't, she doesn't pick, and gives you every mind's answer marked unverified. A graph model asked as a mind now runs only its model.

## Known limits

These are the next job: they will be fixed in upcoming releases.

- On AMD and Intel cards the prompt pass doesn't use their matrix cores yet, so long prompts read slower than they could.
- The faster quantised mixture-of-experts kernels and low-bit kernels are on NVIDIA cards so far; the Vulkan engine still runs those models the 1.1.0 way.
- llama.cpp is still faster on a quantised mixture-of-experts model that fits the card (Qwen3-30B-A3B Q3_K_S: 214 tokens/s writing against 181, and 5,726–6,173 tokens/s reading a 512-token prompt against 805), and at reading prompts on a dense model that fits the card unless you turn on the opt-in switches, which cost accuracy.
- Short prompts on a streamed mixture-of-experts model are bound by the link to the card (about 30 tokens a second on Qwen3-30B-A3B; llama.cpp with its experts in RAM 19–22).
- Streaming experts from a drive has been measured on a PCIe 5.0 NVMe drive only.
- Stage 10's levels were set on one small model, so on Qwen3-30B-A3B q6 lands further from the original than Q6_K, and from q6 down its single worst position is worse than the ggml format's.
- Stage 10 unpacks its layers to 16 bits on the card before computing with them, so it needs more VRAM than a ggml file of the same size, and where both stream their experts the ggml file writes faster (Qwen3-30B-A3B through 16 slots: Q3_K_S 42 tokens/s against q3's 21, Q6_K 23 against q6's 18). Stage 10's gain is disk space, RAM and traffic over the link to the card; computing on its format directly is the next step.
- Integrated graphics have been tested on AMD's Radeon only.
- Faster replies aren't used with mixture-of-experts models yet, with a helper model or without: there every word checked needs experts of its own, so checking guesses cost more than it saved (Qwen3-30B-A3B: 0.59–0.96× in our tests, guessing from the conversation).
- The installer isn't code-signed yet, so Windows SmartScreen may warn you: **More info → Run anyway**.

## Upgrading

Install over 1.1.0 or 1.0.0. Everything in `%LOCALAPPDATA%\Leviathan` is kept, and your `.lev` and `.shn5` files work unchanged.

---


# Leviathan 1.1.0

Leviathan 1.1.0 runs on AMD and Intel graphics cards as well as NVIDIA ones. It adds `.shn5`, a compressed model format that loses nothing, and it can run mixture-of-experts models that are bigger than your graphics card, faster than llama.cpp runs them on the same card. Install it over 1.0.0: your models, memories and knowledge banks stay where they are.

## AMD and Intel graphics cards, through Vulkan

- **A second engine, built on Vulkan, is in the download.** Leviathan picks the engine by itself: an NVIDIA card (GTX 16 / RTX 20 or newer) runs on CUDA, the faster path there, and an AMD or Intel graphics card runs on Vulkan. If CUDA is chosen but won't start, Leviathan falls back to Vulkan and says so.
- **The same answers on both.** In testing, the Vulkan engine matches the CUDA engine token for token on all fifteen models compared. Its newest kernels (for expert streaming, DeepSeek's attention and the `.shn5` decoder) give the same bits as CUDA's on the same card.
- **The whole engine runs on it**, `.shn5`, expert streaming and the helper model for faster replies included.
- **On the Vulkan engine, two graphics cards can share one model** too big for either: set the environment variable `LEVIATHAN_SPLIT=auto`.
- **Tested so far** on NVIDIA's own Vulkan driver and on an AMD Radeon. Intel Arc is next. If you run it on an AMD or Intel card, we would like to hear how it goes.

## `.shn5`: the same model, about 1.5× smaller

- **Lossless.** A `.shn5` gives exactly the same numbers as the `.lev` of the same model, bit for bit. Every piece is checked as it unpacks, so a damaged file is refused instead of run.
- **About 1.5× smaller** for full-precision models (measured 1.51× on BF16, 1.48× on F16). Quantised files (Q4_K_M, Q8_0, the IQ types and so on) stay the size they are, because their numbers are already packed.
- **No slower to load.** It unpacks on the graphics card at about 110 GB/s on an RTX 4080: a 2 GB model loaded in 0.70 s, against 0.77 s for its `.lev`.
- **Studio → Convert makes a `.shn5` by default** from any `.gguf`, showing each model's expected size first. Add it with **Models → 🗜 Add compressed model**.

## Models bigger than your graphics card: expert streaming

A mixture-of-experts (MoE) model splits each layer into many small experts, and each word uses only a few of them. Leviathan now keeps the experts in your computer's memory and brings to the card only the ones each word needs. It does this by itself whenever a MoE model doesn't fit, and the answers are exactly the same as if the model fitted (checked bit for bit).

**Faster than llama.cpp at this.** Qwen3-30B-A3B in BF16 (61 GB, or 40 GB as a `.shn5`) on one RTX 4080 with 16 GB, both programs keeping the experts in system memory:

| | Writing a reply | Reading a 700-token prompt | Reading a 2,048-token prompt |
|---|---|---|---|
| **Leviathan 1.1.0** (16 experts a layer on the card, 10.3 GiB of VRAM) | **16.2 tokens/s** | **683 tokens/s** | **1,234 tokens/s** |
| llama.cpp, CUDA build, `--cpu-moe` | 9.5–10.4 tokens/s | 247–365 tokens/s | 801 tokens/s |

llama.cpp's figures come from its own timer, on the same prompt tokens and the same PC. Its prompt figures use the fastest settings we found for it (`-b 2048 -ub 2048 --no-mmap -fa on`). Leviathan computes this in full fp32: on a 700-token document it comes within 6.5e-5 nats of a float64 reference of the model.

## Faster on DeepSeek, and Gemma 4

- **DeepSeek-V2-Lite's attention runs on the graphics card.** In 1.0.0 it ran on the CPU. A 600-token prompt now reads at 102 tokens a second (was 6), and writing at the end of it takes 20 ms a token (was 290), with the same answers.
- **Gemma 4 runs**, through its own engine path: sliding-window layers, its global layers and final soft-cap. Over a 3,367-token prompt it matches llama.cpp's CUDA build to within 0.033 nats.

## Fixes

- **MoE models read their prompts wrong in 1.0.0.** A mixture-of-experts model that is neither DeepSeek-style nor a hybrid recurrent model (Qwen3-MoE, Qwen2-MoE, Mixtral, OLMoE and the like) read every prompt with its experts skipped, so it answered from a wrongly read prompt. None of the models we had validated were affected, but 1.0.0 would load one. Fixed, and checked against llama.cpp.
- **The right chat format for every model through the runner.** Programs that use Leviathan through its runner process gave models outside four built-in formats (Mistral's `[INST]`, Gemma, Phi and others) a ChatML prompt. Those models now get their own template, as the chat window already did.
- **A graphics card that stops answering is now an error, not a reply.** After Windows reset the GPU, a model could go on "answering" from leftover memory. Now Leviathan says the GPU stopped answering and asks you to reload the model.
- **Unsupported models are always refused before they load.** A model whose metadata confused the blueprint reader skipped the check for tensors the engine doesn't read; it now runs every time.
- Non-square F32 weight matrices were read wrong in the prompt pass (rare: most F32 tensors are norms).
- The converter keeps `tokenizer.ggml.suppress_tokens`, and a model with no chat template of its own gets its blueprint's when it loads.
- The Bridge switch now loads models with an 8,192-token context (was 2,048).
- The bridge server no longer prints an error when a client hangs up.
- Fabricator and Planner checks could stall for up to a minute. Fixed.

## Also new

- **Graph:** drop a second model's weight folder onto a model graph to show both side by side. ▶ Run runs each model exactly as it would run alone, and a Mind node can name either model. The canvas zooms out to 10 %.
- **Fabricator:** a **Check colours** switch (off shows the part in one plain colour).
- The Tensor Scan blueprint export carries the model's chat template.

## Known limits

These are the next job: they will be fixed in upcoming releases.

- **Integrated graphics aren't used yet.** The AMD Radeon integrated GPU we test on gives wrong results after even a few milliseconds of idle (5 ms spoiled 8 runs in 12; 20 ms or more spoiled every one), so Leviathan won't run a model on integrated graphics until a guard against it is in. A PC with integrated graphics only can still use Leviathan with models served by Ollama or an OpenAI-compatible server.
- On AMD and Intel cards the prompt pass doesn't use their matrix cores yet, so long prompts read slower than they could.
- Expert streaming needs system memory for the experts: Qwen3-30B-A3B in BF16 keeps about 36 GB of them in RAM. Short prompts are bound by the link to the card (about 30 tokens a second on the model above; llama.cpp 23–27).
- On a model that fits entirely on your card, llama.cpp was still the faster of the two when we last compared.
- The helper model for faster replies isn't used with mixture-of-experts models yet.
- The installer isn't code-signed yet, so Windows SmartScreen may warn you: **More info → Run anyway**.

## Upgrading

Install over 1.0.0. Everything in `%LOCALAPPDATA%\Leviathan` is kept, and your `.lev` files work unchanged. To have a model as a `.shn5`, convert its `.gguf` again in **Studio → Convert**.

---

Built with Claude Code by Michael Ricky Neal and Claude, Anthropic's AI: Michael set the direction and made the calls; Claude wrote most of the code and ran every test and benchmark. MIT licence.

<sub>Vulkan and the Vulkan logo are registered trademarks of the Khronos Group Inc. Claude and Claude Code are trademarks of Anthropic PBC.</sub>
