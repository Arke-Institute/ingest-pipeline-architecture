# Arke Institute Ingest Pipeline - Architecture Overview

This repository contains the comprehensive architecture documentation for the Arke Institute Ingest Pipeline system.

## What This Repository Contains

- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Complete technical architecture documentation covering:
  - System overview and architecture diagrams
  - 14 main components across 6 architectural layers
  - Complete data flow from upload to archive
  - 7-phase orchestration pipeline
  - Queue architecture and storage design
  - API contracts and service bindings
  - Deployment and operations guide

## Component Repositories

The actual implementation is distributed across multiple component repositories:

### Client Layer
- `arke-upload-client` - Portable TypeScript SDK
- `arke-upload-frontend` - Web UI (Cloudflare Worker)

### Ingest Layer
- `arke-ingest-worker` - Upload coordination service

### Preprocessing Layer
- `arke-preprocessing-orchestrator` - Preprocessing coordinator
- `arke-preproccess-tiff-worker` - TIFF converter (Fly.io)
- `arke-preprocess-image-resize` - Image resizer (Fly.io)

### Orchestration Layer
- `arke-orchestrator` - Main batch processor (7-phase pipeline)

### AI Services Layer
- `arke-ocr-service` - OCR extraction
- `ai-services/metadata-service` - PINAX metadata extraction
- `ai-services/organizer-service` - File reorganization
- `ai-services/description-service` - Narrative generation
- `ai-services/linking-service` - Knowledge graph extraction

### Storage & CDN Layer
- `arke-cdn-worker` - Asset registration and CDN

## System Overview

The Arke Institute Ingest Pipeline is a distributed, cloud-native system for ingesting, processing, and archiving digital collections. It combines:

- **Cloudflare Workers** for serverless compute and orchestration
- **Durable Objects** for atomic state management
- **Fly.io** ephemeral machines for resource-intensive preprocessing
- **IPFS** for content-addressed entity storage with versioning
- **R2** for binary object storage
- **LLM Services** (DeepInfra) for OCR, metadata extraction, and descriptions

### Key Innovation: Incremental Publishing

Entities are published **immediately as v1** during the discovery phase, then atomically updated to v2, v3, v4, v5+ as subsequent phases complete. This enables early access to partial archival data while maintaining versioned integrity.

## Quick Start

For detailed information about the architecture, see [ARCHITECTURE.md](ARCHITECTURE.md).

For component-specific documentation, refer to each component's repository.

## Technology Stack

- **Compute**: Cloudflare Workers, Durable Objects, Fly.io
- **Storage**: R2, IPFS
- **Queues**: Cloudflare Queues
- **Languages**: TypeScript, Node.js
- **Frameworks**: Hono, Sharp, esbuild
- **External Services**: DeepInfra (LLM APIs)

## Documentation

- [ARCHITECTURE.md](ARCHITECTURE.md) - Complete architecture documentation
- Each component repository contains detailed `CLAUDE.md` development guides
- API specifications in component repositories (`API.md`, `QUEUE_MESSAGE_SPEC.md`)

## License

[Add license information here]

## Contact

[Add contact information here]
