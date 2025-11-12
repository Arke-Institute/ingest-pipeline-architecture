# Arke Institute Ingest Pipeline - Architecture Documentation

## Table of Contents
- [Executive Summary](#executive-summary)
- [System Overview](#system-overview)
- [Architecture Diagram](#architecture-diagram)
- [Component Breakdown](#component-breakdown)
- [Data Flow](#data-flow)
- [Orchestrator Pipeline Phases](#orchestrator-pipeline-phases)
- [Queue Architecture](#queue-architecture)
- [Storage Architecture](#storage-architecture)
- [API Contracts](#api-contracts)
- [Technology Stack](#technology-stack)
- [Key Design Patterns](#key-design-patterns)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Monitoring & Operations](#monitoring--operations)

---

## Executive Summary

The Arke Institute Ingest Pipeline is a distributed, cloud-native system for ingesting, processing, and archiving digital collections. It combines:

- **Cloudflare Workers** for serverless compute and orchestration
- **Durable Objects** for atomic state management
- **Fly.io** ephemeral machines for resource-intensive preprocessing
- **IPFS** for content-addressed entity storage with versioning
- **R2** for binary object storage
- **LLM Services** (DeepInfra) for OCR, metadata extraction, file organization, and descriptions

### Key Innovation: Incremental Publishing

The system publishes entities **immediately** as v1 during the discovery phase, then atomically updates them to v2, v3, v4, v5+ as subsequent phases complete. This enables early access to partial archival data while maintaining versioned integrity.

---

## System Overview

The pipeline processes user uploads through 7 phases:

1. **Upload** - Files uploaded directly to R2 via presigned URLs
2. **Preprocessing** - TIFF conversion and image resizing (Fly.io workers)
3. **Discovery** - Build directory tree, convert binaries to refs, publish v1 entities
4. **OCR** - Extract text from images, update entities to v2+
5. **Reorganization** - LLM-powered file grouping, create child directories
6. **PINAX Extraction** - Extract Dublin Core metadata, update entities to v3+
7. **Description** - Generate narrative descriptions, update entities to v4+

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER (Web/SDK)                               │
│  ┌──────────────┐   ┌──────────────────┐   ┌────────────────────────────┐  │
│  │  Desktop     │   │ arke-upload-     │   │ arke-upload-client SDK     │  │
│  │  Client      │──▶│ frontend         │◀──│ (Node.js/Browser/Deno)     │  │
│  │ (Electron)   │   │ (Cloudflare      │   │ Direct R2 uploads via      │  │
│  │              │   │  Worker)         │   │ presigned URLs             │  │
│  └──────────────┘   └──────────────────┘   └────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┬─┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     INGEST LAYER (Cloudflare Workers)                       │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ arke-ingest-worker (Cloudflare Worker)                              │   │
│  │ • POST /api/batches/init - Initialize upload batch                  │   │
│  │ • POST /api/batches/:id/files/start - Get presigned URLs            │   │
│  │ • POST /api/batches/:id/files/complete - Mark file uploaded         │   │
│  │ • POST /api/batches/:id/finalize - Enqueue to batch queue           │   │
│  │                                                                       │   │
│  │ State: Durable Objects (BatchStateObject) - atomic operations       │   │
│  │ Storage: R2 bucket (arke-staging)                                   │   │
│  │ Queue: arke-batch-jobs (for orchestrator)                           │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┬─┘
                                   │
                    Queue: arke-batch-jobs
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              PREPROCESSING LAYER (Cloudflare + Fly.io)                      │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ arke-preprocessing-orchestrator (Cloudflare Worker)                  │   │
│  │ • Consumes: arke-preprocess-jobs queue                              │   │
│  │ • Phases: TIFF conversion → Image resizing → Done                   │   │
│  │ • Spawns: Fly.io ephemeral machines for parallel processing         │   │
│  │ • State: Durable Objects (PreprocessingDurableObject)               │   │
│  │                                                                       │   │
│  │ Workers (Fly.io):                                                    │   │
│  │ • arke-preproccess-tiff-worker (Node.js) - TIFF → JPEG              │   │
│  │ • arke-preprocess-image-resize (Node.js) - Smart image variants     │   │
│  │   (thumb, medium, large, original)                                  │   │
│  │                                                                       │   │
│  │ Output: Transformed file list (TIFF + JPEG, images + variants)      │   │
│  │ Callback: Posts results back to arke-ingest-worker                  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┬─┘
                                   │
              After preprocessing complete, enqueue to:
              Queue: arke-batch-jobs (with transformed files)
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│           ORCHESTRATION LAYER (Cloudflare Workers - Main Pipeline)          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ arke-orchestrator (Cloudflare Worker)                                │   │
│  │ • Consumes: arke-batch-jobs queue                                   │   │
│  │ • Multi-phase state machine (per-batch Durable Object)              │   │
│  │ • Phases:                                                            │   │
│  │   - DISCOVERY: Build directory tree, convert binaries to refs,      │   │
│  │               publish initial entity snapshots (v1)                 │   │
│  │   - OCR: Extract text from images, update entities (v2+)            │   │
│  │   - REORGANIZATION: LLM-powered file grouping, create child dirs    │   │
│  │   - PINAX_EXTRACTION: Extract metadata, publish (v3+)               │   │
│  │   - CHEIMARROS: Extract knowledge graphs                            │   │
│  │   - DESCRIPTION: Generate AI descriptions, publish (v4+)            │   │
│  │   - PUBLISHING: Final versioning and archival                       │   │
│  │                                                                       │   │
│  │ Service Bindings (Worker-to-worker RPC):                            │   │
│  │ • CDN - arke-cdn-worker (register assets)                           │   │
│  │ • OCR_SERVICE - arke-ocr-service (DeepInfra LLM)                    │   │
│  │ • ORGANIZER_SERVICE - ai-services/organizer-service                 │   │
│  │ • METADATA_SERVICE - ai-services/metadata-service (PINAX)           │   │
│  │ • LINKING_SERVICE - ai-services/linking-service (Cheimarros)        │   │
│  │ • DESCRIPTION_SERVICE - ai-services/description-service             │   │
│  │ • IPFS_WRAPPER - arke-ipfs-api (entity CRUD)                        │   │
│  │                                                                       │   │
│  │ Storage:                                                              │   │
│  │ • R2: arke-staging (working files), arke-archive (permanent assets) │   │
│  │ • Durable Objects: Per-batch state persistence                      │   │
│  │ • Alarms: Scheduled phase processing (resumable, incremental)       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┬─┘
                                   │
                        IPFS Publishing
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    AI SERVICES LAYER (Cloudflare Workers)                   │
│  ┌─────────────────┐ ┌──────────────────┐ ┌──────────────────────────────┐  │
│  │ OCR Service     │ │ Metadata Service │ │ Description Service          │  │
│  │ DeepInfra       │ │ (PINAX)          │ │ (Narrative generation)       │  │
│  │ ollm-OCR-2      │ │ Mistral-Small    │ │ GPT-OSS-20B                  │  │
│  └─────────────────┘ └──────────────────┘ └──────────────────────────────┘  │
│  ┌─────────────────┐ ┌──────────────────┐                                   │
│  │ Organizer       │ │ Linking Service  │                                   │
│  │ (File grouping) │ │ (Knowledge graph)│                                   │
│  │ DeepSeek-V3     │ │ LLM-powered      │                                   │
│  └─────────────────┘ └──────────────────┘                                   │
└────────────────────────────────────────────────────────────────────────────┬─┘
                                   │
                            Entity Publishing
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      STORAGE & CDN LAYER                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ arke-cdn-worker (Cloudflare Worker)                                  │   │
│  │ • Asset registration and retrieval                                   │   │
│  │ • Generates asset IDs (ULIDs)                                        │   │
│  │ • Maps assets to CDN-accessible URLs                                 │   │
│  │ • Serves: cdn.arke.institute                                         │   │
│  │                                                                       │   │
│  │ R2 Storage:                                                           │   │
│  │ • arke-staging: Working directory (temporary)                        │   │
│  │ • arke-archive: Permanent asset storage (binaries, processed files)  │   │
│  │                                                                       │   │
│  │ IPFS Network:                                                         │   │
│  │ • Entity versioning via arke-ipfs-api                                │   │
│  │ • Content-addressed storage for all components                       │   │
│  │ • Incremental updates with atomic versioning                         │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### 1. CLIENT LAYER

#### arke-upload-client (@arke/upload-client)
- **Type**: Portable TypeScript SDK
- **Runtimes**: Node.js, Browser, Deno
- **Location**: `arke-upload-client/`
- **Key Responsibilities**:
  - File selection and scanning
  - CID computation using multiformats
  - Presigned URL-based uploads
  - Parallel uploads with configurable concurrency
  - Multipart uploads for files ≥5MB
  - Progress callbacks with token counting
  - Automatic retry with exponential backoff
- **Key Files**:
  - `src/uploader.ts` - Main SDK class
  - `src/platforms/` - Platform-specific adapters (Node.js, Browser)
  - `src/lib/worker-client-fetch.ts` - Worker API client
  - `src/lib/simple-fetch.ts`, `multipart-fetch.ts` - Upload engines

#### arke-upload-frontend
- **Type**: Web application (Cloudflare Worker)
- **Location**: `arke-upload-frontend/`
- **Tech Stack**: TypeScript, Hono, esbuild
- **Key Responsibilities**:
  - Serve HTML/CSS/JS upload UI
  - Proxy API calls to ingest worker and orchestrator
  - Progress tracking across three phases:
    1. Upload phase (direct to R2)
    2. Ingest queue polling
    3. Orchestrator polling (discovery → OCR → metadata → description)
- **Endpoints**:
  - `GET /` - Serve HTML/CSS/JS bundle
  - `POST /api/ingest/*` - Proxy to ingest worker
  - `POST /api/orchestrator/*` - Proxy to orchestrator
- **Service Bindings**:
  - `ORCHESTRATOR` → arke-orchestrator
  - `INGEST` → arke-ingest-worker

### 2. INGEST LAYER

#### arke-ingest-worker
- **Type**: Upload coordination service (Cloudflare Worker)
- **Location**: `arke-ingest-worker/`
- **Tech Stack**: TypeScript, Hono, AWS4Fetch, ULIDX
- **Key Responsibilities**:
  - Coordinate file uploads to R2 via presigned URLs
  - Manage batch state atomically via Durable Objects
  - Enqueue batches for preprocessing/orchestration
- **Architecture**:
  - HTTP fetch handler for API routes
  - Durable Objects (BatchStateObject) for atomic state management
  - Direct R2 integration (no file processing)
- **API Routes**:
  - `POST /api/batches/init` → Initialize batch (returns batch_id)
  - `GET /api/batches/:batchId/status` → Get batch progress
  - `POST /api/batches/:batchId/files/start` → Get presigned URL(s)
  - `POST /api/batches/:batchId/files/complete` → Mark file done
  - `POST /api/batches/:batchId/finalize` → Finalize and enqueue
- **R2 Structure**:
  ```
  staging/
  ├── {batchId}/
  │   ├── file1.tiff
  │   ├── subdir/
  │   │   └── file2.pdf
  │   └── ...
  ```
- **Queue Producer**: Sends to `arke-batch-jobs` (if no preprocessing) or `arke-preprocess-jobs`
- **Key Files**:
  - `src/index.ts` - Hono app entry point
  - `src/durable-objects/BatchState.ts` - State management DO
  - `src/handlers/` - Route handlers (init-batch, start-file, complete-file, finalize)
  - `src/lib/presigned.ts` - AWS v4 signature generation
  - `QUEUE_MESSAGE_SPEC.md` - Message format documentation
  - `API.md` - API reference

### 3. PREPROCESSING LAYER

#### arke-preprocessing-orchestrator
- **Type**: Phase orchestrator (Cloudflare Worker)
- **Location**: `arke-preproccessing-orchestrator/`
- **Key Responsibilities**: Coordinate TIFF conversion and image resizing
- **Queue Consumer**: `arke-preprocess-jobs`
- **State Machine**: Per-batch Durable Object tracking phase progression
- **Phases**:
  1. **TIFF Conversion** - Convert TIFF → JPEG
  2. **Image Processing** - Generate variants (thumb, medium, large)
  3. **Done** - Callback to ingest worker with transformed files
- **Fly.io Machine Spawning**:
  - Ephemeral machines created per task
  - Machines download from staging, process, upload to archive
  - Machines POST callback to `/callback/:batchId/:taskId`
  - Auto-destroy after completion
- **HTTP Endpoints**:
  - `GET /health` - Health check
  - `GET /status/:batchId` - Batch status
  - `POST /callback/:batchId/:taskId` - Machine callback handler
  - `POST /admin/reset/:batchId` - Reset stuck batch

#### arke-preproccess-tiff-worker
- **Type**: Ephemeral Node.js worker (Fly.io)
- **Location**: `arke-preproccess-tiff-worker/`
- **Tech Stack**: Node.js, Sharp, AWS SDK
- **Processing**:
  1. Download TIFF from R2 staging
  2. Convert to JPEG using Sharp (quality: 90, progressive)
  3. Upload JPEG to R2 staging
  4. POST callback with results
  5. Exit with code 0 (success) or 1 (failure)
- **Environment Variables**: `TASK_ID`, `BATCH_ID`, `INPUT_R2_KEY`, `CALLBACK_URL`, R2 credentials

#### arke-preprocess-image-resize
- **Type**: Ephemeral Node.js worker (Fly.io)
- **Location**: `arke-preprocess-image-resize/`
- **Tech Stack**: Node.js, Sharp, AWS SDK, ULID
- **Processing**:
  1. Download image from staging
  2. Analyze metadata (dimensions, format)
  3. Generate variants:
     - **thumb**: 150x150 (WebP)
     - **medium**: 800x800 (WebP)
     - **large**: Full size (WebP or original format)
     - **original**: Preserved
  4. Register with CDN service
  5. Create `.ref.json` in staging with CDN mappings
  6. Upload to archive
  7. POST callback with variant metadata

### 4. ORCHESTRATION LAYER

#### arke-orchestrator
- **Type**: Main batch processing orchestrator (Cloudflare Worker)
- **Location**: `arke-orchestrator/`
- **Tech Stack**: TypeScript, Cloudflare Durable Objects, Service Bindings
- **Key Responsibility**: Multi-phase pipeline execution with atomic versioning
- **Queue Consumer**: `arke-batch-jobs`
- **HTTP Endpoints**:
  - `GET /status/:batchId` - Poll batch status
  - `GET /health` - Health check with optional service verification
  - `POST /admin/reset/:batchId` - Reset stuck batch (admin)
- **State Machine**: Per-batch Durable Object managing 7 phases
- **Key Files**:
  - `src/index.ts` - Queue consumer + HTTP handler
  - `src/batch-do.ts` - Durable Object with state machine
  - `src/types.ts` - TypeScript interfaces
  - `src/phases/` - Phase implementations (discovery, ocr, reorganization, pinax, description)
  - `src/services/` - Service binding wrappers
  - `CLAUDE.md` - Detailed development guide

### 5. AI SERVICES LAYER

#### arke-ocr-service
- **Type**: OCR extraction via LLM (Cloudflare Worker)
- **Location**: `arke-ocr-service/`
- **Tech Stack**: TypeScript, DeepInfra API (ollm-OCR-2)
- **Endpoint**: `POST /ocr`
- **Request**:
  ```json
  {
    "image_url": "https://cdn.arke.institute/asset/...",
    "content_type": "image/jpeg"
  }
  ```
- **Response**:
  ```json
  {
    "text": "Extracted OCR text content...",
    "confidence": 0.95,
    "language": "en"
  }
  ```

#### ai-services/metadata-service
- **Type**: PINAX metadata extraction (Cloudflare Worker)
- **Location**: `ai-services/metadata-service/`
- **Tech Stack**: TypeScript, DeepInfra API (Mistral-Small-3.2-24B)
- **Endpoints**:
  - `POST /extract-metadata` - LLM-based extraction
  - `POST /validate-metadata` - Schema validation only
- **Output**: Dublin Core-compliant metadata (PINAX schema)
- **DCMI Types**: Collection, Dataset, Event, Image, StillImage, Text, etc.
- **Token Budget Logic**:
  - If text tokens < 10,000: include OCR from refs
  - If text tokens ≥ 10,000: exclude OCR (cost optimization)

#### ai-services/organizer-service
- **Type**: LLM-powered file reorganization (Cloudflare Worker)
- **Location**: `ai-services/organizer-service/`
- **Tech Stack**: TypeScript, DeepInfra API (DeepSeek-V3)
- **Endpoint**: `POST /organize`
- **Key Features**:
  - Files can appear in multiple groups
  - IPFS handles deduplication (zero storage overhead)
  - Groups inherit parent's processing_config
  - Creates new child directory entities for each group

#### ai-services/description-service
- **Type**: Narrative description generation (Cloudflare Worker)
- **Location**: `ai-services/description-service/`
- **Tech Stack**: TypeScript, DeepInfra API (GPT-OSS-20B)
- **Endpoint**: `POST /summarize`
- **Output**: Markdown-formatted narrative with structured sections:
  - Overview (50-100 words)
  - Background (50-100 words)
  - Contents (50-100 words)
  - Scope (50-100 words)
- **Target**: 200-350 words total

#### ai-services/linking-service
- **Type**: Knowledge graph extraction (Cloudflare Worker)
- **Location**: `ai-services/linking-service/`
- **Tech Stack**: TypeScript, DeepInfra LLM
- **Responsibility**: Extract entity relationships and knowledge graphs (Cheimarros protocol)

### 6. STORAGE & CDN LAYER

#### arke-cdn-worker
- **Type**: Asset registration and CDN serving (Cloudflare Worker)
- **Location**: `arke-cdn-worker/`
- **Tech Stack**: TypeScript, Hono, R2
- **Routes**:
  - `POST /asset/:assetId` - Register asset metadata
  - `GET /asset/:assetId` - Retrieve asset
  - `GET /asset/:assetId/*` - Handle variant paths
- **Responsibility**:
  - Generate asset IDs (ULIDs)
  - Map assets to public CDN URLs
  - Handle variant retrieval (thumb, medium, large)
  - Serve from R2 archive bucket

#### R2 Storage Buckets
- **arke-staging**:
  - Working directory for batch processing
  - Original uploaded files
  - `.ref.json` files (binary references)
  - Processing outputs (pinax.json, description.md, etc.)
  - Temporary storage (cleaned after archival)
- **arke-archive**:
  - Permanent storage for binary assets
  - Variant images (thumb, medium, large)
  - OCR-processed files
  - Immutable content

#### IPFS Integration (via arke-ipfs-api)
- **Responsibility**: Entity CRUD with versioning
- **Operations**:
  - `POST /upload` - Upload content, get CID
  - `POST /entities` - Create new entity (v1)
  - `PUT /entities/:pi` - Update entity (CAS)
  - `GET /entities/:pi` - Retrieve entity
  - `GET /resolve/:pi` - Resolve PI to latest CID
- **Versioning**:
  - v1: Initial snapshot (discovery phase)
  - v2+: OCR updates
  - v3+: Reorganization updates
  - v4+: PINAX metadata updates
  - v5+: Description updates
- **Bidirectional Relationships**:
  - Setting `parent_pi` automatically appends entity PI to parent's `children_pi`

---

## Data Flow

### Complete Upload-to-Archive Workflow

```
1. USER: Opens arke-upload-frontend
2. FRONTEND: Renders upload form with progress tracking

3. USER: Selects files for upload
4. CLIENT: Scans directory, computes CIDs
5. CLIENT: Calls POST /api/batches/init on arke-ingest-worker
6. INGEST: Generates batch_id (ULID), creates Durable Object
7. INGEST → CLIENT: Returns batch_id, session_id

8. CLIENT: For each file:
   a) Calls POST /api/batches/{id}/files/start
   b) INGEST: Validates file, creates multipart upload, generates presigned URLs
   c) INGEST → CLIENT: Returns presigned_urls
   d) CLIENT: Uploads directly to R2 using presigned URLs (bypasses ingest worker)
   e) CLIENT: Collects ETags from R2 responses
   f) CLIENT: Calls POST /api/batches/{id}/files/complete with ETags
   g) INGEST: Completes multipart upload, marks file as completed

9. CLIENT: Calls POST /api/batches/{id}/finalize
10. INGEST: Verifies all files completed, constructs QueueMessage
11. INGEST: Enqueues to arke-preprocess-jobs (if image preprocessing needed)
    OR
    INGEST: Enqueues directly to arke-batch-jobs (if no preprocessing)

12. [IF PREPROCESSING]
    a) arke-preprocessing-orchestrator consumes from arke-preprocess-jobs
    b) PREPROC: Creates Durable Object for batch, phases through TIFF → image resizing
    c) PREPROC: Spawns Fly.io machines for parallel processing
    d) FLY WORKERS: Download files, process, upload variants to archive
    e) FLY WORKERS: POST callbacks to preprocessing orchestrator
    f) PREPROC: Aggregates results, transforms file list
    g) PREPROC: Calls back to arke-ingest-worker with processed file list
    h) INGEST: Enqueues to arke-batch-jobs with updated files

13. arke-orchestrator consumes from arke-batch-jobs
14. ORCHESTRATOR: Creates Durable Object for batch
15. ORCHESTRATOR: Phase: DISCOVERY
    - Builds directory tree
    - Converts binaries to .ref.json
    - Calls CDN worker to register assets
    - Publishes initial entity snapshots (v1) to IPFS
    - Schedules alarm for OCR phase

16. ORCHESTRATOR: Phase: OCR_IN_PROGRESS
    - Processes directories in parallel batches
    - Calls arke-ocr-service for each image ref
    - Updates .ref.json with OCR text
    - Publishes entity updates (v2+) to IPFS
    - Schedules alarm for REORGANIZATION phase

17. ORCHESTRATOR: Phase: REORGANIZATION
    - Calls organizer-service to group files intelligently
    - Creates new child directory entities for each group
    - Publishes entity updates (v3+) to IPFS
    - Schedules alarm for PINAX phase

18. ORCHESTRATOR: Phase: PINAX_EXTRACTION
    - Calls metadata-service to extract PINAX Dublin Core metadata
    - Publishes entity updates (v3+) with pinax.json component
    - Schedules alarm for CHEIMARROS phase

19. ORCHESTRATOR: Phase: CHEIMARROS_EXTRACTION
    - Calls linking-service for knowledge graph extraction
    - Publishes entity updates with knowledge graph components
    - Schedules alarm for DESCRIPTION phase

20. ORCHESTRATOR: Phase: DESCRIPTION
    - Calls description-service to generate narrative descriptions
    - Publishes entity updates (v4+) with description.md component
    - Transitions to DONE

21. ORCHESTRATOR: Phase: DONE
    - All directories processed
    - root_pi available for archive access
    - Batch marked as completed

22. FRONTEND: Meanwhile polling arke-orchestrator for status
    - After discovery: Shows root_pi link (early access to partial data)
    - Tracks phase progression: OCR → PINAX → Description
    - Shows completion status and final archive link

23. USER: Accesses archive at root_pi IPFS address
    - Hierarchical entity structure accessible
    - Entities versioned (v1-v5+)
    - All components addressed via IPFS
    - CDN variants available for images
    - Metadata, OCR, descriptions available per directory
```

---

## Orchestrator Pipeline Phases

### Phase 1: DISCOVERY (Synchronous)
**Purpose**: Build initial directory tree and publish v1 entities

**Process**:
1. Build hierarchical directory tree from queue message's `directories[]`
2. Classify files:
   - Text files (`.md`, `.txt`, etc.) → read directly, include as component CIDs
   - Binary assets (images, PDFs) → copy to ARCHIVE_BUCKET, register with CDN, convert to `.ref.json`
   - Pre-existing `.ref.json` files → parse and validate
3. Apply per-directory `processing_config` (OCR/describe/PINAX flags)
4. **Publish initial entity snapshots (v1)** bottom-up with:
   - Text file contents as CIDs in `components`
   - Ref JSON files as CIDs in `components`
   - Parent-child relationships via `parent_pi`, update parent's `children_pi`
5. Assign PI and version tracking
6. Schedule first alarm

**Location**: `arke-orchestrator/src/phases/discovery.ts`

### Phase 2: OCR_IN_PROGRESS (Alarm-driven)
**Purpose**: Extract text from images

**Process**:
1. Process `BATCH_SIZE_OCR` directories in parallel per alarm
2. For each directory, OCR all refs needing OCR
3. Call OCR service for each image ref
4. Update `RefData.ocr` in staging `.ref.json`
5. **Publish entity update (v2+)** with updated ref components
6. Mark directory as `ocr_complete: true`
7. Reschedule alarm if more remain

**Location**: `arke-orchestrator/src/phases/ocr.ts`

### Phase 3: REORGANIZATION (Alarm-driven)
**Purpose**: LLM-powered intelligent file grouping

**Process**:
1. Process `BATCH_SIZE_REORGANIZATION` directories in parallel
2. Only directories with ≥`MIN_FILES_FOR_REORGANIZATION` or `processing_config.reorganize === true`
3. Call organizer service with local files + OCR text
4. Service returns overlapping file groups
5. **Create new child directory entities** for each group:
   - Share same component CIDs (IPFS deduplication handles storage)
   - Inherit parent's `processing_config`
6. **Update parent entity**:
   - Remove grouped files from components
   - Add `reorganization-description.txt` component
   - Update `children_pi` with new groups
7. Mark directory as `reorganization_complete: true`
8. New groups added to processing queue

**Location**: `arke-orchestrator/src/phases/reorganization.ts`

### Phase 4: PINAX_EXTRACTION (Alarm-driven, bottom-up)
**Purpose**: Extract Dublin Core-compliant metadata

**Process**:
1. Process `BATCH_SIZE_METADATA` directories in parallel (deepest first)
2. Only if `processing_config.pinax === true`
3. Aggregate content:
   - Local text files
   - OCR text from refs (if available)
   - PINAX metadata from child directories (if available)
4. Call metadata service for structured extraction
5. Save `pinax.json` to staging
6. **Publish entity update (v3+)** with new `pinax.json` component
7. Mark directory as `pinax_complete: true`

**Location**: `arke-orchestrator/src/phases/pinax.ts`

### Phase 5: CHEIMARROS_EXTRACTION (Alarm-driven, bottom-up)
**Purpose**: Extract knowledge graphs and entity relationships

**Process**:
1. Process `BATCH_SIZE_CHEIMARROS` directories in parallel
2. Extract knowledge graphs and entity relationships
3. Call linking service for LLM-powered extraction
4. Generate structured knowledge graph
5. Mark directory as `cheimarros_complete: true`

### Phase 6: DESCRIPTION (Alarm-driven, bottom-up)
**Purpose**: Generate narrative descriptions

**Process**:
1. Process `BATCH_SIZE_DESCRIPTION` directories in parallel (deepest first)
2. Only if `processing_config.describe === true`
3. Aggregate content:
   - Local text files
   - OCR text from refs (token budget-aware)
   - Descriptions from child directories
   - PINAX metadata
4. Call description service for narrative generation
5. Save `description.md` to staging
6. **Publish entity update (v4+)** with new `description.md` component
7. Mark directory as `description_complete: true`

**Location**: `arke-orchestrator/src/phases/description.ts`

### Phase 7: DONE
**Purpose**: Finalize batch processing

**State**:
- All directories processed
- Root entity PI available in `state.root_pi`
- Set `completed_at` timestamp
- No further alarms scheduled

---

## Queue Architecture

### Queue Names and Consumers

#### arke-preprocess-jobs
- **Consumer**: arke-preprocessing-orchestrator
- **Purpose**: Batch preprocessing tasks (TIFF conversion, image resizing)
- **Max Batch Size**: 1
- **Max Retries**: 3

#### arke-batch-jobs
- **Consumer**: arke-orchestrator
- **Purpose**: Main ingestion pipeline batches
- **Max Batch Size**: 1
- **Max Retries**: 3

### Queue Message Schema (arke-batch-jobs)

```typescript
interface QueueMessage {
  batch_id: string;                          // ULID
  r2_prefix: string;                         // "staging/{batchId}/"
  uploader: string;                          // User/system identity
  root_path: string;                         // Logical root path
  parent_pi?: string;                        // Optional parent entity
  total_files: number;
  total_bytes: number;
  uploaded_at: string;                       // ISO 8601
  finalized_at: string;                      // ISO 8601
  metadata: Record<string, any>;
  directories: DirectoryGroup[];
}

interface DirectoryGroup {
  directory_path: string;
  processing_config: ProcessingConfig;
  file_count: number;
  total_bytes: number;
  files: QueueFileInfo[];
}

interface ProcessingConfig {
  ocr: boolean;
  reorganize?: boolean;                      // undefined = threshold, true = always, false = never
  pinax: boolean;
  cheimarros: boolean;
  describe: boolean;
}

interface QueueFileInfo {
  r2_key: string;                           // Full key: "staging/{batchId}/path/file"
  logical_path: string;                     // Relative: "/path/file"
  file_name: string;                        // Just filename
  file_size: number;
  content_type: string;
  cid?: string;                             // Optional from upload
}
```

**Documentation**: See `arke-ingest-worker/QUEUE_MESSAGE_SPEC.md`

---

## Storage Architecture

### R2 Buckets

#### arke-staging
- **Purpose**: Working directory for batch processing
- **Contents**:
  - Original uploaded files
  - `.ref.json` files (binary references)
  - Working files from preprocessing
  - Temporary outputs (pinax.json, description.md)
- **Lifecycle**: Cleaned after archival

#### arke-archive
- **Purpose**: Permanent binary asset storage
- **Contents**:
  - Permanent binary assets
  - Variant images (thumb, medium, large)
  - OCR-processed content
- **Lifecycle**: Immutable storage

### IPFS Integration

**Entity Versioning**:
- v1: Initial snapshot (discovery phase)
- v2+: OCR updates
- v3+: Reorganization + PINAX metadata updates
- v4+: Description updates

**Bidirectional Relationships**:
- Setting `parent_pi` on entity automatically appends entity PI to parent's `children_pi`
- No manual parent updates needed in orchestrator

**CAS (Compare-and-Swap)**: Prevents race conditions during entity updates

### RefData Schema

Binary assets are converted to lightweight `.ref.json` files containing CDN URLs:

```typescript
interface RefData {
  url: string;              // CDN URL (REQUIRED)
  ipfs_cid?: string;        // Original IPFS CID
  type?: string;            // MIME type
  size?: number;            // File size
  filename?: string;        // Original name
  ocr?: string;             // Added during OCR phase
}
```

---

## API Contracts

### arke-ingest-worker

**POST /api/batches/init**
- Initialize batch for upload
- Request: `{ uploader, file_count, total_size, metadata }`
- Response: `{ batch_id, session_id }`

**GET /api/batches/:batchId/status**
- Poll batch status and progress
- Response: Full batch state with file list

**POST /api/batches/:batchId/files/start**
- Get presigned URL(s) for file
- Request: `{ file_name, file_size, logical_path, content_type }`
- Response (simple): `{ method: "PUT", url, headers }`
- Response (multipart): `{ upload_id, urls: [...], part_size }`

**POST /api/batches/:batchId/files/complete**
- Mark file as uploaded
- Request (multipart): `{ r2_key, upload_id, parts: [{ part_number, etag }, ...] }`

**POST /api/batches/:batchId/finalize**
- Finalize batch, trigger preprocessing/orchestration
- Response: `{ status: "enqueued", batch_id }`

**Documentation**: See `arke-ingest-worker/API.md`

### arke-orchestrator

**GET /status/:batchId**
- Poll batch processing status
- Response: Full batch state with phase progress

**GET /health**
- Health check with optional service verification
- Query param: `?check=services`
- Response: `{ status, timestamp, services }`

**POST /admin/reset/:batchId**
- Reset stuck batch (admin only)
- Response: `{ status: "reset", batch_id }`

### Service Binding Call Format

```typescript
// All service calls use service bindings (not HTTP URLs)
const response = await env.SERVICE_NAME.fetch('https://service/path', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload)
});
```

**Note**: Hostname is ignored by service bindings; can be anything semantic

---

## Technology Stack

### Compute Platforms
- **Cloudflare Workers**: Edge computing for orchestration, AI services, frontend
- **Durable Objects**: Atomic state management per batch/preprocessing task
- **Fly.io**: Ephemeral machines for TIFF conversion, image resizing
- **Node.js**: Script runtime for preprocessing workers

### Storage
- **R2**: Object storage (staging, archive buckets)
- **IPFS**: Content-addressed storage via arke-ipfs-api

### APIs & External Services
- **DeepInfra**: LLM access (GPT-OSS-20B for OCR, Mistral-Small for metadata, DeepSeek-V3 for organization)
- **AWS SDK**: R2 credential signing
- **Cloudflare Queue**: Asynchronous batch processing

### Languages & Frameworks
- **TypeScript**: All workers and services
- **Hono**: Lightweight web framework for workers
- **Sharp**: Image processing (TIFF conversion, resizing)
- **Multiformats**: CID computation (upload-client)
- **esbuild**: Client bundle compilation

### Key Libraries
- **ulidx**: Unique ID generation
- **aws4fetch**: AWS v4 signature for presigned URLs
- **vitest**: Testing framework
- **wrangler**: Cloudflare CLI and deployment tool

---

## Key Design Patterns

### 1. Incremental Publishing
- Entities published immediately as v1 during discovery (not after all phases complete)
- Each phase updates entity to new version (v2, v3, v4, v5+)
- Enables early access to partial data
- Atomic versioning per entity
- Graceful degradation (v1 entities usable even if downstream phases fail)

### 2. Resumability
- All state persisted after every operation
- Phase flags track completion (`snapshot_published`, `ocr_complete`, etc.)
- If DO crashes or times out, next alarm resumes from last checkpoint
- Already-processed items skipped based on flags

### 3. Binary-to-Ref Conversion
- Binary assets converted to lightweight `.ref.json` files during discovery
- Original binary: R2 archive (permanent)
- Ref JSON: Contains CDN URL, size, MIME type, original IPFS CID, OCR text (added later)
- Keeps IPFS lightweight while maintaining CDN access

### 4. Bottom-Up Processing
- PINAX and description phases process deepest directories first
- Child content available for parent aggregation
- Discovery publishes bottom-up so child PIs exist for parent's `children_pi`

### 5. Bidirectional Relationships
- Setting `parent_pi` on entity automatically appends entity PI to parent's `children_pi`
- No manual parent updates needed in orchestrator

### 6. Service Bindings Over HTTP
- All inter-service calls use Cloudflare service bindings (not HTTP URLs)
- Direct worker-to-worker RPC
- No CORS configuration needed
- Lower latency, guaranteed delivery

### 7. Parallel Durable Object Alarms
- Each phase has separate alarm with configurable delay
- Multiple batches can progress through phases in parallel
- Batch-level parallelism via separate DO instances

### 8. Fly.io Ephemeral Machines
- Spawned only when needed for resource-intensive work
- Auto-destroy after completion
- Callback-based completion (not polling)
- Horizontally scalable preprocessing

---

## Configuration

### Environment Variables (Critical)

**R2 Storage**:
- `R2_ACCOUNT_ID` - R2 account identifier
- `R2_ACCESS_KEY_ID` - Access key for R2
- `R2_SECRET_ACCESS_KEY` - Secret key for R2

**DeepInfra API**:
- `DEEPINFRA_API_KEY` - API key for LLM services
- `DEEPINFRA_BASE_URL` - API endpoint
- `MODEL_NAME` - Model identifier

**Fly.io**:
- `FLY_API_TOKEN` - API token for machine spawning
- `FLY_TIFF_APP_NAME` - TIFF worker app name
- `FLY_IMAGE_APP_NAME` - Image resizer app name
- `FLY_REGION` - Deployment region (e.g., "ord")

**URLs**:
- `CDN_PUBLIC_URL` - CDN base URL
- `ORCHESTRATOR_URL` - Orchestrator endpoint
- `WORKER_URL` - Worker endpoint

### Configuration Files

**wrangler.jsonc** (per component):
- R2 bucket bindings
- Queue producers/consumers
- Durable Object bindings
- Service bindings (worker-to-worker RPC)
- Environment variables

**Example** (`arke-orchestrator/wrangler.jsonc`):
```jsonc
{
  "r2_buckets": [
    { "binding": "STAGING_BUCKET", "bucket_name": "arke-staging" },
    { "binding": "ARCHIVE_BUCKET", "bucket_name": "arke-archive" }
  ],
  "queues": {
    "consumers": [
      { "queue": "arke-batch-jobs", "max_batch_size": 1, "max_retries": 3 }
    ]
  },
  "durable_objects": [
    { "name": "BATCH_DO", "class_name": "BatchDurableObject" }
  ],
  "services": [
    { "binding": "CDN", "service": "arke-cdn-worker" },
    { "binding": "OCR_SERVICE", "service": "arke-ocr-service" },
    { "binding": "ORGANIZER_SERVICE", "service": "arke-organizer-service" },
    { "binding": "METADATA_SERVICE", "service": "arke-metadata-service" },
    { "binding": "LINKING_SERVICE", "service": "arke-linking-service" },
    { "binding": "DESCRIPTION_SERVICE", "service": "arke-description-service" },
    { "binding": "IPFS_WRAPPER", "service": "arke-ipfs-api" }
  ],
  "vars": {
    "BATCH_SIZE_OCR": "10",
    "BATCH_SIZE_REORGANIZATION": "5",
    "BATCH_SIZE_METADATA": "5",
    "BATCH_SIZE_CHEIMARROS": "3",
    "BATCH_SIZE_DESCRIPTION": "5",
    "ALARM_DELAY_OCR": "3000",
    "ALARM_DELAY_REORGANIZATION": "3000"
  }
}
```

### Tunable Parameters

**Batch Sizes** (adjust parallelism):
- `BATCH_SIZE_OCR`: Directories processed per OCR alarm
- `BATCH_SIZE_REORGANIZATION`: Directories processed per reorganization alarm
- `BATCH_SIZE_METADATA`: Directories processed per PINAX alarm
- `BATCH_SIZE_CHEIMARROS`: Directories processed per Cheimarros alarm
- `BATCH_SIZE_DESCRIPTION`: Directories processed per description alarm

**Alarm Delays** (balance CPU usage vs speed):
- `ALARM_DELAY_OCR`: Milliseconds between OCR alarms
- `ALARM_DELAY_REORGANIZATION`: Milliseconds between reorganization alarms
- `ALARM_DELAY_METADATA`: Milliseconds between PINAX alarms
- `ALARM_DELAY_DESCRIPTION`: Milliseconds between description alarms

**Retry Attempts**:
- `MAX_RETRY_ATTEMPTS`: Failure resilience for processing operations

**Token Budgets**:
- Cost optimization for LLM calls (e.g., OCR token threshold)

**File Size Limits**:
- `MAX_FILE_SIZE`: Maximum file size (default: 5GB)
- `MAX_BATCH_SIZE`: Maximum batch size (default: 100GB)

---

## Deployment

### Deployment Order

1. Deploy shared libraries (`arke-shared`)
2. Deploy storage layer (`arke-cdn-worker`)
3. Deploy AI services (`metadata-service`, `description-service`, `organizer-service`, `linking-service`)
4. Deploy OCR service (`arke-ocr-service`)
5. Deploy orchestrator (`arke-orchestrator`)
6. Deploy preprocessing orchestrator (`arke-preprocessing-orchestrator`)
7. Deploy TIFF and image workers (`arke-preproccess-tiff-worker`, `arke-preprocess-image-resize`)
8. Deploy ingest worker (`arke-ingest-worker`)
9. Deploy frontend (`arke-upload-frontend`)
10. Deploy client SDK (`@arke/upload-client`)

### Prerequisites

- Cloudflare account with Workers, Durable Objects, Queues, R2 enabled
- Fly.io account for ephemeral machine spawning
- DeepInfra API key for LLM services
- IPFS wrapper service (arke-ipfs-api) deployed
- DNS records for custom domains (ingest.arke.institute, orchestrator.arke.institute, etc.)

### Deployment Commands

**Per component**:
```bash
cd component-directory
npm install
npm run build
wrangler deploy
```

**Secrets**:
```bash
wrangler secret put DEEPINFRA_API_KEY
wrangler secret put R2_SECRET_ACCESS_KEY
wrangler secret put FLY_API_TOKEN
```

---

## Monitoring & Operations

### Logging

**Stream logs from deployed worker**:
```bash
wrangler tail {worker-name}
```

**Examples**:
```bash
wrangler tail arke-ingest-worker
wrangler tail arke-orchestrator
wrangler tail arke-preprocessing-orchestrator
```

### Status Endpoints

**Poll batch status**:
```bash
curl https://orchestrator.arke.institute/status/{batchId}
```

**Health check**:
```bash
curl https://orchestrator.arke.institute/health
curl https://orchestrator.arke.institute/health?check=services
```

### Queue Metrics

**Cloudflare Dashboard**:
- Throughput (messages/second)
- Lag (queue depth)
- Retries (failed message count)
- Dead Letter Queue (DLQ) size

### Batch State Inspection

Query Durable Object state via status endpoints:
```bash
curl https://ingest.arke.institute/api/batches/{batchId}/status
curl https://orchestrator.arke.institute/status/{batchId}
curl https://preprocessing.arke.institute/status/{batchId}
```

### Performance Tuning

**Increase processing speed**:
- Decrease alarm delays (`ALARM_DELAY_*`)
- Increase batch sizes (`BATCH_SIZE_*`)
- Increase client-side concurrency

**Reduce CPU usage**:
- Increase alarm delays
- Decrease batch sizes

**Cost optimization**:
- Tune token budgets for LLM calls
- Adjust OCR token threshold
- Configure file size limits

### Troubleshooting

**Reset stuck batch**:
```bash
curl -X POST https://orchestrator.arke.institute/admin/reset/{batchId}
curl -X POST https://preprocessing.arke.institute/admin/reset/{batchId}
```

**Check service bindings**:
```bash
curl https://orchestrator.arke.institute/health?check=services
```

---

## Security Considerations

### Authentication & Authorization
- **Ingest worker**: Open API (restrict CORS in production)
- **Orchestrator**: No authentication (internal queue consumer)
- **Service bindings**: No external authentication (Cloudflare internal)

### Data Privacy
- Consider encrypting sensitive metadata
- Rotate R2 credentials regularly
- Implement audit logging
- Monitor DLQ for failed messages

### Resource Limits
- Presigned URL expiry: 1 hour (default)
- Max file size: 5GB per file
- Max batch size: 100GB per batch
- Queue batch sizes: Prevent memory exhaustion
- Timeouts: Long-running operation protection

---

## Future Enhancements

**Completed**:
- Multi-phase orchestration (DISCOVERY through DESCRIPTION)
- Incremental publishing with versioning
- Service bindings for inter-service communication
- Preprocessing pipeline with Fly.io
- PINAX metadata extraction
- LLM-powered description generation
- LLM-powered file reorganization

**Planned**:
- Dashboard (`arke-ingest-ui`)
- Status tracking (`arke-ingest-status`)
- Direct IPFS upload support
- More preprocessing (PDF, video, formats)
- Full Cheimarros knowledge graph implementation
- Template service for structural patterns
- Cost tracking and billing
- Webhook notifications
- Multi-tenant support
- Batch prioritization

---

## Additional Documentation

Each component has detailed documentation in its directory:

- `CLAUDE.md` - Development guide with implementation details
- `API.md` - API reference (ingest-worker)
- `QUEUE_MESSAGE_SPEC.md` - Message format documentation (ingest-worker)
- `wrangler.jsonc` - Configuration files with all bindings
- `README.md` - Component-specific setup and usage

---

## Conclusion

The Arke Institute Ingest Pipeline is a sophisticated, cloud-native architecture designed to ingest, process, and publish digital collections with minimal operational overhead. By leveraging Cloudflare's edge compute, serverless databases, and queue systems, combined with specialized LLM services for content analysis, the system achieves scalability, resilience, and cost-effectiveness.

The incremental publishing model enables early access to archival content while maintaining atomic versioning for each processing phase, creating a foundation for federated, decentralized archival systems.

**Key Innovations**:
- Incremental publishing with atomic versioning
- Binary-to-ref conversion pattern
- Service bindings for zero-latency inter-service RPC
- Resumable, alarm-driven phase processing
- Bottom-up content aggregation
- IPFS deduplication for cost efficiency
