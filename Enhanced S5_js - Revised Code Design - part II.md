# Enhanced S5.js - Revised Code Design - Part II

by Jules Lai, 27th June 2025

## Overview

This document details the media processing pipeline implementation for Enhanced S5.js, covering months 4-5 of the grant milestones. The design focuses on efficient client-side image processing with WASM, thumbnail generation, and metadata extraction whilst maintaining a small bundle size and broad browser compatibility.

### Key Objectives

**Month 4: WASM Foundation & Basic Media**

- Modular WASM architecture with code splitting
- Lazy loading for optimal initial bundle size
- Basic image metadata extraction (dimensions, format, EXIF)
- Browser compatibility layer
- Performance baseline establishment

**Month 5: Advanced Media Processing**

- High-quality thumbnail generation for JPEG/PNG/WebP
- Progressive image loading support
- Configurable quality/size options
- Bundle size ≤ 700 kB compressed
- Average thumbnail size ≤ 64 kB

## Architecture Overview

The media processing pipeline consists of three main layers:

1. **Core API Layer** - TypeScript interface integrated with FS5
2. **WASM Processing Layer** - High-performance image operations
3. **Fallback Layer** - Graceful degradation when WASM unavailable

```
┌─────────────────────────────────────────────────────────┐
│                    Application Code                      │
├─────────────────────────────────────────────────────────┤
│                  Media Processing API                    │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │  Metadata   │  │  Thumbnail   │  │  Progressive  │ │
│  │ Extraction  │  │  Generation  │  │   Loading     │ │
│  └─────────────┘  └──────────────┘  └───────────────┘ │
├─────────────────────────────────────────────────────────┤
│                   WASM Bridge Layer                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Lazy Loading | Memory Management | Threading   │   │
│  └─────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│              WASM Processing Module                      │
│  ┌───────────┐  ┌───────────┐  ┌─────────────────┐    │
│  │   libvips  │  │  mozjpeg  │  │  libwebp/sharp  │    │
│  └───────────┘  └───────────┘  └─────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## Phase 1: WASM Foundation (Month 4)

### 1.1 Module Structure

Create modular structure for WASM integration:

```typescript
// src/media/index.ts - Main entry point with lazy loading
export class MediaProcessor {
  private static wasmModule?: WASMModule;
  private static loadingPromise?: Promise<WASMModule>;

  static async initialise(): Promise<void> {
    if (this.wasmModule) return;

    if (!this.loadingPromise) {
      this.loadingPromise = this.loadWASM();
    }

    this.wasmModule = await this.loadingPromise;
  }

  private static async loadWASM(): Promise<WASMModule> {
    // Dynamic import for code splitting
    const { WASMModule } = await import(
      /* webpackChunkName: "media-wasm" */
      /* webpackPreload: false */
      "./wasm/module"
    );

    return WASMModule.initialise();
  }

  static async extractMetadata(blob: Blob): Promise<ImageMetadata | undefined> {
    try {
      await this.initialise();
      return this.wasmModule!.extractMetadata(blob);
    } catch (error) {
      // Fallback to basic extraction
      return this.basicMetadataExtraction(blob);
    }
  }

  private static async basicMetadataExtraction(
    blob: Blob
  ): Promise<ImageMetadata | undefined> {
    // Client-side Canvas API fallback
    return CanvasMetadataExtractor.extract(blob);
  }
}
```

### 1.2 WASM Module Wrapper

```typescript
// src/media/wasm/module.ts
export class WASMModule {
  private wasmInstance?: WebAssembly.Instance;
  private memory?: WebAssembly.Memory;
  private allocatedBuffers: Set<number> = new Set();

  static async initialise(): Promise<WASMModule> {
    const module = new WASMModule();

    // Load WASM binary with progress tracking
    const wasmUrl = new URL("./media-processor.wasm", import.meta.url);
    const response = await fetch(wasmUrl);

    if (!response.ok) {
      throw new Error("Failed to load WASM module");
    }

    const contentLength = response.headers.get("content-length");
    const reader = response.body!.getReader();
    const chunks: Uint8Array[] = [];
    let receivedLength = 0;

    // Stream with progress
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      chunks.push(value);
      receivedLength += value.length;

      if (contentLength) {
        const progress = (receivedLength / parseInt(contentLength)) * 100;
        module.onProgress?.(progress);
      }
    }

    const wasmBuffer = new Uint8Array(receivedLength);
    let position = 0;
    for (const chunk of chunks) {
      wasmBuffer.set(chunk, position);
      position += chunk.length;
    }

    // Initialise WASM instance
    const wasmModule = await WebAssembly.compile(wasmBuffer);

    // Create memory with initial size of 256 pages (16MB)
    module.memory = new WebAssembly.Memory({
      initial: 256,
      maximum: 4096, // 256MB max
      shared: false,
    });

    const imports = {
      env: {
        memory: module.memory,
        abort: (msg: number, file: number, line: number, col: number) => {
          console.error("WASM abort:", { msg, file, line, col });
        },
        log: (ptr: number, len: number) => {
          const msg = module.readString(ptr, len);
          console.log("WASM:", msg);
        },
      },
    };

    module.wasmInstance = await WebAssembly.instantiate(wasmModule, imports);

    // Initialise the WASM module
    const init = module.wasmInstance.exports.initialise as Function;
    init();

    return module;
  }

  async extractMetadata(blob: Blob): Promise<ImageMetadata> {
    const arrayBuffer = await blob.arrayBuffer();
    const bytes = new Uint8Array(arrayBuffer);

    // Allocate memory in WASM
    const ptr = this.allocate(bytes.length);
    this.writeMemory(ptr, bytes);

    try {
      // Call WASM function
      const metadataPtr = (
        this.wasmInstance!.exports.extract_metadata as Function
      )(ptr, bytes.length);

      if (!metadataPtr) {
        throw new Error("Failed to extract metadata");
      }

      // Read metadata from WASM memory
      return this.readMetadata(metadataPtr);
    } finally {
      // Clean up allocated memory
      this.free(ptr);
    }
  }

  private allocate(size: number): number {
    const alloc = this.wasmInstance!.exports.allocate as Function;
    const ptr = alloc(size);
    this.allocatedBuffers.add(ptr);
    return ptr;
  }

  private free(ptr: number): void {
    if (this.allocatedBuffers.has(ptr)) {
      const free = this.wasmInstance!.exports.free as Function;
      free(ptr);
      this.allocatedBuffers.delete(ptr);
    }
  }

  private writeMemory(ptr: number, data: Uint8Array): void {
    const memory = new Uint8Array(this.memory!.buffer);
    memory.set(data, ptr);
  }

  private readString(ptr: number, len: number): string {
    const memory = new Uint8Array(this.memory!.buffer);
    const bytes = memory.slice(ptr, ptr + len);
    return new TextDecoder().decode(bytes);
  }

  private readMetadata(ptr: number): ImageMetadata {
    const view = new DataView(this.memory!.buffer, ptr);

    // Read metadata structure
    const width = view.getUint32(0, true);
    const height = view.getUint32(4, true);
    const format = view.getUint8(8);
    const hasExif = view.getUint8(9) === 1;
    const exifOffset = view.getUint32(10, true);
    const exifLength = view.getUint32(14, true);

    const metadata: ImageMetadata = {
      width,
      height,
      format: this.formatCodeToString(format),
      mimeType: this.formatCodeToMimeType(format),
      fileSize: 0, // Will be set by caller
    };

    if (hasExif && exifLength > 0) {
      metadata.exif = this.readExifData(exifOffset, exifLength);
    }

    return metadata;
  }

  private readExifData(offset: number, length: number): ExifData {
    const memory = new Uint8Array(this.memory!.buffer);
    const exifBytes = memory.slice(offset, offset + length);

    // Parse EXIF data structure
    const view = new DataView(exifBytes.buffer);
    const exif: ExifData = {};

    // Read common EXIF fields
    let pos = 0;
    while (pos < length) {
      const tag = view.getUint16(pos, true);
      const type = view.getUint16(pos + 2, true);
      const count = view.getUint32(pos + 4, true);
      const valueOffset = view.getUint32(pos + 8, true);

      // Parse based on tag
      switch (tag) {
        case 0x010f: // Make
          exif.make = this.readExifString(valueOffset, count);
          break;
        case 0x0110: // Model
          exif.model = this.readExifString(valueOffset, count);
          break;
        case 0x0112: // Orientation
          exif.orientation = view.getUint16(valueOffset, true);
          break;
        case 0x9003: // DateTimeOriginal
          exif.dateTime = this.readExifString(valueOffset, count);
          break;
        case 0x829a: // ExposureTime
          exif.exposureTime =
            view.getUint32(valueOffset, true) /
            view.getUint32(valueOffset + 4, true);
          break;
        case 0x829d: // FNumber
          exif.fNumber =
            view.getUint32(valueOffset, true) /
            view.getUint32(valueOffset + 4, true);
          break;
        case 0x8827: // ISO
          exif.iso = view.getUint16(valueOffset, true);
          break;
      }

      pos += 12;
    }

    return exif;
  }

  private formatCodeToString(code: number): ImageFormat {
    switch (code) {
      case 0:
        return "jpeg";
      case 1:
        return "png";
      case 2:
        return "webp";
      case 3:
        return "gif";
      case 4:
        return "bmp";
      default:
        return "unknown";
    }
  }

  private formatCodeToMimeType(code: number): string {
    switch (code) {
      case 0:
        return "image/jpeg";
      case 1:
        return "image/png";
      case 2:
        return "image/webp";
      case 3:
        return "image/gif";
      case 4:
        return "image/bmp";
      default:
        return "application/octet-stream";
    }
  }

  // Cleanup method
  destroy(): void {
    // Free all allocated buffers
    for (const ptr of this.allocatedBuffers) {
      this.free(ptr);
    }
    this.allocatedBuffers.clear();

    // Clear references
    this.wasmInstance = undefined;
    this.memory = undefined;
  }

  onProgress?: (progress: number) => void;
}
```

### 1.3 Type Definitions

```typescript
// src/media/types.ts
export interface ImageMetadata {
  width: number;
  height: number;
  format: ImageFormat;
  mimeType: string;
  fileSize: number;
  exif?: ExifData;
  colorSpace?: "srgb" | "rgb" | "cmyk" | "gray";
  hasAlpha?: boolean;
  bitDepth?: number;
  // Histogram data for exposure analysis
  histogram?: {
    r: Uint32Array;
    g: Uint32Array;
    b: Uint32Array;
    luminance: Uint32Array;
  };
}

