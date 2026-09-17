# Soundry AI — Inference Implementation Documentation

## 1. Overview

The inference pipeline generates music from text input (lyrics + genre/style descriptions). It operates as a **multi-stage system** that transforms text into discrete audio tokens, then decodes those tokens into high-fidelity audio waveforms (48kHz stereo, up to 4.5 minutes).

The pipeline runs on **AWS Inferentia2** or **Trainium** hardware using the **Neuron SDK**, replacing the original CUDA-based inference path.

---

## 2. Models Involved in Inference

| Model | Parameters | Role | Trained by Soundry? |
|-------|-----------|------|-------------------|
| Primary Llama Transformer | 2.83B–5.12B (28–36 layers) | Generates mixed audio token stream (codebook 0) | Yes |
| Secondary Llama Transformer | ~500M (12 layers) | Generates vocal + BGM token streams (codebooks 1-2) | Yes |
| Qwen2-7B Text Tokenizer | ~7B | Converts lyrics text into token IDs | No (pre-trained, frozen) |
| Flow1dVAE Audio Codec | ~3GB | Decodes tokens → latents → audio waveform | No (pre-trained, frozen) |
| Demucs HTDemucs | ~80MB | Source separation (splits song into vocal + BGM) | No (Meta pre-trained) |
| DaSheng Genre Classifier | ~200MB | Extracts style/genre features from reference audio | No (pre-trained, frozen) |

**Memory Footprint at Inference:**
- Primary + Secondary Transformers: ~6-10 GB
- Flow1dVAE (decoder): ~3 GB
- Demucs: ~80 MB
- Qwen2-7B Tokenizer: ~200 MB
- Auto-prompt library: ~50 MB
- Total: ~10-14 GB active (with offloading, down to ~4-5 GB on NeuronCore)

---

## 3. Inference Pipeline — Step by Step

### Step 1: Load Models (Server Startup)

At server startup, all models are loaded into memory:
- The primary and secondary transformers are loaded from checkpoint (either to GPU/NeuronCore directly, or to CPU if using memory offloading)
- The audio tokenizer (Flow1dVAE) is loaded for encoding prompt audio
- The separate tokenizer (diffusion-based decoder) is loaded for final audio generation
- Demucs source separator is initialized for processing reference audio

**Logic:** The system checks available GPU/NeuronCore memory and decides whether to use normal mode (all weights on device) or low-memory mode (offloading enabled). If available memory is greater than 24 GB for base models or 36 GB for large models, normal mode is used. Otherwise, micro-offloading is activated.

### Step 2: Process Reference Audio (Optional)

If the user provides reference audio for style transfer:

1. **Source Separation:** Demucs splits the reference audio into:
   - Full mix
   - Vocal track
   - Background music (BGM) track (computed as full_mix - vocals)
   
2. **Truncation:** Only the first 10 seconds of audio are used (at 48kHz = 480,000 samples)

3. **Encoding:** The audio tokenizer (Flow1dVAE encoder) converts the three audio tracks into discrete token representations

4. **Alternative — Auto-Prompt:** If no reference audio is provided but an `auto_prompt_audio_type` is specified (Pop, R&B, Dance, Jazz, Folk, Rock, Chinese Style, Metal, Reggae, etc.), the system selects a pre-encoded reference from a curated library of prompt tokens

### Step 3: Prepare Text Conditioning

Text conditioning prepares all non-audio inputs for the model:

1. **Lyrics Processing:**
   - Lyrics are formatted with section markers: `[verse]`, `[chorus]`, `[bridge]`, `[intro-short]`, `[intro-medium]`, `[outro-short]`, `[outro-medium]`, `[inst-short]`, `[inst-medium]`
   - Sentences within sections are separated by periods
   - Sections are separated by semicolons
   - The lyrics are tokenized using the Qwen2-7B tokenizer

