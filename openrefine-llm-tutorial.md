# Running OpenRefine with a Local LLM (Ollama + Ministral 3B)

This tutorial walks through wiring an LLM into OpenRefine so you can use natural-language prompts directly on your columns, fully offline. We run the model with [Ollama](https://ollama.com) and connect it to OpenRefine through the community-built [AI Extension](https://github.com/sunilnatraj/llm-extension) presented on the [OpenRefine blog](https://openrefine.org/blog/2025/02/21/AIExtension).

We assume OpenRefine 3.8.7 or later is already installed. If not, grab it from [openrefine.org/download](https://openrefine.org/download).

## What you will end up with

A local OpenRefine instance with an **AI** menu in the Extensions bar, talking to a Ministral 3B model served by Ollama at `http://localhost:11434`. No API key, no data leaving your machine, no per-token cost. Ministral 3 has no chain-of-thought mode, so responses are short and clean by default — a good fit for column-level data wrangling.

## Hardware requirements

Ministral 3B in its default 4-bit quantization weighs around 2 GB on disk and needs roughly 3 to 4 GB of free RAM at inference time. Add the memory OpenRefine itself uses (1 to 2 GB on a typical project), plus headroom for the operating system, and a usable minimum sits around 8 GB of RAM. The model is small enough to run on CPU only, but throughput depends heavily on memory bandwidth and core count.

**Apple Silicon (M1, M2, M3, M4).** Any Mac with an M-series chip and 8 GB of unified memory will run the model comfortably. Ollama uses the Metal backend automatically and the unified memory architecture means the GPU has direct access to the model weights without copying. Expect roughly 30 to 50 tokens per second on an M1 base model, faster on Pro and Max variants. A 16 GB machine is more relaxed when OpenRefine is also loaded with a large project. Intel-based Macs work but are noticeably slower — count on 5 to 10 tokens per second.

**Linux PC, CPU only.** A modern Intel i7 (10th generation or later) or AMD Ryzen 5/7 with 8 GB of RAM is the practical floor. You will see roughly 8 to 15 tokens per second on CPU inference, which is enough for column-level work on a few hundred rows but slow on the full 23,000-row dataset (count on several hours per pass). Use AVX2-capable CPUs; older chips without AVX2 work but drop to single-digit token rates. A dedicated NVIDIA GPU with 4 GB of VRAM or more (GTX 1650, RTX 3050, RTX 4060) brings throughput up to 60 to 100 tokens per second via CUDA, which is the difference between an overnight run and a coffee-break run on the full dataset.

**Windows.** Same expectations as the Linux numbers above. An NVIDIA GPU is the single biggest upgrade you can make if you plan to process more than a few thousand rows.

**Disk space.** Reserve at least 5 GB free: 2 GB for the model, plus space for Ollama's cache and OpenRefine workspace.

**If your machine falls short.** Drop to a smaller quantization (`Q3_K_M` saves around 500 MB of RAM at modest quality cost) or run shorter prompts. If you cannot meet the memory floor at all, point the AI Extension at a cloud-hosted OpenAI-compatible endpoint instead — the rest of the setup is identical.

## 1. Install Ollama

Ollama is the local model server. Install it for your platform:

**macOS** — Download the installer from [ollama.com/download/mac](https://ollama.com/download/mac), open the `.dmg`, and drag Ollama to Applications. Launching it once starts the background service on port `11434`. Alternative via Homebrew:

```bash
brew install --cask ollama
```

**Linux** — One-line install script (works on most distributions, including Ubuntu, Debian, Fedora, Arch):

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This registers a `systemd` service. Verify it is running:

```bash
systemctl status ollama
```

**Windows** — Download the installer from [ollama.com/download/windows](https://ollama.com/download/windows) and run `OllamaSetup.exe`. Ollama then runs as a tray app and exposes the same `http://localhost:11434` endpoint.

Confirm the server replies on any OS:

```bash
curl http://localhost:11434/api/tags
```

You should get a JSON response (likely an empty `models` array on a fresh install).

## 2. Pull Ministral 3B

Ministral 3B is Mistral's edge model: 3 billion parameters, a 256K context window, around 3 GB on disk, and no reasoning mode to filter out. Pull it from the official Ollama library:

```bash
ollama pull ministral-3:3b
```

Verify the model is registered:

```bash
ollama list
```

Copy the value from the **NAME** column exactly as Ollama prints it — that is what you will paste into OpenRefine in step 4. It will look like `ministral-3:3b`.

Optional smoke test, to make sure inference works before involving OpenRefine:

```bash
ollama run ministral-3:3b "Say hello in one short sentence."
```

## 3. Install the AI Extension in OpenRefine

Download the latest release `.zip` from the [llm-extension releases page](https://github.com/sunilnatraj/llm-extension/releases) (0.1.2.3 or later). Unzip it, then move the resulting folder into your OpenRefine **extensions** directory. The recommended location is the per-user workspace, because it survives OpenRefine upgrades.

| OS | Workspace extensions path |
|---|---|
| macOS | `~/Library/Application Support/OpenRefine/extensions/` |
| Linux | `~/.local/share/openrefine/extensions/` |
| Windows | `%APPDATA%\OpenRefine\extensions\` (typically `C:\Users\<you>\AppData\Roaming\OpenRefine\extensions\`) |

If the `extensions` subfolder does not exist yet, create it. The final layout should look like `…/OpenRefine/extensions/llm-extension/module/…` — i.e. the unzipped folder sits directly inside `extensions/`, not nested one level too deep.

Restart OpenRefine. Open any project and look at the top-right bar: a new **AI** menu should appear next to **Wikibase**. If it does not, the most common cause is an extra folder level — check that `module/` is one directory below `extensions/llm-extension/`.

## 4. Configure Ollama as a provider in OpenRefine

Open the **AI** menu and choose **Manage providers** (or the equivalent "Add provider" entry), then fill the form:

| Field | Value |
|---|---|
| Label | `Ollama Ministral` (any name you want) |
| Server URL | `http://localhost:11434/v1/chat/completions` |
| Model | `ministral-3:3b` (exact name from `ollama list`) |
| API Key | `ollama` (Ollama ignores it, but the field cannot be empty) |
| Temperature | `0.3` for extraction and cleaning, `0.7` for free-form rewriting |
| Top-P | `0.95` |
| Seed | `42` if you want reproducible runs, blank otherwise |
| Max Tokens | `1024` is a reasonable starting point |
| Wait Time | `0` on a local model; raise to `200`–`500` ms only if you see the server choke |

Ollama exposes an OpenAI-compatible endpoint at `/v1/chat/completions`, which is exactly what the extension expects, so no proxy is needed.

Click **Test Service**. A green confirmation means the extension reached Ollama and got a valid chat completion back. Click **Save**.

## 5. Use it on a column

Open a project, click the dropdown of any column, and choose **Edit column** then **Add column by AI** (the entry added by the extension). You will get a dialog where you:

1. Pick the provider you just configured.
2. Write a natural-language instruction, for example *"Extract the country from this address and return only the ISO 3166-1 alpha-2 code."*
3. Choose a response format — **Text** for free-form answers, **JSON Object** for loosely-structured output, **JSON Schema** when you want the model to fill a strict shape.
4. Use **Preview** to run the prompt against the first row only. Iterate on the wording until the preview looks right, then run it on the full column.

The extension also stores up to 100 recent prompts and lets you star the ones you reuse, which is convenient when you iterate on a cleaning workflow.

## Troubleshooting

**No AI menu after restart.** The unzipped folder is probably nested one level too deep. Inside your `extensions/` directory you should see `llm-extension/module/…`, not `llm-extension/llm-extension/module/…`.

**Test Service fails with a connection error.** Ollama is not running, or it is bound to a different host. Re-check with `curl http://localhost:11434/api/tags`. On Linux, `systemctl restart ollama` usually fixes it.

**Test Service fails with a 404 on the model.** The string in the **Model** field does not match Ollama's catalogue. Re-run `ollama list` and copy the name verbatim, including the `:3b` tag.

**Model is too small for the task.** Ministral 3B is great for short extractions and reformatting, but heavier reasoning or multilingual rewriting may need more capacity. Swap to a larger Ollama model (`mistral:7b`, `llama3.1:8b`, `qwen2.5:7b`) by pulling it and updating the **Model** field — the rest of the setup stays the same.
