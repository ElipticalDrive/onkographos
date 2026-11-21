# Raw 2D Image Stream Loading

Also called: Image Sequence to Voxel Grid Import, voglRaw, Voxmap Import from Image Sequence

## Overview
Imports a sequence of 2D images into a 3D voxel grid (Voxmap), storing dimensions, metadata, and raw voxel data in separate files [S1, unofficial]. It accepts input from a list of files containing raw 2D images instead of a single 3D TIFF file [S2]. Unlike [Voxel data loading and quantization](2478-voxel-data-loading-and-quantization.md), which handles quantization, this method focuses on the structural assembly of a voxel grid from an image stream, potentially preceded by fixed-length headers [S2].

## How it works
**Inputs and Outputs:**
- **Input:** A directory path containing image files (extensions: `*.jpg`, `*.jpeg`, `*.tif`, `*.tiff`, `*.png`) [S1, unofficial]. Alternatively, command-line arguments specifying file names, image sizes, header length, and voxel data type for raw 2D files (voxel streams in row-major order) [S2].
- **Output:** A 3D voxel grid stored in memory, with metadata saved to `<outbase>.json` and raw voxel data saved to `<outbase>.data` [S1, unofficial].

**Data Structures:**
- **Voxmap:** Stores dimensions $X, Y, Z$ and a contiguous memory `store` of size $X \times Y \times Z$ [S1, unofficial].
- **Stride:**
  - $ystride = X$
  - $zstride = X \times Y$ [S1, unofficial]

**Steps:**
1. **Clear existing data:** Set $X=Y=Z=0$, resize `store` to 0, reset metadata and transforms [S1, unofficial].
2. **Collect files:** Recursively traverse the source directory to collect all image files with extensions `*.jpg`, `*.jpeg`, `*.tif`, `*.tiff`, `*.png` into a list `ifns` [S1, unofficial].
3. **Determine dimensions:** Load the first image `ifns[0]` to determine width $X$ and height $Y$. Set $Z$ to the total number of files in `ifns` [S1, unofficial].
4. **Validate size:** Check if $X \times Y \times Z$ exceeds $MAXMVOX \times 1024 \times 1024$. If so, clear and return false [S1, unofficial].
5. **Allocate memory:** Resize `store` to $X \times Y \times Z$. Set $ystride = X$, $zstride = X \times Y$. Set data pointer to `store.data()` [S1, unofficial].
6. **Load and copy:** Loop $z$ from 0 to $Z-1$:
   - Load image `ifns[z]`.
   - Verify width/height match $X/Y$.
   - Loop $y$ from 0 to $Y-1$:
     - `memcpy` `img.scanLine(y)` into `data + z*zstride + y*X` with length $X$ [S1, unofficial].
7. **Save:** Save metadata to `outbase + ".json"` using `saveJson`. Save raw voxel data to `outbase + ".data"` by writing `zstride` bytes for each $z$-slice [S1, unofficial].

**Parameters:**
- **Header Length:** For raw 2D files, a fixed-length header may precede the voxel stream [S2].
- **Data Type:** Specified on command line (e.g., `ushort`) [S2].
- **Data Range:** Optional range for visualization (e.g., `0 4095`) [S2].
- **LUT:** Optional lookup table indices (e.g., `135 155`) [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_raw_2d_image_stream(source_dir, outbase):
    # 1. Clear existing data
    X = 0
    Y = 0
    Z = 0
    store = []
    meta = {}
    transforms = []

    # 2. Collect files
    ifns = []
    for root, dirs, files in os.walk(source_dir):
        for f in files:
            if f.lower().endswith(('.jpg', '.jpeg', '.tif', '.tiff', '.png')):
                ifns.append(os.path.join(root, f))
    
    if not ifns:
        return False

    # 3. Determine dimensions
    first_img = load_image(ifns[0])
    X = first_img.width
    Y = first_img.height
    Z = len(ifns)

    # 4. Validate size
    MAXMVOX = 1024 # Example constant, not given in sources
    if X * Y * Z > MAXMVOX * 1024 * 1024:
        # Clear and return false
        return False

    # 5. Allocate memory
    store = [0] * (X * Y * Z)
    ystride = X
    zstride = X * Y
    data = store # Pointer to store

    # 6. Load and copy
    for z in range(Z):
        img = load_image(ifns[z])
        if img.width != X or img.height != Y:
            # Handle mismatch, e.g., raise error or skip
            pass
        
        for y in range(Y):
            # memcpy img.scanLine(y) into data + z*zstride + y*X with length X
            # Assuming img.scanLine(y) returns a bytes-like object of length X
            scanline = img.scanLine(y)
            offset = z * zstride + y * X
            for i in range(X):
                data[offset + i] = scanline[i]

    # 7. Save
    save_json(outbase + ".json", meta)
    with open(outbase + ".data", "wb") as f:
        for z in range(Z):
            # Write zstride bytes for each z-slice
            # Note: zstride is in elements, assuming 1 byte per element for raw write
            # If data type is ushort, zstride should be in bytes or adjusted
            slice_start = z * zstride
            slice_end = (z + 1) * zstride
            # In Python, slicing creates a copy; for raw binary, we might need to pack
            # Assuming store is already in correct binary format or we pack it
            chunk = store[slice_start:slice_end]
            f.write(pack_data(chunk)) # Helper to pack based on data type

    return True
```

## Subtopics

### Image-Only Camera Reader
Reads cameras from a directory of images, assuming fixed FoV or computing it from width [S3, unofficial]. Lists and sorts filenames, opens image, converts to RGBA, and composites with background [S3, unofficial]. If FovY is None, computes focal_length_x from FovX and width, then FovY from focal_length_x and height [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of images $Z$ and the number of pixels per image $X \times Y$.
- **Memory Complexity:** $O(X \times Y \times Z)$ for the voxel grid.
- **Strengths:** Handles large datasets that cannot fit into a single 3D TIFF file; supports raw binary formats with headers [S2].
- **Limitations:** Requires all images to be of the same dimensions; no built-in quantization or compression [S1, unofficial][S2].
- **When to use:** When volume data is available as a sequence of 2D slices (e.g., from a microscope or 3D printer) rather than a single 3D file.

## Background
Originated in the SGI (Silicon Graphics) volume rendering tools, specifically `voglRaw`, which provides functionality identical to `voglCache` but accepts input from a list of files of raw 2D images [S2]. It is part of the `vogl` suite for volume visualization [S2].

## Sources
[S1] wagenadl/gvox, https://github.com/wagenadl/gvox, fetched 2026-10-28 (unofficial)
[S2] Appendix A. Volume Rendering Examples, https://techpubs.jurassic.nl/library/manuals/3000/007-3720-002/sgi_html/apa.html, fetched 2026-10-30
[S3] jibo27/3DGS_Hierarchical_Training, https://github.com/jibo27/3DGS_Hierarchical_Training, fetched 2026-10-27 (unofficial)