2. **Genre/Style Descriptions:**
   - Style tags (gender, timbre, genre, emotion, instrument, BPM) are processed
   - Tags are shuffled and randomly trimmed (keeping at least 1) for diversity during training
   - For inference, all provided tags are used

3. **Structure Processing:**
   - Structure text is parsed and normalized into model-compatible section labels
   - Section alignment maps structure labels to time positions

4. **Melody/Chroma Processing:**
   - If melody chromagram features are available, they are normalized to [T, 12] tensors
   - Time axis is resampled to match the target sequence length

### Step 4: Language Model Token Generation (Autoregressive)

This is the core computation — the dual-transformer system generates audio tokens autoregressively:

#### 4.1 Delayed Pattern Interleaving

The model uses a **delayed codebook pattern** with delays `[0, 250, 250]`:
- Codebook 0 (mixed audio): generated with 0 delay
- Codebook 1 (vocal): delayed by 250 timesteps
- Codebook 2 (BGM): delayed by 250 timesteps

This means at each generation step, the model sees:
- Codebook 0: current timestep token
- Codebook 1: token from 250 steps ago
- Codebook 2: token from 250 steps ago

The pattern provider creates an interleaved sequence from these three codebook streams and handles the mapping between sequence positions and codebook timesteps.

#### 4.2 Forward Pass Logic

For each autoregressive step:

1. **Embed codebook 0** through the primary embedding layer
2. **Embed codebooks 1-2** through separate embedding layers (summed)
3. **Primary Transformer** processes codebook 0 embeddings → outputs logits for next codebook 0 token + hidden states
4. **MLP Bridge** concatenates the secondary embeddings with primary hidden states → fused representation
5. **Secondary Transformer** processes the fused representation → hidden states
6. **Linear heads** (one per codebook 1..K-1) project secondary hidden states to token logits

#### 4.3 Sampling Strategy

- **Temperature:** 0.9 (controls randomness)
- **Top-k:** 50 (only consider top 50 most likely tokens)
- **Top-p:** 0.0 (disabled by default)
- **Classifier-Free Guidance (CFG):** Scale of 1.5 applied to the LM logits
- **Repetition Penalty:** Applied to reduce repeated patterns
- **Record Window:** 50 tokens (for repetition penalty tracking)

#### 4.4 Generation Length

- Each "timestep" of tokens corresponds to approximately 40ms of audio
- For 30 seconds of audio: ~750 token generation steps
- For 4.5 minutes (270s): ~6,750 token generation steps
- Maximum new tokens: 3,000 per segment, with multi-segment support

### Step 5: Diffusion Decode (Tokens → Latents)

After all audio tokens are generated by the LM:

1. The generated token sequence is fed to the **GPT2-RoPE diffusion backbone** (16 layers)
2. The diffusion model performs 10-50 denoising steps to convert discrete tokens into continuous latent representations
3. **Flow guidance scale** of 1.5 is applied during the diffusion process
4. This step handles both vocal and BGM tracks separately when separate output is requested
5. Chunked decoding is used for memory efficiency (128-token chunks)

### Step 6: VAE Decode (Latents → Audio)

1. The **Stable Audio VAE decoder** converts continuous latents into raw audio waveforms
2. Output format: 48kHz stereo audio
3. Separate decode passes for:
   - Mixed output (vocals + instruments combined)
   - Vocal-only track
   - BGM-only track (when separate mode is requested)

### Step 7: Output Delivery

1. Generated audio saved as FLAC files to S3
2. Metadata (lyrics, settings, generation time) stored alongside audio
3. Audio accessible via CloudFront CDN within 10 seconds
4. Generation time logged (typical: 30-60 seconds for 30s audio on Inferentia2)

---

## 4. Memory Offloading System (Low-Memory Mode)

For GPUs/NeuronCores with insufficient memory to hold the full model, a sophisticated **micro-offloading** system is used:

### 4.1 Core Concept

