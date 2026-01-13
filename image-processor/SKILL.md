---
name: Image Processor
description: Process images in a directory: compress, convert format, and/or crop to specific dimensions.
---

# Image Processor

Process images in a directory: compress, convert format, and/or crop to specific dimensions.

**Input**: $ARGUMENTS

Parse the arguments to extract:
- **Directory path**: The target directory containing images (required)
- **Operations**: What to do - compress, convert, crop (default: all)
- **Output format**: webp, png, jpg (default: webp)
- **Target dimensions**: WIDTHxHEIGHT for cropping (optional)
- **Quality**: 1-100 for compression (default: 85)
- **Crop mode**: cover, contain, fill, square (default: cover)

## Argument Parsing Examples

```
/image-processor apps/web/public/images/templates
→ Directory: apps/web/public/images/templates
→ Operations: compress + convert to webp
→ Quality: 85

/image-processor apps/web/public/images/showcase --size 512x512 --quality 80
→ Directory: apps/web/public/images/showcase
→ Operations: compress + convert + crop to 512x512
→ Quality: 80

/image-processor ./images --format png --crop square --size 256
→ Directory: ./images
→ Operations: convert to PNG + crop to square 256x256

/image-processor path/to/images --compress-only --quality 90
→ Directory: path/to/images
→ Operations: compress only (keep format)
→ Quality: 90
```

## Implementation Steps

### Step 1: Validate Input
1. Parse the directory path from arguments
2. Verify the directory exists
3. Extract optional parameters (size, quality, format, crop mode)
4. List all image files in the directory (png, jpg, jpeg, webp, gif)

### Step 2: Show Preview
Before processing, display:
- Number of images found
- Current total size
- Planned operations
- Target format and dimensions (if applicable)

Ask for confirmation before proceeding with destructive operations.

### Step 3: Create Processing Script
Generate a TypeScript script using sharp that:

```typescript
import { readdir, stat, unlink, rename } from "node:fs/promises";
import { extname, join, basename } from "node:path";
import sharp from "sharp";

interface ProcessOptions {
  directory: string;
  outputFormat: "webp" | "png" | "jpg";
  quality: number;
  targetWidth?: number;
  targetHeight?: number;
  cropMode: "cover" | "contain" | "fill" | "square";
  compressOnly: boolean;
}

async function processImages(options: ProcessOptions) {
  const {
    directory,
    outputFormat,
    quality,
    targetWidth,
    targetHeight,
    cropMode,
    compressOnly,
  } = options;

  // Get all image files
  const files = await readdir(directory);
  const imageFiles = files.filter((f) => {
    const ext = extname(f).toLowerCase();
    return [".png", ".jpg", ".jpeg", ".webp", ".gif"].includes(ext);
  });

  console.log(`Found ${imageFiles.length} images to process\n`);

  let totalOriginalSize = 0;
  let totalNewSize = 0;

  for (const file of imageFiles) {
    const inputPath = join(directory, file);
    const originalSize = (await stat(inputPath)).size;
    totalOriginalSize += originalSize;

    // Determine output filename
    const baseName = basename(file, extname(file));
    const outputExt = compressOnly ? extname(file) : `.${outputFormat}`;
    const outputPath = join(directory, `${baseName}${outputExt}`);
    const tempPath = `${outputPath}.tmp`;

    console.log(`Processing: ${file}`);
    console.log(`  Original: ${(originalSize / 1024).toFixed(1)}KB`);

    let pipeline = sharp(inputPath);

    // Get metadata for cropping
    const metadata = await pipeline.metadata();
    const origWidth = metadata.width || 0;
    const origHeight = metadata.height || 0;

    // Apply crop if dimensions specified
    if (targetWidth && targetHeight) {
      if (cropMode === "square") {
        const size = Math.min(origWidth, origHeight);
        const left = Math.floor((origWidth - size) / 2);
        const top = Math.floor((origHeight - size) / 2);
        pipeline = pipeline.extract({ left, top, width: size, height: size });
        pipeline = pipeline.resize(targetWidth, targetHeight, { fit: "fill" });
      } else if (cropMode === "cover") {
        pipeline = pipeline.resize(targetWidth, targetHeight, { fit: "cover" });
      } else if (cropMode === "contain") {
        pipeline = pipeline.resize(targetWidth, targetHeight, { fit: "contain" });
      } else {
        pipeline = pipeline.resize(targetWidth, targetHeight, { fit: "fill" });
      }
    } else if (targetWidth) {
      // Square crop with single dimension
      const size = Math.min(origWidth, origHeight);
      const left = Math.floor((origWidth - size) / 2);
      const top = Math.floor((origHeight - size) / 2);
      pipeline = pipeline.extract({ left, top, width: size, height: size });
      pipeline = pipeline.resize(targetWidth, targetWidth, { fit: "fill" });
    }

    // Apply format conversion
    if (outputFormat === "webp" || (!compressOnly && outputFormat === "webp")) {
      pipeline = pipeline.webp({ quality, effort: 6, smartSubsample: true });
    } else if (outputFormat === "png") {
      pipeline = pipeline.png({ quality, compressionLevel: 9 });
    } else if (outputFormat === "jpg") {
      pipeline = pipeline.jpeg({ quality, mozjpeg: true });
    }

    await pipeline.toFile(tempPath);

    // Replace original
    if (inputPath !== outputPath) {
      await unlink(inputPath);
    }
    await rename(tempPath, outputPath);

    const newSize = (await stat(outputPath)).size;
    totalNewSize += newSize;

    const reduction = ((originalSize - newSize) / originalSize * 100).toFixed(1);
    console.log(`  Output: ${(newSize / 1024).toFixed(1)}KB (${reduction}% reduction)\n`);
  }

  console.log("=".repeat(50));
  console.log(`Total: ${(totalOriginalSize / 1024 / 1024).toFixed(2)}MB -> ${(totalNewSize / 1024 / 1024).toFixed(2)}MB`);
  console.log(`Overall reduction: ${((totalOriginalSize - totalNewSize) / totalOriginalSize * 100).toFixed(1)}%`);
}

// Execute with parsed options
processImages({
  directory: "TARGET_DIRECTORY",
  outputFormat: "webp",
  quality: 85,
  targetWidth: undefined,
  targetHeight: undefined,
  cropMode: "cover",
  compressOnly: false,
});
```

### Step 4: Execute Processing
1. Save the script to `scripts/process-images-temp.ts`
2. Run with: `pnpm tsx scripts/process-images-temp.ts`
3. Report results
4. Clean up temp script file

## Safety Checks
- Always create temp files first, then replace
- Verify output file was created before deleting original
- Show file sizes before and after
- Ask for confirmation before processing large batches (>20 images)

## Output Report
After processing, provide:
- Number of images processed
- Total size before/after
- Overall compression ratio
- Any errors encountered
- List of processed files with individual stats