export interface ExifData {
  make?: string;
  model?: string;
  orientation?: number;
  dateTime?: string;
  exposureTime?: number;
  fNumber?: number;
  iso?: number;
  focalLength?: number;
  flash?: boolean;
  gps?: {
    latitude: number;
    longitude: number;
    altitude?: number;
  };
  // Additional EXIF fields as needed
  [key: string]: any;
}

export type ImageFormat = "jpeg" | "png" | "webp" | "gif" | "bmp" | "unknown";

// File extension mappings
export const IMAGE_EXTENSIONS: Record<string, ImageFormat> = {
  jpg: "jpeg",
  jpeg: "jpeg",
  png: "png",
  webp: "webp",
  gif: "gif",
  bmp: "bmp",
};

// Helper to detect format from filename
export function detectImageFormat(filename: string): ImageFormat {
  const ext = filename.toLowerCase().split(".").pop();
  return IMAGE_EXTENSIONS[ext || ""] || "unknown";
}

export interface ThumbnailOptions {
  maxWidth?: number;
  maxHeight?: number;
  quality?: number; // 0-100
  format?: "jpeg" | "webp" | "png";
  maintainAspectRatio?: boolean;
  smartCrop?: boolean; // Use face/object detection for cropping
  progressive?: boolean; // Progressive encoding
  // Target size in bytes (will adjust quality to meet target)
  targetSize?: number;
}

export interface ThumbnailResult {
  blob: Blob;
  width: number;
  height: number;
  format: string;
  quality: number; // Actual quality used
  processingTime: number; // milliseconds
}

export interface ProgressiveLoadingOptions {
  // Number of progressive scans for JPEG
  progressiveScans?: number;
  // Enable interlacing for PNG
  interlace?: boolean;
  // Quality levels for progressive loading
  qualityLevels?: number[];
}
```

### 1.4 Canvas Fallback Implementation

```typescript
// src/media/fallback/canvas.ts
export class CanvasMetadataExtractor {
  static async extract(blob: Blob): Promise<ImageMetadata | undefined> {
    return new Promise((resolve) => {
      const img = new Image();
      const url = URL.createObjectURL(blob);

      img.onload = () => {
        URL.revokeObjectURL(url);

        // Create canvas to read image data
        const canvas = document.createElement("canvas");
        const ctx = canvas.getContext("2d");

        if (!ctx) {
          resolve(undefined);
          return;
        }

        // Set canvas size to image size (limited for memory)
        const maxSize = 100; // Sample size for analysis
        const scale = Math.min(1, maxSize / Math.max(img.width, img.height));

        canvas.width = img.width * scale;
        canvas.height = img.height * scale;

        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

        // Detect format from blob type
        const format = this.detectFormat(blob.type);

        // Extract basic metadata
        const metadata: ImageMetadata = {
          width: img.width,
          height: img.height,
          format,
          mimeType: blob.type || "image/unknown",
          fileSize: blob.size,
        };

        // Try to detect transparency
        if (format === "png" || format === "webp") {
          const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
          metadata.hasAlpha = this.hasTransparency(imageData);
        }

        resolve(metadata);
      };

      img.onerror = () => {
        URL.revokeObjectURL(url);
        resolve(undefined);
      };

      img.src = url;
    });
  }

  private static detectFormat(mimeType: string): ImageFormat {
    // Handle both jpg and jpeg in MIME types
    if (mimeType.includes("jpeg") || mimeType.includes("jpg")) return "jpeg";
    if (mimeType.includes("png")) return "png";
    if (mimeType.includes("webp")) return "webp";
    if (mimeType.includes("gif")) return "gif";
    if (mimeType.includes("bmp")) return "bmp";
    return "unknown";
  }

  private static hasTransparency(imageData: ImageData): boolean {
    const data = imageData.data;
    // Check alpha channel (every 4th byte)
    for (let i = 3; i < data.length; i += 4) {
      if (data[i] < 255) return true;
    }
    return false;
  }
}
```

### 1.5 Browser Compatibility Layer

```typescript
// src/media/compat/browser.ts
export class BrowserCompat {
  private static capabilities?: BrowserCapabilities;

  static async checkCapabilities(): Promise<BrowserCapabilities> {
    if (this.capabilities) return this.capabilities;

    const caps: BrowserCapabilities = {
      webAssembly: false,
      webAssemblyStreaming: false,
      sharedArrayBuffer: false,
      webWorkers: false,
      offscreenCanvas: false,
      webP: false,
      avif: false,
      createImageBitmap: false,
      webGL: false,
      webGL2: false,
    };

    // Check WebAssembly support
    try {
      if (typeof WebAssembly === "object") {
        caps.webAssembly = true;
        caps.webAssemblyStreaming =
          typeof WebAssembly.instantiateStreaming === "function";
      }
    } catch {}

    // Check SharedArrayBuffer (may be disabled due to Spectre)
    try {
      new SharedArrayBuffer(1);
      caps.sharedArrayBuffer = true;
    } catch {}

    // Check Web Workers
    caps.webWorkers = typeof Worker !== "undefined";

    // Check OffscreenCanvas
    caps.offscreenCanvas = typeof OffscreenCanvas !== "undefined";

    // Check image format support
    caps.webP = await this.checkImageFormatSupport("image/webp");
    caps.avif = await this.checkImageFormatSupport("image/avif");

    // Check createImageBitmap
    caps.createImageBitmap = typeof createImageBitmap === "function";

    // Check WebGL support
    try {
      const canvas = document.createElement("canvas");
      caps.webGL = !!(
        canvas.getContext("webgl") || canvas.getContext("experimental-webgl")
      );
      caps.webGL2 = !!canvas.getContext("webgl2");
    } catch {}

    this.capabilities = caps;
    return caps;
  }

  private static checkImageFormatSupport(mimeType: string): Promise<boolean> {
    return new Promise((resolve) => {
      const img = new Image();
      img.onload = () => resolve(true);
      img.onerror = () => resolve(false);

      // 1x1 pixel test images
      if (mimeType === "image/webp") {
        img.src =
          "data:image/webp;base64,UklGRiIAAABXRUJQVlA4IBYAAAAwAQCdASoBAAEADsD+JaQAA3AAAAAA";
      } else if (mimeType === "image/avif") {
        img.src =
          "data:image/avif;base64,AAAAHGZ0eXBhdmlmAAAAAGF2aWZtaWYxbWlhZgAAAPBtZXRhAAAAAAAAAC...";
      }
    });
  }