Instead of loading all transformer layers onto the device simultaneously, the system:
- Keeps model weights on **CPU memory** permanently
- Copies only 1-2 layers to the NeuronCore at a time during generation
- Uses a **background thread** for asynchronous prefetching
- Reduces NeuronCore memory from ~6 GB to ~400 MB active

### 4.2 Offloading Process (8 Steps)

**Step 1 — Initialize OffloadProfiler:**
Creates a controller object with a background copy thread, work queue, condition variables, ring buffer, and tracking variables.

**Step 2 — Read Configuration:**
A YAML config defines what to offload:
- `offload_module`: Which module is the root (e.g., "self" = the audiolm model)
- `cpu_mem_gb`: CPU memory budget (0 = unlimited)
- `pre_copy_step`: How many layers to prefetch ahead (typically 1)
- `dtype`: Weight precision (torch.bfloat16 for Neuron)
- `offload_layer_dict`: Maps module names to depth levels (e.g., `transformer: 4` means offload at depth 4 in the transformer tree)
- `ignore_layer_list`: Layers to skip offloading (stay on device)

**Step 3 — Walk Model Structure:**
Recursively traverses the model tree:
- Modules NOT in offload_layer_dict → moved to NeuronCore permanently (small, always needed)
- Modules IN offload_layer_dict at target depth → marked as offload targets (stay on CPU)
- Check against ignore_layer_list for exceptions

**Step 4 — Install Forward Wrappers:**
For each offload target:
- Cast weights to bfloat16 on CPU (no device round-trip)
- Replace the `forward()` method with a custom wrapper that handles copy-run-delete

**Step 5 — Permanent Layers:**
Small/shared layers stay on NeuronCore:
- `condition_provider` (text/audio conditioners)
- `pattern_provider` (codebook interleaving)
- `linears` (output projection heads)
- These are needed every forward pass and are small enough to keep permanently

**Step 6 — Background Copy Thread:**
Runs an infinite loop:
- Waits for layer name in the queue
- Copies all weights from CPU → NeuronCore
- Synchronizes transfer
- Signals main thread: "layer ready"

**Step 7 — Generation Runtime:**
During autoregressive generation:
- First pass: discovers execution order (slower, no prefetch)
- Subsequent passes: prefetch enabled
  - While layer N runs, background thread copies layer N+1
  - When N finishes, N+1 is already on device (or nearly)
  - Device weights deleted after use (free memory)
  - Uses `torch.func.functional_call()` to run with temporary weights

**Step 8 — Cleanup:**
- Stop flag set, copy thread exits gracefully
- All memory references dropped
- NeuronCore freed for next stage (audio decode)

### 4.3 Memory Timeline

```
Without Offloading:  All 44 layers on device = ~6+ GB
With Offloading:     2 layers on device at any time = ~400 MB

Device at any moment:
  [Layer N running] [Layer N+1 prefetched] = ~200 MB × 2 = 400 MB
  + permanent layers (condition_provider, linears) = ~200 MB
  Total: ~600 MB vs 6+ GB
```

---

## 5. Generation Modes

| Mode | Flag | Output |
|------|------|--------|
| Mixed (default) | `--generate_type mixed` | Full song (vocals + instruments combined) |
| Vocal only | `--generate_type vocal` | Isolated vocal track |
| BGM only | `--generate_type bgm` | Instrumental track only |
| Separated | `--generate_type separate` | All three tracks saved separately |

---

## 6. Neuron-Specific Inference Optimizations

### 6.1 Bucketed Compilation
- Neuron requires fixed sequence lengths at compile time
- Multiple compilation buckets handle variable-length inputs
- Sequence length finalized before design phase

### 6.2 NKI Fused RoPE Kernel
- Optimized rotary position embedding computation
- Loads cos/sin tables once per sequence tile, reuses across all attention heads
- Coalesced DMA with 8-tile batching for efficient memory access
- Native [Batch, Heads, Sequence, Dimension] layout (no reshape overhead)
- Supports forward and backward passes

