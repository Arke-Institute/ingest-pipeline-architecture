# Chunking Implementation Plan

## Overview

This document specifies the implementation of text file chunking in the Arke ingest pipeline. Chunking enables:

1. **Knowledge Graph Precision** - Linking service can create relationships to specific chunks
2. **RAG Retrieval** - Semantic search returns relevant sections, not entire documents
3. **Aligned IDs** - Chunk identifiers are consistent across linking-service and index-sync

---

## Part 1: Chunk Addressing Scheme

### Virtual Chunk Addresses

Chunks are addressed using a URL-like scheme:

```
{pi}:{filename}              # Whole file reference
{pi}:{filename}#chunk_0      # First chunk
{pi}:{filename}#chunk_N      # Nth chunk
```

**Examples:**
```
01K75HQQXNTDG7BBP7PS9AWYAN:document.txt
01K75HQQXNTDG7BBP7PS9AWYAN:document.txt#chunk_0
01K75HQQXNTDG7BBP7PS9AWYAN:document.txt#chunk_5
```

### Why This Scheme

- **Stable** - Chunk IDs don't change unless the file is re-chunked
- **Hierarchical** - Clear relationship: PI → file → chunk
- **URL-like** - Familiar pattern (origin:path#fragment)
- **No PI explosion** - Chunks are not separate entities
- **Consistent** - Same IDs used by linking-service and index-sync

---

## Part 2: chunks.json Schema

Each PI can have an optional `chunks.json` component that describes how text files are chunked.

### Schema Definition

```typescript
interface ChunksManifest {
  /**
   * Schema version for future compatibility
   */
  version: 1;

  /**
   * Chunking configuration used to generate these chunks
   */
  config: ChunkingConfig;

  /**
   * Map of filename to file chunk metadata
   */
  files: {
    [filename: string]: FileChunks;
  };
}

interface ChunkingConfig {
  /**
   * Algorithm used for chunking
   * "adaptive" = chunk size scales with file size
   * "fixed" = constant chunk size
   */
  algorithm: 'adaptive' | 'fixed';

  /**
   * Target number of chunks per file (for adaptive)
   * Actual may vary based on min/max constraints
   */
  target_chunks?: number;

  /**
   * Minimum chunk size in characters
   */
  min_chunk_size: number;

  /**
   * Maximum chunk size in characters
   */
  max_chunk_size: number;

  /**
   * Overlap between chunks in characters
   * Ensures context continuity at boundaries
   */
  overlap: number;
}

interface FileChunks {
  /**
   * Original file CID (for reference)
   */
  original_cid: string;

  /**
   * Total character count of original file
   */
  total_chars: number;

  /**
   * Array of chunks (empty if file wasn't chunked)
   */
  chunks: Chunk[];
}

interface Chunk {
  /**
   * Chunk identifier (e.g., "chunk_0", "chunk_1")
   * Used in the full address: {pi}:{filename}#{id}
   */
  id: string;

  /**
   * CID of the chunk content (stored separately in IPFS)
   */
  cid: string;

  /**
   * Start position in original file (0-indexed)
   */
  char_start: number;

  /**
   * End position in original file (exclusive)
   */
  char_end: number;

  /**
   * Character count of this chunk (including overlap)
   */
  char_count: number;
}
```

### Example chunks.json

```json
{
  "version": 1,
  "config": {
    "algorithm": "adaptive",
    "target_chunks": 50,
    "min_chunk_size": 1000,
    "max_chunk_size": 10000,
    "overlap": 200
  },
  "files": {
    "report.txt": {
      "original_cid": "bafkreihdwdcefgh4dqkjv67uzcmw7o...",
      "total_chars": 45000,
      "chunks": [
        {
          "id": "chunk_0",
          "cid": "bafkreiabc123...",
          "char_start": 0,
          "char_end": 1000,
          "char_count": 1000
        },
        {
          "id": "chunk_1",
          "cid": "bafkreidef456...",
          "char_start": 800,
          "char_end": 1800,
          "char_count": 1000
        }
      ]
    },
    "notes.md": {
      "original_cid": "bafkreixyz789...",
      "total_chars": 500,
      "chunks": []
    }
  }
}
```

**Note:** `notes.md` has no chunks because it's below the minimum threshold (500 < 1000 chars).

---

## Part 3: Adaptive Chunking Algorithm

### Design Goals

1. **Prevent chunk explosion** - Large files shouldn't create thousands of chunks
2. **Maintain utility** - Chunks should be meaningful for retrieval
3. **Preserve context** - Overlap ensures no information is lost at boundaries
4. **Skip small files** - Files below threshold stay as single units

### Chunk Size Calculation

```typescript
function calculateChunkSize(fileSize: number, config: ChunkingConfig): number {
  const { min_chunk_size, max_chunk_size, target_chunks } = config;

  // Files smaller than minimum stay as single chunk
  if (fileSize <= min_chunk_size) {
    return fileSize;
  }

  // Calculate ideal chunk size to hit target count
  const idealChunkSize = Math.ceil(fileSize / target_chunks);

  // Clamp to min/max bounds
  return Math.max(min_chunk_size, Math.min(max_chunk_size, idealChunkSize));
}
```

### Default Configuration

```typescript
const DEFAULT_CHUNKING_CONFIG: ChunkingConfig = {
  algorithm: 'adaptive',
  target_chunks: 50,          // Aim for ~50 chunks per file
  min_chunk_size: 1000,       // 1k chars minimum (~200 words)
  max_chunk_size: 10000,      // 10k chars maximum (~2000 words)
  overlap: 200,               // 200 chars overlap (~40 words)
};

// Separators in order of priority (paragraph → line → sentence → word → char)
const SEPARATORS = [
  '\n\n',     // Paragraph break (highest priority)
  '\n',       // Line break
  '. ',       // Sentence end
  ', ',       // Clause break
  ' ',        // Word boundary
  '',         // Character-level (last resort)
];
```

### Chunk Count Examples

| File Size | Chunk Size | Chunks | Notes |
|-----------|------------|--------|-------|
| 500 chars | N/A | 0 | Below minimum, no chunking |
| 1,000 chars | 1,000 | 1 | Single chunk |
| 10,000 chars | 1,000 | ~12 | Min chunk size (10k/50=200, clamped to 1k) |
| 50,000 chars | 1,000 | ~62 | Min chunk size |
| 100,000 chars | 2,000 | ~62 | Scaled: 100k/50 = 2k |
| 250,000 chars | 5,000 | ~62 | Scaled: 250k/50 = 5k |
| 500,000 chars | 10,000 | ~62 | Max chunk size |
| 1,000,000 chars | 10,000 | ~124 | Max chunk size, more chunks |

**Key insight:** The algorithm naturally caps chunk count. Most files will have 50-125 chunks maximum.

### Recursive Separator-Based Chunking

The chunking algorithm uses a recursive approach that prioritizes natural text boundaries:

1. **Paragraph breaks** (`\n\n`) - Highest priority, preserves document structure
2. **Line breaks** (`\n`) - Respects formatting
3. **Sentence ends** (`. `) - Maintains semantic completeness
4. **Clause breaks** (`, `) - Keeps related ideas together
5. **Word boundaries** (` `) - Avoids mid-word splits
6. **Character-level** (`""`) - Absolute last resort

```typescript
interface ChunkResult {
  text: string;
  char_start: number;
  char_end: number;
}

/**
 * Recursively chunk text using natural boundaries.
 * Prioritizes paragraph breaks, then lines, then sentences, etc.
 * Character-level splitting is only used as a last resort.
 */
function chunkText(
  text: string,
  chunkSize: number,
  overlap: number,
  separators: string[] = SEPARATORS
): ChunkResult[] {
  // If text fits in one chunk, return as-is
  if (text.length <= chunkSize) {
    return [{
      text,
      char_start: 0,
      char_end: text.length,
    }];
  }

  // Try each separator in priority order
  for (let i = 0; i < separators.length; i++) {
    const separator = separators[i];
    const chunks = splitBySeparator(text, chunkSize, overlap, separator, separators.slice(i + 1));

    if (chunks.length > 0) {
      return chunks;
    }
  }

  // Fallback: character-level split (should rarely happen)
  return characterLevelSplit(text, chunkSize, overlap);
}

/**
 * Split text by a specific separator, recursing to finer separators if needed.
 */
function splitBySeparator(
  text: string,
  chunkSize: number,
  overlap: number,
  separator: string,
  remainingSeparators: string[]
): ChunkResult[] {
  // Empty separator = character-level split
  if (separator === '') {
    return characterLevelSplit(text, chunkSize, overlap);
  }

  const parts = text.split(separator);

  // If no splits occurred, try next separator
  if (parts.length === 1) {
    return [];
  }

  const chunks: ChunkResult[] = [];
  let currentChunk = '';
  let currentStart = 0;
  let position = 0;

  for (let i = 0; i < parts.length; i++) {
    const part = parts[i];
    const isLast = i === parts.length - 1;
    const partWithSep = isLast ? part : part + separator;

    // Check if adding this part would exceed chunk size
    const candidate = currentChunk + partWithSep;

    if (candidate.length <= chunkSize) {
      // Part fits, add to current chunk
      currentChunk = candidate;
    } else if (currentChunk.length === 0) {
      // Part alone is too big, recurse with finer separators
      if (part.length > chunkSize && remainingSeparators.length > 0) {
        const subChunks = chunkText(part, chunkSize, overlap, remainingSeparators);
        for (const sub of subChunks) {
          chunks.push({
            text: sub.text,
            char_start: position + sub.char_start,
            char_end: position + sub.char_end,
          });
        }
      } else {
        // Can't split further, take what we can
        chunks.push({
          text: partWithSep.slice(0, chunkSize),
          char_start: position,
          char_end: position + Math.min(partWithSep.length, chunkSize),
        });
      }
      position += partWithSep.length;
      currentStart = position;
    } else {
      // Current chunk is full, save it and start new one
      chunks.push({
        text: currentChunk,
        char_start: currentStart,
        char_end: currentStart + currentChunk.length,
      });

      // Apply overlap: include end of previous chunk in new chunk
      const overlapText = getOverlapText(currentChunk, overlap, separator);
      currentStart = currentStart + currentChunk.length - overlapText.length;
      currentChunk = overlapText + partWithSep;
    }

    if (currentChunk.length === 0) {
      position += partWithSep.length;
      currentStart = position;
    }
  }

  // Don't forget the last chunk
  if (currentChunk.length > 0) {
    chunks.push({
      text: currentChunk,
      char_start: currentStart,
      char_end: currentStart + currentChunk.length,
    });
  }

  return chunks;
}

/**
 * Get overlap text from end of chunk, preferring natural boundaries.
 */
function getOverlapText(chunk: string, overlap: number, currentSeparator: string): string {
  if (overlap <= 0 || chunk.length <= overlap) {
    return '';
  }

  // Take last `overlap` characters
  let overlapText = chunk.slice(-overlap);

  // Try to start at a natural boundary (space or separator)
  const spaceIndex = overlapText.indexOf(' ');
  if (spaceIndex > 0 && spaceIndex < overlap / 2) {
    overlapText = overlapText.slice(spaceIndex + 1);
  }

  return overlapText;
}

/**
 * Last resort: split at character boundaries.
 */
function characterLevelSplit(
  text: string,
  chunkSize: number,
  overlap: number
): ChunkResult[] {
  const chunks: ChunkResult[] = [];
  let start = 0;

  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length);

    chunks.push({
      text: text.slice(start, end),
      char_start: start,
      char_end: end,
    });

    // Move forward, accounting for overlap
    const step = chunkSize - overlap;
    if (step <= 0) {
      // Prevent infinite loop if overlap >= chunkSize
      start = end;
    } else {
      start += step;
    }

    // Don't create tiny trailing chunks
    if (text.length - start < overlap) {
      break;
    }
  }

  return chunks;
}
```

### Algorithm Behavior Examples

**Example 1: Well-structured document**
```
Input: "Paragraph one here.\n\nParagraph two is longer and has more content.\n\nParagraph three."
Chunk size: 50

Result: Splits at \n\n boundaries
- Chunk 0: "Paragraph one here."
- Chunk 1: "Paragraph two is longer and has more content."
- Chunk 2: "Paragraph three."
```

**Example 2: Long paragraph**
```
Input: "This is a very long sentence that goes on and on. And another sentence. More text here."
Chunk size: 40

Result: Falls back to ". " separator
- Chunk 0: "This is a very long sentence that goes on and on."
- Chunk 1: "And another sentence. More text here."
```

**Example 3: No natural boundaries**
```
Input: "abcdefghijklmnopqrstuvwxyz0123456789"
Chunk size: 10, Overlap: 2

Result: Character-level split (last resort)
- Chunk 0: "abcdefghij" (0-10)
- Chunk 1: "ijklmnopqr" (8-18, with 2-char overlap)
- Chunk 2: "qrstuvwxyz" (16-26)
- etc.
```

---

## Part 4: Implementation in Ingest Worker

### Modified File Processing Flow

```
1. Client uploads file to R2
2. Initial Discovery reads file from R2
3. If text file:
   a. Check file size against min_chunk_size
   b. If >= min_chunk_size: chunk the file
   c. Upload original file CID (as before)
   d. Upload each chunk as separate CID
   e. Track chunk metadata
4. After all files processed:
   a. Build chunks.json manifest
   b. Upload chunks.json to IPFS
   c. Add chunks.json to entity components
5. Create entity with all components
```

### Changes to initial-discovery.ts

#### New Types

```typescript
// Add to types.ts

interface DiscoveryChunk {
  id: string;           // "chunk_0", "chunk_1", etc.
  text: string;         // Chunk content
  char_start: number;
  char_end: number;
  cid?: string;         // Set after upload
}

interface DiscoveryTextFile {
  filename: string;
  r2_key: string;
  cid?: string;                    // Original file CID
  total_chars?: number;            // Set after reading
  chunks?: DiscoveryChunk[];       // Chunk data (if chunked)
  chunks_uploaded?: boolean;       // True when all chunks uploaded
}
```

#### New Constants

```typescript
// Chunking configuration
const CHUNKING_CONFIG = {
  algorithm: 'adaptive' as const,
  target_chunks: 50,
  min_chunk_size: 1000,
  max_chunk_size: 10000,
  overlap: 200,
};

// Batch size for chunk uploads (chunks are small, can do more)
const CHUNK_UPLOAD_BATCH_SIZE = 200;
```

#### Modified uploadFileBatch Function

```typescript
export async function uploadFileBatch(
  state: DiscoveryState,
  env: Env,
  batchSize: number = UPLOAD_BATCH_SIZE
): Promise<boolean> {
  const ipfsClient = new IPFSWrapperClient(env.ARKE_IPFS_API);

  // Phase 1: Upload original files (and prepare chunks)
  const pendingFiles = getFilesNeedingUpload(state).slice(0, batchSize);

  if (pendingFiles.length === 0) {
    // Check if we need to upload chunks
    const pendingChunks = getChunksNeedingUpload(state);
    if (pendingChunks.length > 0) {
      return uploadChunkBatch(state, env, CHUNK_UPLOAD_BATCH_SIZE);
    }

    // All files and chunks uploaded
    state.phase = 'PUBLISHING';
    return true;
  }

  // Upload files and prepare chunks
  await Promise.all(
    pendingFiles.map(async ({ node, file }) => {
      try {
        const obj = await env.STAGING_BUCKET.get(file.r2_key);
        if (obj) {
          const content = await obj.text();

          // Upload original file
          const cid = await ipfsClient.uploadContent(content, file.filename);
          file.cid = cid;
          file.total_chars = content.length;

          // Prepare chunks if file is large enough
          if (content.length >= CHUNKING_CONFIG.min_chunk_size) {
            const chunkSize = calculateChunkSize(content.length, CHUNKING_CONFIG);
            const chunkData = chunkText(content, chunkSize, CHUNKING_CONFIG.overlap);

            file.chunks = chunkData.map((chunk, index) => ({
              id: `chunk_${index}`,
              text: chunk.text,
              char_start: chunk.start,
              char_end: chunk.end,
              // cid will be set during chunk upload phase
            }));
          } else {
            file.chunks = []; // Mark as processed, no chunks needed
          }

          state.files_uploaded++;
        }
      } catch (error) {
        console.error(`[Discovery] Failed to upload ${file.filename}:`, error);
        file.cid = '';
        file.chunks = [];
        state.files_uploaded++;
      }
    })
  );

  return true;
}

async function uploadChunkBatch(
  state: DiscoveryState,
  env: Env,
  batchSize: number
): Promise<boolean> {
  const ipfsClient = new IPFSWrapperClient(env.ARKE_IPFS_API);

  const pendingChunks = getChunksNeedingUpload(state).slice(0, batchSize);

  if (pendingChunks.length === 0) {
    state.phase = 'PUBLISHING';
    return true;
  }

  await Promise.all(
    pendingChunks.map(async ({ file, chunk }) => {
      try {
        const cid = await ipfsClient.uploadContent(
          chunk.text,
          `${file.filename}#${chunk.id}`
        );
        chunk.cid = cid;
      } catch (error) {
        console.error(`[Discovery] Failed to upload chunk ${chunk.id}:`, error);
        chunk.cid = ''; // Mark as failed
      }
    })
  );

  return true;
}
```

#### Modified publishDirectory Function

```typescript
async function publishDirectory(
  node: DiscoveryNode,
  state: DiscoveryState,
  ipfsClient: IPFSWrapperClient
): Promise<void> {
  // Build components from uploaded files
  const components: Record<string, string> = {};

  for (const file of node.text_files) {
    if (file.cid && file.cid.length > 0) {
      components[file.filename] = file.cid;
    }
  }

  // Build chunks.json if any files were chunked
  const chunkedFiles = node.text_files.filter(
    f => f.chunks && f.chunks.length > 0
  );

  if (chunkedFiles.length > 0) {
    const chunksManifest = buildChunksManifest(chunkedFiles);
    const chunksJson = JSON.stringify(chunksManifest, null, 2);
    const chunksCid = await ipfsClient.uploadContent(chunksJson, 'chunks.json');
    components['chunks.json'] = chunksCid;
  }

  // Get child PIs and create entity (same as before)
  const childPis: string[] = node.children_paths
    .map((path) => state.node_pis[path])
    .filter((pi): pi is string => !!pi);

  const result = await ipfsClient.createEntity({
    type: 'PI',
    components,
    children_pi: childPis,
    note: 'Initial discovery snapshot',
  });

  // Update tracking (same as before)
  node.pi = result.id;
  node.tip = result.tip;
  node.version = result.ver;
  node.published = true;

  state.node_pis[node.path] = result.id;
  state.node_tips[node.path] = result.tip;
  state.node_versions[node.path] = result.ver;
  state.directories_published++;
}