  static selectProcessingStrategy(
    caps: BrowserCapabilities
  ): ProcessingStrategy {
    if (caps.webAssembly && caps.webWorkers) {
      return "wasm-worker";
    } else if (caps.webAssembly) {
      return "wasm-main";
    } else if (caps.offscreenCanvas && caps.webWorkers) {
      return "canvas-worker";
    } else {
      return "canvas-main";
    }
  }
}

interface BrowserCapabilities {
  webAssembly: boolean;
  webAssemblyStreaming: boolean;
  sharedArrayBuffer: boolean;
  webWorkers: boolean;
  offscreenCanvas: boolean;
  webP: boolean;
  avif: boolean;
  createImageBitmap: boolean;
  webGL: boolean;
  webGL2: boolean;
}

type ProcessingStrategy =
  | "wasm-worker" // Best: WASM in Web Worker
  | "wasm-main" // Good: WASM in main thread
  | "canvas-worker" // OK: Canvas in Web Worker
  | "canvas-main"; // Fallback: Canvas in main thread
```

## Phase 2: Advanced Media Processing (Month 5)

### 2.1 Thumbnail Generation Pipeline

```typescript
// src/media/thumbnail/generator.ts
export class ThumbnailGenerator {
  private static worker?: Worker;
  private static taskQueue: TaskQueue<ThumbnailTask> = new TaskQueue();

  static async generateThumbnail(
    blob: Blob,
    options: ThumbnailOptions = {}
  ): Promise<ThumbnailResult> {
    const startTime = performance.now();

    // Apply defaults
    const opts: Required<ThumbnailOptions> = {
      maxWidth: options.maxWidth ?? 256,
      maxHeight: options.maxHeight ?? 256,
      quality: options.quality ?? 85,
      format: options.format ?? "jpeg",
      maintainAspectRatio: options.maintainAspectRatio ?? true,
      smartCrop: options.smartCrop ?? false,
      progressive: options.progressive ?? true,
      targetSize: options.targetSize ?? 65536, // 64KB default
    };

    // Check capabilities and select strategy
    const caps = await BrowserCompat.checkCapabilities();
    const strategy = BrowserCompat.selectProcessingStrategy(caps);

    let result: ThumbnailResult;

    switch (strategy) {
      case "wasm-worker":
        result = await this.generateWithWASMWorker(blob, opts);
        break;
      case "wasm-main":
        result = await this.generateWithWASM(blob, opts);
        break;
      case "canvas-worker":
        result = await this.generateWithCanvasWorker(blob, opts);
        break;
      default:
        result = await this.generateWithCanvas(blob, opts);
    }

    result.processingTime = performance.now() - startTime;

    // Ensure we meet target size
    if (opts.targetSize && result.blob.size > opts.targetSize) {
      result = await this.optimiseToTargetSize(result, opts);
    }

    return result;
  }

  private static async generateWithWASMWorker(
    blob: Blob,
    options: Required<ThumbnailOptions>
  ): Promise<ThumbnailResult> {
    if (!this.worker) {
      this.worker = new Worker(
        new URL("./workers/thumbnail.worker.ts", import.meta.url),
        { type: "module" }
      );
    }

    return new Promise((resolve, reject) => {
      const task: ThumbnailTask = {
        id: crypto.randomUUID(),
        blob,
        options,
        resolve,
        reject,
      };

      this.taskQueue.add(task);
      this.processNextTask();
    });
  }

  private static async generateWithWASM(
    blob: Blob,
    options: Required<ThumbnailOptions>
  ): Promise<ThumbnailResult> {
    await MediaProcessor.initialise();

    const arrayBuffer = await blob.arrayBuffer();
    const input = new Uint8Array(arrayBuffer);

    // Get WASM module instance
    const wasm = (MediaProcessor as any).wasmModule as WASMModule;

    // Allocate memory for input
    const inputPtr = wasm.allocate(input.length);
    wasm.writeMemory(inputPtr, input);

    // Allocate memory for output (estimate max size)
    const maxOutputSize = options.maxWidth * options.maxHeight * 4;
    const outputPtr = wasm.allocate(maxOutputSize);

    try {
      // Call WASM thumbnail function
      const resultSize = (
        wasm.wasmInstance!.exports.generate_thumbnail as Function
      )(
        inputPtr,
        input.length,
        outputPtr,
        maxOutputSize,
        options.maxWidth,
        options.maxHeight,
        options.quality,
        this.formatToCode(options.format),
        options.maintainAspectRatio ? 1 : 0,
        options.smartCrop ? 1 : 0,
        options.progressive ? 1 : 0
      );

      if (resultSize <= 0) {
        throw new Error("Thumbnail generation failed");
      }

      // Read result
      const resultBytes = new Uint8Array(
        wasm.memory!.buffer,
        outputPtr,
        resultSize
      );
      const resultBlob = new Blob([resultBytes], {
        type: `image/${options.format}`,
      });

      // Read dimensions from result
      const dimensionsPtr = (
        wasm.wasmInstance!.exports.get_last_dimensions as Function
      )();
      const dimensions = new Uint32Array(wasm.memory!.buffer, dimensionsPtr, 2);

      return {
        blob: resultBlob,
        width: dimensions[0],
        height: dimensions[1],
        format: options.format,
        quality: options.quality,
        processingTime: 0, // Will be set by caller
      };
    } finally {
      wasm.free(inputPtr);
      wasm.free(outputPtr);
    }
  }

  private static async generateWithCanvas(
    blob: Blob,
    options: Required<ThumbnailOptions>
  ): Promise<ThumbnailResult> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      const url = URL.createObjectURL(blob);

      img.onload = async () => {
        URL.revokeObjectURL(url);

        try {
          // Calculate dimensions
          let { width, height } = this.calculateDimensions(
            img.width,
            img.height,
            options.maxWidth,
            options.maxHeight,
            options.maintainAspectRatio
          );

          // Create canvas
          const canvas = document.createElement("canvas");
          canvas.width = width;
          canvas.height = height;

          const ctx = canvas.getContext("2d", {
            alpha: options.format === "png",
          });

          if (!ctx) {
            throw new Error("Failed to get canvas context");
          }

          // Apply image smoothing
          ctx.imageSmoothingEnabled = true;
          ctx.imageSmoothingQuality = "high";

          // Smart crop if requested
          let sx = 0,
            sy = 0,
            sw = img.width,
            sh = img.height;
          if (options.smartCrop && !options.maintainAspectRatio) {
            const crop = await this.calculateSmartCrop(img, width, height);
            ({ sx, sy, sw, sh } = crop);
          }

          // Draw image
          ctx.drawImage(img, sx, sy, sw, sh, 0, 0, width, height);

          // Convert to blob
          const thumbnailBlob = await new Promise<Blob>((resolve, reject) => {
            canvas.toBlob(
              (blob) => {
                if (blob) resolve(blob);
                else reject(new Error("Failed to create blob"));
              },
              `image/${options.format}`,
              options.quality / 100
            );
          });

          resolve({
            blob: thumbnailBlob,
            width,
            height,
            format: options.format,
            quality: options.quality,
            processingTime: 0,
          });
        } catch (error) {
          reject(error);
        }
      };

      img.onerror = () => {
        URL.revokeObjectURL(url);
        reject(new Error("Failed to load image"));
      };