### 6.3 Model Compilation
- Both Stage 1 (7B) and Stage 2 (1B) compiled via `torch_neuronx.trace()`
- Compilation artifacts (.neuron/.neff files) stored in S3 for reuse
- Loaded at SageMaker endpoint startup

### 6.4 Eager Mode for Offloading
- Micro-offloading uses eager mode (no torch.compile)
- Each operation executes immediately using temporary device weights
- Compatible with dynamic layer swapping

---

## 7. Inference Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `temperature` | 0.9 | Sampling temperature (higher = more random) |
| `top_k` | 50 | Top-K filtering (consider only top K tokens) |
| `top_p` | 0.0 | Nucleus sampling threshold (0 = disabled) |
| `cfg_coef` | 1.5 | Classifier-free guidance scale |
| `flow_guidance_scale` | 1.5 | Diffusion decode guidance scale |
| `max_duration` | 270s (4.5 min) | Maximum song duration |
| `extend_stride` | 5 | Stride for extending generation |
| `record_tokens` | true | Enable repetition penalty tracking |
| `record_window` | 50 | Window for repetition penalty |

---

## 8. Inference Input/Output Specification

### Input
- **Lyrics** (required): Structured text with section labels (`[verse]`, `[chorus]`, `[bridge]`, etc.)
- **Descriptions** (optional): Genre, mood, timbre, instrument, BPM descriptors
- **Reference Audio** (optional): 10-second audio file for style transfer
- **Auto-Prompt Type** (optional): Predefined style category (Pop, R&B, Dance, Jazz, Folk, Rock, etc.)

### Output
- Audio file: FLAC format, 48kHz stereo
- Optional separate tracks: vocal, BGM, mixed
- Metadata: generation time, model version, parameters used

---

## 9. Conditioning System

The inference pipeline supports multiple conditioning inputs that guide the generation:

| Condition | Purpose | Processing |
|-----------|---------|-----------|
| `prompt_audio` | Style transfer from reference audio | Encoded to tokens via Flow1dVAE |
| `lyrics` | Text content for the song | Tokenized via Qwen2-7B |
| `genre` | Genre classification descriptors | Processed by genre conditioner |
| `style` | Style embedding from DaSheng | Vector-space style guidance |
| `melody_chroma` | 24-dimension melody contour | Chromagram features at 62.5 FPS |
| `structure` | Section layout of the song | Appended to lyrics channel |

The `resolve_enabled_conditions()` function allows selective enabling/disabling of conditions during both training and inference.

---

## 10. Deployment Architecture

```
User Request → CloudFront → API Gateway → Backend (ECS/Lambda)
                                                │
                                                ▼
                                    SageMaker Real-Time Endpoint
                                    (inf2.xlarge / trn1.2xlarge)
                                                │
                                                ▼
                                    Neuron-compiled models execute
                                    (Stage 1 → Stage 2 → Decode)
                                                │
                                                ▼
                                    Generated audio → S3 → CDN → User
```

### Endpoint Specifications
- Instance: `inf2.xlarge` or `trn1.2xlarge`
- Deployment time: <15 minutes
- Update strategy: Blue/green (zero downtime)
- Target latency: <5 minutes for 2-segment generation
- Health monitoring via SageMaker endpoint status

---

## 11. System Monitoring During Inference

A dedicated **System Monitor API** provides real-time hardware metrics:

| Endpoint | Returns |
|----------|---------|
| `GET /cpu` | CPU usage, per-core utilization, frequency, load averages |
| `GET /memory` | RAM total/used/available, swap usage |
| `GET /gpu` | Per-GPU memory, utilization percentage, temperature |
| `GET /overview` | Combined snapshot of all hardware metrics + OS info |

This runs as a FastAPI server alongside inference workloads, enabling real-time monitoring of NeuronCore/GPU utilization during generation.
