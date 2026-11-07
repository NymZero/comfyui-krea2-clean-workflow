# Krea 2 — Clean ComfyUI Workflow

A compact ComfyUI workflow for Krea 2 text-to-image generation, with optional image refinement, LoRA support, and a comparison view. The main path keeps generation and refinement on the same diffusion model to balance quality, speed, and model loading overhead.

**Workflow:** [Krea2_V2.2.json](Krea2_V2.2.json)

The supplied V2.2 export also contains a separate **SeedVR2 Upscale** branch, muted by default. This branch uses additional models; it is distinct from the lightweight Krea 2 refinement path described below.

## Features

- Krea 2 text-to-image generation.
- Optional 1.5× image resize followed by low-denoise refinement with Krea 2.
- An **Image Comparer** connected to the base image and the Krea-refined image.
- Two **Power LoRA Loader** nodes for individual LoRAs or stacks, configured separately for generation and refinement.
- Per-layer conditioning control through **ConditioningKrea2Rebalance** on the generation path.
- An **Options** panel using **Fast Groups Muter** to enable or mute workflow groups.
- PNG output through **SaveImageWithMetaData**.

## How the upscale works

The active `Upscale` group uses this sequence:

```text
Krea 2 generation → VAE decode → ImageScaleBy (bilinear, 1.5×)
                 → VAE encode → Krea 2 refinement (denoise 0.15) → VAE decode
```

At the saved base resolution of **1280 × 1728**, the resize produces **1920 × 2592** before refinement. Both samplers receive the diffusion model from the same `Initial Loader`, through their own LoRA loaders. No additional diffusion checkpoint is needed for this path.

This is a quality/speed tradeoff. It is not a claim of maximum reconstruction quality or superiority to dedicated upscalers such as SeedVR2.

The separate `SeedVR2 Upscale` branch receives the **base decoded image**, rather than the output of the Krea refinement pass. Its model loaders, upscaler, and save node are muted in the export (`mode: 2`). A connected `FastFilmGrain` node is bypassed (`mode: 4`). The comparer is not connected to SeedVR2 output.

## Requirements

### ComfyUI