      img.src = url;
    });
  }

  private static calculateDimensions(
    srcWidth: number,
    srcHeight: number,
    maxWidth: number,
    maxHeight: number,
    maintainAspectRatio: boolean
  ): { width: number; height: number } {
    if (!maintainAspectRatio) {
      return { width: maxWidth, height: maxHeight };
    }

    const aspectRatio = srcWidth / srcHeight;
    let width = maxWidth;
    let height = maxHeight;

    if (width / height > aspectRatio) {
      width = height * aspectRatio;
    } else {
      height = width / aspectRatio;
    }

    return {
      width: Math.round(width),
      height: Math.round(height),
    };
  }

  private static async calculateSmartCrop(
    img: HTMLImageElement,
    targetWidth: number,
    targetHeight: number
  ): Promise<{ sx: number; sy: number; sw: number; sh: number }> {
    // Simple edge detection based smart crop
    // In production, this could use face detection or saliency detection

    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d")!;

    // Sample the image at lower resolution for performance
    const sampleSize = 100;
    canvas.width = sampleSize;
    canvas.height = sampleSize;

    ctx.drawImage(img, 0, 0, sampleSize, sampleSize);
    const imageData = ctx.getImageData(0, 0, sampleSize, sampleSize);

    // Calculate energy map using edge detection
    const energyMap = this.calculateEnergyMap(imageData);

    // Find region with highest energy
    const targetAspect = targetWidth / targetHeight;
    const region = this.findBestRegion(energyMap, targetAspect);

    // Scale back to original dimensions
    const scale = img.width / sampleSize;

    return {
      sx: region.x * scale,
      sy: region.y * scale,
      sw: region.width * scale,
      sh: region.height * scale,
    };
  }

  private static calculateEnergyMap(imageData: ImageData): Float32Array {
    const { width, height, data } = imageData;
    const energy = new Float32Array(width * height);

    // Sobel edge detection
    for (let y = 1; y < height - 1; y++) {
      for (let x = 1; x < width - 1; x++) {
        const idx = y * width + x;

        // Calculate gradients
        let gx = 0,
          gy = 0;

        for (let dy = -1; dy <= 1; dy++) {
          for (let dx = -1; dx <= 1; dx++) {
            const nIdx = (y + dy) * width + (x + dx);
            const pixel = data[nIdx * 4]; // Use red channel

            gx += pixel * SOBEL_X[dy + 1][dx + 1];
            gy += pixel * SOBEL_Y[dy + 1][dx + 1];
          }
        }

        energy[idx] = Math.sqrt(gx * gx + gy * gy);
      }
    }

    return energy;
  }

  private static findBestRegion(
    energyMap: Float32Array,
    targetAspect: number
  ): { x: number; y: number; width: number; height: number } {
    const size = Math.sqrt(energyMap.length);
    let bestRegion = { x: 0, y: 0, width: size, height: size };
    let maxEnergy = -Infinity;

    // Try different region sizes
    for (let h = size * 0.5; h <= size; h += size * 0.1) {
      const w = h * targetAspect;
      if (w > size) continue;

      // Slide window across image
      for (let y = 0; y <= size - h; y += size * 0.05) {
        for (let x = 0; x <= size - w; x += size * 0.05) {
          // Calculate total energy in region
          let energy = 0;
          for (let dy = 0; dy < h; dy++) {
            for (let dx = 0; dx < w; dx++) {
              const idx = Math.floor(y + dy) * size + Math.floor(x + dx);
              energy += energyMap[idx];
            }
          }

          if (energy > maxEnergy) {
            maxEnergy = energy;
            bestRegion = { x, y, width: w, height: h };
          }
        }
      }
    }

    return bestRegion;
  }

  private static async optimiseToTargetSize(
    result: ThumbnailResult,
    options: Required<ThumbnailOptions>
  ): Promise<ThumbnailResult> {
    let quality = result.quality;
    let blob = result.blob;

    // Binary search for optimal quality
    let minQuality = 10;
    let maxQuality = quality;

    while (maxQuality - minQuality > 5) {
      const midQuality = Math.floor((minQuality + maxQuality) / 2);

      // Re-encode with new quality
      const tempResult = await this.reencodeWithQuality(
        blob,
        midQuality,
        options.format
      );

      if (tempResult.size <= options.targetSize) {
        minQuality = midQuality;
        blob = tempResult;
        quality = midQuality;
      } else {
        maxQuality = midQuality;
      }
    }

    return {
      ...result,
      blob,
      quality,
    };
  }

  private static async reencodeWithQuality(
    blob: Blob,
    quality: number,
    format: string
  ): Promise<Blob> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      const url = URL.createObjectURL(blob);

      img.onload = () => {
        URL.revokeObjectURL(url);

        const canvas = document.createElement("canvas");
        canvas.width = img.width;
        canvas.height = img.height;

        const ctx = canvas.getContext("2d")!;
        ctx.drawImage(img, 0, 0);

        canvas.toBlob(
          (blob) => {
            if (blob) resolve(blob);
            else reject(new Error("Failed to re-encode"));
          },
          `image/${format}`,
          quality / 100
        );
      };

      img.onerror = () => {
        URL.revokeObjectURL(url);
        reject(new Error("Failed to load image for re-encoding"));
      };

      img.src = url;
    });
  }

  private static formatToCode(format: string): number {
    switch (format) {
      case "jpeg":
        return 0;
      case "png":
        return 1;
      case "webp":
        return 2;
      default:
        return 0;
    }
  }

  private static processNextTask(): void {
    const task = this.taskQueue.next();
    if (!task || !this.worker) return;

    const channel = new MessageChannel();

    channel.port1.onmessage = (event) => {
      if (event.data.error) {
        task.reject(new Error(event.data.error));
      } else {
        task.resolve(event.data.result);
      }

      this.processNextTask();
    };

    this.worker.postMessage(
      {
        id: task.id,
        blob: task.blob,
        options: task.options,
      },
      [channel.port2]
    );
  }
}

// Sobel operators for edge detection
const SOBEL_X = [
  [-1, 0, 1],
  [-2, 0, 2],
  [-1, 0, 1],
];

const SOBEL_Y = [
  [-1, -2, -1],
  [0, 0, 0],
  [1, 2, 1],
];

interface ThumbnailTask {
  id: string;
  blob: Blob;
  options: Required<ThumbnailOptions>;
  resolve: (result: ThumbnailResult) => void;
  reject: (error: Error) => void;
}

class TaskQueue<T> {
  private queue: T[] = [];

  add(task: T): void {
    this.queue.push(task);
  }

  next(): T | undefined {
    return this.queue.shift();
  }

  get length(): number {
    return this.queue.length;
  }
}
```

### 2.2 Progressive Loading Implementation

```typescript
// src/media/progressive/loader.ts
export class ProgressiveImageLoader {
  static async createProgressive(
    blob: Blob,
    options: ProgressiveLoadingOptions = {}
  ): Promise<ProgressiveImage> {
    const format = await this.detectFormat(blob);

    switch (format) {
      case "jpeg":
        return this.createProgressiveJPEG(blob, options);
      case "png":
        return this.createProgressivePNG(blob, options);
      case "webp":
        return this.createProgressiveWebP(blob, options);
      default:
        throw new Error(
          `Unsupported format for progressive loading: ${format}`
        );
    }
  }

  private static async createProgressiveJPEG(
    blob: Blob,
    options: ProgressiveLoadingOptions
  ): Promise<ProgressiveImage> {
    await MediaProcessor.initialise();

    const scans = options.progressiveScans ?? 3;
    const qualityLevels = options.qualityLevels ?? [20, 50, 85];

    const arrayBuffer = await blob.arrayBuffer();
    const input = new Uint8Array(arrayBuffer);

    const wasm = (MediaProcessor as any).wasmModule as WASMModule;

    // Allocate memory
    const inputPtr = wasm.allocate(input.length);
    wasm.writeMemory(inputPtr, input);

    const layers: ProgressiveLayer[] = [];

    try {
      for (let i = 0; i < scans; i++) {
        const quality = qualityLevels[i] ?? 85;
        const isBaseline = i === 0;

        // Allocate output buffer
        const outputSize = input.length; // Worst case
        const outputPtr = wasm.allocate(outputSize);

        const resultSize = (
          wasm.wasmInstance!.exports.create_progressive_jpeg as Function
        )(
          inputPtr,
          input.length,
          outputPtr,
          outputSize,
          quality,
          i, // scan number
          scans, // total scans
          isBaseline ? 1 : 0
        );

        if (resultSize > 0) {
          const layerData = new Uint8Array(
            wasm.memory!.buffer,
            outputPtr,
            resultSize
          );

          layers.push({
            data: new Uint8Array(layerData),
            quality,
            isBaseline,
            scanNumber: i,
          });
        }

        wasm.free(outputPtr);
      }
    } finally {
      wasm.free(inputPtr);
    }

    return new ProgressiveJPEG(layers);
  }

  private static async createProgressivePNG(
    blob: Blob,
    options: ProgressiveLoadingOptions
  ): Promise<ProgressiveImage> {
    // PNG uses Adam7 interlacing
    const interlace = options.interlace ?? true;

    if (!interlace) {
      // Return non-interlaced PNG as single layer
      return new ProgressivePNG([
        {
          data: new Uint8Array(await blob.arrayBuffer()),
          quality: 100,
          isBaseline: true,
          scanNumber: 0,
        },
      ]);
    }

    // Create interlaced PNG
    await MediaProcessor.initialise();

    const arrayBuffer = await blob.arrayBuffer();
    const input = new Uint8Array(arrayBuffer);

    const wasm = (MediaProcessor as any).wasmModule as WASMModule;

    const inputPtr = wasm.allocate(input.length);
    wasm.writeMemory(inputPtr, input);

    const outputSize = input.length * 1.5; // Allow for some expansion
    const outputPtr = wasm.allocate(outputSize);

    try {
      const resultSize = (
        wasm.wasmInstance!.exports.create_interlaced_png as Function
      )(inputPtr, input.length, outputPtr, outputSize);

      if (resultSize > 0) {
        const interlacedData = new Uint8Array(
          wasm.memory!.buffer,
          outputPtr,
          resultSize
        );

        // PNG interlacing creates 7 passes
        return new ProgressivePNG([
          {
            data: new Uint8Array(interlacedData),
            quality: 100,
            isBaseline: true,
            scanNumber: 0,
          },
        ]);
      }

      throw new Error("Failed to create interlaced PNG");
    } finally {
      wasm.free(inputPtr);
      wasm.free(outputPtr);
    }
  }

