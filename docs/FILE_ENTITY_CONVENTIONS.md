# File Entity Conventions

This document describes the conventions used for file entities in the Arke platform. File entities are built on the [Eidos schema](../../ipfs_wrapper/docs/ENTITY-SCHEMA.md) but follow specific patterns for representing uploaded files.

## Overview

When files are uploaded and processed, the preprocessing pipeline creates **file entities** - one entity per file. These entities store metadata about the file and link back to their parent PI.

## Entity Structure

### Type

File entities use `type: "file"` to distinguish them from other entity types (PI, person, place, etc.).

### ID Format

File entity IDs use a deterministic format:
```
F{HASH}
```

Where:
- `F` prefix identifies this as a file entity
- `{HASH}` is a 25-character base32 hash derived from: `SHA256(parent_pi + "/" + filename)`

This ensures:
- Same file uploaded to same location always gets the same ID
- No collisions between different files
- Predictable IDs for deduplication

### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `id` | Deterministic file ID | `FQQ8FN65GYEY8VENMVB6Y5DQFB` |
| `type` | Always `"file"` | `"file"` |
| `label` | Original filename | `"document.pdf"` |
| `source_pi` | Parent PI that owns this file | `"01KCZ86R3DMMB0SENG89TGGAWD"` |

### Inline Properties

File metadata is stored as **inline properties** directly in the manifest (not as a separate CID component). This provides immediate access without additional API calls.

| Property | Type | Description |
|----------|------|-------------|
| `filename` | string | Original filename |
| `content_type` | string | MIME type (e.g., `"image/jpeg"`) |
| `cdn_url` | string | CDN URL for processed file |
| `archive_key` | string | R2 storage key for original |
| `original_size` | number | Original file size in bytes |
| `original_cid` | string | (optional) IPFS CID of original upload |

### Example File Entity

```json
{
  "id": "FQQ8FN65GYEY8VENMVB6Y5DQFB",
  "type": "file",
  "created_at": "2025-12-21T01:26:25.611Z",
  "label": "a092_001_001.jpg",
  "ver": 1,
  "ts": "2025-12-21T01:26:25.611Z",
  "manifest_cid": "bafyreigdz7x7zygtipqkz7nibfpf3tbq5ttw6sr7pmmksbsq4cmkjeetf4",
  "components": {},
  "properties": {
    "filename": "a092_001_001.jpg",
    "content_type": "image/jpeg",
    "cdn_url": "https://cdn.arke.institute/asset/01KCZ87KK79MHYRY37HBRGWXDW",
    "archive_key": "archive/01KCZ86JQEBV8CSPB971H42BHS/01KCZ87KK79MHYRY37HBRGWXDW/original.jpg",
    "original_size": 318066
  },
  "source_pi": "01KCZ86R3DMMB0SENG89TGGAWD",
  "note": "Created by preprocessing orchestrator"
}
```

## Components

File entities may include these optional components (stored as CIDs):

| Component | Description |
|-----------|-------------|
| `ocr` | OCR text extracted from images |

## Relationship to Parent PI

File entities are linked to their parent PI via:

1. **`source_pi`**: Points to the PI that owns this file (provenance)
2. **`files.json`**: The parent PI maintains a `files.json` component listing all file entities

### files.json Format

```json
{
  "schema": "arke/files@v1",
  "files": {
    "a092_001_001.jpg": {
      "file_id": "FQQ8FN65GYEY8VENMVB6Y5DQFB",
      "content_cid": ""
    },
    "document.pdf": {
      "file_id": "FABCDEFGHIJKLMNOPQRSTUVWXY",
      "content_cid": ""
    }
  }
}
```

## File Processing Flow

1. **Upload**: Files are uploaded to R2 staging
2. **Image Processing**: TIFF files converted to JPEG, thumbnails generated, OCR performed
3. **Binary Finalization**: Non-image files are archived
4. **Entity Creation**: File entities created with inline properties
5. **Manifest Update**: Parent PI's `files.json` component updated

## CDN URLs

After processing, files are available via the CDN:

```
https://cdn.arke.institute/asset/{ASSET_ID}
https://cdn.arke.institute/asset/{ASSET_ID}/thumb
```

## Archive Storage

Original files are archived in R2:

```
archive/{BATCH_ID}/{ASSET_ID}/original.{ext}
```

## Best Practices

1. **Always use `source_pi`**: Links file to its owning PI for provenance
2. **Include all properties**: Metadata should be complete at creation time
3. **Use deterministic IDs**: Enables deduplication and predictable lookups
4. **Store properties inline**: Not as a separate CID component