Use a [ComfyUI](https://github.com/comfyanonymous/ComfyUI) installation that supports Krea 2, the `krea2` text-encoder type, subgraphs, and the `er_sde` sampler. The export contains an `Initial Loader` subgraph, not a missing custom node with a UUID-like name.

The JSON records frontend version `1.52.7` and mixed per-node core version metadata. These are export details, not a verified minimum-version requirement or a reproducible environment lock.

### Models

The main Krea 2 path uses `.safetensors` files with the standard ComfyUI loaders. It does not contain GGUF loaders.

| Component | Model or selection guidance | Default location |
| --- | --- | --- |
| Krea 2 diffusion model | Any compatible Krea 2 `.safetensors` model of your choice | `ComfyUI/models/diffusion_models/` |
| Qwen3-VL 4B text encoder | `qwen3vl_4b_fp8_scaled.safetensors` | `ComfyUI/models/text_encoders/` |
| Qwen Image VAE | `qwen_image_vae.safetensors` | `ComfyUI/models/vae/` |
| Optional Krea 2 LoRAs | A local Iris file is enabled in both saved loaders; see below | `ComfyUI/models/loras/` |

The author also uses **krea2TurboFP8** (approximately **12 GB**), a Qwen3-VL 4B encoder variant (approximately **5 GB**), and the Qwen Image VAE (approximately **250 MB**). A **ComfyOrg nvfp4** Krea 2 variant (approximately **7 GB**) is another reported option. These are approximate file sizes supplied by the author, not VRAM requirements; the nvfp4 variant is not selected in this export or validated here.

**Select your preferred Krea 2 model in `Initial Loader`.** Any Krea 2 model compatible with the workflow's standard `.safetensors` loader can be used. Saved model selections reflect local testing; choose the file installed on your machine without renaming it to match the export.

Model weights are not included in this repository. Obtain compatible weights from their publishers and follow their model cards. Different quantizations and model variants may need different runtime support and settings.

### Custom nodes

| Package | Nodes used | Role in this export |
| --- | --- | --- |
| [rgthree-comfy](https://github.com/rgthree/rgthree-comfy) | `Power Lora Loader (rgthree)`, `Fast Groups Muter (rgthree)`, `Image Comparer (rgthree)` | LoRA loading, group controls, comparison |
| [ComfyUI-Easy-Use](https://github.com/yolain/ComfyUI-Easy-Use) | `easy setNode`, `easy getNode` | Shared MODEL, CLIP, and VAE routing |
| [nova452's conditioning nodes](https://github.com/nova452/ComfyUI-ConditioningKrea2Rebalance), now documented as [Rebalance-Pack](https://github.com/nova452/Rebalance-Pack) | `ConditioningKrea2Rebalance` | Generation conditioning adjustment |
| [Santodan's revived metadata extension](https://github.com/Santodan/revived_comfyui_image_metadata_extension) | `SaveImageWithMetaData` | Image saving with metadata |
| [ComfyUI-SeedVR2_VideoUpscaler](https://github.com/numz/ComfyUI-SeedVR2_VideoUpscaler) | `SeedVR2LoadDiTModel`, `SeedVR2LoadVAEModel`, `SeedVR2VideoUpscaler` | Additional branch, muted by default |
| [VRGDG nodes recorded in the export](https://github.com/vrgamegirl19/comfyui-vrgamedevgirl), with utility nodes now in [VRGDG Custom Node Pack](https://github.com/vrgamegirl19/VRGDG_CustomNodePack) | `FastFilmGrain` | Bypassed node after SeedVR2 |

SeedVR2 and FastFilmGrain are additional dependencies present in V2.2 beyond the original four-package list. Their processing is not needed for the main path, but their node definitions may still be needed to load the complete canvas without missing-node placeholders.

Some saved registry identifiers are inconsistent with node names: the two Power LoRA loaders and the conditioning node carry `cnr_id: comfyui-workflow-encrypt`; FastFilmGrain carries `cnr_id: comfyui-aspect-ratio-crop-node`. Use the actual node types and upstream sources when resolving dependencies. These identifiers alone do not establish a need for a workflow-encryption package.

The metadata extension's upstream README currently points to a maintained fork. The table identifies the provider recorded in this workflow; compatibility with replacement forks has not been tested. Install only one provider for a given node type.

## Installation

1. Install or update ComfyUI following its upstream instructions, including the appropriate PyTorch/GPU setup for your hardware.
2. Install the custom node packages above using **ComfyUI-Manager**, where available, or each package's documented manual installation procedure. For manual installations, use ComfyUI's own Python environment for any package requirements. Restart ComfyUI and refresh its interface afterward.
3. Place your diffusion model, text encoder, VAE, and optional LoRAs in the model directories listed above, or use your existing ComfyUI model-path configuration.
4. Download [Krea2_V2.2.json](Krea2_V2.2.json) and open it in ComfyUI by dragging it onto the canvas or using the workflow open command.
5. Resolve any missing node types before queueing. Check the dependency table if automatic detection suggests an unrelated package.
6. In `Initial Loader`, select the files actually installed on your machine. Inside the subgraph, the `CLIPLoader` type is saved as **`krea2`**, with device `default`; the `UNETLoader` weight dtype is `default`.
7. Review **both** Power LoRA loaders. Each export row is enabled at strength **1.0** and refers to `My_LoRa/K2_Iris_KSC_9.safetensors`. Replace that selection with a compatible local LoRA, or switch the row off in both loaders for a run without LoRAs. The local file is not bundled or required.

Make these runtime selections in your own ComfyUI session or local workflow copy. Keep the repository's original JSON as the reference export.

## Usage

1. Write your positive prompt in the `Generation` group's text encoder (node **500**). Its saved text is simply `woman`; the base negative prompt (node **505**) is empty.
2. Set width, height, and batch size in `EmptyLatentImage` (node **508**). Start with the saved settings below and reduce resolution if memory is insufficient.
3. Choose the processing path using the **Options** group controls:
   - **Base generation only:** keep `Krea2 MODEL`, `Base model`, `Generation`, and `Save Image` active; mute `Upscale` and keep `SeedVR2 Upscale` muted. Also mute the separate **Image Comparer** node (**32**), which expects the refinement output on its second input.
   - **Krea refinement:** enable `Upscale` and the Image Comparer. Keep `SeedVR2 Upscale` muted. This is the active upscale path in the saved export.
4. Review the refinement prompt (node **18**) and its Power LoRA Loader (node **536**). This prompt is empty in the export and is independent of the generation prompt. The refinement negative conditioning is produced by `ConditioningZeroOut`; the generation rebalancing node does not feed this pass.
5. Queue a generation. For refinement runs, inspect the base preview and the Image Comparer: **A** receives the base image and **B** receives the saved Krea-refined image.
6. Find the PNG outputs under ComfyUI's configured output directory:
   - Base images: `Krea2/`
   - Krea-refined images: `Krea2/Krea2_Upscale/`

The save nodes use timestamp-based prefixes and `metadata_scope: full`. The comparer stores temporary image references in the JSON; these are not example images distributed with the repository. Run the workflow to populate it on your machine.

### Separate SeedVR2 branch

Keep this branch muted for the lightweight workflow. If you choose to use it, follow the SeedVR2 package's installation and hardware guidance first. The saved branch selects:

- DiT: `seedvr2_ema_7b_sharp_fp16.safetensors`
- VAE: `ema_vae_fp16.safetensors`
- Device: `cuda:0`, with CPU offloading
- Resolution: `2000`; maximum resolution: `2592`

Its upstream documentation uses `ComfyUI/models/SEEDVR2/` for these models. This is an additional model set, separate from the Krea model and Qwen VAE. The branch saves under `Krea2/Krea2_SeedVR/`; its timing, memory use, and image quality are not established by the performance figures below. Preserve the FastFilmGrain bypass unless you intentionally want that processing.

## Saved parameters

These values describe the supplied JSON, not universal recommendations for every Krea 2 model or LoRA.

| Parameter | Base generation | Krea refinement |
| --- | --- | --- |
| Sampler node | `504` | `19` |
| Steps | `8` | `2` |
| CFG | `1` | `1` |
| Sampler | `euler` | `er_sde` |
| Scheduler | `beta` | `simple` |
| Denoise | `1.0` | `0.15` |
| Saved seed | `6439` | `5047` |
| Seed control | `increment` | `increment` |
| Positive prompt | `woman` | Empty |
| Negative conditioning | Empty text encoding | Zeroed conditioning |

Other relevant settings:

- Base latent: **1280 × 1728**, batch size **1**.
- Resize: **bilinear**, factor **1.5**.
- Generation conditioning multiplier: **0.5**.
- Generation per-layer weights: `1.0,1.0,1.0,1.0,1.0,1.0,1.0,2.5,5.0,1.1,4.0,1.0`.
- Both saved LoRA rows: enabled, strength **1.0**, configured independently.

For repeatable comparisons, fix the seeds in both samplers and keep the models, LoRAs, prompts, resolution, and other settings unchanged. The saved `increment` controls change seeds between runs. If you change LoRAs for generation, review the refinement loader too: its MODEL/CLIP inputs come from the shared initial models, not from the generation LoRA loader.

## Indicative performance

Author-observed timings on an **NVIDIA RTX 5070 Ti with 16 GB VRAM** and **48 GB system RAM**:

| Path | Approximate observed time |
| --- | --- |
| Base generation | 29 seconds |
| Generation with the Krea upscale/refinement path | 60 seconds total |

These are practical reference observations, not universal benchmarks or measurements reproduced for this documentation. The observations do not establish timings for the exact model selection in the current export, and do not cover SeedVR2. Model variant, resolution, LoRA stack, software versions, offloading, and model-loading state can affect runtime and memory use.

## Optional LoRA example

[Iris V2 — Krea 2 Character LoRA](https://huggingface.co/ShadowFrame7/Krea-2-Iris-V2-Character-LoRA) is an optional Krea 2 character LoRA published by the workflow author. It is not required for generation.

Consult its model card for the supported base model, trigger words, and recommended strength before selecting it in the loaders. The export's local `K2_Iris_KSC_9.safetensors` filename is not proof that it matches a particular public Iris V2 file. The model card and weights were not independently inspected during this documentation pass; end-to-end compatibility has not been tested here.

## Example images

Example images and workflow screenshots have not been added yet. A suggested layout for future assets is:

```text
docs/images/
├── workflow-overview.png
├── base-example.png
├── krea-refined-example.png
└── base-vs-refined.png
```

These are proposed filenames, not files currently included in the repository. When adding examples, record the model and quantization, LoRA names and strengths, prompts, both seeds, resolution, and relevant sampler settings. Label base, Krea-refined, and any future SeedVR2 examples separately.

## Verification scope

This documentation was checked against the nodes, connections, subgraph, and saved settings in `Krea2_V2.2.json`, with upstream documentation/source checks for the listed node providers. The workflow JSON was left unchanged. No ComfyUI generation or GPU performance test was run as part of this documentation update.