  private static async createProgressiveWebP(
    blob: Blob,
    options: ProgressiveLoadingOptions
  ): Promise<ProgressiveImage> {
    // WebP doesn't have true progressive loading like JPEG
    // We'll create multiple quality versions instead
    const qualityLevels = options.qualityLevels ?? [30, 60, 90];
    const layers: ProgressiveLayer[] = [];

    for (let i = 0; i < qualityLevels.length; i++) {
      const quality = qualityLevels[i];

      const result = await ThumbnailGenerator.generateThumbnail(blob, {
        quality,
        format: "webp",
        maxWidth: Infinity,
        maxHeight: Infinity,
      });

      layers.push({
        data: new Uint8Array(await result.blob.arrayBuffer()),
        quality,
        isBaseline: i === 0,
        scanNumber: i,
      });
    }

    return new ProgressiveWebP(layers);
  }

  private static async detectFormat(blob: Blob): Promise<ImageFormat> {
    const header = new Uint8Array(await blob.slice(0, 16).arrayBuffer());

    // JPEG: FF D8 FF
    if (header[0] === 0xff && header[1] === 0xd8 && header[2] === 0xff) {
      return "jpeg";
    }

    // PNG: 89 50 4E 47 0D 0A 1A 0A
    if (
      header[0] === 0x89 &&
      header[1] === 0x50 &&
      header[2] === 0x4e &&
      header[3] === 0x47
    ) {
      return "png";
    }

    // WebP: RIFF....WEBP
    if (
      header[0] === 0x52 &&
      header[1] === 0x49 &&
      header[2] === 0x46 &&
      header[3] === 0x46 &&
      header[8] === 0x57 &&
      header[9] === 0x45 &&
      header[10] === 0x42 &&
      header[11] === 0x50
    ) {
      return "webp";
    }

    return "unknown";
  }
}

interface ProgressiveLayer {
  data: Uint8Array;
  quality: number;
  isBaseline: boolean;
  scanNumber: number;
}

abstract class ProgressiveImage {
  constructor(protected layers: ProgressiveLayer[]) {}

  abstract getLayer(index: number): ProgressiveLayer | undefined;
  abstract get layerCount(): number;
  abstract toBlob(): Blob;

  getAllLayers(): ProgressiveLayer[] {
    return this.layers;
  }
}

class ProgressiveJPEG extends ProgressiveImage {
  getLayer(index: number): ProgressiveLayer | undefined {
    return this.layers[index];
  }

  get layerCount(): number {
    return this.layers.length;
  }

  toBlob(): Blob {
    // Combine all layers for final image
    const totalSize = this.layers.reduce(
      (sum, layer) => sum + layer.data.length,
      0
    );
    const combined = new Uint8Array(totalSize);

    let offset = 0;
    for (const layer of this.layers) {
      combined.set(layer.data, offset);
      offset += layer.data.length;
    }

    return new Blob([combined], { type: "image/jpeg" });
  }
}

class ProgressivePNG extends ProgressiveImage {
  getLayer(index: number): ProgressiveLayer | undefined {
    // PNG interlacing is handled internally
    return index === 0 ? this.layers[0] : undefined;
  }

  get layerCount(): number {
    return 1; // PNG progressive is a single interlaced file
  }

  toBlob(): Blob {
    return new Blob([this.layers[0].data], { type: "image/png" });
  }
}

class ProgressiveWebP extends ProgressiveImage {
  getLayer(index: number): ProgressiveLayer | undefined {
    return this.layers[index];
  }

  get layerCount(): number {
    return this.layers.length;
  }

  toBlob(): Blob {
    // Return highest quality version
    const bestLayer = this.layers[this.layers.length - 1];
    return new Blob([bestLayer.data], { type: "image/webp" });
  }
}
```

### 2.3 Integration with FS5 Path API

```typescript
// src/fs/media-extensions.ts
import { FS5 } from "./fs5";
import { MediaProcessor } from "../media";
import {
  ThumbnailGenerator,
  ThumbnailOptions,
} from "../media/thumbnail/generator";
import { ProgressiveImageLoader } from "../media/progressive/loader";

// Extend FS5 class with media capabilities
declare module "./fs5" {
  interface FS5 {
    putImage(
      path: string,
      blob: Blob,
      options?: PutImageOptions
    ): Promise<ImageReference>;
    getThumbnail(
      path: string,
      options?: ThumbnailOptions
    ): Promise<Blob | undefined>;
    getImageMetadata(path: string): Promise<ImageMetadata | undefined>;
    createImageGallery(path: string, images: ImageUpload[]): Promise<void>;
  }
}

interface PutImageOptions extends PutOptions {
  generateThumbnail?: boolean;
  thumbnailOptions?: ThumbnailOptions;
  extractMetadata?: boolean;
  progressive?: boolean;
}

interface ImageReference {
  path: string;
  cid: string;
  thumbnailPath?: string;
  thumbnailCid?: string;
  metadata?: ImageMetadata;
}

interface ImageUpload {
  name: string;
  blob: Blob;
  metadata?: Record<string, any>;
}

// Implementation
FS5.prototype.putImage = async function (
  path: string,
  blob: Blob,
  options: PutImageOptions = {}
): Promise<ImageReference> {
  const result: ImageReference = {
    path,
    cid: "", // Will be set after upload
  };

  // Extract metadata if requested
  if (options.extractMetadata !== false) {
    try {
      const metadata = await MediaProcessor.extractMetadata(blob);
      if (metadata) {
        result.metadata = metadata;
        options.metadata = {
          ...options.metadata,
          image: {
            width: metadata.width,
            height: metadata.height,
            format: metadata.format,
            colorSpace: metadata.colorSpace,
            hasAlpha: metadata.hasAlpha,
            exif: metadata.exif,
          },
        };
      }
    } catch (error) {
      console.warn("Failed to extract image metadata:", error);
    }
  }

  // Generate thumbnail if requested
  if (options.generateThumbnail !== false) {
    try {
      const thumbnailResult = await ThumbnailGenerator.generateThumbnail(
        blob,
        options.thumbnailOptions
      );

      // Support both .jpg and .jpeg extensions
      const thumbnailPath = path.replace(
        /\.(jpg|jpeg|png|webp|gif|bmp)$/i,
        "_thumb.jpg"
      );

      await this.put(thumbnailPath, thumbnailResult.blob, {
        mediaType: `image/${thumbnailResult.format}`,
        metadata: {
          isThumbnail: true,
          originalPath: path,
          dimensions: {
            width: thumbnailResult.width,
            height: thumbnailResult.height,
          },
          quality: thumbnailResult.quality,
          processingTime: thumbnailResult.processingTime,
        },
      });

      result.thumbnailPath = thumbnailPath;
    } catch (error) {
      console.warn("Failed to generate thumbnail:", error);
    }
  }

  // Handle progressive encoding
  if (options.progressive) {
    try {
      const progressive = await ProgressiveImageLoader.createProgressive(blob);
      // Store progressive layers information in metadata
      options.metadata = {
        ...options.metadata,
        progressive: {
          layerCount: progressive.layerCount,
          format: result.metadata?.format,
        },
      };
    } catch (error) {
      console.warn("Failed to create progressive image:", error);
    }
  }

  // Store the main image
  await this.put(path, blob, {
    ...options,
    mediaType: options.mediaType || blob.type,
  });

  return result;
};

