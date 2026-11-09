# Bounded-Memory Large Volume Export

Also called: Bounded-Memory Export, Crop-and-Pack Export

## Overview
Bounded-Memory Large Volume Export processes large 3D volumes (such as OME-Zarr or RAW formats) with bounded memory usage by streaming input chunks, computing intensity baselines, cropping data to disjoint aligned bricks, and writing compressed outputs without loading the full volume into RAM [S1, unofficial]. It differs from sibling methods by explicitly enforcing a memory bound through chunked streaming and packing only foreground voxels into a compact, offset-indexed raw file, discarding background values entirely [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A volume source supporting chunked iteration (e.g., `Source` for OME-Zarr or `RawSource` for sequential slabs), an output directory path, a cutoff threshold, and optional brick dimensions [S1, unofficial].
- **Output:** A directory containing `density.raw` (packed foreground data), metadata files (implied by structure), and progress logs [S1, unofficial].

**Data Structures:**
1.  **Source:** An iterator yielding tuples `(origin, array, received_bytes, chunk_hash)` for each chunk [S1, unofficial].
2.  **Pages Array:** A 4D array `pages` of shape `grid` (derived from volume shape and brick size) with 4 `uint32` fields per cell:
    -   `pages[cell][0]`: Packed offset + 1 (address of data in `density.raw`) [S1, unofficial].
    -   `pages[cell][1]`: Packed local XYZ origin (5 bits per axis) [S1, unofficial].
    -   `pages[cell][2]`: Packed local XYZ size (6 bits per axis) [S1, unofficial].
    -   `pages[cell][3]`: Reserved/unused [S1, unofficial].
3.  **Histogram:** 1D array of size $2^{8 \times \text{itemsize}}$ (256 for uint8, 65536 for uint16) [S1, unofficial].
4.  **MIPs:** 3 projection arrays, one per axis, storing maximum intensity projections [S1, unofficial].
5.  **Bounds:** `bounds_lo` and `bounds_hi` arrays tracking the global bounding box of all packed bricks [S1, unofficial].

**Parameters:**
-   `brick`: Tuple of integers (1-32). If not provided, computed as the largest divisor $\le 16$ of the source chunk dimensions [S1, unofficial].
-   `cutoff`: Intensity threshold. Voxels $\le$ cutoff are discarded. Must be $<$ max dtype value [S1, unofficial].
-   `workers`: Number of threads for chunk fetching. Default: 4 [S1, unofficial].

**Steps:**
1.  **Initialize Source:** Create a `Source` or `RawSource` object. Validate that source chunk origins align with the brick grid (i.e., `origin % brick == 0`) [S1, unofficial].
2.  **Compute Brick Size:** If `brick` is not specified, calculate `b = max([d for d in range(4, 17) if c % d == 0])` for each chunk dimension `c` [S1, unofficial]. Validate $1 \le b \le 32$ and $0 \le \text{cutoff} < \text{max\_dtype}$ [S1, unofficial].
3.  **Profile Source:** Stream all blocks to build a histogram. Compute Otsu threshold using `skimage.filters.threshold_otsu(hist, np.arange(len(hist)))`. Compute CDF and quantiles at [0.5, 0.9, 0.95, 0.98, 0.99, 0.995] [S1, unofficial].
4.  **Initialize Output:** Create output directory. Initialize `pages`, `hist`, `offset`, `selected`, `bricks`, `compressed_bytes`, `full_brick_voxels`, `payload_hash`, `input_chunks_hash`, `bounds_lo`, `bounds_hi`, and `mips` [S1, unofficial].
5.  **Stream and Process:** Iterate through source blocks using `workers` threads:
    a. Fetch chunk, decompress, and validate size [S1, unofficial].
    b. Update histogram and MIPs [S1, unofficial].
    c. Call `crop_bricks` to yield disjoint aligned cells [S1, unofficial].
    d. For each cell:
        i. Check if `offset + data.size` exceeds $2^{32}$ [S1, unofficial].
        ii. Pack metadata into `pages[cell]` [S1, unofficial].
        iii. Write raw data to `density.raw` and update `payload_hash` [S1, unofficial].
        iv. Update global bounds and counters [S1, unofficial].
    e. Print progress every 32 chunks or at the end [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import hashlib
import time
from pathlib import Path
from skimage.filters import threshold_otsu
from concurrent.futures import ThreadPoolExecutor
import itertools

def crop_bricks(array, origin, brick, cutoff):
    """Yields (cell, lo, data, count) for occupied bricks."""
    mask = array > cutoff
    # Pad array to brick grid
    padded_shape = tuple(((n + b - 1) // b) * b for n, b in zip(array.shape, brick))
    padded = np.zeros(padded_shape, dtype=bool)
    slices = tuple(slice(0, n) for n in array.shape)
    padded[slices] = mask
    
    nz, ny, nx = (n // b for n, b in zip(padded_shape, brick))
    bz, by, bx = brick
    
    # Reshape to (nz, bz, ny, by, nx, bx) and find occupied bricks
    occupied = padded.reshape(nz, bz, ny, by, nx, bx).any(axis=(1, 3, 5))
    
    for iz, iy, ix in np.argwhere(occupied):
        local = (int(iz) * bz, int(iy) * by, int(ix) * bx)
        sl = tuple(slice(o, o + b) for o, b in zip(local, brick))
        submask = mask[sl]
        
        coords = np.where(submask)
        if not coords[0].size: continue # Empty brick after mask
        
        lo = tuple(int(c.min()) for c in coords)
        hi = tuple(int(c.max()) + 1 for c in coords)
        crop = tuple(slice(a, b) for a, b in zip(lo, hi))
        
        # Crop data, filling non-masked areas with 0
        data = np.where(submask[crop], array[sl][crop], 0).astype(array.dtype)
        
        # Calculate cell index in the global brick grid
        cell = tuple((o + l) // b for o, l, b in zip(origin, local, brick))
        
        yield cell, lo, data, int(submask.sum())

def profile(source):
    """Compute histogram and Otsu threshold."""
    hist_size = 65536 if source.dtype.itemsize == 2 else 256
    hist = np.zeros(hist_size, dtype=np.int64)
    
    for _, a, _, _ in source.blocks():
        hist += np.bincount(a.ravel(), minlength=hist_size)
        
    threshold = int(threshold_otsu(hist=(hist, np.arange(len(hist)))))
    return threshold

def export(source, output, cutoff, brick=None, workers=4, save_original=False):
    start = time.perf_counter()
    out = Path(output)
    if out.exists() and any(out.iterdir()):
        raise ValueError('Choose an empty export directory')
    out.mkdir(parents=True, exist_ok=True)
    
    # Compute brick size if not provided
    if brick is None:
        brick = tuple(max(b for b in range(4, 17) if c % b == 0) for c in source.chunks)
        
    if any(b > 32 or b < 1 for b in brick) or not 0 <= cutoff < np.iinfo(source.dtype).max:
        raise ValueError('Invalid brick or cutoff')
        
    grid = tuple((n + b - 1) // b for n, b in zip(source.shape, brick))
    
    # Initialize data structures
    pages = np.zeros((*grid, 4), dtype='<u4')
    hist = np.zeros(1 << (8 * source.dtype.itemsize), dtype=np.int64)
    offset = 0
    selected = 0
    bricks = 0
    compressed_bytes = 0
    full_brick_voxels = 0
    payload_hash = hashlib.sha256()
    input_chunks_hash = hashlib.sha256()
    bounds_lo = np.array(source.shape)
    bounds_hi = np.zeros(3, dtype=int)
    
    # Initialize MIPs
    mips = []
    for axis in range(3):
        shape = tuple(n for j, n in enumerate(source.shape) if j != axis)
        mips.append(np.zeros(shape, dtype=source.dtype))
        
    total_chunks = np.prod(grid)
    
    with open(out / 'density.raw', 'wb') as target:
        for index, (origin, array, received, chunk_hash) in enumerate(source.blocks(workers)):
            # Update MIPs
            for axis in range(3):
                sl = tuple(slice(o, o + n) for j, (o, n) in enumerate(zip(origin, array.shape)) if j != axis)
                np.maximum(mips[axis][sl], array.max(axis=axis), out=mips[axis][sl])
                
            # Update histogram
            hist += np.bincount(array.ravel(), minlength=len(hist))
            
            # Process bricks
            for cell, lo, data, count in crop_bricks(array, origin, brick, cutoff):
                if offset + data.size >= 2**32:
                    raise ValueError('Packed element address exceeds uint32')
                    
                # Pack metadata: base+1, local XYZ origin (5 bits/axis), XYZ size (6 bits/axis)
                xyz = lo[::-1]
                dims = data.shape[::-1]
                pages[cell] = (
                    offset + 1,
                    xyz[0] | xyz[1] << 5 | xyz[2] << 10,
                    dims[0] | dims[1] << 6 | dims[2] << 12,
                    0
                )
                
                raw = data.tobytes()
                target.write(raw)
                payload_hash.update(raw)
                
                p = np.array(cell) * brick + lo
                bounds_lo = np.minimum(bounds_lo, p)
                bounds_hi = np.maximum(bounds_hi, p + data.shape)
                
                offset += data.size
                selected += count
                bricks += 1
                full_brick_voxels += int(np.prod(brick))
                
            compressed_bytes += received
            input_chunks_hash.update(bytes.fromhex(chunk_hash))
            
            if index % 32 == 0 or index + 1 == total_chunks:
                print({
                    'chunks': index + 1,
                    'total': total_chunks,
                    'selected_voxels': selected,
                    'packed_MiB': round(offset * source.dtype.itemsize / 2**20, 2),
                    'seconds': round(time.perf_counter() - start, 1)
                }, flush=True)

```

**Edge Cases and Pitfalls:**
-   **Alignment:** Source chunk origins must align with the brick grid. If `origin % brick != 0`, a `ValueError` is raised [S1, unofficial].
-   **Offset Overflow:** If the total packed data size exceeds $2^{32}$ bytes, a `ValueError` is raised because the offset is stored in `uint32` [S1, unofficial].
-   **Stale Metadata:** Zarr chunk metadata may be stale; validation requires checking the decompressed chunk size against the expected size [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of input voxels, $O(N)$, where $N$ is the total volume size. The complexity includes streaming, histogramming, and per-voxel masking/cropping [S1, unofficial].
-   **Memory Complexity:** Bounded by the size of a single chunk plus the `pages` array and histogram. The `pages` array size is $O(\text{grid\_size})$, which is typically much smaller than the full volume if the brick size is small relative to the volume [S1, unofficial].
-   **Performance:** Measured performance depends on I/O speed and CPU for masking. The use of `ThreadPoolExecutor` with a default of 4 workers allows for parallel chunk fetching [S1, unofficial].
-   **Strengths:** Handles volumes larger than available RAM; compact output format; fast streaming [S1, unofficial].
-   **Limitations:** Requires aligned chunks; limited by $2^{32}$ byte offset for the packed file; background voxels are discarded and cannot be recovered [S1, unofficial].

## Background
The method is implemented in the `VolumeSegmenter` project by Winson-L [S1, unofficial]. It is designed for preprocessing large microscopy volumes (OME-Zarr) for segmentation tasks, where memory constraints and efficient storage of foreground data are critical [S1, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