function buildChunksManifest(files: DiscoveryTextFile[]): ChunksManifest {
  const manifest: ChunksManifest = {
    version: 1,
    config: CHUNKING_CONFIG,
    files: {},
  };

  for (const file of files) {
    manifest.files[file.filename] = {
      original_cid: file.cid!,
      total_chars: file.total_chars!,
      chunks: (file.chunks || [])
        .filter(c => c.cid && c.cid.length > 0)
        .map(c => ({
          id: c.id,
          cid: c.cid!,
          char_start: c.char_start,
          char_end: c.char_end,
          char_count: c.text.length,
        })),
    };
  }

  return manifest;
}
```

---

## Part 5: Discovery State Changes

### Updated DiscoveryState

```typescript
interface DiscoveryState {
  nodes: Record<string, DiscoveryNode>;
  directories_total: number;
  directories_published: number;
  node_pis: Record<string, string>;
  node_tips: Record<string, string>;
  node_versions: Record<string, number>;
  phase: DiscoveryPhase;
  current_depth: number;

  // File tracking
  files_total: number;
  files_uploaded: number;

  // NEW: Chunk tracking
  chunks_total: number;
  chunks_uploaded: number;

  error?: string;
  retry_count?: number;
}

// Updated phase to include CHUNKING
type DiscoveryPhase =
  | 'UPLOADING'      // Upload original files, prepare chunks
  | 'CHUNKING'       // Upload chunk CIDs (NEW)
  | 'PUBLISHING'     // Create entities with components
  | 'RELATIONSHIPS'  // Set parent-child relationships
  | 'DONE'
  | 'ERROR';