FS5.prototype.getThumbnail = async function (
  path: string,
  options?: ThumbnailOptions
): Promise<Blob | undefined> {
  // First, check if a pre-generated thumbnail exists
  const thumbnailPath = path.replace(
    /\.(jpg|jpeg|png|webp|gif|bmp)$/i,
    "_thumb.jpg"
  );

  try {
    const thumbnailData = await this.get(thumbnailPath);
    if (thumbnailData instanceof Blob) {
      return thumbnailData;
    }
    if (thumbnailData instanceof ArrayBuffer) {
      return new Blob([thumbnailData], { type: "image/jpeg" });
    }
  } catch {
    // Thumbnail doesn't exist, try to generate on-demand
  }

  // Get the original image
  const imageData = await this.get(path);
  if (!imageData) return undefined;

  let blob: Blob;
  if (imageData instanceof Blob) {
    blob = imageData;
  } else if (imageData instanceof ArrayBuffer) {
    blob = new Blob([imageData]);
  } else {
    return undefined;
  }

  // Generate thumbnail on-demand
  try {
    const result = await ThumbnailGenerator.generateThumbnail(blob, options);

    // Optionally cache the generated thumbnail
    await this.put(thumbnailPath, result.blob, {
      mediaType: `image/${result.format}`,
      metadata: {
        isThumbnail: true,
        originalPath: path,
        generatedOnDemand: true,
      },
    });

    return result.blob;
  } catch (error) {
    console.error("Failed to generate thumbnail:", error);
    return undefined;
  }
};

FS5.prototype.getImageMetadata = async function (
  path: string
): Promise<ImageMetadata | undefined> {
  // First check if metadata is stored
  const fileMetadata = await this.getMetadata(path);
  if (fileMetadata?.custom?.image) {
    return fileMetadata.custom.image as ImageMetadata;
  }

  // Otherwise, extract from image
  const imageData = await this.get(path);
  if (!imageData) return undefined;

  let blob: Blob;
  if (imageData instanceof Blob) {
    blob = imageData;
  } else if (imageData instanceof ArrayBuffer) {
    blob = new Blob([imageData]);
  } else {
    return undefined;
  }

  try {
    return await MediaProcessor.extractMetadata(blob);
  } catch (error) {
    console.error("Failed to extract metadata:", error);
    return undefined;
  }
};

FS5.prototype.createImageGallery = async function (
  galleryPath: string,
  images: ImageUpload[]
): Promise<void> {
  // Create gallery directory
  const segments = galleryPath.split("/").filter((s) => s);
  const galleryName = segments.pop() || "gallery";
  const parentPath = segments.join("/") || "home";

  await this.createDirectory(parentPath, galleryName);

  // Process images in parallel with concurrency limit
  const concurrency = 4;
  const queue = [...images];
  const processing: Promise<void>[] = [];

  const processImage = async (image: ImageUpload) => {
    const imagePath = `${galleryPath}/${image.name}`;

    await this.putImage(imagePath, image.blob, {
      generateThumbnail: true,
      extractMetadata: true,
      progressive: true,
      metadata: image.metadata,
      thumbnailOptions: {
        maxWidth: 256,
        maxHeight: 256,
        quality: 85,
        smartCrop: true,
      },
    });
  };

  while (queue.length > 0 || processing.length > 0) {
    // Start new tasks up to concurrency limit
    while (processing.length < concurrency && queue.length > 0) {
      const image = queue.shift()!;
      const promise = processImage(image).then(() => {
        // Remove from processing array when done
        const index = processing.indexOf(promise);
        if (index > -1) processing.splice(index, 1);
      });
      processing.push(promise);
    }

    // Wait for at least one to complete
    if (processing.length > 0) {
      await Promise.race(processing);
    }
  }

  // Create gallery manifest
  await this.put(`${galleryPath}/manifest.json`, {
    type: "image-gallery",
    created: new Date().toISOString(),
    imageCount: images.length,
    version: "1.0",
  });
};
```

### 2.4 Web Hosting Support

```typescript
// src/fs/web-hosting.ts
export class WebHostingResolver {
  constructor(private fs: FS5) {}

  async resolveWebPath(path: string): Promise<WebResource | undefined> {
    // Normalise path
    const normalised = path.endsWith("/") ? path : path + "/";

    // Try exact file first
    const exactFile = await this.fs.get(path);
    if (exactFile !== undefined) {
      return {
        data: exactFile,
        mimeType: this.guessMimeType(path),
        isDefaultFile: false,
      };
    }

    // Try with common extensions
    const extensions = [".html", ".htm", ".json", ".js", ".css"];
    for (const ext of extensions) {
      const withExt = await this.fs.get(path + ext);
      if (withExt !== undefined) {
        return {
          data: withExt,
          mimeType: this.guessMimeType(path + ext),
          isDefaultFile: false,
        };
      }
    }

    // Try default files in directory
    const defaultFiles = ["index.html", "index.htm", "default.html"];
    for (const defaultFile of defaultFiles) {
      const defaultPath = normalised + defaultFile;
      const data = await this.fs.get(defaultPath);
      if (data !== undefined) {
        return {
          data,
          mimeType: "text/html",
          isDefaultFile: true,
          resolvedPath: defaultPath,
        };
      }
    }

    // Check if it's a directory and generate listing
    const metadata = await this.fs.getMetadata(path);
    if (metadata?.type === "directory") {
      return {
        data: await this.generateDirectoryListing(path),
        mimeType: "text/html",
        isDirectoryListing: true,
      };
    }

    return undefined;
  }

  private async generateDirectoryListing(path: string): Promise<string> {
    const items: string[] = [];

    for await (const item of this.fs.list(path)) {
      const icon = item.type === "directory" ? "📁" : "📄";
      const href = item.type === "directory" ? `${item.name}/` : item.name;

      items.push(`<li>${icon} <a href="${href}">${item.name}</a></li>`);
    }

    return `
      <!DOCTYPE html>
      <html>
      <head>
        <title>Directory: ${path}</title>
        <meta charset="utf-8">
        <style>
          body { font-family: system-ui, sans-serif; margin: 2em; }
          ul { list-style: none; }
          li { margin: 0.5em 0; }
          a { text-decoration: none; color: #0066cc; }
          a:hover { text-decoration: underline; }
        </style>
      </head>
      <body>
        <h1>Directory: ${path}</h1>
        <ul>${items.join("\n")}</ul>
      </body>
      </html>
    `;
  }

  private guessMimeType(path: string): string {
    const ext = path.toLowerCase().split(".").pop();
    const mimeTypes: Record<string, string> = {
      html: "text/html",
      htm: "text/html",
      css: "text/css",
      js: "application/javascript",
      json: "application/json",
      jpg: "image/jpeg",
      jpeg: "image/jpeg",
      png: "image/png",
      gif: "image/gif",
      webp: "image/webp",
      svg: "image/svg+xml",
      pdf: "application/pdf",
      txt: "text/plain",
      md: "text/markdown",
    };

    return mimeTypes[ext || ""] || "application/octet-stream";
  }
}

interface WebResource {
  data: any;
  mimeType: string;
  isDefaultFile?: boolean;
  isDirectoryListing?: boolean;
  resolvedPath?: string;
}
```

### 2.5 Path-Based API Updates for Default File Resolution

Update the FS5 class to support default file resolution:

```typescript
// Additions to src/fs/fs5.ts

// Default file resolution configuration
interface DefaultFileConfig {
  enabled: boolean;
  filenames: string[];
  extensions?: string[];
}

const DEFAULT_FILE_CONFIG: DefaultFileConfig = {
  enabled: true,
  filenames: ['index.html', 'index.htm', 'default.html'],
  extensions: ['.html', '.htm'],
};

// Add GetOptions interface
interface GetOptions {
  resolveDefaults?: boolean;
}

// Modify the get method to support default file resolution
async get(path: string, options?: GetOptions): Promise<any | undefined> {
  const { directory, fileName } = await this._resolvePath(path);

  if (!fileName) {
    // Path points to a directory - try default file resolution
    if (options?.resolveDefaults !== false && DEFAULT_FILE_CONFIG.enabled) {
      return this._resolveDefaultFile(directory, path);
    }
    // Return directory metadata if no default file
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
      return JSON.parse(new TextDecoder().decode(data));
    } catch {
      return new TextDecoder().decode(data);
    }
  }
}

// New method for default file resolution
private async _resolveDefaultFile(
  directory: DirV1,
  directoryPath: string
): Promise<any | undefined> {
  // Try each default filename in order
  for (const filename of DEFAULT_FILE_CONFIG.filenames) {
    const file = await this._getFileFromDirectory(directory, filename);
    if (file) {
      // Found default file - load it
      const data = await this.api.downloadBlobAsBytes(file.hash);
      try {
        return new TextDecoder().decode(data);
      } catch {
        return data; // Return raw data if not text
      }
    }
  }

  // No default file found
  return undefined;
}
```

## Phase 3: Performance Monitoring & Testing

### 3.1 Performance Baseline

```typescript
// src/media/performance/monitor.ts
export class PerformanceMonitor {
  private static metrics: Map<string, PerformanceMetric[]> = new Map();

