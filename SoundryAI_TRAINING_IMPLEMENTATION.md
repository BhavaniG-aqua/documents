# Soundry AI — Training Implementation Documentation

## 1. Overview

The training pipeline trains the **LeVo2 dual-transformer codec language model** to generate high-quality music from text descriptions. Training runs on **AWS Trainium2** hardware using the **Neuron SDK** with **Native PyTorch 2.10**, migrated from the original CUDA/GPU-based pipeline.

Only the two Llama transformer models are trained — all other models (text tokenizer, audio codec, source separator) are pre-trained and frozen.

---

## 2. Model Architecture

### 2.1 What Gets Trained

| Component | Parameters | Layers | Purpose |
|-----------|-----------|--------|---------|
| Primary Llama Transformer | 2.83B (medium) / 5.12B (large) | 28–36 | Processes codebook 0 (mixed audio tokens) |
| Secondary Llama Transformer | ~500M | 12 | Processes codebooks 1-2 (vocal + BGM tokens) |
| MLP Bridge | Small | 2 linear layers | Fuses primary hidden states with secondary embeddings |

### 2.2 Architecture Details

**Primary Transformer (Transformer 1):**
- Llama architecture with RMSNorm, SwiGLU MLP, Grouped-Query Attention
- Hidden size: 2048
- Intermediate size: 11008
- Attention heads: 16, Head dimension: 128
- RoPE position embeddings (theta = 100,000)
- Max sequence length: 8,196 tokens

**Secondary Transformer (Transformer 2):**
- Same Llama architecture, fewer layers (12)
- RoPE theta: 500,000 (longer context support)
- Max sequence length: 10,000 tokens

**MLP Bridge:**
- Takes concatenated [secondary_embeddings, primary_hidden_states] (dim × 2)
- Linear(dim×2, dim) → GELU → Linear(dim, dim)
- Produces fused representation for secondary transformer

### 2.3 Vocabulary

- **Code size:** 16,384 audio tokens per codebook
- **Special tokens:** +1 EOS, +1 EOP (end-of-pattern)
- **Total input embedding size:** 16,386
- **Code depth:** 3 codebooks (mixed, vocal, BGM)

---

## 3. Training Data Preparation (Offline — Not on Trainium)

Before training begins, raw audio data is preprocessed into WebDataset tar shards:

