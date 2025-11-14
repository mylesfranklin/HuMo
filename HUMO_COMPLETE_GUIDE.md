# HuMo: Complete Getting Started & Maximization Guide

**Last Updated**: November 2025
**Project**: HuMo (Human-Centric Video Generation via Collaborative Multi-Modal Conditioning)
**Version**: 1.0

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [What is HuMo?](#what-is-humo)
3. [Ecosystem & Dependencies](#ecosystem--dependencies)
4. [Getting Started](#getting-started)
5. [Architecture Deep Dive](#architecture-deep-dive)
6. [Using HuMo](#using-humo)
7. [Configuration Guide](#configuration-guide)
8. [Advanced Usage & Optimization](#advanced-usage--optimization)
9. [Troubleshooting & FAQ](#troubleshooting--faq)
10. [Best Practices](#best-practices)

---

## Executive Summary

HuMo is a state-of-the-art deep learning framework for **human-centric video generation** that synthesizes coherent, audio-synchronized videos from multimodal inputs (text descriptions, reference images, and audio). It combines:

- **17B-parameter and 1.7B-parameter models** for different hardware capabilities
- **Multi-GPU distributed inference** using FSDP + Sequence Parallel
- **Audio-visual synchronization** to ensure generated motions match audio input
- **Subject preservation** to maintain consistent character appearance across videos
- **Multiple input modalities**: Text-Audio (TA) or Text-Image-Audio (TIA) modes

**Key Stats:**
- Output: 97 frames @ 25 FPS (3.88 seconds)
- Resolutions: 480P (832×480) or 720P (1280×720)
- Hardware: HuMo-17B requires 8× GPUs (80GB+ VRAM); HuMo-1.7B runs on 32GB GPU
- Generation Time: ~15-30 min (17B, 720P) or ~8 min (1.7B, 480P)

---

## What is HuMo?

### Project Overview

HuMo stands for "Human-Centric Video Generation via Collaborative Multi-Modal Conditioning." It's developed by **Tsinghua University** and **ByteDance Research** and represents the state-of-the-art in controllable human video synthesis.

### Key Capabilities

#### 1. **Text-Audio to Video (TA Mode)**
- **Input**: Text description + Audio file
- **Output**: Audio-synchronized human video
- **Use Cases**:
  - Generating character-driven dialogue scenes
  - Creating music videos with choreographed motion
  - Producing AI-generated performances
- **Advantage**: No need for reference image; enables creative freedom

Example:
```
Text: "A woman in formal business attire stands confidently in a modern office,
       gesturing as she presents ideas. She speaks clearly and with purpose."
Audio: interview_audio.wav (2-3 minute audio track)
→ Output: Video of woman synchronized to audio speech/presentation
```

#### 2. **Text-Image-Audio to Video (TIA Mode)**
- **Input**: Text description + Reference image + Audio file
- **Output**: Character-consistent, audio-synchronized video
- **Use Cases**:
  - Creating realistic videos of real people from photos
  - Generating character-driven content with specified appearance
  - Consistency-critical applications (movie production, advertising)
- **Advantage**: Maximum control over character appearance

Example:
```
Text: "The man walks through a forest, looking around cautiously"
Image: reference_photo.jpg (photo of a specific person)
Audio: forest_ambience.wav + speech
→ Output: Video of that specific person in the described scenario
```

#### 3. **Text-Image to Video (TI Mode - Coming Soon)**
- **Input**: Text description + Reference image
- **Output**: Silent video of character in scene
- **Use Cases**: Visual content without audio requirements

---

## Ecosystem & Dependencies

### Technology Stack Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        HuMo Application                          │
└────────────────┬────────────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┬──────────────┬──────────────┐
    ▼            ▼            ▼              ▼              ▼
┌─────────┐ ┌────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐
│ PyTorch │ │ Config │ │ Audio    │ │ Vision     │ │ Video    │
│ 2.5.1   │ │Manager │ │Processing│ │Transformer │ │ Encoder  │
│ (DL)    │ │        │ │(Whisper) │ │(CLIP, T5)  │ │(VAE)     │
└─────────┘ └────────┘ └──────────┘ └────────────┘ └──────────┘
    │
    ├─ Distributed: FSDP, Sequence Parallel
    ├─ Efficiency: Flash Attention 2/3
    └─ Hardware: CUDA 12.4

├─────────────────────┬─────────────────────────┤
▼                     ▼                         ▼
GPU Computation    Model Weights           Pre-trained Models
(NVIDIA CUDA)    (HuggingFace Hub)       (T5, CLIP, Whisper)
```

### Core Libraries

| Category | Library | Version | Purpose |
|----------|---------|---------|---------|
| **Deep Learning** | PyTorch | 2.5.1 | Core tensor operations |
| | torchvision | 0.20.1 | Image processing |
| | torchaudio | 2.5.1 | Audio processing |
| **Generative AI** | diffusers | ≥0.31.0 | Diffusion pipeline |
| | transformers | ≥4.49.0 | Pre-trained models (T5, CLIP) |
| **Distributed** | accelerate | ≥1.1.1 | Multi-GPU training |
| | xfuser | ≥0.4.1 | Distributed attention |
| **Performance** | flash_attn | 2.6.3 | Efficient attention computation |
| **Media** | moviepy | 1.0.3 | Video compositing with audio |
| | imageio-ffmpeg | - | Video I/O (FFmpeg wrapper) |
| | librosa | - | Audio analysis |
| **Audio** | audio-separator | 0.24.1 | Vocal/background separation |
| | onnxruntime-gpu | - | Inference optimization |
| **Config** | omegaconf | - | YAML-based configuration |
| **Optional UI** | gradio | ≥5.0.0 | Web interface (if needed) |

### External Resources

#### Pre-trained Model Weights

All models are hosted on HuggingFace Hub:

| Model | Size | Download | Purpose |
|-------|------|----------|---------|
| **HuMo-17B** | ~34GB | [Link](https://huggingface.co/bytedance-research/HuMo/tree/main/HuMo-17B) | Main 17B diffusion model |
| **HuMo-1.7B** | ~3.4GB | [Link](https://huggingface.co/bytedance-research/HuMo/tree/main/HuMo-1.7B) | Lightweight variant |
| Wan2.1-T2V | ~2.5GB | [Link](https://huggingface.co/Wan-AI/Wan2.1-T2V-1.3B) | VAE + base T2V model |
| T5-XXL (UMT5) | ~15GB | [Link](https://huggingface.co/google/umt5-xxl) | Text encoder |
| Whisper-large-v3 | ~3GB | [Link](https://huggingface.co/openai/whisper-large-v3) | Audio encoder |
| Kim_Vocal_2 | ~500MB | [Link](https://huggingface.co/huangjackson/Kim_Vocal_2) | Audio separator (optional) |

**Total Storage Required**: ~60-75GB for all models + weights

#### Online Platforms

| Platform | URL | Purpose |
|----------|-----|---------|
| **HuggingFace Space** | [HuMo Demo](https://huggingface.co/spaces/alexnasa/HuMo_local) | Web UI for quick testing |
| **OpenBayes Playground** | [Tutorial](https://openbayes.com/console/public/tutorials/KhniTI5hwrf) | Free GPU hours for testing |
| **Project Page** | [HuMo Visualizations](https://phantom-video.github.io/HuMo/) | Example outputs, paper, resources |
| **ArXiv Paper** | [2509.08519](https://arxiv.org/abs/2509.08519) | Technical details |

#### Integration Platforms

- **ComfyUI**: [HuMo Support](https://blog.comfy.org/p/humo-and-chroma1-radiance-support) - Node-based workflow
- **ComfyUI-Wan**: [GitHub](https://github.com/kijai/ComfyUI-WanVideoWrapper) - Runs on RTX 3090

---

## Getting Started

### Step 1: Environment Setup

#### 1.1 Clone Repository

```bash
# Clone the repository
cd ~/projects  # or your preferred location
git clone https://github.com/Phantom-video/HuMo.git
cd HuMo
```

#### 1.2 Create Conda Environment

```bash
# Create isolated Python 3.11 environment
conda create -n humo python=3.11
conda activate humo

# Verify Python version
python --version  # Should show Python 3.11.x
```

#### 1.3 Install PyTorch with CUDA 12.4 Support

```bash
# Install PyTorch for CUDA 12.4
pip install torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 \
  --index-url https://download.pytorch.org/whl/cu124

# Verify CUDA availability
python -c "import torch; print(f'CUDA Available: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0)}')"
```

#### 1.4 Install Flash Attention (Critical for Performance)

```bash
# Flash Attention 2 - Essential for GPU memory efficiency
pip install flash_attn==2.6.3

# Note: Flash Attention compilation requires:
# - CUDA 12.0+
# - nvcc (NVIDIA CUDA compiler) in PATH
# - If compilation fails, check your CUDA installation
```

#### 1.5 Install Project Dependencies

```bash
# Install all Python dependencies
pip install -r requirements.txt

# Install FFmpeg (required for video processing)
conda install -c conda-forge ffmpeg

# Verify FFmpeg installation
ffmpeg -version
```

#### 1.6 Verify Installation

```bash
# Test PyTorch + CUDA
python -c "
import torch
print(f'PyTorch Version: {torch.__version__}')
print(f'CUDA Available: {torch.cuda.is_available()}')
print(f'GPU Count: {torch.cuda.device_count()}')
if torch.cuda.is_available():
    for i in range(torch.cuda.device_count()):
        print(f'  GPU {i}: {torch.cuda.get_device_name(i)}')
"

# Test imports
python -c "
from humo.generate import Generator
from humo.generate_1_7B import Generator as Generator_1_7B
print('✓ HuMo imports successful')
"
```

### Step 2: Download Model Weights

#### 2.1 Create Weights Directory

```bash
# Create directory structure for models
mkdir -p weights/{HuMo,Wan2.1-T2V-1.3B,whisper-large-v3,audio_separator}

# Verify structure
tree weights/ -L 1
```

#### 2.2 Install HuggingFace CLI (if not already installed)

```bash
pip install huggingface-hub
huggingface-cli --version

# Optional: Login to HuggingFace (for faster downloads)
huggingface-cli login
```

#### 2.3 Download Models

```bash
# Base models (Required)
huggingface-cli download Wan-AI/Wan2.1-T2V-1.3B \
  --local-dir ./weights/Wan2.1-T2V-1.3B

# HuMo checkpoints (Required)
huggingface-cli download bytedance-research/HuMo \
  --local-dir ./weights/HuMo

# Audio encoder (Required)
huggingface-cli download openai/whisper-large-v3 \
  --local-dir ./weights/whisper-large-v3

# Audio separator (Optional - for background noise removal)
huggingface-cli download huangjackson/Kim_Vocal_2 \
  --local-dir ./weights/audio_separator
```

**Note**: Total download size ~60GB. On slower connections, expect 1-3 hours.

#### 2.4 Verify Downloaded Files

```bash
# Check model directory sizes
du -sh weights/*

# Expected output:
# 3.4G  weights/HuMo/HuMo-1.7B (if downloaded)
# 34G   weights/HuMo/HuMo-17B (if downloaded)
# 2.5G  weights/Wan2.1-T2V-1.3B
# 3.0G  weights/whisper-large-v3
# 0.5G  weights/audio_separator
```

### Step 3: Prepare Input Data

#### 3.1 Prepare Text Prompts

Create a JSON file with detailed text descriptions:

```json
{
    "character_name": {
        "prompt": "A detailed, vivid description of the character and scene..."
    }
}
```

**Example** (`examples/my_test.json`):
```json
{
    "office_presentation": {
        "prompt": "A professional woman in business attire stands in a modern corporate office. She wears a navy blazer and white blouse. She gestures expressively while speaking, with confident body language. The office has large windows with city views in the background. She maintains eye contact with the camera as she presents ideas clearly."
    },
    "forest_explorer": {
        "prompt": "A bearded man with weathered features wearing an olive-green jacket navigates through a dense forest. He moves carefully, looking around cautiously as if exploring unfamiliar terrain. Sunlight filters through the canopy above. His expression shows focus and determination."
    }
}
```

**Prompt Tips**:
- Be **specific and detailed** about appearance, clothing, environment
- Include **emotional context** (confident, nervous, excited)
- Describe **body language and movements** expected
- Mention **camera perspective** and framing
- Use **color and texture descriptions**

#### 3.2 Prepare Reference Images (for TIA mode)

**For Text-Image-Audio (TIA) mode**, provide reference images:

```bash
# Supported formats: PNG, JPG, BMP
# Recommended specs:
# - Resolution: 512×512 to 1024×1024 pixels
# - Format: PNG or JPG
# - Size: <5MB per image
# - Content: Clear, well-lit frontal face/full body shot

# Create directory
mkdir -p input_data/reference_images

# Add your reference images
cp /path/to/person_photo.jpg input_data/reference_images/
```

**Image Tips**:
- Use **clear, well-lit photos** for best results
- **Frontal or semi-profile** poses work best
- Include **full body** or at least chest-up views
- Avoid **extreme angles** or partial occlusion
- Higher resolution improves consistency

#### 3.3 Prepare Audio Files

**For Text-Audio (TA) and Text-Image-Audio (TIA) modes**:

```bash
# Supported formats: WAV, MP3, FLAC
# Recommended specs:
# - Duration: 60-120 seconds (3-5 minutes max)
# - Sample rate: 16kHz or 48kHz
# - Channels: Mono or Stereo
# - Bitrate: 128-320 kbps

# Create directory
mkdir -p input_data/audio

# Convert audio to supported format (if needed)
ffmpeg -i original_audio.m4a -ar 16000 -ac 1 output_audio.wav
```

**Audio Tips**:
- **Speech-driven**: Dialogue, presentations, narration work best
- **Music-driven**: Instrumental or beat-synchronized content
- **Clean audio**: Low background noise improves sync
- **Pacing**: Match speech pace to intended motion
- **Duration**: 3-5 minutes for coherent narrative

---

## Architecture Deep Dive

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Input                               │
│         (Text Prompt, Reference Image, Audio)                   │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │   Input Processing Layer    │
        ├─────────────────────────────┤
        │ • Text Tokenization (T5)    │
        │ • Image Encoding (VAE)      │
        │ • Audio Feature Extraction  │
        │   (Whisper-large-v3)        │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │   Conditioning Embedding     │
        ├─────────────────────────────┤
        │ • Text embeddings (2560-D)  │
        │ • Image latents (4-D)       │
        │ • Audio embeddings (1536-D) │
        │ Projected to unified space  │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │   Diffusion Model (DiT)      │
        ├─────────────────────────────┤
        │  Diffusion Transformer      │
        │  • 14B/1.3B parameters      │
        │  • 40/30 layers             │
        │  • RoPE positional encoding │
        │  • Flash Attention 2/3      │
        │  • 50-step denoising        │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │     Latent Space Video      │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │   VAE Decoder               │
        │   (Pixel Space Conversion)  │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │   Video Encoder             │
        ├─────────────────────────────┤
        │ • Audio re-sync (moviepy)   │
        │ • H.264 encoding            │
        │ • MP4 container format      │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │    Output Video (MP4)       │
        │  480P/720P × 97 frames @25fps
        └─────────────────────────────┘
```

### Core Model Components

#### 1. **Text Encoder** (T5-XXL)
- **Input**: Text prompt (up to 512 tokens)
- **Architecture**: Sequence-to-sequence transformer
- **Output**: 2560-dimensional embeddings
- **Role**: Converts text descriptions to semantic vectors
- **Language**: Supports multiple languages via UMT5

#### 2. **Audio Processor** (Whisper-large-v3)
- **Input**: Audio file (.wav, .mp3, .flac)
- **Processing**:
  - Extracts audio features every frame
  - Generates 768-dimensional embeddings per frame
  - Synchronized to video frame rate (25 FPS)
- **Features**:
  - Speech recognition-level audio understanding
  - Robust to background noise
  - Multilingual support
- **Output**: Sequence of audio embeddings

#### 3. **Audio Projection Module**
- **Input**: Whisper audio embeddings (768-D)
- **Network**: 13 transformer blocks
- **Output**: 1536-dimensional features aligned to video latent space
- **Purpose**: Bridge audio space to visual generation space

#### 4. **Image Encoder** (VAE)
- **Input**: Reference image (for TIA mode)
- **Function**: Encodes image to latent space
- **Stride**: [4, 8, 8] (temporal, height, width)
- **Scaling Factor**: 0.9152 (for stable diffusion)
- **Role**: Provides appearance consistency conditioning

#### 5. **Diffusion Transformer (DiT)**
- **Base Model**: Wan-AI 2.1 architecture
- **Two Variants**:
  - **HuMo-17B**: 40 layers, 5120-D, 40 attention heads
  - **HuMo-1.7B**: 30 layers, 1536-D, 12 attention heads
- **Key Features**:
  - Flow matching diffusion (smooth, stable generation)
  - Rotary position embeddings (RoPE)
  - QK-normalization (training stability)
  - Gradient checkpointing (memory efficiency)
  - Flash Attention 2/3 support
- **Input**: Noisy latent video + conditioning
- **Output**: Denoised video latents
- **Process**: 50-step iterative denoising (Euler sampler)

#### 6. **VAE Decoder**
- **Input**: Denoised latent video
- **Function**: Upsamples from latent to pixel space
- **Output**: RGB video (480P or 720P)
- **Causal Structure**: Maintains temporal coherence

### Distributed Inference Strategy

For the 17B model on 8 GPUs:

```
8 GPUs with FSDP + Sequence Parallel:

┌─────────┬─────────┬─────────┬─────────┐
│ GPU 0   │ GPU 1   │ GPU 2   │ GPU 3   │  FSDP Sharding
│ DiT-1   │ DiT-1   │ DiT-1   │ DiT-1   │  (Model Split)
├─────────┼─────────┼─────────┼─────────┤
│ GPU 4   │ GPU 5   │ GPU 6   │ GPU 7   │  Sequence Parallel
│ DiT-2   │ DiT-2   │ DiT-2   │ DiT-2   │  (Time Chunks)
└─────────┴─────────┴─────────┴─────────┘

• FSDP: Horizontal model sharding (ZeRO-2)
• Sequence Parallel: Vertical temporal sharding (Ulysses)
• Communication: Ring AllReduce for gradients
• Memory: ~10GB per GPU (vs 40GB single GPU)
```

---

## Using HuMo

### Quick Start: Running Your First Generation

#### Option A: Text-Audio Mode (TA)

Generate video from text + audio without reference image:

```bash
# Activate environment
conda activate humo

# Run 17B model (requires 8 GPUs)
bash scripts/infer_ta.sh

# Or run 1.7B model (single GPU, 32GB)
bash scripts/infer_ta_1_7B.sh
```

**What happens**:
1. Loads `humo/configs/inference/generate.yaml`
2. Reads test case from `examples/test_case.json`
3. Launches 8 torchrun processes (17B) or 1 process (1.7B)
4. Performs 50-step diffusion sampling
5. Saves video to `output/` directory

**Output Files**:
```
output/
├── case_1.mp4          # Generated video
├── case_1_info.txt     # Generation metadata
└── case_1.log          # Generation log
```

#### Option B: Text-Image-Audio Mode (TIA)

Generate video with character consistency:

```bash
# Edit configuration to include image path
nano humo/configs/inference/generate.yaml
# Set: generation.mode=TIA

# Run 17B model
bash scripts/infer_tia.sh

# Or 1.7B model
bash scripts/infer_tia_1_7B.sh
```

### Advanced: Custom Configuration

#### Configuration File: `humo/configs/inference/generate.yaml`

Key parameters to customize:

```yaml
generation:
  # Video properties
  frames: 97                    # Number of frames (max ~97 recommended)
  fps: 25                       # Frames per second
  height: 720                   # 720 or 480
  width: 1280                   # 1280 (720p) or 832 (480p)

  # Guidance strengths (higher = stronger adherence)
  scale_t: 5.0                  # Text guidance strength [3-8]
  scale_a: 5.5                  # Audio guidance strength [3-8]
  scale_i: 4.0                  # Image guidance (TIA mode) [2-5]

  # Sampling
  seed: 666666                  # Reproducibility seed
  batch_size: 1                 # Videos per batch
  mode: "TIA"                   # "TA" or "TIA"

  # Input/Output paths
  positive_prompt: ./examples/test_case.json
  output:
    dir: ./output

  # Input files (can override at command line)
  sample_neg_prompt: '色调艳丽,过曝,静态,...'  # Negative prompt

diffusion:
  # Sampling configuration
  sampler:
    type: euler                 # Sampling algorithm
    prediction_type: v_lerp     # Velocity prediction

  timesteps:
    sampling:
      type: uniform_trailing
      steps: 50                 # Denoising steps [30-50]
      shift: 5.0

dit:
  # Model configuration
  sp_size: 8                    # Sequence parallel size = # GPUs

  # Checkpoint paths
  checkpoint_dir: ./weights/HuMo/HuMo-17B
  zero_vae_path: ./weights/HuMo/zero_vae_129frame.pt
  zero_vae_720p_path: ./weights/HuMo/zero_vae_720p_161frame.pt

  # Optimization
  gradient_checkpoint: True     # Save memory
  compile: False                # torch.compile (experimental)

  fsdp:
    sharding_strategy: _HYBRID_SHARD_ZERO2  # Multi-GPU strategy
```

#### Command-Line Parameter Override

Override config parameters without editing YAML:

```bash
# Generate with custom text
python main.py humo/configs/inference/generate.yaml \
  generation.mode=TA \
  generation.height=480 \
  generation.frames=97 \
  generation.scale_a=7.0

# Switch between models
python main.py humo/configs/inference/generate.yaml \
  dit.checkpoint_dir=./weights/HuMo/HuMo-1.7B \
  dit.sp_size=1

# Adjust output directory
python main.py humo/configs/inference/generate.yaml \
  generation.output.dir=./my_outputs

# Faster generation (fewer steps, lower quality)
python main.py humo/configs/inference/generate.yaml \
  diffusion.timesteps.sampling.steps=30 \
  generation.seed=12345
```

### Creating Custom Prompts

#### JSON Format

```json
{
    "scene_name": {
        "img_paths": ["path/to/image.jpg"],  // Optional, for TIA mode
        "audio_path": "path/to/audio.wav",   // Required for TA/TIA
        "prompt": "Your detailed text description here"
    }
}
```

#### Complete Example

```json
{
    "interview_scenario": {
        "img_paths": ["./input_images/professional.jpg"],
        "audio_path": "./input_audio/interview.wav",
        "prompt": "A confident professional woman with shoulder-length blonde hair wearing a dark business blazer over a light blouse sits at a desk. She faces the camera with a friendly expression, occasionally nodding and gesturing as she speaks. Natural light streams from a window behind her. The office background shows bookshelves and a modern desk setup. She maintains good posture and makes periodic hand gestures to emphasize points."
    },

    "dance_performance": {
        "audio_path": "./input_audio/upbeat_music.wav",
        "prompt": "A young man wearing colorful street-style clothing performs energetic dance moves. He moves fluidly with the beat, incorporating hip movements, arm waves, and footwork. His expression shows enjoyment and concentration. The setting is a bright, open studio space with wooden floors. Warm golden lighting creates depth and emphasizes his movements."
    },

    "narrative_scene": {
        "img_paths": ["./input_images/actor.jpg"],
        "audio_path": "./input_audio/monologue.wav",
        "prompt": "A middle-aged man with graying temples and a weathered face sits on a park bench at dusk. He wears a worn leather jacket and jeans. As he speaks, his eyes focus on a distant point, and he occasionally looks down pensively. He uses subtle hand movements to punctuate his words. The autumn park behind him has golden leaves on trees, and the lighting is warm and cinematographic."
    }
}
```

---

## Configuration Guide

### Understanding YAML Configuration Files

HuMo uses OmegaConf for configuration management. Files are organized hierarchically:

```
configs/
├── inference/
│   ├── generate.yaml          # Main 17B inference config
│   └── generate_1_7B.yaml     # Main 1.7B inference config
└── models/
    ├── Wan_14B.yaml           # 14B model architecture
    ├── Wan_14B_I2V.yaml       # 14B Image-to-Video variant
    ├── Wan_1.3B.yaml          # 1.3B model architecture
    └── Wan_1.3B_I2V.yaml      # 1.3B I2V variant
```

### Configuration Inheritance

Configs support inheritance via `__inherit__`:

```yaml
# generate.yaml
dit:
  model:
    __inherit__: humo/configs/models/Wan_14B_I2V.yaml
    insert_audio: True  # Override inherited value
```

This loads `Wan_14B_I2V.yaml` and then applies overrides.

### Object Creation from Config

Special `__object__` key instantiates Python classes:

```yaml
__object__:
  path: humo.generate        # Python module path
  name: Generator            # Class name
  # Arguments passed to __init__

dit:
  model:
    __object__:
      path: humo.models.wan_modules.model_humo
      name: WanModel
```

Equivalent Python:
```python
from humo.models.wan_modules.model_humo import WanModel
config = load_config(...)
model = WanModel(**config.dit.model)
```

### Key Configuration Parameters

#### Generation Parameters

```yaml
generation:
  frames: 97                    # Video length
  fps: 25                       # Frame rate (fixed)
  height: 720                   # 720 or 480
  width: 1280                   # 1280 or 832
  mode: "TIA"                   # Input mode
  seed: 666666                  # Random seed

  scale_t: 5.0                  # Text strength
  scale_a: 5.5                  # Audio strength
  scale_i: 4.0                  # Image strength

  step_change: 980              # Guidance step threshold
  batch_size: 1                 # Batch size
  sequence_parallel: 8          # # GPUs for sequence parallel
```

#### Diffusion Parameters

```yaml
diffusion:
  schedule:
    type: lerp                  # Schedule type
    T: 1000.0                   # Total timesteps

  sampler:
    type: euler                 # Sampling algorithm
    prediction_type: v_lerp     # Velocity/noise prediction

  timesteps:
    sampling:
      type: uniform_trailing
      steps: 50                 # Denoising iterations
      shift: 5.0
```

#### Model Parameters

```yaml
dit:
  checkpoint_dir: ./weights/HuMo/HuMo-17B
  sp_size: 8                    # Sequence parallel GPUs
  gradient_checkpoint: True     # Memory optimization
  compile: False                # Torch.compile

  fsdp:
    sharding_strategy: _HYBRID_SHARD_ZERO2
```

#### Encoder Parameters

```yaml
text:
  t5_checkpoint: ./weights/Wan2.1-T2V-1.3B/models_t5_umt5-xxl-enc-bf16.pth
  t5_tokenizer: ./weights/Wan2.1-T2V-1.3B/google/umt5-xxl
  dtype: bfloat16
  fsdp:
    enabled: True
    sharding_strategy: HYBRID_SHARD

audio:
  vocal_separator: ./weights/audio_separator/Kim_Vocal_2.onnx
  wav2vec_model: ./weights/whisper-large-v3
```

---

## Advanced Usage & Optimization

### Memory Optimization Strategies

#### Strategy 1: Gradient Checkpointing
```yaml
dit:
  gradient_checkpoint: True   # Save activation memory
```
- Trades compute for memory (~30% slower, 50% less VRAM)
- Recommended for VRAM < 40GB

#### Strategy 2: Sequence Parallelism
```yaml
dit:
  sp_size: 8                  # Distribute across 8 GPUs
```
- Splits temporal dimension across GPUs
- Requires multiple GPUs with high-bandwidth interconnect
- 8 GPUs with sp_size=8 vs 1 GPU: ~8x speedup, 8x memory reduction

#### Strategy 3: Lower Batch Size
```yaml
generation:
  batch_size: 1               # Process one video at a time
```
- Default is optimal (only 1 video at a time)

#### Strategy 4: Reduce Sampling Steps
```yaml
diffusion:
  timesteps:
    sampling:
      steps: 30               # Fast but lower quality
```
- 30 steps: ~50% faster, slight quality loss
- 50 steps: Default quality
- 70+ steps: Better quality but diminishing returns

### Quality Optimization Strategies

#### Strategy 1: Guidance Scale Tuning
```yaml
generation:
  scale_t: 7.0                # Higher: more text adherence
  scale_a: 7.0                # Higher: better audio sync
  scale_i: 5.0                # Higher: more subject consistency
```
- Recommend testing range: 4-8
- Higher values increase prompt following but reduce diversity
- Audio guidance especially important for lip-sync quality

#### Strategy 2: Resolution Trade-offs
```yaml
generation:
  height: 720                 # Higher quality
  # vs
  height: 480                 # Faster generation
```
- 720P: ~2x slower than 480P, 30% more VRAM
- 480P: 8 minutes on 32GB GPU (1.7B)
- 720P: 15-30 minutes on 8× GPUs (17B)

#### Strategy 3: Negative Prompting
```yaml
generation:
  sample_neg_prompt: '色调艳丽,过曝,静态,细节模糊不清...'
```
- Guides model away from undesired features
- Current prompt is in Chinese; customize for your use case

#### Strategy 4: Seed Control for Reproducibility
```yaml
generation:
  seed: 42                    # Same seed = same video
```
- Useful for iterating on prompt changes
- Try multiple seeds (42, 123, 999) for variations

### Using HuMo with ComfyUI

For node-based workflow users:

```bash
# Install ComfyUI (if not already)
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI

# Install HuMo wrapper
cd custom_nodes
git clone https://github.com/kijai/ComfyUI-WanVideoWrapper

# Restart ComfyUI
# HuMo node should now be available
```

**Advantages**:
- Visual node-based interface
- Real-time parameter adjustment
- Runs on RTX 3090 (with optimizations)
- Easy to chain with other video generation nodes

### Using HuMo with HuggingFace Spaces

For quick, browser-based testing:

1. Visit: https://huggingface.co/spaces/alexnasa/HuMo_local
2. Upload reference image (if TIA mode)
3. Upload audio file
4. Write text prompt
5. Adjust parameters via sliders
6. Click "Generate"
7. Download output video

**Free Alternative**: OpenBayes playground provides 3 hours free GPU time

---

## Troubleshooting & FAQ

### Installation Issues

#### **Issue: CUDA not detected**
```
RuntimeError: CUDA is not available
```
**Solution**:
```bash
# Check CUDA installation
nvidia-smi              # Should show GPU info
nvcc --version          # Should show CUDA compiler

# If missing, install CUDA 12.4:
# https://developer.nvidia.com/cuda-12-4-download

# Reinstall PyTorch with correct CUDA version
pip install torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 \
  --index-url https://download.pytorch.org/whl/cu124 --force-reinstall
```

#### **Issue: Flash Attention compilation fails**
```
ModuleNotFoundError: No module named 'flash_attn'
```
**Solution**:
```bash
# Requires CUDA 12.0+ and nvcc in PATH
export CUDA_HOME=/usr/local/cuda-12.4

# Try pre-compiled wheel
pip install flash-attn --no-build-isolation

# Or use CPU fallback (much slower)
# Comment out flash_attn imports in code
```

#### **Issue: Out of Memory (OOM)**
```
RuntimeError: CUDA out of memory
```
**Solution**:
```bash
# For HuMo-17B with 8 GPUs (8GB per GPU minimum):
# Ensure sp_size matches number of GPUs
python main.py ... dit.sp_size=8

# For HuMo-1.7B (single GPU):
# Use generate_1_7B.yaml
python main.py humo/configs/inference/generate_1_7B.yaml

# Reduce to lower resolution
generation.height=480 generation.width=832

# Reduce batch size (though 1 is already minimum)
generation.batch_size=1

# Enable gradient checkpointing
dit.gradient_checkpoint=True
```

### Runtime Issues

#### **Issue: Slow inference (40+ min for 1 video)**
```
[Very slow processing]
```
**Solution**:
```bash
# Check GPU utilization
watch -n 1 nvidia-smi

# If GPU utilization < 50%, check:
# 1. Correct sp_size (should match # GPUs)
# 2. Check PyTorch + Flash Attention installation
# 3. Verify no CPU bottlenecks (check system monitor)

# Reduce sampling steps for faster (lower quality) generation
diffusion.timesteps.sampling.steps=30

# Reduce resolution
generation.height=480
```

#### **Issue: Poor audio-visual synchronization**
```
[Audio and video movements misaligned]
```
**Solution**:
```yaml
# Increase audio guidance strength
generation:
  scale_a: 7.0        # Up from 5.5

# Ensure audio file is:
# - 16kHz sample rate
# - Mono channel
# - Clean without excessive background noise

# Check audio processing
ffmpeg -i audio.wav -af "loudnorm" normalized_audio.wav
```

#### **Issue: Character appearance inconsistency (TIA mode)**
```
[Subject looks different in generated video]
```
**Solution**:
```yaml
# Increase image guidance strength
generation:
  scale_i: 6.0        # Up from 4.0

# Improve reference image:
# - Use well-lit frontal photo
# - Clear facial features
# - Minimal extreme angles or occlusions
# - Higher resolution (512×512+)

# Try different seeds
generation:
  seed: 123           # Try 456, 789, etc.
```

### Output Issues

#### **Issue: Generated video has artifacts**
```
[Blurry, distorted, or corrupted frames]
```
**Solution**:
```bash
# Check video encoding
ffmpeg -i output.mp4 -v verbose 2>&1 | head -20

# Re-encode with H.264
ffmpeg -i corrupted.mp4 -c:v libx264 -preset slow -crf 23 -c:a aac repaired.mp4

# Check input audio for corruption
ffmpeg -i audio.wav -f null -
```

#### **Issue: Video has no audio or audio misaligned**
```
[Silent video or out-of-sync audio]
```
**Solution**:
```bash
# Verify audio file exists and is accessible
ffmpeg -i audio.wav -f null -

# Check audio synchronization in output
ffprobe -v error -show_format -show_streams output.mp4 | grep duration

# Re-mux audio if corrupted
ffmpeg -i silent_video.mp4 -i audio.wav -c:v copy -c:a aac -shortest output_fixed.mp4
```

### FAQ

**Q: What's the difference between 17B and 1.7B models?**
- 17B: Better quality, higher resolution, multi-GPU required
- 1.7B: Faster, lower quality, single 32GB GPU sufficient
- Choose based on hardware availability

**Q: Can I train HuMo on custom data?**
- Training code not yet released
- Only inference supported currently
- Training dataset (Stage-1) available on GitHub for reference

**Q: How long should prompts be?**
- Ideal: 150-300 words
- Minimum: 50 words
- Maximum: 512 tokens (auto-truncated)
- More detail = better results

**Q: What audio formats are supported?**
- WAV, MP3, FLAC, OGG
- Recommended: 16kHz WAV mono
- Duration: 60-120 seconds optimal

**Q: Can I generate longer videos (>97 frames)?**
- Not recommended currently
- Model trained on 97 frames
- Longer generation checkpoint coming Q4 2025

**Q: Is there a REST API?**
- Not official
- HuggingFace Space provides web interface
- Can wrap main.py in FastAPI for custom API

**Q: How do I handle Mandarin prompts?**
- T5-XXL/UMT5 supports multilingual input
- Works with Chinese prompts directly
- Mixed-language prompts also supported

**Q: Can I run on lower-end GPUs?**
- RTX 3090: Possible with optimizations
- RTX 2080: Use 1.7B model with lower resolution
- A10G: Single GPU inference supported
- T4: Not sufficient VRAM (~20GB minimum)

---

## Best Practices

### Prompt Engineering

#### Do's ✓
- **Be specific**: "A woman in a navy blazer" not "A woman in professional clothes"
- **Describe appearance**: Hair color, facial features, clothing details
- **Include emotion**: "Confident expression" or "Nervous gestures"
- **Set the scene**: "Modern office with floor-to-ceiling windows"
- **Detail movements**: "Gestures expressively while speaking"
- **Specify camera**: "Medium shot from chest up"

#### Example High-Quality Prompt
```
A professional woman aged 30-40 with shoulder-length auburn hair, pale complexion,
and sharp cheekbones stands in a modern glass-walled office. She wears a charcoal
gray blazer over a cream silk blouse. Her expression conveys confidence and focus.
As she speaks, she makes deliberate hand gestures to emphasize points. She maintains
steady eye contact with the camera. Natural light from floor-to-ceiling windows
illuminates her face, creating soft shadows. Her posture is straight and professional.
The background shows a blurred cityscape and bookshelf. She occasionally glances to
the side as if reading from a prompter.
```

### Audio Selection

#### Best Practices
1. **Clear speech**: Dialogue with good pronunciation
2. **Appropriate pacing**: Neither too fast nor too slow
3. **Clean audio**: Minimal background noise
4. **Matched emotions**: Audio tone matches visual intent
5. **Professional quality**: 16-bit, 16kHz+ sample rate

#### Audio Preparation Workflow
```bash
# 1. Normalize audio levels
ffmpeg -i original.wav -af "loudnorm" normalized.wav

# 2. Reduce background noise (optional)
ffmpeg -i normalized.wav -af "anlmdn" cleaned.wav

# 3. Convert to recommended format
ffmpeg -i cleaned.wav -acodec pcm_s16le -ar 16000 -ac 1 final.wav

# 4. Verify
ffprobe final.wav
```

### Reference Image Selection

#### Ideal Reference Image Characteristics
- **Frontal or 3/4 angle** (avoid extreme profile)
- **Clear facial features** (good lighting, in-focus)
- **Full body or chest-up** view
- **Neutral or slight smile** expression
- **Simple background** (or will be masked)
- **High resolution** (512×512+)
- **Consistent lighting** (no harsh shadows)

#### Image Preparation Workflow
```bash
# 1. Crop to subject
convert original.jpg -crop 80%x80%+10%+10% cropped.jpg

# 2. Enhance if needed
convert cropped.jpg -enhance enhanced.jpg

# 3. Verify dimensions
identify enhanced.jpg

# 4. Copy to input directory
cp enhanced.jpg ./input_data/reference_images/
```

### Iterative Workflow

Recommended approach for optimal results:

```
1. PLANNING (1-2 hours)
   ├─ Define character details (appearance, personality)
   ├─ Script dialogue/narration if needed
   ├─ Select or record audio
   └─ Find or prepare reference images

2. INITIAL GENERATION (30 mins)
   ├─ Write basic prompt
   ├─ Run with default parameters
   ├─ Evaluate output quality
   └─ Note issues

3. REFINEMENT (1-2 hours)
   ├─ If poor audio-sync: Increase scale_a, check audio
   ├─ If low subject consistency: Increase scale_i, improve reference image
   ├─ If prompt not followed: Increase scale_t, refine text
   ├─ Try different seeds (3-5 variations)
   └─ Select best output

4. FINALIZATION (optional)
   ├─ Post-process video (color grade, add effects)
   ├─ Enhance audio if needed
   └─ Export to desired format
```

### Multi-Video Projects

Generating coherent multi-clip videos:

```bash
# Create directory for project
mkdir project_name
cd project_name

# Organize inputs
mkdir input/{audio,images,prompts}

# Create prompt file with multiple scenes
cat > input/prompts/scenes.json << 'EOF'
{
    "scene_1_intro": { ... },
    "scene_2_main": { ... },
    "scene_3_conclusion": { ... }
}
EOF

# Generate videos
python ../main.py ../humo/configs/inference/generate.yaml \
  generation.positive_prompt=./input/prompts/scenes.json \
  generation.output.dir=./output

# Combine videos
ffmpeg -f concat -safe 0 -i <(for f in output/*.mp4; do echo "file '$PWD/$f'"; done) \
  -c copy final_project.mp4
```

### Performance Monitoring

Monitor generation during inference:

```bash
# Terminal 1: Run generation
conda activate humo
python main.py ...

# Terminal 2: Monitor GPU
watch -n 1 nvidia-smi

# Terminal 3: Monitor system
htop
```

**Healthy Metrics**:
- GPU Utilization: 95-99%
- Memory: 90-95% of available
- Temperature: < 85°C
- Power: Consistent (no spikes)

### Safety & Ethical Considerations

1. **Consent**: Always have permission to generate videos of real people
2. **Misinformation**: Don't use to create misleading content
3. **Copyright**: Respect audio and music copyrights
4. **Harmful Content**: Don't generate violence, harassment, or illegal content
5. **Disclosure**: Disclose AI-generated content in professional contexts

---

## Additional Resources

### Official Links
- **GitHub**: https://github.com/Phantom-video/HuMo
- **Paper**: https://arxiv.org/abs/2509.08519
- **Project Page**: https://phantom-video.github.io/HuMo/
- **HuggingFace Hub**: https://huggingface.co/bytedance-research/HuMo
- **Model Card**: https://huggingface.co/bytedance-research/HuMo

### Related Projects
- **Phantom**: Original human video generation framework
- **SeedVR**: Video representation learning
- **Wan-AI**: Base text-to-video model architecture
- **Whisper**: Audio understanding model

### Community Resources
- **ComfyUI Integration**: https://github.com/kijai/ComfyUI-WanVideoWrapper
- **HuggingFace Space**: https://huggingface.co/spaces/alexnasa/HuMo_local
- **OpenBayes Playground**: Free GPU access for testing

### Citation
If you use HuMo in research or projects, please cite:
```bibtex
@misc{chen2025humo,
      title={HuMo: Human-Centric Video Generation via Collaborative Multi-Modal Conditioning},
      author={Liyang Chen and Tianxiang Ma and Jiawei Liu and Bingchuan Li and Zhuowei Chen and Lijie Liu and Xu He and Gen Li and Qian He and Zhiyong Wu},
      year={2025},
      eprint={2509.08519},
      archivePrefix={arXiv},
      primaryClass={cs.CV},
      url={https://arxiv.org/abs/2509.08519}
}
```

---

## Document Information

| Aspect | Details |
|--------|---------|
| **Guide Version** | 1.0 |
| **Last Updated** | November 2025 |
| **HuMo Version** | Latest (Sep 2025 release) |
| **Coverage** | Installation, Configuration, Usage, Optimization, Troubleshooting |
| **Audience** | Beginners to Advanced Users |
| **Estimated Read Time** | 45-60 minutes (full) or 15 minutes (quick start) |

---

**End of Guide**