```

### Updated buildDiscoveryTree

Add chunk counting:

```typescript
export function buildDiscoveryTree(manifest: BatchManifest): DiscoveryState {
  // ... existing code ...

  return {
    // ... existing fields ...
    chunks_total: 0,      // Updated during UPLOADING phase
    chunks_uploaded: 0,   // Updated during CHUNKING phase
  };
}
```

---

## Part 6: Retrieval Endpoint

### New Endpoint: GET /chunk/:address

Add to IPFS Wrapper API:

```typescript
// GET /chunk/01K75HQQ...:document.txt#chunk_3
app.get('/chunk/:address', async (c) => {
  const address = c.req.param('address');

  // Parse address: {pi}:{filename}#{chunk_id}
  const parsed = parseChunkAddress(address);
  if (!parsed) {
    return c.json({ error: 'Invalid chunk address' }, 400);
  }

  const { pi, filename, chunkId } = parsed;

  // Get entity manifest
  const entity = await getEntity(pi);
  if (!entity) {
    return c.json({ error: 'Entity not found' }, 404);
  }

  // If no chunk requested, return whole file
  if (!chunkId) {
    const fileCid = entity.components[filename];
    if (!fileCid) {
      return c.json({ error: 'File not found' }, 404);
    }
    const content = await fetchCid(fileCid);
    return c.text(content);
  }

  // Get chunks.json
  const chunksJsonCid = entity.components['chunks.json'];
  if (!chunksJsonCid) {
    // Fallback: return whole file if no chunks.json
    const fileCid = entity.components[filename];
    if (!fileCid) {
      return c.json({ error: 'File not found' }, 404);
    }
    const content = await fetchCid(fileCid);
    return c.text(content);
  }

  // Find chunk in manifest
  const chunksManifest = await fetchJson(chunksJsonCid);
  const fileChunks = chunksManifest.files[filename];
  if (!fileChunks) {
    return c.json({ error: 'File not in chunks manifest' }, 404);
  }

  const chunk = fileChunks.chunks.find(c => c.id === chunkId);
  if (!chunk) {
    // Fallback: return whole file if chunk not found
    const content = await fetchCid(fileChunks.original_cid);
    return c.text(content);
  }

  // Return chunk content
  const content = await fetchCid(chunk.cid);
  return c.text(content);
});

