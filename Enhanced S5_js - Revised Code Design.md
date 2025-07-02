# Enhanced S5.js - Revised Code Design

by Jules Lai, 27th June 2025

## Overview

This enhancement extends S5.js with modern directory handling capabilities aligned with the S5 v1 specification and Rust implementation, incorporating DAG-CBOR serialisation and HAMT sharding for efficient management of large-scale file systems.

### Key Features

**Developer-Friendly Path-Based APIs**

- `get(path)` - Direct path-based retrieval of any data
- `put(path, data, options)` - Store data at any path with automatic directory creation and optional metadata
- `getMetadata(path)` - Retrieve metadata for any file or directory
- `list(path, options)` - Efficient paginated directory listing with cursor support and async iteration

**DirV1 Format with DAG-CBOR**

- Implements S5 v1 directory format matching Rust implementation
- DAG-CBOR serialisation for compact binary representation
- Deterministic encoding for consistent content addressing
- No legacy support - clean S5 v1 implementation

**HAMT Sharding Integration**

- Hash Array Mapped Trie for O(log n) access to directories with millions of entries
- Automatic node splitting for large directories (>1000 entries by default)
- Lazy loading - only fetch required nodes, not entire directory structures
- Configurable sharding parameters in directory header

**Performance Optimisations**

- In-memory caching of frequently accessed nodes
- Efficient batch operations for bulk updates
- Streaming support for large data sets
- Minimal overhead for small directories
- Cursor-based pagination for stateless directory traversal

### Benefits

- **Scalability**: Handle directories with 10M+ entries efficiently
- **Simplicity**: Path-based operations instead of complex manifest manipulation
- **Performance**: Sub-millisecond lookups even in massive directories
- **Compatibility**: Full compatibility with Rust S5 v1 implementation
- **Resumability**: Cursor-based pagination enables resumable operations

## Key Concepts

### DirV1 Format

The directory format matches the Rust implementation:

```typescript
// DirV1 structure matching Rust
interface DirV1 {
  magic: string; // "S5.pro"
  header: DirHeader;
  dirs: Map<string, DirRef>;
  files: Map<string, FileRef>;
}

interface FileRef {
  hash: Uint8Array; // 32-byte hash
  size: number; // u64 file size
  media_type?: string; // Optional MIME type
  timestamp?: number; // Optional timestamp seconds
  timestamp_subsec_nanos?: number; // Optional subsecond precision
  locations?: BlobLocation[]; // Optional locations
  hash_type?: number; // Optional hash type
  extra?: Map<string, any>; // Optional extra metadata
  prev?: FileRef; // Optional previous version
}
```

### DAG-CBOR Serialisation

DAG-CBOR provides:

