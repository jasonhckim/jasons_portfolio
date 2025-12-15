# Fashion AI Studio

Production-grade AI system that automates fashion product image generation, replacing traditional photography workflows for color variants, fabric swaps, and editorial styling.

## Problem

Fashion brands spend 8-10 minutes per color variant in traditional photography workflows, requiring physical samples, studio time, and retouching. For a collection with 20 styles and 5 color variants each, this translates to 13+ hours of photographer time at $185/hour ($2,400+ per collection). The system eliminates this bottleneck by generating photorealistic product variants on-demand using AI, reducing time per variant from 8.5 minutes to ~30 seconds.

## Constraints

**Technical:**
- AI model APIs (Google Gemini, FASHN) are rate-limited and have variable latency (2-120 seconds per image)
- Image generation must preserve exact garment structure, proportions, and non-fabric elements
- Output quality must meet e-commerce standards (4K resolution, no AI artifacts)
- Multi-tenant architecture requires strict workspace isolation and encrypted credential storage

**Business:**
- Users need to maintain brand consistency across transformations
- Integration with Shopify requires bidirectional sync without data loss
- Cost per transformation must remain predictable despite variable API pricing

**Operational:**
- Failed generations must be retryable without manual intervention
- Batch operations (100+ images) must handle partial failures gracefully
- Prompt engineering changes must be versioned and rollback-capable

## System Architecture

```
User Upload → Image Group Creation → View Type Assignment
    ↓
Transformation Request (Color/Fabric/Ghost/Editorial/Composite)
    ↓
Prompt Compilation (System Prompt + Template + Variables)
    ↓
Job Queue → Per-Output Processing:
    ├─ Source Image Retrieval (Object Storage or Shopify CDN)
    ├─ AI Generation (Google Gemini with retry logic)
    ├─ Optional: FASHN Virtual Try-On (if model selected)
    ├─ Output Validation (format, dimensions, file integrity)
    └─ Storage & Metadata Update
    ↓
Real-time Status Polling → Frontend Updates
    ↓
Batch Export (ZIP download or Shopify push)
```

**Key Components:**
- **Frontend**: React + TypeScript with TanStack Query for state management
- **Backend**: Express.js with PostgreSQL (Drizzle ORM) and Google Cloud Storage
- **AI Services**: Google Gemini (image generation), FASHN API (virtual try-on)
- **Integrations**: Shopify Admin API (product import/export)
- **Authentication**: Session-based with workspace-scoped access control

## AI vs Deterministic Logic

**AI is used for:**
- **Image transformation** (color variants, fabric swaps, ghost mannequin removal, editorial styling) — Google Gemini handles photorealistic pixel-level edits that require understanding garment structure, lighting, and material properties
- **Virtual try-on** (FASHN API) — Generates model-worn images from flat-lay garments, requiring body geometry and fabric draping understanding
- **View angle generation** — Creates new camera angles (front/back/side) from a single source image, requiring 3D understanding

**Deterministic logic is used for:**
- **Prompt compilation** — Template variable substitution, conditional block processing, and Pantone color specification mapping
- **Job orchestration** — Output creation, status tracking, retry scheduling with exponential backoff
- **Image validation** — File format checks, dimension verification, storage path normalization
- **Shopify sync** — Product metadata mapping, variant matching, image position assignment
- **Access control** — Workspace membership verification, role-based permission checks

**Why these decisions:**
- AI handles tasks requiring visual understanding and creative generation that cannot be algorithmically defined
- Deterministic logic ensures reliability, reproducibility, and cost control for business-critical operations
- Hybrid approach balances quality (AI) with predictability (deterministic) for production use

## Failure Modes & Safeguards

**Invalid AI Output:**
- **Detection**: Format validation (PNG/JPEG), dimension checks, file size thresholds
- **Handling**: Automatic retry with exponential backoff (max 3 attempts), job-level failure isolation (one failed output doesn't cancel entire batch)
- **User Action**: Regeneration button per output, job-level cancellation, error messages with retryable vs. permanent failure classification

**Edge Cases:**
- **Missing source images**: Job creation validates image group existence; processing phase checks image availability before AI calls
- **API rate limits**: Request queuing with `p-limit` (concurrent request throttling), workspace-level credential validation before job start
- **Partial batch failures**: Per-output status tracking; completed outputs remain available even if job partially fails
- **Shopify sync conflicts**: Import uses style number as unique key; export batches are atomic (all-or-nothing per batch)

**Data Quality Issues:**
- **Corrupted uploads**: Pre-upload validation (file type, size limits), post-upload integrity checks before storage
- **Invalid view types**: Dropdown selection with predefined options, validation on assignment
- **Malformed prompts**: Template compilation validates all required variables, fallback to default templates if custom templates fail
- **Credential expiration**: Workspace credential validation endpoint, UI warnings for invalid API keys before job submission

**Operational Safeguards:**
- **Job cancellation**: Database flag checked before expensive AI calls; cleanup of in-progress outputs on cancellation
- **Timeout handling**: 5-minute max wait per job; automatic failure after timeout with user notification
- **Storage quota**: Object storage ACL policies prevent cross-workspace access; workspace-scoped queries enforce isolation

## Results & Impact

**Measured Metrics (from production usage):**
- **Time savings**: 8.5 minutes → 0.5 minutes per color variant (94% reduction)
- **Cost efficiency**: $26.25 → $1.54 per variant at $185/hour photographer rate (94% cost reduction)
- **Throughput**: 100+ images per batch processed asynchronously vs. sequential photography
- **Quality**: E-commerce-ready output meeting brand standards (validated by design team acceptance)

**Business Impact:**
- **Faster time-to-market**: Collections can launch with full color variant coverage in hours vs. weeks
- **Reduced sample production**: Physical samples only needed for final approval, not every variant
- **Scalability**: Handles seasonal collections with 500+ SKUs without linear cost increase
- **Shopify integration**: Direct export reduces manual upload time by ~80% for catalog management

## What I'd Improve Next

**Performance:**
- Implement Redis-based job queue for better horizontal scaling and job prioritization
- Add image caching layer (CDN) for frequently regenerated variants to reduce API costs
- Batch API requests where possible (e.g., Gemini batch API if available) to reduce latency

**Reliability:**
- Add comprehensive integration tests for transformation pipelines with golden image comparisons
- Implement A/B testing framework for prompt templates to measure quality improvements
- Add monitoring/alerting for API failure rates and job completion times

**Features:**
- Support for video generation (product videos, 360° views) using video-capable AI models
- Automated quality scoring (ML-based) to flag outputs needing manual review before export
- Multi-model fallback (e.g., OpenAI DALL-E as backup if Gemini fails) for critical jobs

**User Experience:**
- Real-time WebSocket updates for job status instead of polling
- Preview generation at lower resolution before full-quality output to reduce wasted API calls
- Collaborative features (comments, approvals) for team workflows