  static startOperation(name: string): OperationTimer {
    const start = performance.now();
    const startMemory = this.getMemoryUsage();

    return {
      end: (metadata?: Record<string, any>) => {
        const duration = performance.now() - start;
        const endMemory = this.getMemoryUsage();

        const metric: PerformanceMetric = {
          name,
          duration,
          timestamp: Date.now(),
          memoryDelta: endMemory - startMemory,
          metadata,
        };

        this.recordMetric(name, metric);
        return metric;
      },
    };
  }

  private static recordMetric(name: string, metric: PerformanceMetric): void {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }

    const metrics = this.metrics.get(name)!;
    metrics.push(metric);

    // Keep only last 1000 metrics per operation
    if (metrics.length > 1000) {
      metrics.shift();
    }
  }

  static getStats(name: string): PerformanceStats | undefined {
    const metrics = this.metrics.get(name);
    if (!metrics || metrics.length === 0) return undefined;

    const durations = metrics.map((m) => m.duration);
    const sorted = [...durations].sort((a, b) => a - b);

    return {
      count: metrics.length,
      mean: durations.reduce((a, b) => a + b, 0) / durations.length,
      median: sorted[Math.floor(sorted.length / 2)],
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)],
      min: sorted[0],
      max: sorted[sorted.length - 1],
    };
  }

  static generateReport(): PerformanceReport {
    const report: PerformanceReport = {
      timestamp: Date.now(),
      operations: {},
    };

    for (const [name, _] of this.metrics) {
      const stats = this.getStats(name);
      if (stats) {
        report.operations[name] = stats;
      }
    }

    return report;
  }

  private static getMemoryUsage(): number {
    if ("memory" in performance) {
      return (performance as any).memory.usedJSHeapSize;
    }
    return 0;
  }
}

interface OperationTimer {
  end: (metadata?: Record<string, any>) => PerformanceMetric;
}

interface PerformanceMetric {
  name: string;
  duration: number;
  timestamp: number;
  memoryDelta: number;
  metadata?: Record<string, any>;
}

interface PerformanceStats {
  count: number;
  mean: number;
  median: number;
  p95: number;
  p99: number;
  min: number;
  max: number;
}

interface PerformanceReport {
  timestamp: number;
  operations: Record<string, PerformanceStats>;
}
```

### 3.2 Browser Test Matrix

```typescript
// src/media/testing/browser-tests.ts
export class BrowserTestSuite {
  static async runCompatibilityTests(): Promise<TestReport> {
    const tests: Test[] = [
      this.testWebAssemblySupport(),
      this.testImageFormatSupport(),
      this.testCanvasOperations(),
      this.testWorkerSupport(),
      this.testMemoryLimits(),
      this.testConcurrency(),
    ];

    const results = await Promise.all(tests.map((test) => this.runTest(test)));

    return {
      browser: this.getBrowserInfo(),
      timestamp: Date.now(),
      results,
      passed: results.every((r) => r.passed),
    };
  }

  private static async runTest(test: Test): Promise<TestResult> {
    const timer = PerformanceMonitor.startOperation(`test_${test.name}`);

    try {
      await test.fn();
      const metric = timer.end({ status: "passed" });

      return {
        name: test.name,
        passed: true,
        duration: metric.duration,
      };
    } catch (error) {
      const metric = timer.end({ status: "failed", error: String(error) });

      return {
        name: test.name,
        passed: false,
        duration: metric.duration,
        error: String(error),
      };
    }
  }

  private static testWebAssemblySupport(): Test {
    return {
      name: "WebAssembly Support",
      fn: async () => {
        // Test basic WebAssembly
        const wasmCode = new Uint8Array([
          0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00,
        ]);

        const module = await WebAssembly.compile(wasmCode);
        const instance = await WebAssembly.instantiate(module);

        // Test streaming compilation
        const response = new Response(wasmCode, {
          headers: { "Content-Type": "application/wasm" },
        });

        await WebAssembly.instantiateStreaming(response);
      },
    };
  }

  private static testImageFormatSupport(): Test {
    return {
      name: "Image Format Support",
      fn: async () => {
        const formats = ["webp", "avif"];
        const supported: Record<string, boolean> = {};

        for (const format of formats) {
          supported[format] = await this.checkImageFormat(format);
        }

        // WebP should be supported in modern browsers
        if (!supported.webp) {
          throw new Error("WebP not supported");
        }
      },
    };
  }

  private static testCanvasOperations(): Test {
    return {
      name: "Canvas Operations",
      fn: async () => {
        // Test 2D canvas
        const canvas = document.createElement("canvas");
        canvas.width = 1024;
        canvas.height = 1024;

        const ctx = canvas.getContext("2d");
        if (!ctx) throw new Error("2D context not available");

        // Test image smoothing
        if (!("imageSmoothingEnabled" in ctx)) {
          throw new Error("Image smoothing not supported");
        }

        // Test toBlob
        await new Promise<void>((resolve, reject) => {
          canvas.toBlob(
            (blob) => {
              if (blob) resolve();
              else reject(new Error("toBlob failed"));
            },
            "image/jpeg",
            0.9
          );
        });

        // Test OffscreenCanvas if available
        if (typeof OffscreenCanvas !== "undefined") {
          const offscreen = new OffscreenCanvas(256, 256);
          const offCtx = offscreen.getContext("2d");
          if (!offCtx) throw new Error("OffscreenCanvas 2D context failed");
        }
      },
    };
  }

  private static testWorkerSupport(): Test {
    return {
      name: "Worker Support",
      fn: async () => {
        // Test basic worker
        const workerCode = `
          self.addEventListener('message', (e) => {
            self.postMessage({ received: e.data });
          });
        `;

        const blob = new Blob([workerCode], { type: "application/javascript" });
        const worker = new Worker(URL.createObjectURL(blob));

        await new Promise<void>((resolve, reject) => {
          worker.onmessage = () => resolve();
          worker.onerror = reject;
          worker.postMessage("test");
        });

        worker.terminate();
      },
    };
  }

  private static testMemoryLimits(): Test {
    return {
      name: "Memory Limits",
      fn: async () => {
        // Try to allocate 100MB
        const size = 100 * 1024 * 1024;
        const buffer = new ArrayBuffer(size);

        // Verify allocation
        if (buffer.byteLength !== size) {
          throw new Error("Failed to allocate 100MB");
        }

        // Test transferable objects
        const channel = new MessageChannel();
        channel.port1.postMessage(buffer, [buffer]);

        // Buffer should be detached
        if (buffer.byteLength !== 0) {
          throw new Error("Transferable objects not working correctly");
        }
      },
    };
  }

  private static testConcurrency(): Test {
    return {
      name: "Concurrency",
      fn: async () => {
        const tasks = 10;
        const promises: Promise<number>[] = [];

        for (let i = 0; i < tasks; i++) {
          promises.push(
            new Promise((resolve) => {
              setTimeout(() => resolve(i), Math.random() * 100);
            })
          );
        }

        const results = await Promise.all(promises);

        if (results.length !== tasks) {
          throw new Error("Promise.all failed");
        }
      },
    };
  }

  private static async checkImageFormat(format: string): Promise<boolean> {
    return new Promise((resolve) => {
      const img = new Image();
      img.onload = () => resolve(true);
      img.onerror = () => resolve(false);

      // Test images for different formats
      const testImages: Record<string, string> = {
        webp: "data:image/webp;base64,UklGRiIAAABXRUJQVlA4IBYAAAAwAQCdASoBAAEADsD+JaQAA3AAAAAA",
        avif: "data:image/avif;base64,AAAAHGZ0eXBhdmlmAAAAAGF2aWZtaWYxbWlhZgAAAPBtZXRhAAAAAAAAAC...",
      };

      img.src = testImages[format] || "";
    });
  }

  private static getBrowserInfo(): BrowserInfo {
    const ua = navigator.userAgent;
    const browser: BrowserInfo = {
      userAgent: ua,
      vendor: navigator.vendor,
      platform: navigator.platform,
      cores: navigator.hardwareConcurrency || 1,
    };

    // Detect browser type
    if (ua.includes("Chrome")) browser.name = "Chrome";
    else if (ua.includes("Firefox")) browser.name = "Firefox";
    else if (ua.includes("Safari")) browser.name = "Safari";
    else if (ua.includes("Edge")) browser.name = "Edge";
    else browser.name = "Unknown";

    // Extract version
    const versionMatch = ua.match(new RegExp(`${browser.name}/(\\d+)`));
    if (versionMatch) {
      browser.version = parseInt(versionMatch[1]);
    }

    return browser;
  }
}

interface Test {
  name: string;
  fn: () => Promise<void>;
}