- Binary efficiency (30-50% smaller than JSON)
- Native support for binary data (Uint8Array)
- Deterministic encoding for consistent hashes
- Direct CID references without base encoding
- Integer key mapping during serialisation (matching Rust's `#[n(X)]` attributes)

### HAMT Configuration

HAMT parameters are stored in the directory header:

```typescript
interface DirHeader {
  // Empty base header matching Rust
  sharding?: {
    type: "hamt";
    config: {
      bitsPerLevel: number; // Default: 5 (32-way branching)
      maxInlineEntries: number; // Default: 1000
      hashFunction: 0 | 1; // 0=xxhash, 1=blake3
    };
    root?: {
      cid: Uint8Array; // Root HAMT node CID
      totalEntries: number;
      depth: number;
    };
  };
}
```

## Phase 1: Core Infrastructure

### 1.1 Add CBOR Dependencies

Update `package.json`:

```json
{
  "dependencies": {
    "cbor-x": "^1.5.0",
    "xxhash-wasm": "^1.0.0",
    "@noble/hashes": "^1.3.0"
  }
}
```

### 1.2 Create DirV1 Types Matching Rust

New file `src/fs/dirv1/types.ts`:

```typescript
// Match Rust's DirV1 structure
export interface DirV1 {
  magic: string; // "S5.pro"
  header: DirHeader;
  dirs: Map<string, DirRef>;
  files: Map<string, FileRef>;
}

// Match Rust's DirHeader
export interface DirHeader {
  // Empty for now, matching Rust implementation
  // Sharding fields will be added as extensions
  sharding?: HAMTShardingConfig;
}

// Match Rust's DirRef
export interface DirRef {
  link: DirLink;
  ts_seconds?: number; // Optional timestamp seconds
  ts_nanos?: number; // Optional timestamp nanoseconds
  extra?: any; // Optional extra metadata
}

// Match Rust's FileRef
export interface FileRef {
  hash: Uint8Array; // 32-byte hash
  size: number; // u64 file size
  media_type?: string; // Optional MIME type
  timestamp?: number; // Optional timestamp seconds
  timestamp_subsec_nanos?: number; // Optional subsecond precision
  locations?: BlobLocation[]; // Optional locations
  hash_type?: number; // Optional hash type
  extra?: Map<string, any>; // Optional extra metadata
  prev?: FileRef; // Optional previous version
}

// Match Rust's BlobLocation enum
export type BlobLocation =
  | { type: "identity"; data: Uint8Array }
  | { type: "http"; url: string }
  | { type: "multihash_sha1"; hash: Uint8Array } // 20 bytes
  | { type: "multihash_sha2_256"; hash: Uint8Array } // 32 bytes
  | { type: "multihash_blake3"; hash: Uint8Array } // 32 bytes
  | { type: "multihash_md5"; hash: Uint8Array }; // 16 bytes

// Match Rust's DirLink enum
export type DirLink =
  | { type: "fixed_hash_blake3"; hash: Uint8Array } // 32 bytes
  | { type: "mutable_registry_ed25519"; publicKey: Uint8Array }; // 32 bytes

// Add HAMT sharding extension to header
export interface HAMTShardingConfig {
  type: "hamt";
  config: {
    bitsPerLevel: number;
    maxInlineEntries: number;
    hashFunction: number;
  };
  root?: {
    cid: Uint8Array;
    totalEntries: number;
    depth: number;
  };
}

// Put options with metadata support
export interface PutOptions {
  mediaType?: string;
  metadata?: Record<string, any>;
  locations?: BlobLocation[];
  encryption?: {
    algorithm: "xchacha20-poly1305";
    key?: Uint8Array; // If not provided, will generate
  };
}

// List options with cursor support
export interface ListOptions {
  limit?: number;
  cursor?: string; // Opaque cursor for pagination
  filter?: (item: ListResult) => boolean; // Optional client-side filter
}

// List result with cursor
export interface ListResult {
  path: string;
  name: string;
  type: "file" | "directory";
  metadata: any;
  cursor: string; // Cursor to resume from this position
}

// Cursor data structure
interface CursorData {
  type: "file" | "directory";
  name: string;
  hamtPath?: number[]; // Path through HAMT nodes
  position: number; // Item count for validation
}
```

### 1.3 Create CBOR Configuration

New file `src/fs/dirv1/cbor-config.ts`:

```typescript
import { Encoder, Decoder, addExtension } from "cbor-x";

// Create configured encoder with S5-required settings
export const s5Encoder = new Encoder({
  mapsAsObjects: false, // Use Maps, not objects
  sequential: true, // CRITICAL: Deterministic encoding for consistent hashes
  structuredClone: true, // Handle complex objects properly
  bundleStrings: false, // Don't bundle strings (affects determinism)
  variableMapSize: true, // Allow variable-sized maps
  useRecords: false, // Don't use records (affects compatibility)
  tagUint8Array: false, // Don't tag Uint8Arrays
});

// Create configured decoder
export const s5Decoder = new Decoder({
  mapsAsObjects: false, // Decode as Maps
  variableMapSize: true, // Handle variable-sized maps
  useRecords: false, // Match encoder settings
  tagUint8Array: false, // Match encoder settings
});

// Helper functions for consistent encoding/decoding
export function encodeS5(data: any): Uint8Array {
  return s5Encoder.encode(data);
}

export function decodeS5(data: Uint8Array): any {
  return s5Decoder.decode(data);
}

// Helper to create ordered maps from objects
export function createOrderedMap<V>(obj: Record<string, V>): Map<string, V> {
  const entries = Object.entries(obj).sort(([a], [b]) => a.localeCompare(b));
  return new Map(entries);
}
```

### 1.4 Implement CBOR Serialisation Matching Rust

New file `src/fs/dirv1/serialisation.ts`:

```typescript
import { encodeS5, decodeS5 } from "./cbor-config";

// CBOR integer key mappings matching Rust's #[n(X)] attributes
const DIRREF_KEYS = {
  link: 2,
  ts_seconds: 7,
  ts_nanos: 8,
  extra: 0x16,
} as const;

const FILEREF_KEYS = {
  hash: 3,
  size: 4,
  media_type: 6,
  timestamp: 7,
  timestamp_subsec_nanos: 8,
  locations: 9,
  hash_type: 0x13,
  extra: 0x16,
  prev: 0x17,
} as const;

const BLOB_LOCATION_TAGS = {
  identity: 0,
  http: 1,
  multihash_sha1: 0x11,
  multihash_sha2_256: 0x12,
  multihash_blake3: 0x1e,
  multihash_md5: 0xd5,
} as const;

export class DirV1Serialiser {
  private static readonly MAGIC_BYTE = 0x5f;
  private static readonly DIR_V1_TYPE = 0x5d; // Standard FS5 directory type

  static serialise(dir: DirV1): Uint8Array {
    // Convert to CBOR structure matching Rust
    const cborData = [
      dir.magic,
      this.serialiseHeader(dir.header),
      this.serialiseDirRefs(dir.dirs),
      this.serialiseFileRefs(dir.files),
    ];

    // Use configured encoder for deterministic encoding
    const encoded = encodeS5(cborData);

    // Add magic bytes
    return new Uint8Array([this.MAGIC_BYTE, this.DIR_V1_TYPE, ...encoded]);
  }

  static deserialise(data: Uint8Array): DirV1 {
    // Check magic bytes
    if (data[0] !== this.MAGIC_BYTE || data[1] !== this.DIR_V1_TYPE) {
      throw new Error("Invalid DirV1 format");
    }

    // Use configured decoder
    const decoded = decodeS5(data.subarray(2));

    if (!Array.isArray(decoded) || decoded.length !== 4) {
      throw new Error("Invalid DirV1 structure");
    }

    return {
      magic: decoded[0],
      header: this.deserialiseHeader(decoded[1]),
      dirs: this.deserialiseDirRefs(decoded[2]),
      files: this.deserialiseFileRefs(decoded[3]),
    };
  }

  private static serialiseHeader(header: DirHeader): any {
    const result: any = {};

    // Add sharding config if present
    if (header.sharding) {
      result.sharding = header.sharding;
    }

    return result;
  }

  private static deserialiseHeader(data: any): DirHeader {
    const header: DirHeader = {};

    if (data.sharding) {
      header.sharding = data.sharding;
    }

    return header;
  }

  private static serialiseDirRefs(dirs: Map<string, DirRef>): any {
    const result: Record<string, any> = {};

    for (const [name, dirRef] of dirs) {
      const encoded: Record<number, any> = {
        [DIRREF_KEYS.link]: this.serialiseDirLink(dirRef.link),
      };

      if (dirRef.ts_seconds !== undefined) {
        encoded[DIRREF_KEYS.ts_seconds] = dirRef.ts_seconds;
      }
      if (dirRef.ts_nanos !== undefined) {
        encoded[DIRREF_KEYS.ts_nanos] = dirRef.ts_nanos;
      }
      if (dirRef.extra !== undefined) {
        encoded[DIRREF_KEYS.extra] = dirRef.extra;
      }

      result[name] = encoded;
    }

    return result;
  }

  private static serialiseFileRefs(files: Map<string, FileRef>): any {
    const result: Record<string, any> = {};

    for (const [name, fileRef] of files) {
      const encoded: Record<number, any> = {
        [FILEREF_KEYS.hash]: fileRef.hash,
        [FILEREF_KEYS.size]: fileRef.size,
      };

      if (fileRef.media_type !== undefined) {
        encoded[FILEREF_KEYS.media_type] = fileRef.media_type;
      }
      if (fileRef.timestamp !== undefined) {
        encoded[FILEREF_KEYS.timestamp] = fileRef.timestamp;
      }
      if (fileRef.timestamp_subsec_nanos !== undefined) {
        encoded[FILEREF_KEYS.timestamp_subsec_nanos] =
          fileRef.timestamp_subsec_nanos;
      }
      if (fileRef.locations !== undefined) {
        encoded[FILEREF_KEYS.locations] = fileRef.locations.map((loc) =>
          this.serialiseBlobLocation(loc)
        );
      }
      if (fileRef.hash_type !== undefined) {
        encoded[FILEREF_KEYS.hash_type] = fileRef.hash_type;
      }
      if (fileRef.extra !== undefined) {
        encoded[FILEREF_KEYS.extra] = Object.fromEntries(fileRef.extra);
      }
      if (fileRef.prev !== undefined) {
        encoded[FILEREF_KEYS.prev] = this.serialiseFileRef(fileRef.prev);
      }

      result[name] = encoded;
    }

    return result;
  }

  private static serialiseDirLink(link: DirLink): Uint8Array {
    const bytes = new Uint8Array(33);

    switch (link.type) {
      case "fixed_hash_blake3":
        bytes[0] = 0x1e;
        bytes.set(link.hash, 1);
        break;
      case "mutable_registry_ed25519":
        bytes[0] = 0xed;
        bytes.set(link.publicKey, 1);
        break;
    }

    return bytes;
  }

  private static deserialiseDirLink(bytes: Uint8Array): DirLink {
    if (bytes.length !== 33) {
      throw new Error("Invalid DirLink length");
    }

    switch (bytes[0]) {
      case 0x1e:
        return {
          type: "fixed_hash_blake3",
          hash: bytes.slice(1),
        };
      case 0xed:
        return {
          type: "mutable_registry_ed25519",
          publicKey: bytes.slice(1),
        };
      default:
        throw new Error(`Unknown DirLink tag: ${bytes[0]}`);
    }
  }

  private static serialiseBlobLocation(location: BlobLocation): [number, any] {
    switch (location.type) {
      case "identity":
        return [BLOB_LOCATION_TAGS.identity, location.data];
      case "http":
        return [BLOB_LOCATION_TAGS.http, location.url];
      case "multihash_sha1":
        return [BLOB_LOCATION_TAGS.multihash_sha1, location.hash];
      case "multihash_sha2_256":
        return [BLOB_LOCATION_TAGS.multihash_sha2_256, location.hash];
      case "multihash_blake3":
        return [BLOB_LOCATION_TAGS.multihash_blake3, location.hash];
      case "multihash_md5":
        return [BLOB_LOCATION_TAGS.multihash_md5, location.hash];
    }
  }

  private static deserialiseDirRefs(data: any): Map<string, DirRef> {
    const result = new Map<string, DirRef>();

    for (const [name, encoded] of Object.entries(data)) {
      const dirRef: DirRef = {
        link: this.deserialiseDirLink(encoded[DIRREF_KEYS.link]),
      };

      if (DIRREF_KEYS.ts_seconds in encoded) {
        dirRef.ts_seconds = encoded[DIRREF_KEYS.ts_seconds];
      }
      if (DIRREF_KEYS.ts_nanos in encoded) {
        dirRef.ts_nanos = encoded[DIRREF_KEYS.ts_nanos];
      }
      if (DIRREF_KEYS.extra in encoded) {
        dirRef.extra = encoded[DIRREF_KEYS.extra];
      }

      result.set(name, dirRef);
    }

    return result;
  }

  private static deserialiseFileRefs(data: any): Map<string, FileRef> {
    const result = new Map<string, FileRef>();

    for (const [name, encoded] of Object.entries(data)) {
      const fileRef: FileRef = {
        hash: new Uint8Array(encoded[FILEREF_KEYS.hash]),
        size: encoded[FILEREF_KEYS.size],
      };

      if (FILEREF_KEYS.media_type in encoded) {
        fileRef.media_type = encoded[FILEREF_KEYS.media_type];
      }
      if (FILEREF_KEYS.timestamp in encoded) {
        fileRef.timestamp = encoded[FILEREF_KEYS.timestamp];
      }
      if (FILEREF_KEYS.timestamp_subsec_nanos in encoded) {
        fileRef.timestamp_subsec_nanos =
          encoded[FILEREF_KEYS.timestamp_subsec_nanos];
      }
      if (FILEREF_KEYS.locations in encoded) {
        fileRef.locations = encoded[FILEREF_KEYS.locations].map(
          (loc: [number, any]) => this.deserialiseBlobLocation(loc)
        );
      }
      if (FILEREF_KEYS.hash_type in encoded) {
        fileRef.hash_type = encoded[FILEREF_KEYS.hash_type];
      }
      if (FILEREF_KEYS.extra in encoded) {
        fileRef.extra = new Map(Object.entries(encoded[FILEREF_KEYS.extra]));
      }
      if (FILEREF_KEYS.prev in encoded) {
        fileRef.prev = this.deserialiseFileRef(encoded[FILEREF_KEYS.prev]);
      }

      result.set(name, fileRef);
    }

    return result;
  }

  private static serialiseFileRef(fileRef: FileRef): any {
    const encoded: Record<number, any> = {
      [FILEREF_KEYS.hash]: fileRef.hash,
      [FILEREF_KEYS.size]: fileRef.size,
    };

    if (fileRef.media_type !== undefined) {
      encoded[FILEREF_KEYS.media_type] = fileRef.media_type;
    }
    if (fileRef.timestamp !== undefined) {
      encoded[FILEREF_KEYS.timestamp] = fileRef.timestamp;
    }
    if (fileRef.timestamp_subsec_nanos !== undefined) {
      encoded[FILEREF_KEYS.timestamp_subsec_nanos] =
        fileRef.timestamp_subsec_nanos;
    }
    if (fileRef.extra !== undefined) {
      encoded[FILEREF_KEYS.extra] = Object.fromEntries(fileRef.extra);
    }

    return encoded;
  }

  private static deserialiseFileRef(encoded: any): FileRef {
    const fileRef: FileRef = {
      hash: new Uint8Array(encoded[FILEREF_KEYS.hash]),
      size: encoded[FILEREF_KEYS.size],
    };

    if (FILEREF_KEYS.media_type in encoded) {
      fileRef.media_type = encoded[FILEREF_KEYS.media_type];
    }
    if (FILEREF_KEYS.timestamp in encoded) {
      fileRef.timestamp = encoded[FILEREF_KEYS.timestamp];
    }
    if (FILEREF_KEYS.extra in encoded) {
      fileRef.extra = new Map(Object.entries(encoded[FILEREF_KEYS.extra]));
    }

    return fileRef;
  }

  private static deserialiseBlobLocation(data: [number, any]): BlobLocation {
    const [tag, value] = data;

    switch (tag) {
      case BLOB_LOCATION_TAGS.identity:
        return { type: "identity", data: new Uint8Array(value) };
      case BLOB_LOCATION_TAGS.http:
        return { type: "http", url: value };
      case BLOB_LOCATION_TAGS.multihash_sha1:
        return { type: "multihash_sha1", hash: new Uint8Array(value) };
      case BLOB_LOCATION_TAGS.multihash_sha2_256:
        return { type: "multihash_sha2_256", hash: new Uint8Array(value) };
      case BLOB_LOCATION_TAGS.multihash_blake3:
        return { type: "multihash_blake3", hash: new Uint8Array(value) };
      case BLOB_LOCATION_TAGS.multihash_md5:
        return { type: "multihash_md5", hash: new Uint8Array(value) };
      default:
        throw new Error(`Unknown BlobLocation tag: ${tag}`);
    }
  }
}
```

## Phase 2: Path-Based API Implementation

### 2.1 Extend FS5 Class

Modify `src/fs/fs5.ts`:

```typescript
import {
  DirV1,
  FileRef,
  DirRef,
  PutOptions,
  ListOptions,
  ListResult,
  CursorData,
} from "./dirv1/types";
import { DirV1Serialiser } from "./dirv1/serialisation";
import { encodeS5, decodeS5 } from "./dirv1/cbor-config";
import { HAMT } from "./hamt/hamt";

export class FS5 {
  private nodeCache: Map<string, DirV1> = new Map();

  // Path-based get - retrieves any data
  async get(path: string): Promise<any | undefined> {
    const { directory, fileName } = await this._resolvePath(path);
    if (!fileName) {
      // Return directory metadata if path points to directory
      return this._getDirectoryMetadata(directory);
    }

    const file = await this._getFileFromDirectory(directory, fileName);
    if (!file) return undefined;

    // Load and decode file data
    const data = await this.api.downloadBlobAsBytes(file.hash);
    try {
      // Try CBOR first with configured decoder
      return decodeS5(data);
    } catch {
      try {
        return JSON.parse(new TextDecoder().decode(data)); // Try JSON
      } catch {
        return new TextDecoder().decode(data); // Fallback to text
      }
    }
  }

  // Path-based put - stores any data with optional metadata
  async put(path: string, data: any, options: PutOptions = {}): Promise<void> {
    const segments = path.split("/").filter((s) => s);
    const fileName = segments.pop();
    if (!fileName) throw new Error("Invalid path: must include filename");

    const dirPath = segments.join("/");

    // Serialise data
    let encoded: Uint8Array;
    let mediaType = options.mediaType;

    if (data instanceof Uint8Array) {
      encoded = data;
      mediaType = mediaType || "application/octet-stream";
    } else if (typeof data === "string") {
      encoded = new TextEncoder().encode(data);
      mediaType = mediaType || "text/plain";
    } else {
      // Use deterministic CBOR for objects
      encoded = encodeS5(data);
      mediaType = mediaType || "application/cbor";
    }

    // Handle encryption if requested
    if (options.encryption) {
      const key =
        options.encryption.key || this.api.crypto.generateSecureRandomBytes(32);
      const nonce = this.api.crypto.generateSecureRandomBytes(24);

      encoded = await this.api.crypto.encryptXChaCha20Poly1305(
        key,
        nonce,
        encoded
      );

      // Store encryption info in metadata
      options.metadata = {
        ...options.metadata,
        encryption: {
          algorithm: options.encryption.algorithm,
          nonce: base64UrlNoPaddingEncode(nonce),
          // Note: key management is application's responsibility
        },
      };
    }

    const blob = new Blob([encoded]);
    const blobId = await this.api.uploadBlob(blob);

    // Create file entry with metadata
    const now = Date.now();
    const fileEntry: FileRef = {
      hash: blobId.hash,
      size: encoded.length,
      media_type: mediaType,
      timestamp: Math.floor(now / 1000),
      timestamp_subsec_nanos: (now % 1000) * 1000000,
      locations: options.locations,
      extra: options.metadata
        ? new Map(Object.entries(options.metadata))
        : undefined,
    };

    // Update directory
    await this._updateDirectory(dirPath, (dir) => {
      // Preserve previous version if exists
      const existingFile = dir.files.get(fileName);
      if (existingFile) {
        fileEntry.prev = existingFile;
      }

      dir.files.set(fileName, fileEntry);
      return dir;
    });
  }

  // Get metadata for a file or directory
  async getMetadata(path: string): Promise<Record<string, any> | undefined> {
    const { directory, fileName } = await this._resolvePath(path);

    if (!fileName) {
      // Directory metadata
      return {
        type: "directory",
        name: path.split("/").pop() || "root",
        fileCount: directory.files.size,
        directoryCount: directory.dirs.size,
        sharding: directory.header.sharding,
        created: this._getOldestTimestamp(directory),
        modified: this._getNewestTimestamp(directory),
      };
    }

    const file = await this._getFileFromDirectory(directory, fileName);
    if (!file) return undefined;

    // File metadata
    const metadata: Record<string, any> = {
      type: "file",
      name: fileName,
      size: file.size,
      mediaType: file.media_type,
      timestamp: file.timestamp
        ? new Date(file.timestamp * 1000).toISOString()
        : undefined,
      hashType: file.hash_type,
      locations: file.locations,
      hasHistory: !!file.prev,
    };

    // Include custom metadata from extra field
    if (file.extra) {
      metadata.custom = Object.fromEntries(file.extra);
    }

    return metadata;
  }

  // List with pagination and cursor support
  async *list(
    prefix: string,
    options?: ListOptions
  ): AsyncIterator<ListResult> {
    const { directory, pathInDir } = await this._resolvePath(prefix);

    // Parse cursor if provided
    const cursorData = options?.cursor
      ? this._parseCursor(options.cursor)
      : null;

    // Check if directory uses HAMT sharding
    if (directory.header.sharding?.root) {
      yield* this._listWithHAMT(
        directory,
        prefix,
        pathInDir,
        options,
        cursorData
      );
    } else {
      yield* this._listRegular(
        directory,
        prefix,
        pathInDir,
        options,
        cursorData
      );
    }
  }

  // Delete a file or directory
  async delete(path: string): Promise<boolean> {
    const segments = path.split("/").filter((s) => s);
    const name = segments.pop();
    if (!name) return false;

    const dirPath = segments.join("/");

    return await this._updateDirectory(dirPath, (dir) => {
      const hadFile = dir.files.has(name);
      const hadDir = dir.dirs.has(name);

      dir.files.delete(name);
      dir.dirs.delete(name);

      if (hadFile || hadDir) {
        return dir;
      }
      return null; // No changes
    });
  }

  // Cursor encoding/decoding with deterministic CBOR
  private _encodeCursor(data: CursorData): string {
    // IMPORTANT: Use deterministic encoding for cursors
    const encoded = encodeS5({
      v: 1, // Version for future compatibility
      t: data.type,
      n: data.name,
      h: data.hamtPath,
      p: data.position,
    });

    return base64UrlNoPaddingEncode(encoded);
  }

  private _parseCursor(cursor: string): CursorData {
    try {
      const decoded = decodeS5(base64UrlNoPaddingDecode(cursor));
      if (decoded.v !== 1) throw new Error("Unsupported cursor version");

      return {
        type: decoded.t,
        name: decoded.n,
        hamtPath: decoded.h,
        position: decoded.p,
      };
    } catch {
      throw new Error("Invalid cursor");
    }
  }

  // Internal: Navigate to directory
  private async _resolvePath(path: string): Promise<{
    directory: DirV1;
    fileName?: string;
    pathInDir?: string;
  }> {
    const segments = path.split("/").filter((s) => s);
    let currentPath = await this._preprocessLocalPath(segments[0] || "home");
    let directory = await this._loadDirectory(currentPath);

    // Navigate through path
    for (let i = 1; i < segments.length; i++) {
      const segment = segments[i];

      // Check if this is a directory
      const dirEntry = directory.dirs.get(segment);
      if (dirEntry) {
        currentPath = `${currentPath}/${segment}`;
        directory = await this._loadDirectory(currentPath);
      } else {
        // Must be a file or doesn't exist
        return {
          directory,
          fileName: segment,
          pathInDir: segments.slice(i).join("/"),
        };
      }
    }

    return { directory };
  }

  // Internal: Load directory with caching
  private async _loadDirectory(path: string): Promise<DirV1> {
    const cached = this.nodeCache.get(path);
    if (cached) return cached;

    const ks = await this.getKeySet(path);
    const metadata = await this._getDirectoryMetadata(ks);

    if (!metadata) {
      // Create empty directory
      return {
        magic: "S5.pro",
        header: {},
        dirs: new Map(),
        files: new Map(),
      };
    }

    const directory = DirV1Serialiser.deserialise(metadata.data);
    this.nodeCache.set(path, directory);
    return directory;
  }

  // Internal: Update directory with transaction and LWW conflict resolution
  private async _updateDirectory(
    path: string,
    updater: (dir: DirV1) => DirV1 | null
  ): Promise<boolean> {
    const maxRetries = 3;
    const retryDelay = 100; // ms

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      const ks = await this.getKeySet(
        await this._preprocessLocalPath(path || "home")
      );

      if (!ks.writeKey) {
        throw new Error("No write access to directory");
      }

      const currentMetadata = await this._getDirectoryMetadata(ks);
      const currentDir = currentMetadata
        ? DirV1Serialiser.deserialise(currentMetadata.data)
        : this._createEmptyDirectory();

      // Apply update
      const updatedDir = updater(currentDir);
      if (!updatedDir) return false; // No changes

      // Check if we need to enable sharding
      const totalEntries = updatedDir.files.size + updatedDir.dirs.size;
      if (totalEntries > 1000 && !updatedDir.header.sharding) {
        updatedDir.header.sharding = {
          type: "hamt",
          config: {
            bitsPerLevel: 5,
            maxInlineEntries: 1000,
            hashFunction: 0,
          },
        };
      }

      // Serialise and store
      const serialised = await this._serialiseDirectory(updatedDir);
      const blob = new Blob([serialised]);
      const cid = await this.api.uploadBlob(blob);

      // Update registry
      const kp = await this.api.crypto.newKeyPairEd25519(ks.writeKey);
      const entry = await createRegistryEntry(
        kp,
        cid.hash,
        (currentMetadata?.entry?.revision ?? 0) + 1,
        this.api.crypto
      );

      try {
        await this.api.registrySet(entry);

        // Success! Update cache and return
        this.nodeCache.delete(path);
        return true;
      } catch (error: any) {
        // Check if this is a revision conflict
        if (
          error.message?.includes("revision") ||
          error.message?.includes("conflict") ||
          error.code === "REVISION_CONFLICT"
        ) {
          if (attempt < maxRetries - 1) {
            // Wait briefly before retry to reduce contention
            await new Promise((resolve) =>
              setTimeout(resolve, retryDelay * (attempt + 1))
            );
            continue; // Retry with fresh data
          }
        }

        // Not a revision conflict or out of retries
        throw error;
      }
    }

    throw new Error(
      `Failed to update after ${maxRetries} attempts due to conflicts`
    );
  }

  // Serialise directory (with HAMT support)
  private async _serialiseDirectory(dir: DirV1): Promise<Uint8Array> {
    if (
      dir.header.sharding &&
      dir.files.size + dir.dirs.size >
        dir.header.sharding.config.maxInlineEntries
    ) {
      return this._serialiseShardedDirectory(dir);
    }
    return DirV1Serialiser.serialise(dir);
  }

  private _createEmptyDirectory(): DirV1 {
    return {
      magic: "S5.pro",
      header: {},
      dirs: new Map(),
      files: new Map(),
    };
  }

  private _getOldestTimestamp(dir: DirV1): string | undefined {
    let oldest: number | undefined;

    for (const [_, file] of dir.files) {
      if (file.timestamp && (!oldest || file.timestamp < oldest)) {
        oldest = file.timestamp;
      }
    }

    for (const [_, subdir] of dir.dirs) {
      if (subdir.ts_seconds && (!oldest || subdir.ts_seconds < oldest)) {
        oldest = subdir.ts_seconds;
      }
    }

    return oldest ? new Date(oldest * 1000).toISOString() : undefined;
  }

  private _getNewestTimestamp(dir: DirV1): string | undefined {
    let newest: number | undefined;

    for (const [_, file] of dir.files) {
      if (file.timestamp && (!newest || file.timestamp > newest)) {
        newest = file.timestamp;
      }
    }

    for (const [_, subdir] of dir.dirs) {
      if (subdir.ts_seconds && (!newest || subdir.ts_seconds > newest)) {
        newest = subdir.ts_seconds;
      }
    }

    return newest ? new Date(newest * 1000).toISOString() : undefined;
  }

  private _extractFileMetadata(file: FileRef): Record<string, any> {
    return {
      size: file.size,
      mediaType: file.media_type,
      timestamp: file.timestamp
        ? new Date(file.timestamp * 1000).toISOString()
        : undefined,
      custom: file.extra ? Object.fromEntries(file.extra) : undefined,
    };
  }

  private _extractDirMetadata(dir: DirRef): Record<string, any> {
    return {
      timestamp: dir.ts_seconds
        ? new Date(dir.ts_seconds * 1000).toISOString()
        : undefined,
      extra: dir.extra,
    };
  }

  // Regular listing with cursor support
  private async *_listRegular(
    directory: DirV1,
    prefix: string,
    pathInDir: string,
    options?: ListOptions,
    cursorData?: CursorData | null
  ): AsyncIterator<ListResult> {
    let count = 0;
    let foundCursor = !cursorData; // If no cursor, start from beginning

    // Combine files and directories, sort by name for consistent ordering
    const allEntries = [
      ...Array.from(directory.files.entries()).map(([name, file]) => ({
        name,
        type: "file" as const,
        data: file,
      })),
      ...Array.from(directory.dirs.entries()).map(([name, dir]) => ({
        name,
        type: "directory" as const,
        data: dir,
      })),
    ].sort((a, b) => a.name.localeCompare(b.name));

    for (const entry of allEntries) {
      // Skip until we find the cursor position
      if (!foundCursor) {
        if (
          entry.name === cursorData!.name &&
          entry.type === cursorData!.type
        ) {
          foundCursor = true;
        }
        continue; // Skip this entry, we've seen it before
      }

      // Apply path filter
      if (pathInDir && !entry.name.startsWith(pathInDir)) continue;

      const result: ListResult = {
        path: `${prefix}/${entry.name}`,
        name: entry.name,
        type: entry.type,
        metadata:
          entry.type === "file"
            ? this._extractFileMetadata(entry.data as FileRef)
            : this._extractDirMetadata(entry.data as DirRef),
        cursor: this._encodeCursor({
          type: entry.type,
          name: entry.name,
          position: count,
        }),
      };

      // Apply optional filter
      if (options?.filter && !options.filter(result)) continue;

      yield result;

      count++;
      if (options?.limit && count >= options.limit) return;
    }
  }

  // HAMT listing with cursor support
  private async *_listWithHAMT(
    directory: DirV1,
    prefix: string,
    pathInDir: string,
    options?: ListOptions,
    cursorData?: CursorData | null
  ): AsyncIterator<ListResult> {
    if (!directory.header.sharding?.root) return;

    const hamtData = await this.api.downloadBlobAsBytes(
      directory.header.sharding.root.cid
    );
    const hamt = await HAMT.deserialise(hamtData, this.api);

    let count = 0;
    let foundCursor = !cursorData;

    // Resume HAMT traversal from cursor position
    const iterator = cursorData?.hamtPath
      ? await hamt.entriesFrom(cursorData.hamtPath)
      : hamt.entries();

    for await (const [key, entry] of iterator) {
      // Extract type and name
      const type = key.startsWith("d:") ? "directory" : "file";
      const name = key.substring(2);

      // Skip until we find the exact cursor position
      if (!foundCursor) {
        if (name === cursorData!.name && type === cursorData!.type) {
          foundCursor = true;
        }
        continue;
      }

      // Apply path filter
      if (pathInDir && !name.startsWith(pathInDir)) continue;

      const result: ListResult = {
        path: `${prefix}/${name}`,
        name,
        type,
        metadata:
          type === "file"
            ? this._extractFileMetadata(entry as FileRef)
            : this._extractDirMetadata(entry as DirRef),
        cursor: this._encodeCursor({
          type,
          name,
          hamtPath: await hamt.getPathForKey(key), // Store HAMT position
          position: count,
        }),
      };

      // Apply optional filter
      if (options?.filter && !options.filter(result)) continue;

      yield result;

      count++;
      if (options?.limit && count >= options.limit) return;
    }
  }

  // Get file from potentially sharded directory
  private async _getFileFromDirectory(
    dir: DirV1,
    fileName: string
  ): Promise<FileRef | undefined> {
    // Check inline files first
    const inlineFile = dir.files.get(fileName);
    if (inlineFile) return inlineFile;

    // Check HAMT if sharded
    if (dir.header.sharding?.root) {
      const hamtData = await this.api.downloadBlobAsBytes(
        dir.header.sharding.root.cid
      );
      const hamt = await HAMT.deserialise(hamtData, this.api);
      return (await hamt.get(`f:${fileName}`)) as FileRef;
    }

    return undefined;
  }

  // Serialise sharded directory
  private async _serialiseShardedDirectory(dir: DirV1): Promise<Uint8Array> {
    // Create HAMT
    const hamt = new HAMT(this.api, dir.header.sharding?.config);

    // Add all entries
    for (const [name, dirRef] of dir.dirs) {
      await hamt.insert(`d:${name}`, dirRef);
    }
    for (const [name, fileRef] of dir.files) {
      await hamt.insert(`f:${name}`, fileRef);
    }

    // Store HAMT root
    const hamtData = hamt.serialise();
    const blob = new Blob([hamtData]);
    const hamtCID = await this.api.uploadBlob(blob);

    // Create directory with HAMT reference only
    const shardedDir: DirV1 = {
      magic: dir.magic,
      header: {
        ...dir.header,
        sharding: {
          ...dir.header.sharding!,
          root: {
            cid: hamtCID.toBytes(),
            totalEntries: dir.files.size + dir.dirs.size,
            depth: await hamt.getDepth(),
          },
        },
      },
      dirs: new Map(), // Empty - all in HAMT
      files: new Map(), // Empty - all in HAMT
    };

    return DirV1Serialiser.serialise(shardedDir);
  }
}
```

## Phase 3: HAMT Integration

### 3.1 HAMT Implementation

Update `src/fs/hamt/hamt.ts`:

```typescript
import { xxhash64 } from "../util/xxhash";
import { blake3 } from "@noble/hashes/blake3";
import { encodeS5, decodeS5 } from "../dirv1/cbor-config";
import { FileRef, DirRef } from "../dirv1/types";

export class HAMT {
  private rootNode: HAMTNode | null = null;
  private config: HAMTConfig;
  private nodeCache: Map<string, HAMTNode>;

  constructor(private api: S5APIInterface, config?: Partial<HAMTConfig>) {
    this.config = {
      bitsPerLevel: 5,
      maxInlineEntries: 8,
      hashFunction: 0,
      ...config,
    };
    this.nodeCache = new Map();
  }

  async insert(key: string, value: FileRef | DirRef): Promise<void> {
    const hash = this.hashKey(key);

    if (!this.rootNode) {
      this.rootNode = this.createEmptyNode(0);
    }

    await this._insertAtNode(this.rootNode, hash, 0, key, value);
  }

  async get(key: string): Promise<FileRef | DirRef | undefined> {
    if (!this.rootNode) return undefined;

    const hash = this.hashKey(key);
    return this._getFromNode(this.rootNode, hash, 0, key);
  }

  async *entries(): AsyncIterator<[string, FileRef | DirRef]> {
    if (!this.rootNode) return;
    yield* this._entriesFromNode(this.rootNode);
  }

  // Resume from a specific path for cursor support
  async *entriesFrom(
    hamtPath: number[]
  ): AsyncIterator<[string, FileRef | DirRef]> {
    if (!this.rootNode) return;
    yield* this._entriesFromPath(this.rootNode, hamtPath, 0);
  }

  // Get the path to a specific key (for cursor generation)
  async getPathForKey(targetKey: string): Promise<number[]> {
    if (!this.rootNode) return [];

    const path: number[] = [];
    const hash = this.hashKey(targetKey);
    await this._findPath(this.rootNode, hash, 0, targetKey, path);
    return path;
  }

  serialise(): Uint8Array {
    if (!this.rootNode) {
      return new Uint8Array();
    }

    // Use deterministic encoding for HAMT nodes
    return encodeS5({
      version: 1,
      config: this.config,
      root: this.rootNode,
    });
  }

  static async deserialise(
    data: Uint8Array,
    api: S5APIInterface
  ): Promise<HAMT> {
    const decoded = decodeS5(data);
    const hamt = new HAMT(api, decoded.config);
    hamt.rootNode = decoded.root;
    return hamt;
  }

  private async _insertAtNode(
    node: HAMTNode,
    hash: bigint,
    depth: number,
    key: string,
    value: FileRef | DirRef
  ): Promise<void> {
    const index = this.getIndex(hash, depth);
    const bit = 1 << index;

    if (!(node.bitmap & bit)) {
      // Empty slot - insert directly
      const childIdx = this.getChildIndex(node.bitmap, index);
      node.bitmap |= bit;
      node.children.splice(childIdx, 0, {
        type: "leaf",
        entries: [[key, value]],
      });
      node.count++;
      return;
    }

    // Slot occupied - check what's there
    const childIdx = this.getChildIndex(node.bitmap, index);
    const child = node.children[childIdx];

    if (child.type === "leaf") {
      // Check if we need to split
      if (child.entries.length >= this.config.maxInlineEntries) {
        // Convert to internal node
        const newNode = await this.splitLeaf(child.entries, depth + 1);
        node.children[childIdx] = {
          type: "node",
          cid: await this.storeNode(newNode),
        };
        // Retry insertion
        await this._insertAtNode(node, hash, depth, key, value);
      } else {
        // Add to leaf
        child.entries.push([key, value]);
        node.count++;
      }
    } else {
      // Navigate to child node
      const childNode = await this.loadNode(child.cid);
      await this._insertAtNode(childNode, hash, depth + 1, key, value);
      // Update stored node
      child.cid = await this.storeNode(childNode);
      node.count++;
    }
  }

  private async splitLeaf(
    entries: Array<[string, FileRef | DirRef]>,
    depth: number
  ): Promise<HAMTNode> {
    const node = this.createEmptyNode(depth);

    // Redistribute entries
    for (const [key, value] of entries) {
      const hash = this.hashKey(key);
      await this._insertAtNode(node, hash, depth, key, value);
    }

    return node;
  }

  private async _getFromNode(
    node: HAMTNode,
    hash: bigint,
    depth: number,
    key: string
  ): Promise<FileRef | DirRef | undefined> {
    const index = this.getIndex(hash, depth);

    if (!this.hasChild(node.bitmap, index)) {
      return undefined;
    }

    const childIdx = this.getChildIndex(node.bitmap, index);
    const child = node.children[childIdx];

    if (child.type === "leaf") {
      const entry = child.entries.find(([k]) => k === key);
      return entry?.[1];
    } else {
      const childNode = await this.loadNode(child.cid);
      return this._getFromNode(childNode, hash, depth + 1, key);
    }
  }

  private async *_entriesFromNode(
    node: HAMTNode
  ): AsyncIterator<[string, FileRef | DirRef]> {
    for (const child of node.children) {
      if (child.type === "leaf") {
        for (const entry of child.entries) {
          yield entry;
        }
      } else {
        const childNode = await this.loadNode(child.cid);
        yield* this._entriesFromNode(childNode);
      }
    }
  }

  private async _findPath(
    node: HAMTNode,
    hash: bigint,
    depth: number,
    targetKey: string,
    path: number[]
  ): Promise<boolean> {
    const index = this.getIndex(hash, depth);

    if (!this.hasChild(node.bitmap, index)) {
      return false;
    }

    const childIdx = this.getChildIndex(node.bitmap, index);
    path.push(childIdx);

    const child = node.children[childIdx];

    if (child.type === "leaf") {
      return child.entries.some(([k]) => k === targetKey);
    } else {
      const childNode = await this.loadNode(child.cid);
      return this._findPath(childNode, hash, depth + 1, targetKey, path);
    }
  }

  private async *_entriesFromPath(
    node: HAMTNode,
    hamtPath: number[],
    depth: number
  ): AsyncIterator<[string, FileRef | DirRef]> {
    const startIdx = depth < hamtPath.length ? hamtPath[depth] : 0;

    for (let i = startIdx; i < node.children.length; i++) {
      const child = node.children[i];

      if (child.type === "leaf") {
        // If we're at the target depth, skip entries we've seen
        const startEntry = depth === hamtPath.length - 1 ? hamtPath[depth] : 0;

        for (
          let j = i === startIdx ? startEntry : 0;
          j < child.entries.length;
          j++
        ) {
          yield child.entries[j];
        }
      } else {
        const childNode = await this.loadNode(child.cid);

        // Continue from path if we're still following it
        if (i === startIdx && depth + 1 < hamtPath.length) {
          yield* this._entriesFromPath(childNode, hamtPath, depth + 1);
        } else {
          yield* this._entriesFromNode(childNode);
        }
      }
    }
  }

  private hashKey(key: string): bigint {
    if (this.config.hashFunction === 0) {
      return xxhash64(key);
    } else {
      const hash = blake3(new TextEncoder().encode(key));
      return new DataView(hash.buffer).getBigUint64(0, false);
    }
  }

  private getIndex(hash: bigint, depth: number): number {
    const shift = BigInt(depth * this.config.bitsPerLevel);
    const mask = BigInt((1 << this.config.bitsPerLevel) - 1);
    return Number((hash >> shift) & mask);
  }

  private hasChild(bitmap: number, index: number): boolean {
    return (bitmap & (1 << index)) !== 0;
  }

  private getChildIndex(bitmap: number, index: number): number {
    const mask = (1 << index) - 1;
    return popcount(bitmap & mask);
  }

  private createEmptyNode(depth: number): HAMTNode {
    return {
      bitmap: 0,
      children: [],
      count: 0,
      depth,
    };
  }

  private async storeNode(node: HAMTNode): Promise<Uint8Array> {
    // CRITICAL: Deterministic encoding for consistent CIDs
    const encoded = encodeS5(node);
    const blob = new Blob([encoded]);
    const cid = await this.api.uploadBlob(blob);

    this.nodeCache.set(base64UrlNoPaddingEncode(cid.toBytes()), node);
    return cid.toBytes();
  }

  private async loadNode(cid: Uint8Array): Promise<HAMTNode> {
    const cidStr = base64UrlNoPaddingEncode(cid);
    const cached = this.nodeCache.get(cidStr);
    if (cached) return cached;

    const data = await this.api.downloadBlobAsBytes(cid);
    const node = decodeS5(data) as HAMTNode;
    this.nodeCache.set(cidStr, node);
    return node;
  }

  async getDepth(): Promise<number> {
    if (!this.rootNode) return 0;
    return this._getMaxDepth(this.rootNode);
  }

  private async _getMaxDepth(node: HAMTNode): Promise<number> {
    let maxDepth = node.depth;

    for (const child of node.children) {
      if (child.type === "node") {
        const childNode = await this.loadNode(child.cid);
        const childDepth = await this._getMaxDepth(childNode);
        maxDepth = Math.max(maxDepth, childDepth);
      }
    }

    return maxDepth;
  }
}

interface HAMTNode {
  bitmap: number;
  children: Array<HAMTChild>;
  count: number;
  depth: number;
}

type HAMTChild =
  | { type: "node"; cid: Uint8Array }
  | { type: "leaf"; entries: Array<[string, FileRef | DirRef]> };

interface HAMTConfig {
  bitsPerLevel: number;
  maxInlineEntries: number;
  hashFunction: number;
}

// Bit manipulation helper
function popcount(n: number): number {
  n = n - ((n >>> 1) & 0x55555555);
  n = (n & 0x33333333) + ((n >>> 2) & 0x33333333);
  return (((n + (n >>> 4)) & 0xf0f0f0f) * 0x1010101) >>> 24;
}
```

## Phase 4: Utility Functions

### 4.1 Directory Walker

New file `src/fs/utils/walker.ts`:

```typescript
export interface WalkOptions {
  recursive?: boolean;
  includeFiles?: boolean;
  includeDirectories?: boolean;
  filter?: (name: string, type: "file" | "directory") => boolean;
  maxDepth?: number;
  cursor?: string; // Resume from cursor
}

export class DirectoryWalker {
  constructor(private fs: FS5, private startPath: string) {}

  async *walk(options: WalkOptions = {}): AsyncIterator<WalkResult> {
    const {
      recursive = true,
      includeFiles = true,
      includeDirectories = true,
      filter,
      maxDepth = Infinity,
      cursor,
    } = options;

    const stack: Array<{ path: string; depth: number; cursor?: string }> = [
      { path: this.startPath, depth: 0, cursor },
    ];

    while (stack.length > 0) {
      const { path, depth, cursor: dirCursor } = stack.pop()!;

      if (depth > maxDepth) continue;

      // List directory contents with cursor support
      for await (const item of this.fs.list(path, { cursor: dirCursor })) {
        const include =
          (item.type === "file" && includeFiles) ||
          (item.type === "directory" && includeDirectories);

        if (include && (!filter || filter(item.name, item.type))) {
          yield {
            path: item.path,
            name: item.name,
            type: item.type,
            metadata: item.metadata,
            depth,
            cursor: item.cursor,
          };
        }

        if (recursive && item.type === "directory") {
          stack.push({ path: item.path, depth: depth + 1 });
        }
      }
    }
  }

  async count(options: WalkOptions = {}): Promise<WalkStats> {
    let files = 0;
    let directories = 0;
    let totalSize = 0;

    for await (const item of this.walk(options)) {
      if (item.type === "file") {
        files++;
        totalSize += item.metadata.size || 0;
      } else {
        directories++;
      }
    }

    return { files, directories, totalSize };
  }
}

interface WalkResult {
  path: string;
  name: string;
  type: "file" | "directory";
  metadata: any;
  depth: number;
  cursor: string;
}

interface WalkStats {
  files: number;
  directories: number;
  totalSize: number;
}
```

### 4.2 Batch Operations

New file `src/fs/utils/batch.ts`:

```typescript
export class BatchOperations {
  constructor(private fs: FS5) {}

  async copyDirectory(
    source: string,
    destination: string,
    options?: CopyOptions
  ): Promise<CopyResult> {
    const walker = new DirectoryWalker(this.fs, source);
    let copied = 0;
    let skipped = 0;
    let errors = 0;
    let lastCursor: string | undefined;

    // Resume from cursor if provided
    const walkOptions: WalkOptions = {
      cursor: options?.resumeCursor,
    };

    // Create destination directory
    await this._ensureDirectory(destination);

    for await (const item of walker.walk(walkOptions)) {
      try {
        const relativePath = item.path.substring(source.length);
        const destPath = `${destination}${relativePath}`;

        if (item.type === "directory") {
          await this._ensureDirectory(destPath);
          copied++;
        } else {
          // Check if file exists
          if (!options?.overwrite) {
            const existing = await this.fs.get(destPath);
            if (existing !== undefined) {
              skipped++;
              continue;
            }
          }

          // Copy file data
          const data = await this.fs.get(item.path);
          const metadata = await this.fs.getMetadata(item.path);

          await this.fs.put(destPath, data, {
            mediaType: metadata?.mediaType,
            metadata: metadata?.custom,
          });
          copied++;
        }

        lastCursor = item.cursor;
      } catch (error) {
        errors++;
        if (options?.stopOnError) throw error;
      }
    }

    return { copied, skipped, errors, lastCursor };
  }

  async deleteDirectory(
    path: string,
    options?: DeleteOptions
  ): Promise<DeleteResult> {
    let deleted = 0;
    let errors = 0;

    if (!options?.recursive) {
      // Just delete the directory entry
      const success = await this.fs.delete(path);
      return { deleted: success ? 1 : 0, errors: success ? 0 : 1 };
    }

    // Collect all items for deletion (bottom-up)
    const toDelete: string[] = [];
    const walker = new DirectoryWalker(this.fs, path);

    for await (const item of walker.walk()) {
      toDelete.push(item.path);
    }

    // Sort by depth (deepest first)
    toDelete.sort((a, b) => b.split("/").length - a.split("/").length);

    // Delete items
    for (const itemPath of toDelete) {
      try {
        const success = await this.fs.delete(itemPath);
        if (success) deleted++;
        else errors++;
      } catch (error) {
        errors++;
        if (options?.stopOnError) throw error;
      }
    }

    // Finally delete the directory itself
    try {
      const success = await this.fs.delete(path);
      if (success) deleted++;
    } catch {
      errors++;
    }

    return { deleted, errors };
  }

  private async _ensureDirectory(path: string): Promise<void> {
    const segments = path.split("/").filter((s) => s);
    let currentPath = "";

    for (const segment of segments) {
      currentPath = currentPath ? `${currentPath}/${segment}` : segment;

      // Check if directory exists
      const metadata = await this.fs.getMetadata(currentPath);
      if (!metadata || metadata.type !== "directory") {
        // Create directory by adding it to parent
        await this.fs.createDirectory(
          currentPath.substring(0, currentPath.lastIndexOf("/")),
          segment
        );
      }
    }
  }
}

interface CopyOptions {
  overwrite?: boolean;
  filter?: (name: string, type: "file" | "directory") => boolean;
  stopOnError?: boolean;
  resumeCursor?: string; // Resume from cursor
}

interface CopyResult {
  copied: number;
  skipped: number;
  errors: number;
  lastCursor?: string; // Cursor to resume if interrupted
}

interface DeleteOptions {
  recursive?: boolean;
  stopOnError?: boolean;
}

interface DeleteResult {
  deleted: number;
  errors: number;
}
```

## Phase 5: Testing Strategy

### 5.1 Core Functionality Tests

```typescript
describe("Enhanced S5.js", () => {
  describe("Path-based API", () => {
    it("should store and retrieve data at any path", async () => {
      const testData = { message: "Hello S5", value: 42 };
      await s5.fs.put("home/test/data.json", testData);

      const retrieved = await s5.fs.get("home/test/data.json");
      expect(retrieved).toEqual(testData);
    });

    it("should handle metadata correctly", async () => {
      const data = { content: "test" };
      const metadata = { author: "Alice", tags: ["test", "example"] };

      await s5.fs.put("home/test/meta.json", data, {
        metadata,
        mediaType: "application/json",
      });

      const fileMeta = await s5.fs.getMetadata("home/test/meta.json");
      expect(fileMeta?.custom).toEqual(metadata);
      expect(fileMeta?.mediaType).toBe("application/json");
    });

    it("should handle binary data efficiently", async () => {
      const binary = new Uint8Array([1, 2, 3, 4, 5]);
      await s5.fs.put("home/binary/test.bin", binary);

      const retrieved = await s5.fs.get("home/binary/test.bin");
      expect(new Uint8Array(retrieved)).toEqual(binary);
    });

    it("should support encrypted storage", async () => {
      const secretData = { secret: "confidential info" };
      const key = crypto.getRandomValues(new Uint8Array(32));

      await s5.fs.put("home/encrypted/secret.json", secretData, {
        encryption: {
          algorithm: "xchacha20-poly1305",
          key,
        },
      });

      // Verify encryption metadata is stored
      const metadata = await s5.fs.getMetadata("home/encrypted/secret.json");
      expect(metadata?.custom?.encryption?.algorithm).toBe(
        "xchacha20-poly1305"
      );
    });

    it("should list directory contents with pagination", async () => {
      // Create test files
      for (let i = 0; i < 25; i++) {
        await s5.fs.put(`home/list-test/file${i}.txt`, `content ${i}`);
      }

      // List with limit
      const items: string[] = [];
      for await (const item of s5.fs.list("home/list-test", { limit: 10 })) {
        items.push(item.name);
      }

      expect(items.length).toBe(10);
    });
  });

  describe("CBOR Determinism", () => {
    it("should produce identical encoding for same data", () => {
      const data = {
        z: "last",
        a: "first",
        m: "middle",
        nested: { y: 2, x: 1 },
      };

      const encoded1 = encodeS5(data);
      const encoded2 = encodeS5(data);

      expect(encoded1).toEqual(encoded2);
    });

    it("should produce same hash for reordered object keys", () => {
      const data1 = { a: 1, b: 2, c: 3 };
      const data2 = { c: 3, a: 1, b: 2 };

      const encoded1 = encodeS5(data1);
      const encoded2 = encodeS5(data2);

      expect(encoded1).toEqual(encoded2);
    });

    it("should handle Maps deterministically", () => {
      const map1 = new Map([
        ["z", 1],
        ["a", 2],
        ["m", 3],
      ]);
      const map2 = new Map([
        ["a", 2],
        ["m", 3],
        ["z", 1],
      ]);

      const encoded1 = encodeS5(map1);
      const encoded2 = encodeS5(map2);

      // Maps preserve insertion order, so these should be different
      expect(encoded1).not.toEqual(encoded2);

      // But ordered maps should be identical
      const ordered1 = createOrderedMap({ z: 1, a: 2, m: 3 });
      const ordered2 = createOrderedMap({ a: 2, m: 3, z: 1 });

      const encodedOrdered1 = encodeS5(ordered1);
      const encodedOrdered2 = encodeS5(ordered2);

      expect(encodedOrdered1).toEqual(encodedOrdered2);
    });
  });

  describe("Cursor-based Pagination", () => {
    it("should support cursor-based pagination", async () => {
      // Create test files
      for (let i = 0; i < 100; i++) {
        await s5.fs.put(
          `home/cursor-test/file${i.toString().padStart(3, "0")}.txt`,
          `content ${i}`
        );
      }

      // First page
      const page1: ListResult[] = [];
      let lastCursor: string | undefined;

      for await (const item of s5.fs.list("home/cursor-test", { limit: 20 })) {
        page1.push(item);
        lastCursor = item.cursor;
      }

      expect(page1.length).toBe(20);
      expect(page1[0].name).toBe("file000.txt");

      // Second page using cursor
      const page2: ListResult[] = [];
      for await (const item of s5.fs.list("home/cursor-test", {
        limit: 20,
        cursor: lastCursor,
      })) {
        page2.push(item);
      }

      expect(page2.length).toBe(20);
      expect(page2[0].name).toBe("file020.txt");

      // Verify no overlap
      const page1Names = new Set(page1.map((item) => item.name));
      const page2Names = new Set(page2.map((item) => item.name));
      const intersection = new Set(
        [...page1Names].filter((x) => page2Names.has(x))
      );
      expect(intersection.size).toBe(0);
    });

    it("should maintain cursor consistency across directory changes", async () => {
      // Create initial files
      for (let i = 0; i < 50; i++) {
        await s5.fs.put(`home/dynamic/file${i}.txt`, `content ${i}`);
      }

      // Get first page
      const page1: ListResult[] = [];
      let cursor: string | undefined;

      for await (const item of s5.fs.list("home/dynamic", { limit: 25 })) {
        page1.push(item);
        cursor = item.cursor;
      }

      // Add new files
      for (let i = 50; i < 60; i++) {
        await s5.fs.put(`home/dynamic/file${i}.txt`, `content ${i}`);
      }

      // Continue with cursor - should not see duplicates
      const page2: ListResult[] = [];
      for await (const item of s5.fs.list("home/dynamic", {
        limit: 25,
        cursor,
      })) {
        page2.push(item);
      }

      // Check continuity
      const allNames = [...page1, ...page2].map((item) => item.name);
      const uniqueNames = new Set(allNames);
      expect(allNames.length).toBe(uniqueNames.size);
    });
  });

  describe("HAMT Sharding", () => {
    it("should automatically shard large directories", async () => {
      // Create directory
      await s5.fs.createDirectory("home", "test-shard");

      // Add files until sharding triggers
      for (let i = 0; i < 1500; i++) {
        await s5.fs.put(`home/test-shard/file-${i}.txt`, `content-${i}`);
      }

      // Verify directory is sharded
      const metadata = await s5.fs.getMetadata("home/test-shard");
      expect(metadata?.sharding).toBeDefined();
      expect(metadata?.sharding?.type).toBe("hamt");
    });

    it("should handle mixed file/directory entries", async () => {
      await s5.fs.createDirectory("home", "mixed");

      // Add mix of files and subdirectories
      for (let i = 0; i < 1000; i++) {
        if (i % 3 === 0) {
          await s5.fs.createDirectory("home/mixed", `subdir-${i}`);
        } else {
          await s5.fs.put(`home/mixed/file-${i}.txt`, `content-${i}`);
        }
      }

      const dirMeta = await s5.fs.getMetadata("home/mixed");
      expect(dirMeta?.fileCount).toBeGreaterThan(600);
      expect(dirMeta?.directoryCount).toBeGreaterThan(300);
    });

    it("should support cursor pagination in sharded directories", async () => {
      // Create sharded directory
      await s5.fs.createDirectory("home", "sharded-cursor");

      // Add enough files to trigger sharding
      for (let i = 0; i < 2000; i++) {
        await s5.fs.put(
          `home/sharded-cursor/file-${i.toString().padStart(4, "0")}.txt`,
          `content-${i}`
        );
      }

      // Paginate through sharded directory
      let totalItems = 0;
      let cursor: string | undefined;
      const pageSize = 100;

      while (true) {
        let pageItems = 0;

        for await (const item of s5.fs.list("home/sharded-cursor", {
          limit: pageSize,
          cursor,
        })) {
          pageItems++;
          cursor = item.cursor;
        }

        totalItems += pageItems;

        if (pageItems < pageSize) break;
      }

      expect(totalItems).toBe(2000);
    });
  });

  describe("Performance", () => {
    it("should handle 1M entries efficiently", async () => {
      await s5.fs.createDirectory("home", "huge");

      const batchSize = 10000;
      const totalBatches = 100;

      console.time("Insert 1M entries");
      for (let batch = 0; batch < totalBatches; batch++) {
        const promises: Promise<void>[] = [];

        for (let i = 0; i < batchSize; i++) {
          const idx = batch * batchSize + i;
          promises.push(s5.fs.put(`home/huge/file-${idx}.txt`, { index: idx }));
        }

        await Promise.all(promises);

        if ((batch + 1) % 10 === 0) {
          console.log(`Inserted ${(batch + 1) * batchSize} entries`);
        }
      }
      console.timeEnd("Insert 1M entries");

      // Test lookup performance
      console.time("Random lookups (1000)");
      for (let i = 0; i < 1000; i++) {
        const idx = Math.floor(Math.random() * 1000000);
        const data = await s5.fs.get(`home/huge/file-${idx}.txt`);
        expect(data.index).toBe(idx);
      }
      console.timeEnd("Random lookups (1000)");

      // Test cursor pagination performance
      console.time("Paginate through 10k items");
      let count = 0;
      let cursor: string | undefined;

      while (count < 10000) {
        for await (const item of s5.fs.list("home/huge", {
          limit: 1000,
          cursor,
        })) {
          count++;
          cursor = item.cursor;
          if (count >= 10000) break;
        }
      }
      console.timeEnd("Paginate through 10k items");

      // Test metadata retrieval
      const hugeMeta = await s5.fs.getMetadata("home/huge");
      expect(hugeMeta?.fileCount).toBe(1000000);
    });
  });

  describe("Directory Utilities", () => {
    it("should walk directories recursively", async () => {
      // Create nested structure
      await s5.fs.put("home/walk/a/file1.txt", "content1");
      await s5.fs.put("home/walk/a/b/file2.txt", "content2");
      await s5.fs.put("home/walk/c/file3.txt", "content3");

      const walker = new DirectoryWalker(s5.fs, "home/walk");
      const files: string[] = [];

      for await (const item of walker.walk()) {
        if (item.type === "file") {
          files.push(item.name);
        }
      }

      expect(files).toContain("file1.txt");
      expect(files).toContain("file2.txt");
      expect(files).toContain("file3.txt");
    });

    it("should support batch copy operations", async () => {
      // Create source directory with metadata
      for (let i = 0; i < 10; i++) {
        await s5.fs.put(`home/source/file${i}.txt`, `content ${i}`, {
          metadata: { index: i, type: "test" },
        });
      }

      // Copy to destination
      const batch = new BatchOperations(s5.fs);
      const result = await batch.copyDirectory("home/source", "home/dest");

      expect(result.copied).toBe(10);

      // Verify copy with metadata
      const copied = await s5.fs.get("home/dest/file5.txt");
      expect(copied).toBe("content 5");

      const copiedMeta = await s5.fs.getMetadata("home/dest/file5.txt");
      expect(copiedMeta?.custom?.index).toBe(5);
    });

    it("should support resumable copy operations", async () => {
      // Create large source directory
      for (let i = 0; i < 100; i++) {
        await s5.fs.put(`home/resume-source/file${i}.txt`, `content ${i}`);
      }

      // First partial copy
      const batch = new BatchOperations(s5.fs);
      let result = await batch.copyDirectory(
        "home/resume-source",
        "home/resume-dest",
        {
          // Simulate interruption by using walk with limit
        }
      );

      // Resume copy using cursor
      if (result.lastCursor) {
        result = await batch.copyDirectory(
          "home/resume-source",
          "home/resume-dest",
          {
            resumeCursor: result.lastCursor,
          }
        );
      }

      // Verify all files copied
      const destFiles = [];
      for await (const item of s5.fs.list("home/resume-dest")) {
        if (item.type === "file") {
          destFiles.push(item.name);
        }
      }

      expect(destFiles.length).toBe(100);
    });
  });
});
```

### 5.2 Performance Benchmarks

```typescript
async function runBenchmarks(s5: S5) {
  const results = [];

  // Test different scales
  for (const size of [100, 1000, 10000, 100000, 1000000]) {
    console.log(`Benchmarking ${size} entries...`);

    // Write performance
    const writeStart = Date.now();
    for (let i = 0; i < size; i++) {
      await s5.fs.put(
        `bench/${size}/item${i}`,
        { value: i },
        {
          metadata: { index: i },
        }
      );
    }
    const writeTime = Date.now() - writeStart;

    // Read performance (random access)
    const readStart = Date.now();
    const reads = Math.min(1000, size);
    for (let i = 0; i < reads; i++) {
      const idx = Math.floor(Math.random() * size);
      await s5.fs.get(`bench/${size}/item${idx}`);
    }
    const readTime = Date.now() - readStart;

    // Metadata performance
    const metaStart = Date.now();
    for (let i = 0; i < 100; i++) {
      const idx = Math.floor(Math.random() * size);
      await s5.fs.getMetadata(`bench/${size}/item${idx}`);
    }
    const metaTime = Date.now() - metaStart;

    // List performance with cursor
    const listStart = Date.now();
    let count = 0;
    let cursor: string | undefined;

    for await (const item of s5.fs.list(`bench/${size}`, { limit: 1000 })) {
      count++;
      cursor = item.cursor;
    }

    // Continue with cursor for next batch
    for await (const item of s5.fs.list(`bench/${size}`, {
      limit: 1000,
      cursor,
    })) {
      count++;
      if (count >= 2000) break;
    }

    const listTime = Date.now() - listStart;

    results.push({
      size,
      writeTime,
      writeOps: (size / writeTime) * 1000,
      readTime,
      readOps: (reads / readTime) * 1000,
      metaTime,
      metaOps: (100 / metaTime) * 1000,
      listTime,
      listOps: (count / listTime) * 1000,
    });
  }

  return results;
}
```

## Phase 6: Documentation

### 6.1 API Reference

````typescript
/**
 * Enhanced S5.js - Path-based Storage API
 *
 * Provides simple path-based access to S5 storage with automatic
 * directory management, metadata support, and efficient handling
 * of large datasets.
 *
 * @example
 * ```typescript
 * const s5 = await S5.create({ initialPeers: [...] });
 *
 * // Store data with metadata
 * await s5.fs.put('users/alice/profile.json', {
 *   name: 'Alice',
 *   email: 'alice@example.com'
 * }, {
 *   metadata: { version: '1.0', lastModified: Date.now() },
 *   mediaType: 'application/json'
 * });
 *
 * // Retrieve data
 * const profile = await s5.fs.get('users/alice/profile.json');
 *
 * // Get metadata
 * const metadata = await s5.fs.getMetadata('users/alice/profile.json');
 * console.log(metadata.custom.version); // '1.0'
 *
 * // Encrypted storage
 * await s5.fs.put('secrets/private.json', secretData, {
 *   encryption: {
 *     algorithm: 'xchacha20-poly1305',
 *     key: mySecretKey
 *   }
 * });
 *
 * // List directory contents with cursor pagination
 * let cursor: string | undefined;
 *
 * // First page
 * for await (const item of s5.fs.list('users/', { limit: 100 })) {
 *   console.log(item.name, item.type, item.metadata);
 *   cursor = item.cursor;
 * }
 *
 * // Next page
 * for await (const item of s5.fs.list('users/', { limit: 100, cursor })) {
 *   console.log(item.name, item.type, item.metadata);
 * }
 *
 * // Delete data
 * await s5.fs.delete('users/alice/profile.json');
 * ```
 */

/**
 * get - Retrieve data from any path
 * @param path - Full path to data (e.g. 'home/documents/file.txt')
 * @returns The stored data, automatically decoded from CBOR/JSON/text
 */
async get(path: string): Promise<any | undefined>

/**
 * put - Store data at any path with optional metadata
 * @param path - Full path including filename
 * @param data - Any serialisable data (objects, arrays, strings, binary)
 * @param options - Optional storage options including metadata and encryption
 */
async put(path: string, data: any, options?: PutOptions): Promise<void>

/**
 * getMetadata - Retrieve metadata for a file or directory
 * @param path - Path to file or directory
 * @returns Metadata object including size, type, timestamps, and custom fields
 */
async getMetadata(path: string): Promise<Record<string, any> | undefined>

/**
 * list - Iterate directory contents with optional pagination
 * @param path - Directory path to list
 * @param options - Options including limit, cursor, and filter
 * @yields Items in directory with path, name, type, metadata, and cursor
 */
async *list(path: string, options?: ListOptions): AsyncIterator<ListResult>

/**
 * delete - Remove file or directory
 * @param path - Path to file or directory to delete
 * @returns true if deleted, false if not found
 */
async delete(path: string): Promise<boolean>
````

## Additional Considerations

### Map Ordering

When creating directories programmatically, ensure consistent ordering:

```typescript
import { createOrderedMap } from "./dirv1/cbor-config";

// Instead of:
const files = new Map([
  ["z.txt", fileRef1],
  ["a.txt", fileRef2],
]);

// Use:
const files = createOrderedMap({
  "z.txt": fileRef1,
  "a.txt": fileRef2,
}); // Will be sorted as ['a.txt', 'z.txt']
```

### Testing Determinism

Always verify that your implementation produces deterministic results:

```typescript
describe("Implementation Compatibility", () => {
  it("should match Rust implementation hashes", async () => {
    const testDir: DirV1 = {
      magic: "S5.pro",
      header: {},
      dirs: createOrderedMap({ subdir1: dirRef1, subdir2: dirRef2 }),
      files: createOrderedMap({ file1: fileRef1, file2: fileRef2 }),
    };

    const serialised = DirV1Serialiser.serialise(testDir);
    const hash = await s5.api.crypto.hashBlake3(serialised);

    // This hash should match the Rust implementation
    expect(base64UrlNoPaddingEncode(hash)).toBe(EXPECTED_HASH_FROM_RUST);
  });
});
```

## Success Criteria

- [ ] Path-based API with metadata support
- [ ] DAG-CBOR serialisation matching Rust implementation
- [ ] Deterministic encoding for all operations
- [ ] HAMT sharding activates automatically at 1000+ entries
- [ ] O(log n) performance maintained to 10M+ entries
- [ ] Cursor-based pagination for stateless traversal
- [ ] Resumable operations using cursors
- [ ] 90%+ test coverage achieved
- [ ] Benchmarks show <100ms access at all scales
- [ ] Complete API documentation
- [ ] Migration guide published
- [ ] Cross-implementation compatibility verified
