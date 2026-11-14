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