### Step 1: Source Separation
- Tool: Demucs HTDemucs (Meta's model)
- Input: Raw song audio files
- Output: Separated vocal track + background music (BGM) track

### Step 2: Audio Tokenization
- Tool: Flow1dVAE audio encoder
- Input: Vocal track, BGM track, full mix
- Output: Discrete token sequences for each track (3 codebook streams)
- Each token represents ~40ms of audio

### Step 3: Lyrics Tokenization
- Tool: Qwen2-7B text tokenizer
- Input: Song lyrics with structure labels
- Output: Token IDs

### Step 4: Style Feature Extraction
- Tool: DaSheng genre classifier
- Input: Audio tracks
- Output: Style/genre embedding vectors

### Step 5: Melody Extraction
- Tool: Chromagram extractor
- Input: Audio (at 32kHz, 512-hop)
- Output: 24-dimension chroma features at 62.5 FPS
- Normalized to [Time, 12] tensors

### Step 6: Packaging
- All features packaged into WebDataset `.tar` shards
- Each sample contains: pre-encoded audio tokens, lyrics tokens, genre features, style embeddings, melody features, structure labels
- Stored in S3: `s3://soundry-datasets/levo/`

---

## 4. Delayed Codebook Pattern (Core Training Logic)

### 4.1 What Is It?

The model generates 3 parallel streams of audio tokens (codebooks). Instead of generating all 3 tokens simultaneously at each timestep, it uses a **delayed pattern** where:
- Codebook 0 (mixed): no delay
- Codebook 1 (vocal): delayed by 250 timesteps
- Codebook 2 (BGM): delayed by 250 timesteps

### 4.2 Why This Pattern?

The delay allows the model to:
1. First generate the overall mixed audio structure (codebook 0)
2. Then generate the vocal and BGM tracks with awareness of what was generated 250 steps earlier
3. This improves coherence between the full mix and individual tracks

### 4.3 How It Works in Training

**Building the Interleaved Sequence:**
1. Given raw codes of shape [Batch, Codebooks=3, Timesteps]
2. The DelayedPatternProvider creates a layout mapping sequence positions to (timestep, codebook) pairs
3. The first sequence position is always empty (start token)
4. For each subsequent position `s`, each codebook `q` maps to timestep `t = s - 1 - delays[q]`
5. Only valid positions (where `0 ≤ t < timesteps`) contribute tokens

**Forward Pass:**
1. Pattern converts [B, K, T] raw codes → [B, K, S] interleaved sequence
2. Model processes the interleaved sequence
3. Model outputs logits in interleaved space [B, K, S, vocab_size]
4. Pattern reverts logits back to [B, K, T, vocab_size] for loss computation

**Reverting Predictions:**
- The revert operation maps each (codebook, timestep) pair back to its sequence position
- Invalid positions (where tokens haven't been generated yet due to delays) are masked out during loss computation

---

## 5. Training Loop Logic

### 5.1 Forward Pass

```
Input: codes [B, K=3, T] (pre-tokenized audio)

1. Apply delayed pattern → interleaved sequence [B, K, S]
2. Embed codebook 0 → primary input [B, S, dim]
3. Embed codebooks 1-2 → secondary input [B, S, dim] (summed)
4. Primary Transformer(primary_input) → logits_cb0 [B, S, vocab], hidden_states [B, S, dim]
5. Concatenate [secondary_input, hidden_states] → [B, S, dim*2]
6. MLP Bridge → fused [B, S, dim]
7. Secondary Transformer(fused) → hidden_states_2 [B, S, dim]
8. Linear heads × (K-1) → logits_cb1, logits_cb2
9. Stack all logits → [B, K, S, vocab]
10. Revert pattern → [B, K, T, vocab] + validity mask [B, K, T]
```

### 5.2 Loss Computation

- **Loss function:** Cross-entropy (per-element)
- **Masking:** Only valid positions (where pattern maps to real timesteps) contribute to loss
- **Reduction:** Sum of masked losses / count of valid positions
- **NaN handling:** Logits from invalid pattern positions contain NaN (from revert) — replaced with 0 before loss computation

**Optional NKI Fused Cross-Entropy:**
- Custom Neuron kernel for more efficient cross-entropy
- Uses online log-sum-exp for numerical stability
- Chunked vocabulary processing (handles 16,385 vocab size)
- Forward + backward computed in a single kernel invocation

### 5.3 Backward Pass

- Standard backpropagation through both transformers
- Gradients flow: loss → linear heads → secondary transformer → MLP bridge → primary transformer → embeddings
- With FSDP: gradients all-reduced across NeuronCores after backward

### 5.4 Optimizer Step

- **Optimizer:** AdamW (standard PyTorch, replaces bitsandbytes Adam8bit from CUDA version)
- **Learning rate:** Configurable (default 1e-4)
- **Precision:** BF16 compute, FP32 gradient accumulation (with FSDP mixed precision)
- **Neuron synchronization:** `torch.neuron.synchronize()` after each step for accurate timing

---

## 6. Training Stages

### Stage 1: Full Conditioning Training
- **All conditions enabled:** lyrics, genre, style, melody_chroma, structure, prompt_audio
- **Config:** `train_remix_stage1_sds.yaml`
- **Dataset:** Full training shards with all metadata
- **Purpose:** Teach the model to respond to all types of conditioning

### Stage 2: Fine-tuning Without Genre
- **Conditions:** lyrics, style, melody_chroma, structure (genre EXCLUDED)
- **Config:** `train_styletransfer_stage2_large_b200.yaml`
- **Purpose:** Improve style transfer quality without relying on genre labels
- **Typically starts from a Stage 1 checkpoint**

---

## 7. Neuron-Specific Training Features

### 7.1 Device Management

- Model moved to Neuron device: `model.to(device="neuron", dtype=torch.bfloat16)`
- No `torch.cuda.*` calls — all replaced with Neuron equivalents
- `torch.neuron.synchronize()` used for timing accuracy (Neuron has async execution)
- `torch.neuron.set_device(local_rank)` for multi-core setups

### 7.2 torch.compile with Neuron Backend

- Optional compilation: `torch.compile(model, backend="neuron")`
- Neuron compiler optimizes the computation graph
- Actual compilation happens on first forward pass (lazy compilation)
- Produces optimized execution plans for NeuronCores

### 7.3 FSDP (Fully Sharded Data Parallelism)

**Purpose:** Distribute model weights across multiple NeuronCores when the model doesn't fit on one core.

**Strategy:**
- Only **LlamaDecoderLayer** modules are wrapped with FSDP
- Embeddings (nn.Embedding) are **NOT** wrapped — FSDP flattens params to 1D which breaks Neuron's `torch.embedding` dispatch (requires 2D weights)
- Decoder layers hold ~95% of parameters, so sharding them is sufficient

**Configuration:**
- Mixed precision: BF16 param/buffer, FP32 reduce
- Sharding strategy: `SHARD_GRAD_OP` (shard gradients and optimizer state)
- `use_orig_params=True` for compatibility

**Manual Gradient All-Reduce:**
- Parameters NOT inside FSDP-wrapped layers (embeddings, norms, lm_heads) need manual gradient synchronization
- After `loss.backward()`, a manual `dist.all_reduce(p.grad, op=ReduceOp.AVG)` is called for these parameters

**Launch Command:**
```
NEURON_RT_VIRTUAL_CORE_SIZE=2
NEURON_RT_NUM_CORES=4
torchrun --nproc_per_node=4 --rdzv_backend=c10d --rdzv_endpoint=localhost:29500
    train_neuron.py --full --fsdp --grad-checkpoint
```

### 7.4 Gradient Checkpointing

- Enabled via `--grad-checkpoint` flag
- Uses `torch.utils.checkpoint` (not PyTorch Lightning)
- Each decoder layer is checkpointed: forward activations recomputed during backward instead of stored
- Saves ~40-60% memory at the cost of ~30-40% slower training
- Essential for fitting the full 36+12 layer model on device

### 7.5 NKI Custom Kernels

#### Fused RoPE (Rotary Position Embeddings)

**Purpose:** Compute rotary position embeddings more efficiently than the PyTorch reference implementation.

**Logic:**
- Standard RoPE: `q_embed = (q * cos) + (rotate_half(q) * sin)` (requires negation, assembly, multiple ops)
- Fused NKI kernel: Uses subtract/add on halves directly (saves 3 ops per tile)
- Loads cos/sin frequency tables ONCE per sequence tile, reuses across all attention heads
- Coalesced DMA with 8-tile batching for efficient memory access
- Native [Batch, Heads, Sequence, Dimension] layout — no flatten/broadcast overhead

**Backward Pass:**
- Implemented via rotation reversal (sin is negated)
- Registered with PyTorch autograd via `register_autograd`

**Validation:**
- Sequence length must be a multiple of 128 × LNC (Logical NeuronCore count)
- Head dimension validated against cos/sin tensor shapes
- Batch size in cos/sin must match query batch size

#### Fused Cross-Entropy

**Purpose:** Compute cross-entropy loss more efficiently for large vocabularies (16,385 tokens).

**Logic:**
- **Forward:** Online log-sum-exp algorithm for numerically stable softmax computation
  - Processes vocabulary in chunks (configurable chunk size, up to 65,536)
  - Each chunk: load logits tile → compute local max → update running max → accumulate exp-sum
  - Final loss: `log_sum_exp - logit_at_target`
  - Memory allocation stays within 24 MiB per kernel invocation
  
- **Backward:** Explicit gradient computation
  - `grad = grad_scale × (softmax(logits) - one_hot(target))`
  - Chunked processing matches forward pass
  - LSE (log-sum-exp) state passed from forward to avoid recomputation
  - Supports both "mean" and "sum" reduction modes

**Constraints:**
- Logits must be 2D: [num_positions, vocab_size]
- Targets must be 1D int32
- Positions per batch ≤ 128 (P_MAX)
- Chunk size ≤ 65,536 (F_MAX)

---

## 8. CUDA-to-Neuron Migration Details

### 8.1 What Was Replaced

| CUDA Component | Neuron Replacement |
|---------------|-------------------|
| `torch.cuda.*` calls | `torch_xla.*` / Neuron device APIs |
| `accelerate` device placement | `xm.xla_device()` / `torch.device("neuron")` |
| DeepSpeed ZeRO-2 | `neuronx_distributed` / FSDP |
| Flash Attention | Standard SDPA / Neuron-compatible kernels |
| `torch.cuda.amp.autocast` | `torch.autocast(device_type="neuron", dtype=torch.bfloat16)` |
| bitsandbytes Adam8bit | Standard `torch.optim.Adam` |
| CUDA streams (for offloading) | Neuron async execution + threading |
| `torch.cuda.empty_cache()` | Not needed on Neuron (automatic) |
| Pinned memory (`pin_memory`) | Not applicable on Neuron |

### 8.2 Key Differences in Neuron

1. **No CUDA streams:** Offloading uses Python threading + condition variables instead
2. **No explicit cache management:** Neuron runtime handles memory automatically
3. **BF16 preferred:** Neuron hardware natively supports BF16 (not FP16)
4. **XLA compilation:** Optional `torch.compile(backend="neuron")` for graph optimization
5. **Embedding handling:** `nn.Embedding` must keep 2D weight tensors (not flattened by FSDP)
6. **Synchronization:** Explicit `torch.neuron.synchronize()` needed for timing

### 8.3 Validation Criteria

- Training loss convergence must match CUDA baseline within ±5% at 1,000 steps
- No NaN or Inf gradients during training
- Memory usage within 512 GB DRAM budget on trn1.32xlarge
- NeuronCore utilization exceeds 70%

---

## 9. Training Configurations

### 9.1 Model Size Configurations

| Config | Layers (Primary + Secondary) | Parameters | Use Case |
|--------|------------------------------|-----------|----------|
| Tiny | 2 + 1 | ~50M | CPU debugging only |
| Small | 4 + 2 | ~500M | Fast iteration, NeuronCore validation |
| Full | 36 + 12 | ~5B | Production training |

### 9.2 Hyperparameters

| Parameter | Default | Range |
|-----------|---------|-------|
| Learning rate | 1e-4 | 1e-5 to 1e-3 |
| Batch size | 2 | 1-8 (limited by memory) |
| Sequence length | 128-512 (training), up to 8196 (full) | Must be divisible by 128 |
| Precision | BF16 | BF16 or FP32 |
| Gradient checkpointing | Enabled for full model | On/Off |
| FSDP world size | 4 (trn2.3xlarge) to 32 (trn1.32xlarge) | 1-32 |

### 9.3 Training Modes

| Mode | Description | Typical Use |
|------|-------------|-------------|
| Full Retraining | All parameters updated | Production model (larger runs, 10K+ steps) |
| LoRA Fine-tuning | Low-rank adapters only | Iterative experimentation |

---

## 10. Data Loading (On Trainium)

### 10.1 Streaming Data Service (SDS)

- Training data streamed from S3 using SDS (vendored in `third_party/sds`)
- WebDataset tar shards loaded on-the-fly
- No need to download entire dataset to local disk
- Supports S3 path patterns: `s3://bucket/data/train-{000000..000100}.tar`

### 10.2 Per-Sample Content

Each training sample from a tar shard contains:
- Pre-encoded audio tokens (3 codebook streams: mixed, vocal, BGM)
- Lyrics tokens (pre-tokenized with Qwen2-7B)
- Genre features (from DaSheng classifier)
- Style embedding vector
- Melody chromagram features ([T, 12] tensor)
- Structure labels (section boundaries and types)
- Language metadata (for language filtering)

---

## 11. Checkpointing

| Aspect | Details |
|--------|---------|
| Frequency | Every 500 steps (configurable) |
| Retention | 3 most recent checkpoints locally |
| Storage | S3: `s3://soundry-checkpoints/{run-id}/step_{N}.pt` |
| Manifest | `checkpoint_manifest.json` with step, epoch, loss, timestamp |
| Resume | `--resume CHECKPOINT_PATH` flag on launch script |
| Spot Recovery | On SageMaker spot interruption, resumes from latest S3 checkpoint |

---

## 12. Monitoring During Training

### CloudWatch Metrics (Namespace: `SoundryAI/Training`)
- Training loss (per step)
- Validation loss (periodic)
- Learning rate schedule
- Tokens per second (throughput)
- NeuronCore utilization percentage
- HBM memory usage percentage
- Step time (seconds)
- Gradient norm

### Console Output (Per Step)
```
  Step         Loss       Time      Tok/s
----------------------------------------------
     0       9.6872      3.45s      2234
     1       9.6451      1.82s      4231
     2       9.5987      1.78s      4326
   ...
```

### Neuron-Specific Monitoring
- `neuron-top`: Real-time NeuronCore utilization
- `neuron-monitor`: Detailed hardware metrics (memory bandwidth, DRAM usage)
- Runtime logs streamed to CloudWatch Logs

---

## 13. Training Launch Scripts

### Stage 1 Training
```bash
./train_stage1.sh [--resume CHECKPOINT_PATH]
```
- Sets PYTHONPATH for Flow1dVAE imports
- Configures PyTorch memory allocator (`expandable_segments:True`)
- Runs `train.py --config ./config/train_remix_stage1_sds.yaml`

### FSDP Multi-Core Training (Neuron)
```bash
NEURON_RT_VIRTUAL_CORE_SIZE=2 NEURON_RT_NUM_CORES=4 \
torchrun --nproc_per_node=4 --rdzv_backend=c10d --rdzv_endpoint=localhost:29500 \
    train_neuron.py --full --fsdp --grad-checkpoint --steps 20 --batch-size 2
```

### Evaluation
```bash
python eval.py \
  --ckpt_path step11000.ckpt \
  --train_config_path config/train_styletransfer_stage2_large_b200.yaml \
  --model_config_path config/songgeneration_large_march14.yaml \
  --include_conditions melody_chroma style lyrics \
  --genre_style_mode style
```

---

## 14. SageMaker Training Container

### Docker Image Contents
- Base: AWS Neuron Deep Learning Container
- PyTorch + `torch-neuronx` + `neuronx-distributed` + `neuronx-cc`
- Training scripts (`train.py`, `train_flow.py`, all model code)
- Tokenizer assets (Qwen2-7B files)
- Audio codec assets (Flow1dVAE, XCodec)
- SDS streaming client
- Entry point: reads SageMaker environment variables (`SM_HP_*`, `SM_CHANNEL_TRAINING`)

### SageMaker Integration
- Data channels: `SM_CHANNEL_TRAINING` → S3 path for training data
- Hyperparameters: `SM_HP_*` → learning rate, batch size, etc.
- Output: `SM_OUTPUT_DATA_DIR` → checkpoints + logs
- Metrics: emitted to CloudWatch custom namespace

---

## 15. Evaluation Metrics

The model is evaluated on:

| Metric | Description |
|--------|-------------|
| PER (↓) | Pronunciation Error Rate — measures lyric clarity |
| CE (↑) | Content Emotion — how well emotional content is conveyed |
| CU (↑) | Content Understanding — semantic accuracy |
| PC (↑) | Production Clarity — audio engineering quality |
| PQ (↑) | Production Quality — overall production value |
| COH (↑) | Coherence — musical flow and consistency |
| MUS (↑) | Musicality — how musical the output sounds |
| MEM (↑) | Memorability — catchiness factor |
| CLA (↑) | Clarity — overall vocal/instrumental clarity |
| NAT (↑) | Naturalness — how natural/human the output sounds |

**Benchmark Performance (SongGeneration-large, English):**
- PER: 14.9% (competitive with commercial systems like Suno at 15.6%)
- Production Quality: 8.46 (highest among open-source models)
- Musicality: 3.94 (second only to Suno at 4.35)