interface TestResult {
  name: string;
  passed: boolean;
  duration: number;
  error?: string;
}

interface TestReport {
  browser: BrowserInfo;
  timestamp: number;
  results: TestResult[];
  passed: boolean;
}

interface BrowserInfo {
  name?: string;
  version?: number;
  userAgent: string;
  vendor: string;
  platform: string;
  cores: number;
}
```

## Phase 4: Bundle Optimisation

### 4.1 Webpack Configuration

```javascript
// webpack.config.js
const path = require("path");
const TerserPlugin = require("terser-webpack-plugin");
const CompressionPlugin = require("compression-webpack-plugin");
const BundleAnalyzerPlugin =
  require("webpack-bundle-analyzer").BundleAnalyzerPlugin;
const WasmPackPlugin = require("@wasm-tool/wasm-pack-plugin");

module.exports = {
  entry: "./src/index.ts",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].js",
    chunkFilename: "[name].[contenthash].js",
    library: "S5",
    libraryTarget: "umd",
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
      {
        test: /\.wasm$/,
        type: "asset/resource",
        generator: {
          filename: "wasm/[name].[hash][ext]",
        },
      },
    ],
  },
  resolve: {
    extensions: [".tsx", ".ts", ".js", ".wasm"],
  },
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
            passes: 2,
          },
          mangle: {
            safari10: true,
          },
          format: {
            comments: false,
          },
        },
        extractComments: false,
      }),
    ],
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        media: {
          test: /[\\/]src[\\/]media[\\/]/,
          name: "media",
          priority: 10,
          reuseExistingChunk: true,
        },
        wasm: {
          test: /\.wasm$/,
          name: "media-wasm",
          priority: 20,
          enforce: true,
        },
      },
    },
  },
  plugins: [
    new WasmPackPlugin({
      crateDirectory: path.resolve(__dirname, "wasm-src"),
      outDir: path.resolve(__dirname, "src/media/wasm"),
      outName: "media-processor",
    }),
    new CompressionPlugin({
      test: /\.(js|wasm)$/,
      algorithm: "brotli",
      compressionOptions: {
        level: 11,
      },
    }),
    new BundleAnalyzerPlugin({
      analyzerMode: "static",
      openAnalyzer: false,
      reportFilename: "bundle-report.html",
    }),
  ],
  experiments: {
    asyncWebAssembly: true,
  },
};
```

### 4.2 TypeScript Configuration for Code Splitting

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "esnext",
    "lib": ["ES2020", "DOM", "WebWorker"],
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "types": ["@types/wicg-file-system-access"],
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "wasm-src"]
}
```

## Testing Strategy

### Media Processing Tests

```typescript
describe("Media Processing", () => {
  describe("Metadata Extraction", () => {
    it("should extract JPEG metadata", async () => {
      const jpegBlob = await loadTestImage("test.jpg");
      const metadata = await MediaProcessor.extractMetadata(jpegBlob);

      expect(metadata).toBeDefined();
      expect(metadata?.format).toBe("jpeg");
      expect(metadata?.width).toBe(1920);
      expect(metadata?.height).toBe(1080);
      expect(metadata?.exif?.make).toBe("Canon");
    });

    it("should fall back to Canvas API when WASM unavailable", async () => {
      // Mock WASM failure
      jest
        .spyOn(WASMModule, "initialise")
        .mockRejectedValue(new Error("WASM failed"));

      const pngBlob = await loadTestImage("test.png");
      const metadata = await MediaProcessor.extractMetadata(pngBlob);

      expect(metadata).toBeDefined();
      expect(metadata?.format).toBe("png");
      expect(metadata?.hasAlpha).toBe(true);
    });
  });

  describe("Thumbnail Generation", () => {
    it("should generate thumbnail within size limit", async () => {
      const largeImage = await loadTestImage("large.jpg"); // 10MB

      const result = await ThumbnailGenerator.generateThumbnail(largeImage, {
        maxWidth: 256,
        maxHeight: 256,
        targetSize: 65536, // 64KB
      });

      expect(result.blob.size).toBeLessThanOrEqual(65536);
      expect(result.width).toBeLessThanOrEqual(256);
      expect(result.height).toBeLessThanOrEqual(256);
    });

    it("should maintain aspect ratio by default", async () => {
      const wideImage = await loadTestImage("wide.jpg"); // 1920x1080

      const result = await ThumbnailGenerator.generateThumbnail(wideImage, {
        maxWidth: 256,
        maxHeight: 256,
      });

      const aspectRatio = result.width / result.height;
      expect(aspectRatio).toBeCloseTo(16 / 9, 2);
    });

    it("should support smart cropping", async () => {
      const portrait = await loadTestImage("portrait.jpg");

      const result = await ThumbnailGenerator.generateThumbnail(portrait, {
        maxWidth: 256,
        maxHeight: 256,
        maintainAspectRatio: false,
        smartCrop: true,
      });

      expect(result.width).toBe(256);
      expect(result.height).toBe(256);
    });
  });

  describe("Progressive Loading", () => {
    it("should create progressive JPEG", async () => {
      const jpeg = await loadTestImage("test.jpg");

      const progressive = await ProgressiveImageLoader.createProgressive(jpeg, {
        progressiveScans: 3,
        qualityLevels: [20, 50, 85],
      });

      expect(progressive.layerCount).toBe(3);

      const firstLayer = progressive.getLayer(0);
      expect(firstLayer?.quality).toBe(20);
      expect(firstLayer?.isBaseline).toBe(true);
    });

    it("should create interlaced PNG", async () => {
      const png = await loadTestImage("test.png");

      const progressive = await ProgressiveImageLoader.createProgressive(png, {
        interlace: true,
      });

      expect(progressive.layerCount).toBe(1); // PNG uses single interlaced file
    });
  });

  describe("Performance", () => {
    it("should process thumbnails within performance budget", async () => {
      const image = await loadTestImage("medium.jpg"); // 1MP image

      const timer = PerformanceMonitor.startOperation("thumbnail_test");

      await ThumbnailGenerator.generateThumbnail(image, {
        maxWidth: 256,
        maxHeight: 256,
        quality: 85,
      });

      const metric = timer.end();

      expect(metric.duration).toBeLessThan(500); // 500ms budget
    });

    it("should handle concurrent operations efficiently", async () => {
      const images = await Promise.all([
        loadTestImage("test1.jpg"),
        loadTestImage("test2.jpg"),
        loadTestImage("test3.jpg"),
        loadTestImage("test4.jpg"),
      ]);

      const timer = PerformanceMonitor.startOperation("concurrent_thumbnails");

      const results = await Promise.all(
        images.map((img) =>
          ThumbnailGenerator.generateThumbnail(img, {
            maxWidth: 128,
            maxHeight: 128,
          })
        )
      );

      const metric = timer.end();

      expect(results).toHaveLength(4);
      expect(metric.duration).toBeLessThan(1000); // Should benefit from parallelism
    });
  });

  describe("Bundle Size", () => {
    it("should keep media module under size limit", async () => {
      const stats = await getWebpackStats();
      const mediaChunk = stats.chunks.find((c) => c.names.includes("media"));

      expect(mediaChunk).toBeDefined();
      expect(mediaChunk!.size).toBeLessThan(700 * 1024); // 700KB limit
    });

    it("should lazy load WASM module", async () => {
      const stats = await getWebpackStats();
      const mainChunk = stats.chunks.find((c) => c.names.includes("main"));
      const wasmChunk = stats.chunks.find((c) =>
        c.names.includes("media-wasm")
      );

      expect(mainChunk).toBeDefined();
      expect(wasmChunk).toBeDefined();

      // WASM should not be in main chunk
      const mainModules = mainChunk!.modules.map((m) => m.name);
      expect(mainModules).not.toContain(expect.stringContaining(".wasm"));
    });
  });
});
```

## Conclusion

This design provides a comprehensive media processing pipeline for Enhanced S5.js with:

1. **Modular Architecture** - WASM modules load only when needed
2. **Graceful Degradation** - Canvas API fallback when WASM unavailable
3. **Performance Optimised** - Worker threads, streaming, and efficient memory usage
4. **Bundle Size Conscious** - Code splitting keeps initial bundle small
5. **Browser Compatible** - Tested across major browsers with appropriate fallbacks
6. **Feature Rich** - Metadata extraction, thumbnail generation, progressive loading
7. **Web Hosting Support** - Default file resolution and proper MIME type detection
8. **Extension Handling** - Both .jpg and .jpeg (and other common variations) are properly supported

The implementation maintains the ≤700KB compressed bundle size target whilst providing professional-grade image processing capabilities directly in the browser.