function parseChunkAddress(address: string): {
  pi: string;
  filename: string;
  chunkId?: string;
} | null {
  // Format: {pi}:{filename} or {pi}:{filename}#{chunk_id}
  const colonIndex = address.indexOf(':');
  if (colonIndex === -1) return null;

  const pi = address.slice(0, colonIndex);
  const rest = address.slice(colonIndex + 1);

  const hashIndex = rest.indexOf('#');
  if (hashIndex === -1) {
    return { pi, filename: rest };
  }

  return {
    pi,
    filename: rest.slice(0, hashIndex),
    chunkId: rest.slice(hashIndex + 1),
  };
}
```

---

## Part 7: Testing Strategy

### Unit Tests

1. **Chunk size calculation**
   - Verify min/max bounds
   - Verify adaptive scaling
   - Verify target chunk count

2. **Text chunking**
   - Verify overlap handling
   - Verify edge cases (empty, single char, exact boundary)
   - Verify chunk count matches expectations

3. **chunks.json generation**
   - Verify schema compliance
   - Verify all chunks have CIDs
   - Verify char positions are accurate

### Integration Tests

1. **Discovery with chunking**
   - Upload batch with mixed file sizes
   - Verify chunks.json created when needed
   - Verify chunks.json not created for small files only

2. **Chunk retrieval**
   - Retrieve specific chunk by address
   - Verify fallback to full file
   - Verify 404 for missing chunks

---

## Part 8: Migration Notes

### No Migration Required

This is a new capability. Existing PIs without chunks.json will continue to work:
- Linking service can still reference files without chunks
- Index-sync can still index files without chunks.json
- Retrieval endpoint falls back to full file

### Backwards Compatibility

- New chunks.json component is optional
- Downstream services should handle missing chunks.json gracefully
- File references without #chunk_N work as before

---

## Part 9: Future Enhancements

### Semantic Chunking (Phase 2)

Instead of fixed character boundaries, chunk at semantic boundaries:
- Paragraph breaks
- Section headers
- Sentence boundaries

Would require NLP processing, could be a post-discovery enhancement.

### Chunk Labels (Phase 2)

Add human-readable labels based on content:
```json
{
  "id": "chunk_3",
  "label": "Background and History",
  "cid": "...",
  ...
}
```

### Re-chunking Support (Phase 2)

Track chunk generation version to support re-chunking:
```json
{
  "version": 1,
  "generation": 2,  // Incremented when re-chunked
  ...
}
```

---

## Summary

| Component | Change | Priority |
|-----------|--------|----------|
| Ingest Worker | Add chunking during discovery | P0 |
| Types | Add chunk-related types | P0 |
| IPFS Wrapper | Add /chunk endpoint | P1 |
| Linking Service | Update to reference chunks | P2 |
| Index-Sync | Update to use chunk IDs | P2 |

**Next Steps:**
1. Implement chunking in initial-discovery.ts
2. Add chunks.json schema validation
3. Test with various file sizes
4. Add /chunk retrieval endpoint
5. Update linking service instructions
6. Update index-sync vector IDs
