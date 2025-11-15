# Production Video Stack: Implementation Guide
## Hybrid Chaining for AI Music Video Generation

**Status**: Pre-production (Phase 1 Ready)
**Last Updated**: November 2025
**Target Stack**: FastAPI + Modal + PostgreSQL + S3 + Next.js

---

## Table of Contents

1. [Vision & Goals](#vision--goals)
2. [System Architecture](#system-architecture)
3. [Phase Roadmap](#phase-roadmap)
4. [Configuration System](#configuration-system)
5. [API Contract](#api-contract)
6. [Core Pipeline Logic](#core-pipeline-logic)
7. [Runners & Model Integration](#runners--model-integration)
8. [Database Schema](#database-schema)
9. [Deployment on Modal](#deployment-on-modal)
10. [ComfyUI Lab Setup](#comfyui-lab-setup)
11. [First Week Implementation](#first-week-implementation)

---

## Vision & Goals

### One-Sentence Vision
A small web app where you pick a brand, drop in cover art + prompt, choose short or long, and a backend "video brain" chains Wan 2.2/Mochi clips, RIFE and ESRGAN into 9:16 music videos that feel handcrafted.

### Core Objectives

| Objective | Metric | Timeline |
|-----------|--------|----------|
| Generate publication-ready 8-12s vertical clips | 1 video per 5 minutes | Week 1 (Phase 1) |
| Chain clips into 25-30s continuous shots | 1 video per 15-20 minutes | Week 3 (Phase 2) |
| Beat-synchronized multi-shot montages | 1 video per 25 minutes + beat analysis | Week 4 (Phase 3) |
| Production-quality output (720p+, 24-30fps) | Consistent visual quality | Ongoing optimization |

---

## System Architecture

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Interface                            │
│              (Next.js / React Single Page App)                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Brand Dropdown │ Image Upload │ Prompt │ Length Toggle  │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS / WebSocket
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FastAPI Orchestrator                          │
│          (Auth, Validation, Job Management)                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ POST /generate                                           │  │
│  │ GET /jobs/{id}                                           │  │
│  │ GET /brands, /profiles, /status                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│  Dependencies: PostgreSQL, S3, Redis (job queue)               │
└────────────────────────┬────────────────────────────────────────┘
                         │ Job Submission
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │ Modal   │    │ Modal   │    │ Modal   │
    │Function │    │Function │    │Function │
    │ Group A │    │ Group B │    │ Group C │
    └─────────┘    └─────────┘    └─────────┘
        │                │                │
        ▼                ▼                ▼
    ┌────────────────────────────────────────┐
    │      GPU Runners (A100 / H100)         │
    │  ┌──────────┐ ┌──────────┐            │
    │  │ Wan 2.2  │ │  Mochi   │            │
    │  ├──────────┤ ├──────────┤            │
    │  │  RIFE    │ │ ESRGAN   │            │
    │  └──────────┘ └──────────┘            │
    │  ┌──────────────────────┐             │
    │  │  Beat Analyzer (aubio)            │
    │  └──────────────────────┘             │
    └────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │   S3   │ │ Redis  │ │ Postgres
    │Storage │ │ Queue  │ │Database
    └────────┘ └────────┘ └────────┘

┌─────────────────────────────────────────────────────────────────┐
│              ComfyUI Lab (Optional R&D)                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ • Wan 2.2 Nodes        • LongCat Nodes                   │ │
│  │ • FramePack Nodes      • RIFE / ESRGAN Nodes             │ │
│  └───────────────────────────────────────────────────────────┘ │
│  Purpose: Prototype workflows, capture hyperparameters          │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow for Single Generation

```
User Input (brand, prompt, image)
    ↓
[FastAPI] Validate & Create Job
    ↓
[Config] Lookup Brand → Profile → Model
    ↓
[Modal Queue] Submit run_generation(job_id)
    ↓
[Modal Worker] Load Models (Wan 2.2 / Mochi)
    ↓
[Model Runner] Generate Raw Clip
    ↓
[Post-Process] RIFE Interpolation (if enabled)
    ↓
[Post-Process] ESRGAN Upscaling (if enabled)
    ↓
[Upload] Save to S3
    ↓
[Database] Update Job Status + Final URL
    ↓
User Retrieves via GET /jobs/{id} → Download video
```

---

## Phase Roadmap

### Phase 1: Get Generating (Weeks 1-2)
**Goal**: Hit a URL and get 6-12s vertical clips that look good.

**Scope**:
- [ ] Profiles: PREVIEW (8s), HOOK (12s)
- [ ] Models: Wan 2.2 primary, optional Mochi
- [ ] Resolution: 720×1280 (vertical 9:16)
- [ ] FPS: 12-16 (baseline)
- [ ] Optional: RIFE up to 24 fps
- [ ] No chaining (single chunks only)
- [ ] No audio sync yet

**Deliverables**:
- FastAPI app with `/generate` and `/jobs/{id}`
- Wan 2.2 Modal runner for 12s clips
- S3 upload + DB job tracking
- Minimal Next.js UI (brand, prompt, upload, submit)

**Success Criteria**:
- Generate 10-12s 720p vertical clips in <5 minutes
- Consistent visual quality across prompts
- Reliable job tracking and retrieval

### Phase 2: Longshot Chaining (Weeks 3-4)
**Goal**: Single continuous 25-30s vertical clip via sliding-window chaining.

**Scope**:
- [ ] Implement profile: LONGSHOT (30s max)
- [ ] Sliding window chaining logic:
  - 5-6s chunks with 16-frame overlaps
  - Last frame of previous chunk → ref image for next
  - Cross-fade blending at overlaps
- [ ] RIFE upscale to 24-30 fps
- [ ] ESRGAN optional final upscale
- [ ] Prompt variation per chunk (e.g., camera direction)

**Deliverables**:
- `generate_longshot()` orchestrator function
- Chunk stitching + overlap blending logic
- Profile config for LONGSHOT
- Updated UI: length selector (short / long)

**Success Criteria**:
- Generate 25-30s clips with smooth transitions
- No visible seams or jitter at chunk boundaries
- Total generation time: 15-20 minutes

### Phase 3: Beat Sync & MONTAGE (Weeks 5-6)
**Goal**: Videos that "breathe" with music; multi-shot sequences.

**Scope**:
- [ ] Implement profile: MONTAGE (40s max, 4 shots)
- [ ] Beat analyzer:
  - Parse audio → extract BPM, beat times
  - Return beat map (frame indices at 30 fps)
- [ ] Montage planning:
  - Split target duration into N scenes
  - Align scene boundaries to beat timestamps
  - Derive per-scene prompts from master prompt
- [ ] Multi-clip generation + beat-aligned concatenation
- [ ] Optional: simple beat effects (zoom, shake) at transitions

**Deliverables**:
- `beat_analyzer(audio_url)` Modal function
- `montage_planner(prompt, beats, duration)` orchestrator
- Profile config for MONTAGE
- Updated job schema: include beat metadata

**Success Criteria**:
- MONTAGE sequences feel musically motivated
- Beat alignment is perceptually correct
- Supports 35-40s total video length

### Phase 4: Hero Sequences & Lip Sync (Weeks 7-8)
**Goal**: Occasional hero shots with legitimate audio-driven facial sync.

**Scope**:
- [ ] Integrate Wan S2V or LTX-Video for audio-driven generation
- [ ] Use primarily in HOOK / hero MONTAGE scenes
- [ ] Fallback to standard Wan 2.2 for less critical shots
- [ ] Update brand config to specify "hero shot frames"

**Deliverables**:
- S2V / LTX Modal runners
- Brand config extension: hero_scene_indices
- Updated orchestrator to route scenes to appropriate model

**Success Criteria**:
- Hero shots show discernible lip sync
- Seamless blending with non-audio-driven scenes
- <10% additional generation time overhead

---

## Configuration System

### Directory Structure

```
production-stack/
├── config/
│   ├── profiles.yaml         # PREVIEW, HOOK, LONGSHOT, MONTAGE
│   ├── brands.yaml           # Brand definitions + style
│   ├── models.yaml           # Model registry (Wan 2.2, Mochi, etc.)
│   └── samplers.yaml         # Sampling hyperparameters per model
├── app/
│   ├── api/
│   │   ├── main.py           # FastAPI app setup
│   │   ├── routes.py         # Endpoints
│   │   └── auth.py           # Simple auth (if needed)
│   ├── database/
│   │   ├── models.py         # SQLAlchemy ORM models
│   │   ├── connection.py     # DB setup
│   │   └── schemas.py        # Pydantic schemas
│   ├── services/
│   │   ├── job_service.py    # Job CRUD + state machine
│   │   ├── brand_service.py  # Brand lookup + defaults
│   │   └── storage.py        # S3 operations
│   └── runners/
│       ├── wan22.py          # Wan 2.2 Modal function
│       ├── mochi.py          # Mochi Modal function
│       ├── rife.py           # RIFE interpolation
│       ├── esrgan.py         # Upscaling
│       ├── beat_analyzer.py  # Audio beat extraction
│       └── orchestrator.py   # Chaining logic
├── ui/
│   ├── pages/
│   │   ├── index.tsx         # Main page
│   │   └── jobs/[id].tsx     # Job detail
│   ├── components/
│   │   ├── GenerateForm.tsx
│   │   ├── JobStatus.tsx
│   │   └── VideoPlayer.tsx
│   └── styles/
├── tests/
│   ├── test_api.py
│   ├── test_pipelines.py
│   └── test_runners.py
├── docker/
│   ├── Dockerfile.app
│   └── docker-compose.yml
├── docs/
│   ├── API.md
│   └── DEPLOYMENT.md
└── README.md
```

### profiles.yaml

```yaml
profiles:
  PREVIEW:
    description: "Quick 8-second preview clip"
    max_seconds: 8
    base_resolution:
      width: 540
      height: 960
    fps: 12
    use_rife: false
    use_esrgan: false
    chaining: false
    cost_estimate: "~$0.50 / clip"
    generation_time_minutes: 2

  HOOK:
    description: "12-second hook shot with smooth motion"
    max_seconds: 12
    base_resolution:
      width: 720
      height: 1280
    fps: 16
    use_rife: true
    use_esrgan: false
    rife_target_fps: 24
    chaining: false
    cost_estimate: "~$1.00 / clip"
    generation_time_minutes: 4

  LONGSHOT:
    description: "Continuous 25-30 second shot via sliding-window chaining"
    max_seconds: 30
    base_resolution:
      width: 720
      height: 1280
    fps: 16
    use_rife: true
    use_esrgan: true
    rife_target_fps: 30
    esrgan_upscale: 1.5
    chaining: true
    chaining_config:
      chunk_seconds: 5
      overlap_frames: 16
      blend_method: "crossfade"  # or "optical_flow"
    cost_estimate: "~$3.00 / clip"
    generation_time_minutes: 15

  MONTAGE:
    description: "40-second multi-shot sequence with beat alignment"
    max_seconds: 40
    base_resolution:
      width: 720
      height: 1280
    fps: 16
    use_rife: true
    use_esrgan: true
    rife_target_fps: 30
    esrgan_upscale: 1.5
    chaining: false  # multi-shot, not sliding-window
    montage_config:
      num_shots: 4
      shot_duration_seconds: 8
      beat_alignment: true
      beat_sync_sensitivity: "high"  # how strictly to sync to beats
    cost_estimate: "~$4.00 / clip"
    generation_time_minutes: 25
```

### brands.yaml

```yaml
brands:
  club_dark:
    display_name: "Dark Club"
    description: "Underground rave, cyberpunk vibes"
    default_profile: LONGSHOT
    default_model: wan22
    style_prompt_suffix: |
      dark warehouse rave, cyberpunk lighting, neon strobes,
      handheld camera with fast motion, crowd energy, DJ booth
    color_grade_hint: "desaturated, cool tones, high contrast"
    recommended_audio: "high-energy EDM, 120-140 BPM"
    example_prompts:
      - "crowd jumping in packed underground club"
      - "DJ mixing tracks, hands on turntables"
    tags: ["nightlife", "electronic", "high-energy"]

  luxury_slow:
    display_name: "Luxury Cinematic"
    description: "High-end, slow cinema, golden light"
    default_profile: HOOK
    default_model: mochi
    style_prompt_suffix: |
      luxury cinematic, shallow depth of field, golden hour,
      smooth dolly camera, elegant compositions, warm lighting
    color_grade_hint: "warm, desaturated, film grain"
    recommended_audio: "slow jazz, ambient, 60-90 BPM"
    example_prompts:
      - "woman in silk dress, window light, contemplative"
      - "minimalist interior, elegant movement"
    tags: ["luxury", "cinematic", "slow"]

  glitch_pop:
    display_name: "Glitch Pop"
    description: "Bright, kinetic, playful digital aesthetics"
    default_profile: HOOK
    default_model: wan22
    style_prompt_suffix: |
      bright neon glitch art, kinetic typography, playful camera shakes,
      digital artifacts as design, fast cuts, colorful
    color_grade_hint: "saturated, neon, high contrast"
    recommended_audio: "pop, synthwave, 100-130 BPM"
    example_prompts:
      - "dancer in neon lighting, glitch effects"
      - "character with digital overlay, playful motion"
    tags: ["pop", "digital", "playful"]
```

### models.yaml

```yaml
models:
  wan22:
    display_name: "Wan 2.2"
    type: "text-to-video"
    base_model: "Wan-AI/Wan2.1-T2V-1.3B"
    runner_module: "app.runners.wan22"
    runner_function: "wan22_runner"
    strengths:
      - "stylized generation"
      - "text-following accuracy"
      - "general purpose"
      - "vertical (9:16) friendly"
    max_seconds_single: 8
    optimal_resolution: [720, 1280]
    fps_range: [12, 24]
    default_sampling:
      num_inference_steps: 50
      guidance_scale: 7.5
      sampler: "euler"
      seed: null  # randomized unless specified
    vram_required_gb: 24
    estimated_time_seconds_per_frame: 0.8

  mochi:
    display_name: "Mochi"
    type: "text-to-video"
    base_model: "genmo/mochi-1"
    runner_module: "app.runners.mochi"
    runner_function: "mochi_runner"
    strengths:
      - "photoreal output"
      - "cinematic quality"
      - "smooth motion"
    max_seconds_single: 8
    optimal_resolution: [720, 1280]
    fps_range: [12, 24]
    default_sampling:
      num_inference_steps: 50
      guidance_scale: 6.0
      sampler: "flow_match"
      seed: null
    vram_required_gb: 32
    estimated_time_seconds_per_frame: 1.2

  # Future models (Phase 4+)
  wan_s2v:
    display_name: "Wan S2V (Audio-Driven)"
    type: "speech-to-video"
    base_model: "Wan-AI/Wan-S2V"
    runner_module: "app.runners.wan_s2v"
    runner_function: "wan_s2v_runner"
    strengths:
      - "lip sync"
      - "facial animation"
      - "audio-driven motion"
    max_seconds_single: 12
    optimal_resolution: [720, 1280]
    vram_required_gb: 32
    use_case: "hero shots, dialogue"
```

### samplers.yaml

```yaml
samplers:
  wan22_default:
    model: wan22
    num_inference_steps: 50
    guidance_scale: 7.5
    negative_prompt: |
      blurry, low quality, distorted, static, watermark,
      text overlay, poor lighting, unfocused
    sampler_type: "euler"
    seed: null

  wan22_fast:
    model: wan22
    num_inference_steps: 30
    guidance_scale: 7.0
    negative_prompt: |
      blurry, low quality, distorted, static, watermark
    sampler_type: "euler"
    seed: null
    use_case: "preview / low-cost"

  wan22_quality:
    model: wan22
    num_inference_steps: 70
    guidance_scale: 8.0
    negative_prompt: |
      blurry, low quality, distorted, static, watermark,
      text overlay, poor lighting, unfocused, jittery motion
    sampler_type: "euler"
    seed: null
    use_case: "final output"

  mochi_default:
    model: mochi
    num_inference_steps: 50
    guidance_scale: 6.0
    negative_prompt: |
      low quality, blurry, static, distorted, watermark
    sampler_type: "flow_match"
    seed: null
```

---

## API Contract

### POST /generate

**Description**: Create a new video generation job.

**Request**:
```json
{
  "brand_id": "club_dark",
  "prompt": "crowd jumping in a packed underground club, camera flying over heads toward DJ",
  "ref_image_url": "https://cdn.example.com/cover.jpg",
  "length": "short",
  "profile_override": null,
  "audio_url": null,
  "seed": null
}
```

**Request Validation**:
- `brand_id`: Must exist in brands.yaml
- `prompt`: 20-2000 characters
- `ref_image_url`: Valid image URL or null
- `length`: "short" | "long" | "montage"
- `profile_override`: null | "PREVIEW" | "HOOK" | "LONGSHOT" | "MONTAGE"
- `audio_url`: Optional, valid audio URL (Phase 3+)
- `seed`: Optional integer for reproducibility

**Backend Logic**:
```python
def post_generate(req: GenerateRequest):
    # 1. Validate inputs
    brand = brands[req.brand_id]  # raises 404 if not found

    # 2. Determine profile
    if req.profile_override:
        profile = profiles[req.profile_override]
    else:
        profile_key = {
            "short": "HOOK",
            "long": "LONGSHOT",
            "montage": "MONTAGE"
        }[req.length]
        profile = profiles[profile_key]

    # 3. Build full prompt
    full_prompt = f"{req.prompt}, {brand.style_prompt_suffix}"

    # 4. Create job record
    job = Job(
        id=generate_uuid(),
        brand_id=req.brand_id,
        profile=profile.name,
        prompt=full_prompt,
        ref_image_url=req.ref_image_url,
        audio_url=req.audio_url,
        seed=req.seed,
        status="PENDING",
        created_at=now()
    )
    db.add(job)
    db.commit()

    # 5. Queue Modal function
    modal.queue_job("run_generation", job.id)

    return {
        "job_id": job.id,
        "status": "PENDING",
        "estimated_wait_minutes": estimate_queue_time()
    }
```

**Response** (202 Accepted):
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PENDING",
  "estimated_wait_minutes": 5,
  "profile": "HOOK",
  "model": "wan22"
}
```

### GET /jobs/{job_id}

**Description**: Poll for job status and retrieve results.

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "profile": "LONGSHOT",
  "brand_id": "club_dark",
  "prompt": "crowd jumping in packed underground club...",
  "created_at": "2025-11-15T10:30:00Z",
  "started_at": "2025-11-15T10:32:15Z",
  "completed_at": "2025-11-15T10:47:30Z",
  "final_video_url": "https://cdn.example.com/videos/550e8400.mp4",
  "preview_images": [
    "https://cdn.example.com/thumbs/550e8400_1.jpg",
    "https://cdn.example.com/thumbs/550e8400_2.jpg"
  ],
  "meta": {
    "model": "wan22",
    "seconds": 28,
    "fps": 30,
    "resolution": [720, 1280],
    "rife_enabled": true,
    "esrgan_enabled": true,
    "total_generation_time_seconds": 915
  },
  "error": null
}
```

**Response if still pending** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PROCESSING",
  "queue_position": 2,
  "estimated_completion_minutes": 8,
  "progress": {
    "stage": "Running post-processors",
    "current_step": 3,
    "total_steps": 5
  }
}
```

**Response if failed** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "FAILED",
  "error": "CUDA out of memory during VAE decode. Try smaller resolution."
}
```

### GET /brands

**Description**: List available brands with metadata.

**Response** (200 OK):
```json
{
  "brands": [
    {
      "id": "club_dark",
      "display_name": "Dark Club",
      "description": "Underground rave, cyberpunk vibes",
      "default_profile": "LONGSHOT",
      "default_model": "wan22",
      "example_prompts": [
        "crowd jumping in packed underground club",
        "DJ mixing tracks, hands on turntables"
      ],
      "tags": ["nightlife", "electronic", "high-energy"]
    },
    {
      "id": "luxury_slow",
      "display_name": "Luxury Cinematic",
      "description": "High-end, slow cinema, golden light",
      "default_profile": "HOOK",
      "default_model": "mochi",
      "example_prompts": [
        "woman in silk dress, window light, contemplative"
      ],
      "tags": ["luxury", "cinematic", "slow"]
    }
  ]
}
```

### GET /profiles

**Description**: List available profiles with specifications.

**Response** (200 OK):
```json
{
  "profiles": [
    {
      "id": "PREVIEW",
      "description": "Quick 8-second preview clip",
      "max_seconds": 8,
      "resolution": [540, 960],
      "fps": 12,
      "use_rife": false,
      "use_esrgan": false,
      "generation_time_minutes": 2,
      "cost_estimate": "$0.50"
    },
    {
      "id": "HOOK",
      "description": "12-second hook shot with smooth motion",
      "max_seconds": 12,
      "resolution": [720, 1280],
      "fps": 16,
      "use_rife": true,
      "use_esrgan": false,
      "generation_time_minutes": 4,
      "cost_estimate": "$1.00"
    }
  ]
}
```

### GET /models

**Description**: List available models with capabilities.

**Response** (200 OK):
```json
{
  "models": [
    {
      "id": "wan22",
      "display_name": "Wan 2.2",
      "type": "text-to-video",
      "strengths": ["stylized", "text-friendly", "general"],
      "max_seconds_single": 8,
      "optimal_resolution": [720, 1280],
      "vram_required_gb": 24
    },
    {
      "id": "mochi",
      "display_name": "Mochi",
      "type": "text-to-video",
      "strengths": ["photoreal", "cinematic", "smooth"],
      "max_seconds_single": 8,
      "optimal_resolution": [720, 1280],
      "vram_required_gb": 32
    }
  ]
}
```

---

## Core Pipeline Logic

### The Orchestrator: run_generation(job_id)

**Location**: `app/runners/orchestrator.py`

```python
import asyncio
from app.database.models import Job
from app.database.connection import get_db
from app.runners import wan22, mochi, rife, esrgan
from app.services.storage import upload_to_s3
from app.services.brand_service import get_brand_config
from config import profiles, brands, models

async def run_generation(job_id: str):
    """
    Main orchestrator: routes job to appropriate pipeline based on profile.
    """
    db = get_db()
    job = db.query(Job).filter(Job.id == job_id).first()

    if not job:
        raise ValueError(f"Job {job_id} not found")

    try:
        # 1. Lookup configs
        profile = profiles[job.profile]
        brand = brands[job.brand_id]
        model_cfg = models[brand.default_model]

        # Download ref image if provided
        ref_image_path = None
        if job.ref_image_url:
            ref_image_path = download_image(job.ref_image_url, f"/tmp/{job_id}_ref.jpg")

        # Update status
        job.status = "PROCESSING"
        job.started_at = datetime.now()
        db.commit()

        # 2. Route based on profile
        if profile.chaining:
            if profile.name == "LONGSHOT":
                video_path = await generate_longshot(
                    job_id=job_id,
                    profile=profile,
                    brand=brand,
                    model_cfg=model_cfg,
                    prompt=job.prompt,
                    ref_image=ref_image_path,
                    seed=job.seed
                )
            elif profile.name == "MONTAGE":
                video_path = await generate_montage(
                    job_id=job_id,
                    profile=profile,
                    brand=brand,
                    model_cfg=model_cfg,
                    prompt=job.prompt,
                    ref_image=ref_image_path,
                    audio_url=job.audio_url,
                    seed=job.seed
                )
        else:
            # Single clip (PREVIEW, HOOK)
            video_path = await generate_single_clip(
                job_id=job_id,
                profile=profile,
                brand=brand,
                model_cfg=model_cfg,
                prompt=job.prompt,
                ref_image=ref_image_path,
                seed=job.seed
            )

        # 3. Post-process
        processed_path = await post_process(
            video_path=video_path,
            profile=profile,
            job_id=job_id
        )

        # 4. Generate thumbnails
        thumbs = generate_thumbnails(processed_path, num_thumbs=3)
        thumb_urls = [upload_to_s3(t) for t in thumbs]

        # 5. Upload final video
        final_url = upload_to_s3(processed_path, f"videos/{job_id}.mp4")

        # 6. Update job record
        job.status = "COMPLETED"
        job.final_video_url = final_url
        job.preview_images = thumb_urls
        job.completed_at = datetime.now()
        job.meta = {
            "model": brand.default_model,
            "profile": profile.name,
            "seconds": profile.max_seconds,
            "fps": 24 if profile.use_rife else profile.fps,
            "resolution": [profile.base_resolution.width,
                          profile.base_resolution.height],
            "rife_enabled": profile.use_rife,
            "esrgan_enabled": profile.use_esrgan,
        }
        db.commit()

    except Exception as e:
        job.status = "FAILED"
        job.error = str(e)
        db.commit()
        raise
```

### Single Clip Generation: generate_single_clip()

```python
async def generate_single_clip(
    job_id: str,
    profile: Profile,
    brand: Brand,
    model_cfg: Model,
    prompt: str,
    ref_image: str = None,
    seed: int = None
) -> str:
    """
    Generate a single clip (PREVIEW or HOOK profile).

    Returns: Path to video file
    """
    # Step 1: Call model runner
    raw_path = await call_model_runner(
        runner_name=model_cfg.runner_function,
        prompt=prompt,
        ref_image=ref_image,
        seconds=profile.max_seconds,
        resolution=[profile.base_resolution.width,
                   profile.base_resolution.height],
        fps=profile.fps,
        seed=seed
    )

    # Step 2: Optional RIFE interpolation
    if profile.use_rife:
        rife_path = await rife.rife_runner(
            input_path=raw_path,
            target_fps=profile.rife_target_fps or 24,
            job_id=job_id
        )
    else:
        rife_path = raw_path

    # Step 3: Optional ESRGAN upscaling
    if profile.use_esrgan:
        final_path = await esrgan.esrgan_runner(
            input_path=rife_path,
            up_scale=profile.esrgan_upscale or 1.5,
            job_id=job_id
        )
    else:
        final_path = rife_path

    return final_path
```

### Longshot Chaining: generate_longshot()

```python
async def generate_longshot(
    job_id: str,
    profile: Profile,
    brand: Brand,
    model_cfg: Model,
    prompt: str,
    ref_image: str = None,
    seed: int = None
) -> str:
    """
    Generate a 25-30s continuous shot via sliding-window chaining.

    Process:
    1. Plan chunks based on target duration
    2. Generate each chunk with last-frame of previous as ref
    3. Stitch chunks with crossfade overlaps
    4. Apply RIFE and ESRGAN post-processing
    """
    secs = profile.max_seconds
    chunk_secs = profile.chaining_config.chunk_seconds  # e.g., 5
    overlap_frames = profile.chaining_config.overlap_frames  # e.g., 16
    blend_method = profile.chaining_config.blend_method  # "crossfade"
    fps = profile.fps

    num_chunks = math.ceil(secs / chunk_secs)
    chunks = []

    for i in range(num_chunks):
        # Determine reference for this chunk
        if i == 0:
            chunk_ref = ref_image
        else:
            # Extract last frame of previous chunk as reference
            prev_last_frame = extract_last_frame(chunks[-1]["path"])
            chunk_ref = prev_last_frame

        # Plan prompt for this chunk
        # e.g., "camera pans left" for chunk 0, "camera continues panning" for chunk 1
        chunk_prompt = plan_chunk_prompt(prompt, i, num_chunks)

        # Calculate actual duration of this chunk
        if i == num_chunks - 1:
            # Last chunk: fill remaining time
            chunk_duration = secs - (i * chunk_secs)
        else:
            chunk_duration = chunk_secs

        # Generate chunk
        chunk_path = await call_model_runner(
            runner_name=model_cfg.runner_function,
            prompt=chunk_prompt,
            ref_image=chunk_ref,
            seconds=chunk_duration,
            resolution=[profile.base_resolution.width,
                       profile.base_resolution.height],
            fps=fps,
            seed=seed + i if seed else None  # Vary seed per chunk
        )

        chunks.append({
            "index": i,
            "path": chunk_path,
            "duration": chunk_duration
        })

    # Step 2: Stitch chunks with overlaps
    stitched_path = stitch_with_overlap(
        chunk_paths=[c["path"] for c in chunks],
        overlap_frames=overlap_frames,
        blend_method=blend_method,
        fps=fps,
        job_id=job_id
    )

    # Step 3: Post-process with RIFE + ESRGAN
    rife_path = await rife.rife_runner(
        input_path=stitched_path,
        target_fps=profile.rife_target_fps,
        job_id=job_id
    )

    final_path = await esrgan.esrgan_runner(
        input_path=rife_path,
        up_scale=profile.esrgan_upscale,
        job_id=job_id
    )

    return final_path
```

### Montage Generation: generate_montage()

```python
async def generate_montage(
    job_id: str,
    profile: Profile,
    brand: Brand,
    model_cfg: Model,
    prompt: str,
    ref_image: str = None,
    audio_url: str = None,
    seed: int = None
) -> str:
    """
    Generate a multi-shot MONTAGE sequence with beat alignment.

    Process:
    1. If audio provided: analyze beats
    2. Plan N shots with beat-aligned transitions
    3. Generate each shot independently
    4. Concatenate with beat-aligned cuts or transitions
    """
    num_shots = profile.montage_config.num_shots  # e.g., 4
    shot_duration = profile.montage_config.shot_duration_seconds  # e.g., 8
    use_beat_alignment = profile.montage_config.beat_alignment

    # Step 1: Analyze beats if audio provided
    beat_data = None
    if audio_url and use_beat_alignment:
        beat_data = await beat_analyzer.analyze_beats(
            audio_url=audio_url,
            target_fps=profile.fps
        )
        # Returns: {"bpm": 120, "beats": [0.5, 1.0, 1.5, ...]}

    # Step 2: Plan shots
    shots = plan_montage_shots(
        prompt=prompt,
        num_shots=num_shots,
        duration_per_shot=shot_duration,
        beat_data=beat_data
    )
    # Returns: [{
    #   "index": 0,
    #   "prompt": "crowd jumping in underground club",
    #   "duration": 8,
    #   "beat_cut_frame": 150
    # }, ...]

    # Step 3: Generate shots
    shot_paths = []
    for shot in shots:
        shot_path = await call_model_runner(
            runner_name=model_cfg.runner_function,
            prompt=shot["prompt"],
            ref_image=ref_image if shot["index"] == 0 else None,
            seconds=shot["duration"],
            resolution=[profile.base_resolution.width,
                       profile.base_resolution.height],
            fps=profile.fps,
            seed=seed + shot["index"] if seed else None
        )
        shot_paths.append(shot_path)

    # Step 4: Concatenate with beat-aligned transitions
    concatenated_path = concat_shots(
        shot_paths=shot_paths,
        shots_metadata=shots,
        beat_data=beat_data,
        job_id=job_id
    )

    # Step 5: Post-process
    rife_path = await rife.rife_runner(
        input_path=concatenated_path,
        target_fps=profile.rife_target_fps,
        job_id=job_id
    )

    final_path = await esrgan.esrgan_runner(
        input_path=rife_path,
        up_scale=profile.esrgan_upscale,
        job_id=job_id
    )

    # If audio provided, sync final video to audio
    if audio_url:
        final_path = sync_video_to_audio(
            video_path=final_path,
            audio_path=audio_url,
            output_path=f"/tmp/{job_id}_final_sync.mp4"
        )

    return final_path
```

---

## Runners & Model Integration

### Wan 2.2 Runner

**Location**: `app/runners/wan22.py`

```python
import modal
from diffusers import DiffusionPipeline
import torch

# Define a Modal image with all dependencies
humo_image = (
    modal.Image.debian_slim()
    .pip_install(
        "torch==2.5.1",
        "torchvision==0.20.1",
        "diffusers>=0.31.0",
        "transformers>=4.49.0",
        "accelerate>=1.1.1",
        "flash_attn==2.6.3",
        "moviepy==1.0.3",
        "imageio-ffmpeg",
    )
    .run_commands(
        "apt-get update && apt-get install -y ffmpeg"
    )
)

# Define app for Modal
app = modal.App("humo-video-gen")

# Persistent container to keep models in VRAM between requests
class ModelCache:
    def __enter__(self):
        # Load models once
        self.device = "cuda" if torch.cuda.is_available() else "cpu"
        self.pipe = DiffusionPipeline.from_pretrained(
            "Wan-AI/Wan2.1-T2V-1.3B",
            torch_dtype=torch.bfloat16
        ).to(self.device)
        return self

    def __exit__(self, *args):
        pass

    def generate(self, prompt, ref_image, seconds, resolution, fps, seed):
        # Inference logic here
        pass

@app.function(
    image=humo_image,
    gpu="A100",  # or "H100" for better throughput
    timeout=3600,  # 1 hour max
    retries=1
)
async def wan22_runner(
    prompt: str,
    ref_image: str = None,
    seconds: int = 8,
    resolution: list = [720, 1280],
    fps: int = 16,
    seed: int = None,
    job_id: str = None
) -> str:
    """
    Generate video using Wan 2.2 model.

    Args:
        prompt: Text description of video
        ref_image: Path to reference image (optional)
        seconds: Duration in seconds
        resolution: [width, height]
        fps: Frames per second
        seed: Random seed for reproducibility
        job_id: For logging / temp file naming

    Returns:
        Path to generated MP4 file
    """
    import os
    from PIL import Image
    from moviepy.editor import ImageSequenceClip

    # Calculate frame count
    num_frames = seconds * fps

    # Load models (cached in Modal)
    with ModelCache() as cache:
        # Prepare inputs
        if ref_image:
            image = Image.open(ref_image).convert("RGB").resize(resolution)
        else:
            image = None

        # Run generation
        with torch.no_grad():
            output = cache.pipe(
                prompt=prompt,
                height=resolution[1],
                width=resolution[0],
                num_frames=num_frames,
                num_inference_steps=50,
                guidance_scale=7.5,
                image=image,
                generator=torch.Generator().manual_seed(seed) if seed else None
            )

        # Extract frames
        frames = output.frames[0]  # List of PIL Images

        # Create video using moviepy
        frame_array = [np.array(f) for f in frames]
        clip = ImageSequenceClip(frame_array, fps=fps)

        output_path = f"/tmp/{job_id or 'wan22'}_output.mp4"
        clip.write_videofile(output_path, codec="libx264")

        return output_path
```

### RIFE Runner (Interpolation)

**Location**: `app/runners/rife.py`

```python
import modal
import subprocess

rife_image = (
    modal.Image.debian_slim()
    .pip_install(
        "torch==2.5.1",
        "torchvision==0.20.1",
        "opencv-python",
    )
    .run_commands(
        "git clone https://github.com/megvii-research/RIFE.git /rife && cd /rife && pip install ."
    )
)

@app.function(
    image=rife_image,
    gpu="A100",
    timeout=1800
)
async def rife_runner(
    input_path: str,
    target_fps: int = 24,
    job_id: str = None
) -> str:
    """
    Interpolate video frames using RIFE.

    Args:
        input_path: Path to input video
        target_fps: Target frames per second (interpolated)
        job_id: For logging / temp file naming

    Returns:
        Path to interpolated video
    """
    import cv2
    from RIFE.inference_video import interpolate_video

    # Determine multiplier needed
    input_fps = get_video_fps(input_path)
    multiplier = target_fps / input_fps

    # Run RIFE
    output_path = f"/tmp/{job_id or 'rife'}_output.mp4"
    interpolate_video(
        input_path=input_path,
        output_path=output_path,
        multiplier=multiplier,
        model="train"  # or "vfifp" for lighter model
    )

    return output_path
```

### ESRGAN Runner (Upscaling)

**Location**: `app/runners/esrgan.py`

```python
import modal
from basicsr.archs.rrdbnet_arch import RRDBNet
from realesrgan import RealESRGANer

esrgan_image = (
    modal.Image.debian_slim()
    .pip_install(
        "torch==2.5.1",
        "torchvision==0.20.1",
        "basicsr",
        "realesrgan",
        "opencv-python"
    )
)

@app.function(
    image=esrgan_image,
    gpu="A100",
    timeout=3600
)
async def esrgan_runner(
    input_path: str,
    up_scale: float = 1.5,
    job_id: str = None
) -> str:
    """
    Upscale video using Real-ESRGAN.

    Args:
        input_path: Path to input video
        up_scale: Scale factor (1.5, 2, 3, or 4)
        job_id: For logging / temp file naming

    Returns:
        Path to upscaled video
    """
    import cv2
    import numpy as np
    from moviepy.editor import VideoFileClip

    # Load model
    upsampler = RealESRGANer(
        scale=up_scale,
        model_name="RealESRGAN_x2plus",
        dni_weight=0.5,
        device="cuda"
    )

    # Process video
    cap = cv2.VideoCapture(input_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))

    output_width = int(width * up_scale)
    output_height = int(height * up_scale)

    output_path = f"/tmp/{job_id or 'esrgan'}_output.mp4"
    writer = cv2.VideoWriter(
        output_path,
        cv2.VideoWriter_fourcc(*'mp4v'),
        fps,
        (output_width, output_height)
    )

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        # Upscale frame
        upscaled, _ = upsampler.enhance(frame)
        writer.write(upscaled)

    cap.release()
    writer.release()

    return output_path
```

### Beat Analyzer (aubio)

**Location**: `app/runners/beat_analyzer.py`

```python
import modal
import librosa
import aubio

beat_image = (
    modal.Image.debian_slim()
    .pip_install(
        "librosa",
        "aubio",
        "numpy"
    )
)

@app.function(
    image=beat_image,
    timeout=300
)
async def analyze_beats(
    audio_url: str,
    target_fps: int = 30
) -> dict:
    """
    Analyze audio and extract beat times.

    Args:
        audio_url: Path or URL to audio file
        target_fps: Video FPS (for frame index conversion)

    Returns:
        {
            "bpm": 120,
            "beats": [0.5, 1.0, 1.5, ...],  # in seconds
            "beat_frames": [15, 30, 45, ...],  # frame indices at target_fps
            "downbeats": [0.0, 2.0, 4.0, ...],  # major beats
            "onsets": [0.1, 0.3, 0.5, ...]  # all detected onsets
        }
    """
    import librosa
    import numpy as np

    # Load audio
    y, sr = librosa.load(audio_url, sr=22050)

    # Detect BPM and beats
    onset_env = librosa.onset.onset_strength(y=y, sr=sr)
    bpm = librosa.beat.tempo(onset_env=onset_env, sr=sr)[0]

    _, beats = librosa.beat.beat_track(bpm=bpm, sr=sr, y=y)
    beat_times = librosa.frames_to_time(beats, sr=sr)

    # Also extract onsets
    onset_frames = librosa.onset.onset_detect(y=y, sr=sr)
    onset_times = librosa.frames_to_time(onset_frames, sr=sr)

    # Downbeats (every 4 beats, typically)
    downbeat_indices = np.arange(0, len(beats), 4)
    downbeats = beat_times[downbeat_indices]

    # Convert to frame indices
    beat_frames = (beat_times * target_fps).astype(int)
    onset_frames_video = (onset_times * target_fps).astype(int)
    downbeat_frames = (downbeats * target_fps).astype(int)

    return {
        "bpm": float(bpm),
        "beats": beat_times.tolist(),
        "beat_frames": beat_frames.tolist(),
        "downbeats": downbeats.tolist(),
        "downbeat_frames": downbeat_frames.tolist(),
        "onsets": onset_times.tolist(),
        "onset_frames": onset_frames_video.tolist(),
    }
```

---

## Database Schema

### PostgreSQL Schema

```sql
-- Jobs table
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_id VARCHAR(100) NOT NULL,
    profile VARCHAR(50) NOT NULL,
    prompt TEXT NOT NULL,
    ref_image_url VARCHAR(500),
    audio_url VARCHAR(500),
    seed INTEGER,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    -- Status values: PENDING, QUEUED, PROCESSING, COMPLETED, FAILED

    final_video_url VARCHAR(500),
    preview_images TEXT[],  -- JSON array of URLs

    error TEXT,
    meta JSONB,  -- Stores {model, fps, resolution, rife_enabled, etc.}

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,

    -- Indexing for common queries
    FOREIGN KEY (brand_id) REFERENCES brands(id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at),
    INDEX idx_brand_id (brand_id)
);

-- Brands table
CREATE TABLE brands (
    id VARCHAR(100) PRIMARY KEY,
    display_name VARCHAR(200) NOT NULL,
    description TEXT,
    default_profile VARCHAR(50) NOT NULL,
    default_model VARCHAR(100) NOT NULL,
    style_prompt_suffix TEXT,
    color_grade_hint VARCHAR(200),
    recommended_audio VARCHAR(200),
    tags TEXT[],
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Analytics table
CREATE TABLE analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(id),
    brand_id VARCHAR(100) NOT NULL,
    profile VARCHAR(50) NOT NULL,
    model VARCHAR(100) NOT NULL,
    seconds INT,
    fps INT,
    rife_enabled BOOLEAN,
    esrgan_enabled BOOLEAN,
    total_time_seconds INT,

    -- Performance metrics
    views INT DEFAULT 0,
    likes INT DEFAULT 0,
    shares INT DEFAULT 0,

    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### SQLAlchemy ORM Models

**Location**: `app/database/models.py`

```python
from sqlalchemy import Column, String, Text, Integer, DateTime, JSONB, Index
from sqlalchemy.dialects.postgresql import UUID, ARRAY
from sqlalchemy.ext.declarative import declarative_base
import uuid
from datetime import datetime

Base = declarative_base()

class Job(Base):
    __tablename__ = "jobs"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    brand_id = Column(String(100), nullable=False)
    profile = Column(String(50), nullable=False)
    prompt = Column(Text, nullable=False)
    ref_image_url = Column(String(500))
    audio_url = Column(String(500))
    seed = Column(Integer)

    status = Column(String(20), default="PENDING")
    final_video_url = Column(String(500))
    preview_images = Column(ARRAY(String))
    error = Column(Text)
    meta = Column(JSONB)

    created_at = Column(DateTime, default=datetime.utcnow)
    started_at = Column(DateTime)
    completed_at = Column(DateTime)

    __table_args__ = (
        Index("idx_status", "status"),
        Index("idx_created_at", "created_at"),
        Index("idx_brand_id", "brand_id"),
    )

class Brand(Base):
    __tablename__ = "brands"

    id = Column(String(100), primary_key=True)
    display_name = Column(String(200), nullable=False)
    description = Column(Text)
    default_profile = Column(String(50), nullable=False)
    default_model = Column(String(100), nullable=False)
    style_prompt_suffix = Column(Text)
    color_grade_hint = Column(String(200))
    recommended_audio = Column(String(200))
    tags = Column(ARRAY(String))

    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

class Analytics(Base):
    __tablename__ = "analytics"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    job_id = Column(UUID(as_uuid=True), nullable=False)
    brand_id = Column(String(100), nullable=False)
    profile = Column(String(50), nullable=False)
    model = Column(String(100), nullable=False)
    seconds = Column(Integer)
    fps = Column(Integer)
    rife_enabled = Column(Boolean, default=False)
    esrgan_enabled = Column(Boolean, default=False)
    total_time_seconds = Column(Integer)

    views = Column(Integer, default=0)
    likes = Column(Integer, default=0)
    shares = Column(Integer, default=0)

    created_at = Column(DateTime, default=datetime.utcnow)
```

---

## Deployment on Modal

### Modal Function Configuration

**Location**: `app/modal_app.py`

```python
import modal
import os
from datetime import timedelta

# Define custom image with all dependencies
video_gen_image = (
    modal.Image.debian_slim()
    .pip_install(
        "torch==2.5.1",
        "torchvision==0.20.1",
        "diffusers>=0.31.0",
        "transformers>=4.49.0",
        "accelerate>=1.1.1",
        "flash_attn==2.6.3",
        "moviepy==1.0.3",
        "imageio-ffmpeg",
        "librosa",
        "aubio",
        "basicsr",
        "realesrgan",
        "opencv-python",
        "sqlalchemy",
        "psycopg2-binary",
        "boto3",
        "pydantic",
    )
    .run_commands(
        "apt-get update && apt-get install -y ffmpeg"
    )
)

app = modal.App("production-video-stack")

# Shared network file system for models (optional, can also download on-demand)
model_cache = modal.Volume.from_name("model-cache", create_if_missing=True)

# ============================================================================
# GPU Functions (Runners)
# ============================================================================

@app.function(
    image=video_gen_image,
    gpu="A100-80GB",  # or "H100" for faster inference
    timeout=3600,
    retries=1,
    container_idle_timeout=600,  # Keep warm for 10 minutes
    volumes={"/models": model_cache}
)
async def wan22_runner(
    prompt: str,
    ref_image_path: str = None,
    seconds: int = 8,
    resolution: list = [720, 1280],
    fps: int = 16,
    seed: int = None,
    job_id: str = None
) -> str:
    """Wan 2.2 video generation"""
    # Implementation from runners/wan22.py
    from app.runners.wan22 import generate_video
    return await generate_video(
        prompt=prompt,
        ref_image=ref_image_path,
        seconds=seconds,
        resolution=resolution,
        fps=fps,
        seed=seed,
        job_id=job_id
    )

@app.function(
    image=video_gen_image,
    gpu="A100-80GB",
    timeout=1800
)
async def rife_runner(
    input_path: str,
    target_fps: int = 24,
    job_id: str = None
) -> str:
    """RIFE frame interpolation"""
    from app.runners.rife import interpolate
    return await interpolate(
        input_path=input_path,
        target_fps=target_fps,
        job_id=job_id
    )

@app.function(
    image=video_gen_image,
    gpu="A100-80GB",
    timeout=3600
)
async def esrgan_runner(
    input_path: str,
    up_scale: float = 1.5,
    job_id: str = None
) -> str:
    """ESRGAN upscaling"""
    from app.runners.esrgan import upscale
    return await upscale(
        input_path=input_path,
        up_scale=up_scale,
        job_id=job_id
    )

@app.function(image=video_gen_image, timeout=300)
async def beat_analyzer_fn(
    audio_url: str,
    target_fps: int = 30
) -> dict:
    """Beat analysis for montage"""
    from app.runners.beat_analyzer import analyze_beats
    return await analyze_beats(audio_url, target_fps)

# ============================================================================
# Orchestrator Function
# ============================================================================

@app.function(
    image=video_gen_image,
    timeout=7200,  # 2 hours max for LONGSHOT
    retries=0
)
async def run_generation(job_id: str):
    """Main orchestrator - called from FastAPI"""
    from app.runners.orchestrator import run_generation as orchestrate
    return await orchestrate(job_id)

# ============================================================================
# FastAPI Web Handler
# ============================================================================

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

api = modal.asgi_app()

@app.asgi_app()
def fastapi_app():
    app = FastAPI(title="Production Video Stack API")

    @app.post("/generate")
    async def generate(req: GenerateRequest):
        from app.services.job_service import create_job
        job = await create_job(req)
        # Queue Modal function
        run_generation.spawn(job.id)
        return {
            "job_id": str(job.id),
            "status": "PENDING"
        }

    @app.get("/jobs/{job_id}")
    async def get_job(job_id: str):
        from app.services.job_service import get_job_status
        job = await get_job_status(job_id)
        if not job:
            raise HTTPException(status_code=404, detail="Job not found")
        return job

    @app.get("/brands")
    async def list_brands():
        from app.services.brand_service import get_all_brands
        return await get_all_brands()

    @app.get("/profiles")
    async def list_profiles():
        from config import profiles
        return {"profiles": list(profiles.values())}

    return app
```

### Local Modal Development

```bash
# Install Modal CLI
pip install modal

# Authenticate with Modal
modal token new

# Deploy app
modal deploy app/modal_app.py

# Deploy specific function
modal run app/modal_app.py::wan22_runner

# Stream logs
modal tail production-video-stack::run_generation

# Test locally
modal run app/modal_app.py
```

---

## ComfyUI Lab Setup

### Purpose

ComfyUI Lab is not production—it's a **prototyping sandbox** where you:
1. Experiment with node graphs (Wan 2.2, RIFE, ESRGAN, LongCat, FramePack)
2. Tweak hyperparameters (CFG, steps, noise schedule, seed)
3. Capture winning workflows → convert to Python runners

### Setup

```bash
# Clone ComfyUI
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI

# Install dependencies
pip install -r requirements.txt

# Install community nodes
cd custom_nodes
git clone https://github.com/kijai/ComfyUI-WanVideoWrapper
git clone https://github.com/Kosinkadink/ComfyUI-Advanced-ControlNet
git clone https://github.com/jags111/efficiency-nodes-comfyui
cd ..

# Start server
python main.py --listen 0.0.0.0 --port 8188
```

### Example Workflow: 12-Second Vertical Clip

```json
{
  "1": {
    "class_type": "CheckpointLoaderSimple",
    "inputs": {
      "ckpt_name": "wan22-t2v-1.3b.safetensors"
    }
  },
  "2": {
    "class_type": "CLIPTextEncode",
    "inputs": {
      "text": "crowd jumping in underground club, camera flying, neon lights",
      "clip": ["1", 1]
    }
  },
  "3": {
    "class_type": "CLIPTextEncode",
    "inputs": {
      "text": "blurry, low quality, static",
      "clip": ["1", 1]
    }
  },
  "4": {
    "class_type": "KSampler",
    "inputs": {
      "seed": 42,
      "steps": 50,
      "cfg": 7.5,
      "sampler_name": "euler",
      "scheduler": "normal",
      "denoise": 1.0,
      "model": ["1", 0],
      "positive": ["2", 0],
      "negative": ["3", 0],
      "latent_image": ["5", 0]
    }
  },
  "5": {
    "class_type": "EmptyVideoLatentImage",
    "inputs": {
      "width": 720,
      "height": 1280,
      "length": 96,
      "batch_size": 1
    }
  },
  "6": {
    "class_type": "VAEDecode",
    "inputs": {
      "samples": ["4", 0],
      "vae": ["1", 2]
    }
  },
  "7": {
    "class_type": "RIFEInterpolation",
    "inputs": {
      "video": ["6", 0],
      "multiplier": 1.5
    }
  },
  "8": {
    "class_type": "VHS_VideoCombine",
    "inputs": {
      "images": ["7", 0],
      "frame_rate": 24,
      "format": "video/mp4",
      "codec": "h264",
      "quality": 95
    }
  }
}
```

When this workflow produces good results:
1. Note the node configuration (CFG=7.5, steps=50, sampler=euler)
2. Extract settings into `samplers.yaml`
3. Implement equivalent Python code in `runners/wan22.py`

---

## First Week Implementation

### Week 1 Deliverables

**By end of Week 1, you should have:**

1. **Skeleton FastAPI app** (2-3 hours)
   ```python
   # app/api/main.py
   from fastapi import FastAPI, HTTPException
   from pydantic import BaseModel
   from datetime import datetime
   import uuid

   app = FastAPI(title="Production Video Stack")

   class GenerateRequest(BaseModel):
       brand_id: str
       prompt: str
       ref_image_url: str = None
       length: str = "short"  # "short" | "long"
       seed: int = None

   @app.post("/generate")
   async def generate(req: GenerateRequest):
       job_id = str(uuid.uuid4())
       # TODO: Save to DB
       # TODO: Queue Modal function
       return {"job_id": job_id, "status": "PENDING"}

   @app.get("/jobs/{job_id}")
   async def get_job(job_id: str):
       # TODO: Fetch from DB
       return {
           "id": job_id,
           "status": "PENDING",  # or PROCESSING, COMPLETED
           "final_video_url": None
       }
   ```

2. **PostgreSQL setup** (1-2 hours)
   ```bash
   # Create database
   createdb production_video_stack

   # Run migrations
   alembic upgrade head  # (or raw SQL from schema.sql)
   ```

3. **Simple brand + profile config** (30 min)
   ```yaml
   # config/brands.yaml
   brands:
     club_dark:
       default_profile: LONGSHOT
       default_model: wan22
       style_prompt_suffix: "dark warehouse rave, cyberpunk lighting"

   # config/profiles.yaml
   profiles:
     PREVIEW:
       max_seconds: 8
       base_resolution: [540, 960]
     HOOK:
       max_seconds: 12
       base_resolution: [720, 1280]
   ```

4. **Wan 2.2 Modal runner** (3-4 hours)
   ```python
   # app/runners/wan22.py
   # (See runner section above)
   ```

5. **Single-clip generate pipeline** (2-3 hours)
   ```python
   # app/runners/orchestrator.py
   async def generate_single_clip(profile, brand, prompt, ref_image, seed):
       # Call wan22_runner → RIFE (optional) → return path
   ```

6. **Minimal Next.js UI** (2-3 hours)
   ```tsx
   // ui/pages/index.tsx
   export default function Home() {
     return (
       <div>
         <h1>Video Generator</h1>
         <select id="brand">{/* brands */}</select>
         <textarea placeholder="Describe your video..." />
         <input type="file" accept="image/*" />
         <button onClick={generate}>Generate</button>
         <div id="result">{/* job status / video */}</div>
       </div>
     );
   }
   ```

7. **Local testing** (1-2 hours)
   - Test API locally with curl / Postman
   - Verify DB saves job records
   - Test one full generation end-to-end

### Week 1 Timeline (assuming full-time effort)

| Time | Task | Duration |
|------|------|----------|
| Day 1 AM | FastAPI skeleton + DB schema | 3-4h |
| Day 1 PM | Config files (brands, profiles) | 1h |
| Day 2 AM | Wan 2.2 Modal runner stub | 2-3h |
| Day 2 PM | Single-clip orchestrator | 2-3h |
| Day 3 AM | Minimal Next.js UI | 2-3h |
| Day 3 PM | E2E testing + debugging | 2-3h |
| Day 4+ | Polish, optimizations, buffer | 8h+ |

**Success Criteria for Phase 1**:
- [ ] POST /generate creates job in DB
- [ ] GET /jobs/{id} returns job status
- [ ] Wan 2.2 generates 8-12s MP4 (even if 2-3 min runtime)
- [ ] UI allows brand selection, prompt entry, file upload
- [ ] Generated video plays, looks reasonable quality

---

## Observability & Feedback Loop

### Logging

```python
# app/services/logger.py
import logging
from pythonjsonlogger import jsonlogger

logger = logging.getLogger()
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
logHandler.setFormatter(formatter)
logger.addHandler(logHandler)

# Log every generation attempt
logger.info("generation_start", extra={
    "job_id": job.id,
    "brand_id": job.brand_id,
    "profile": job.profile,
    "model": brand.default_model,
    "seconds": profile.max_seconds
})

# Log completion
logger.info("generation_complete", extra={
    "job_id": job.id,
    "duration_seconds": (job.completed_at - job.created_at).total_seconds(),
    "status": "success" or "failed"
})
```

### Analytics Dashboard (Later)

Track which profiles, brands, models work best:
```python
# Query: Most popular profile / brand combination
SELECT profile, brand_id, COUNT(*) as count
FROM jobs
WHERE status = 'COMPLETED'
GROUP BY profile, brand_id
ORDER BY count DESC;

# Query: Average generation time by profile
SELECT profile, AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) as avg_seconds
FROM jobs
WHERE status = 'COMPLETED'
GROUP BY profile;
```

---

## Next Steps

Once Phase 1 is complete, Phase 2 adds:
- [ ] Chaining logic in `generate_longshot()`
- [ ] RIFE + ESRGAN post-processors
- [ ] Stitching logic (crossfade overlaps)
- [ ] Update UI to show "short" vs "long" toggle

Then Phase 3 adds beat sync, and Phase 4 adds S2V hero shots.

---

## Document Summary

| Section | Key Takeaway |
|---------|--------------|
| **Vision** | Small web app: brand + prompt + image → publication-ready 9:16 videos |
| **Architecture** | FastAPI + Modal GPU functions + PostgreSQL + S3 |
| **Phases** | 1=single clips, 2=chaining, 3=beat sync, 4=hero lip-sync |
| **Profiles** | PREVIEW, HOOK, LONGSHOT, MONTAGE—each with different chaining/FPS/upscale |
| **Runners** | Wan 2.2, Mochi, RIFE, ESRGAN, beat_analyzer—each a Modal function |
| **First Week** | API skeleton + Wan 2.2 runner + simple UI = ~30-40 hours effort |
| **ComfyUI Lab** | Prototype workflows locally, port settings to Python runners |

---

**Ready to start Week 1? Let's ship it.**
