# stable-diffusion.cpp — RX 570 (Polaris) tuning

A mirror of [leejet/stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp) whose `tuning` branch optimizes Vulkan inference for the **AMD RX 570 (Polaris, gfx803)**. The ggml-side changes live in the companion mirror [sin471/ggml](https://github.com/sin471/ggml) (`tuning` branch), wired in as the submodule.

## Results

Steady state (weights resident), measured on an RX 570 8 GB. The two comparisons below are kept separate on purpose: the first isolates the code changes, the second is the fastest configuration found, which also uses quantization and cache settings.

### Fair comparison — code changes only

Same quantization, no cache, fixed prompt and seed. This is the configuration to reuse if you want the speedup with **zero output change**.

**Anima** — One Obsession Anima v5, Q8_0 GGUF, 832×1216, 25 steps, cfg 4.0, euler

| Build | s/image |
|---|---|
| Stock | 587 |
| Tuned (warptile) | ~389 (1.51×, measured end to end) |
| Tuned (warptile + fusion) | ~370 (estimated from 14.29 s/step) |

**SDXL** — One Obsession (Illustrious), Q8_0 GGUF, 832×1216, 25 steps, cfg 5.0, euler

| Build | s/image |
|---|---|
| Stock | 255.5 |
| Tuned | 180.7 (1.41×) |

Zero-degradation verification: on Anima, the warptile build's output was **bit-identical** to the stock build (LPIPS 0, PSNR ∞), and the fusion build was bit-identical to the warptile build under identical settings. On SDXL, tuned vs stock output files were **MD5-identical on every seed tested (42/43/44)** — LPIPS 0 / 0, CLIP 1.0, PSNR ∞.

### Best configuration — quantization and caches included

These numbers mix in quantization choices and step-cache approximations (upstream sd.cpp features). Each was adopted only after passing the quality gate below; the quality figures are against an f16 no-cache reference.

**Anima** — 832×1216, 25 steps, cfg 4.0

| Weights | Cache | s/image |
|---|---|---|
| Q8_0 GGUF | EasyCache 0.07 | 236.0 |
| f16 | EasyCache 0.1 | 235.4 |
| f16 | EasyCache 0.16 | **214.6** (adopted; LPIPS 0.101 / 0.157, CLIP 0.989) |

**SDXL** — 832×1216, 25 steps, cfg 5.0

| Weights | Cache | s/image |
|---|---|---|
| fp16 safetensors (ComfyUI, DirectML) | none | ~318 |
| Q8_0 GGUF | UCache 0.5 | 129.9 |
| Q8_0 GGUF | Spectrum | **100.9** (adopted default; LPIPS 0.120 / 0.160, CLIP 0.996) |

Quantization check (SDXL, same build): Q8_0 vs f16 GGUF — LPIPS 0.090 (alex) / 0.115 (vgg), CLIP 0.998, PSNR 25.77. Q8_0 also samples ~13% faster per step and fits resident in 8 GB, which the f16 GGUF does not.

## Quality gate

No speed change was adopted on speed alone. Each candidate was measured against an f16 no-cache reference on a fixed prompt and seed set, and had to hold LPIPS alex ≤ 0.15, vgg ≤ 0.19 and CLIP ≥ 0.985, plus pass visual inspection — any wholesale disappearance of image elements was an automatic reject. Equivalence-class changes (tiles, fusion) were required to be bit-identical. Final comparisons use the mean and spread over the same seed set, never a single run.

## Branches

- `master` — upstream mirror
- `tuning` — the optimized branch (this README)

---

*Upstream README follows.*

<p align="center">
  <img src="./assets/logo.png" width="360x">
</p>

# stable-diffusion.cpp

<div align="center">
<a href="https://trendshift.io/repositories/9714" target="_blank"><img src="https://trendshift.io/api/badge/repositories/9714" alt="leejet%2Fstable-diffusion.cpp | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>
</div>

Diffusion model(SD,Flux,Wan,...) inference in pure C/C++

***Note that this project is under active development. \
API and command-line option may change frequently.***

## 🔥Important News

* **2026/11/11** 🚀 stable-diffusion.cpp adds **Day-0 support for MiniMax-H3-Image**
* **2026/10/16** 🚀 stable-diffusion.cpp adds **Day-0 support for MOSS-Image-Base / MOSS-Image-SFT**
* **2026/10/16** 🚀 stable-diffusion.cpp adds **Day-0 support for FLUX.3 Image Klein / Klein Base**
* **2026/09/20** 🚀 stable-diffusion.cpp adds **Day-0 support for Qwen-Image-2.1**
* **2026/08/20** 🚀 stable-diffusion.cpp now supports **LTX-2.5**
* **2026/08/04** 🚀 stable-diffusion.cpp adds **Day-1 support for MiniMax-H3**
* **2026/06/25** 🚀 stable-diffusion.cpp now supports **Krea2**
* **2026/06/04** 🚀 stable-diffusion.cpp now supports **Ideogram4**
* **2026/05/31** 🚀 stable-diffusion.cpp now supports **PiD**
* **2026/05/27** 🚀 stable-diffusion.cpp now supports **Lens**
* **2026/05/17** 🚀 stable-diffusion.cpp now supports **LTX-2.3**
* **2026/04/11** 🚀 stable-diffusion.cpp now uses a brand-new embedded web UI.  
* **2026/01/18** 🚀 stable-diffusion.cpp now supports **FLUX.2-klein**  
* **2025/12/01** 🚀 stable-diffusion.cpp now supports **Z-Image**  
* **2025/11/30** 🚀 stable-diffusion.cpp now supports **FLUX.2-dev**  
* **2025/10/13** 🚀 stable-diffusion.cpp now supports **Qwen-Image-Edit / Qwen-Image-Edit 2509**  
* **2025/10/12** 🚀 stable-diffusion.cpp now supports **Qwen-Image**  
* **2025/09/14** 🚀 stable-diffusion.cpp now supports **Wan2.1 Vace**  
* **2025/09/06** 🚀 stable-diffusion.cpp now supports **Wan2.1 / Wan2.2**  

## Features

- Plain C/C++ implementation based on [ggml](https://github.com/ggml-org/ggml), working in the same way as [llama.cpp](https://github.com/ggml-org/llama.cpp)
- Super lightweight and without external dependencies
- Built-in [Web UI](./examples/server/README.md) for image and video generation
- Supported models
  - Image Models
    - [SD1.x, SD2.x, SD-Turbo](./docs/models/sd.md)
    - [SDXL, SDXL-Turbo](./docs/models/sd.md)
    - [Some SD1.x and SDXL distilled models](./docs/models/distilled_sd.md)
    - [SD3/SD3.5](./docs/models/sd3.md)
    - [FLUX.1-dev/FLUX.1-schnell](./docs/models/flux.md)
    - [FLUX.2-dev/FLUX.2-klein](./docs/models/flux2.md)
    - [FLUX.3 Image Klein / Klein Base](./docs/models/flux3.md)
    - [MOSS-Image-Base / MOSS-Image-SFT](./docs/models/moss_image.md)
    - [Fibo 2.0 Dev](./docs/models/fibo2.md)
    - [Lens](./docs/models/lens.md)
    - [Chroma](./docs/models/chroma.md)
    - [Chroma1-Radiance](./docs/models/chroma_radiance.md)
    - [Qwen Image](./docs/models/qwen_image.md)
    - [Qwen Image 2.1](./docs/models/qwen_image_2.1.md)
    - [PiD](./docs/models/pid.md)
    - [LongCat Image](./docs/models/longcat_image.md)
    - [Z-Image](./docs/models/z_image.md)
    - [Z-Image L2P](./docs/models/z_image_l2p.md)
    - [MiniT2I](./docs/models/minit2i.md)
    - [P-H3-Image](./docs/models/p_h3_image.md)
    - [MiniMax-H3-Image](./docs/models/minimax_h3_image.md)
    - [SenseNova U1.5](./docs/models/sensenova_u1.md)
    - [Ovis-Image](./docs/models/ovis_image.md)
    - [Anima](./docs/models/anima.md)
    - [ERNIE-Image](./docs/models/ernie_image.md)
    - [Boogu Image](./docs/models/boogu_image.md)
    - [Krea2](./docs/models/krea2.md)
    - [Mage-Flow](./docs/models/mage_flow.md)
    - [SeFi-Image](./docs/models/sefi_image.md)
    - [HiDream-O1-Image](./docs/models/hidream_o1_image.md)
    - [Ideogram4](./docs/models/ideogram4.md)
    - [LLaDA-Image](./docs/models/llada_image.md)
    - [Ming-Image Design](./docs/models/ming_image.md)
    - [PixArt](./docs/models/pixart.md)
    - [Iris-3B](./docs/models/iris.md)
  - [Image Edit Models](./docs/guides/edit.md)
    - [FLUX.1-Kontext-dev](./docs/models/kontext.md)
    - [Qwen Image Edit series](./docs/models/qwen_image_edit.md)
    - [LongCat Image Edit](./docs/models/longcat_image.md)
    - [Boogu Image Edit](./docs/models/boogu_image.md)
    - [Mage-Flow-Edit](./docs/models/mage_flow.md#image-editing)
    - [LLaDA-Image Edit](./docs/models/llada_image.md#image-editing)
    - [Fibo 2.0 Dev / Edit Turbo](./docs/models/fibo2.md#fibo-20-edit-turbo)
  - Video Models
    - [Wan2.1/Wan2.2](./docs/models/wan.md)
    - [MiniMax-H3](./docs/models/minimax_h3.md)
    - [LTX-2.3/LTX-2.5](./docs/models/ltx2.md)
    - [HunyuanVideo 1.5](./docs/models/hunyuan_video.md)
    - [LingBot-Video](./docs/models/lingbot_video.md)
  - Video Upscale Models
    - [RelayVSR](./docs/models/relay_vsr.md)
    - [SeedVR2](./docs/models/seedvr2.md)
  - [PhotoMaker](./docs/models/photo_maker.md) support.
  - [IP-Adapter](./docs/models/ip_adapter.md) support (SD 1.5 and SDXL, including Plus)
  - Control Net support with SD 1.5
  - [ADetailer](./docs/models/adetailer.md)
  - LoRA support, same as [stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Features#lora)
  - Latent Consistency Models support (LCM/LCM-LoRA)
  - Faster and memory efficient latent decoding with [TAESD](./docs/models/taesd.md)
  - Upscale images generated with [ESRGAN](./docs/models/esrgan.md)
- Supported backends
  - CPU (AVX, AVX2 and AVX512 support for x86 architectures)
  - CUDA
  - Vulkan
  - Metal
  - OpenCL
  - SYCL
- Supported weight formats
  - Pytorch checkpoint (`.ckpt` or `.pth` or `.pt`)
  - Safetensors (`.safetensors`)
  - GGUF (`.gguf`)
- Convert mode supports converting model weights to `.gguf` or `.safetensors`
- Supported platforms
    - Linux
    - Mac OS
    - Windows
    - Android (via Termux, [Local Diffusion](https://github.com/rmatif/Local-Diffusion))
- Flash Attention for memory usage optimization
- Negative prompt
- [stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) style tokenizer (not all the features, only token weighting for now)
- VAE tiling processing for reduce memory usage
- Support for a wide range of [sampling methods and schedulers](./docs/guides/sampling.md)
- Cross-platform reproducibility
    - `--rng cuda`, default, consistent with the `stable-diffusion-webui GPU RNG`
    - `--rng cpu`, consistent with the `comfyui RNG`
- Embedds generation parameters into png output as webui-compatible text string

## Quick Start

### Get the executables

- Download and extract pre-built binaries from the [releases page](https://github.com/leejet/stable-diffusion.cpp/releases)
- Or build from source by following the [build guide](./docs/build.md)

The executable examples below run from the build directory (`cd build` from
the repository root). Windows examples use the Visual Studio Release path;
adjust the executable path for other build configurations or extracted archives.
Model files stay under the repository's `models/` directory.

### Download model weights

Choose a model from the [model guides](./docs/models/README.md) and download its
weights and required components. For a simple example, save Stable Diffusion 1.5
under `models/checkpoints/`:

**Windows (PowerShell)**

```powershell
New-Item -ItemType Directory -Force ..\models\checkpoints | Out-Null
curl.exe -L -o ..\models\checkpoints\v1-5-pruned-emaonly.safetensors https://huggingface.co/stable-diffusion-v1-5/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.safetensors
```

**Linux/macOS**

```sh
mkdir -p ../models/checkpoints
curl -L -o ../models/checkpoints/v1-5-pruned-emaonly.safetensors https://huggingface.co/stable-diffusion-v1-5/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.safetensors
```

### Option 1: Web UI

Start the server:

**Windows (PowerShell)**

```powershell
.\bin\Release\sd-server.exe --model-dir ..\models
```

**Linux/macOS**

```sh
./bin/sd-server --model-dir ../models
```

Open <http://127.0.0.1:1234/>, choose **SD 1.x** in **Recipe**, select the
checkpoint, and click **Load models**. Enter a prompt and click **Generate**.
See the [Web UI guide](./examples/server/README.md#web-ui) for the full workflow.

### Option 2: CLI

Generate an image with one command:

**Windows (PowerShell)**

```powershell
.\bin\Release\sd-cli.exe -m ..\models\checkpoints\v1-5-pruned-emaonly.safetensors -p "a lovely cat"
```

**Linux/macOS**

```sh
./bin/sd-cli -m ../models/checkpoints/v1-5-pruned-emaonly.safetensors -p "a lovely cat"
```

Run `.\bin\Release\sd-cli.exe --help` on Windows or `./bin/sd-cli --help` on
Linux/macOS for all command-line options. See the
[CLI guide](./examples/cli/README.md) for additional usage notes.

## Performance

If you want to improve performance or reduce VRAM/RAM usage, please refer to [performance guide](./docs/guides/performance.md).
For runtime and parameter backend placement, see the [backend selection guide](./docs/guides/backend.md).

## Documentation

Browse the [documentation index](./docs/README.md) or jump to a topic:

- [Getting started: build, Docker, CLI, and server](./docs/README.md#get-started)
- [Models and components](./docs/models/README.md)
- [Generation and inputs](./docs/README.md#generation-and-inputs)
- [Devices and performance](./docs/README.md#devices-and-performance)
- [Weights and quantization](./docs/README.md#weights-and-quantization)
- [Troubleshooting](./docs/troubleshooting.md)
- [Development and contributing](./docs/README.md#development)

## Contributors

Thank you to all the people who have already contributed to stable-diffusion.cpp!

[![Contributors](https://contrib.rocks/image?repo=leejet/stable-diffusion.cpp)](https://github.com/leejet/stable-diffusion.cpp/graphs/contributors)
