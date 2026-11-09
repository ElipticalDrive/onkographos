# Volume Partitioning Pipeline


## Overview
The Volume Partitioning Pipeline splits a 3D volume into foreground regions, optionally partitions them spatially or by intensity, crops to bounding boxes, downsamples, and exports as raw files with metadata [S1, unofficial]. It differs from the [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by focusing on geometric and intensity-based partitioning of the entire foreground mass rather than semantic or anatomical segmentation, and from the [Preprocessing Pipeline](6733-preprocessing-pipeline.md) parent by specifically handling the decomposition into disjoint components for efficient storage and rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume array (typically `uint8` or `uint16`), processing parameters.
- Output: A directory containing `component_{id}.raw` files and a `metadata.json` file.

**Data Structures:**
- `metadata.json`: JSON object with schema version 2.
  - Top-level fields: `schema_version` (int), `dtype` (str), `array_order` (str, 'ZYX'), `coordinate_order` (str, 'XYZ'), `input_file` (str), `input_sha256` (str), `dimensions_xyz` (list[int]), `original_bytes` (int), `parameters` (dict), `foreground_metrics` (dict), `original_nonzero` (int), `retained_nonzero` (int), `components` (list[dict]), `output_bytes` (int), `processing_seconds` (float), `reconstruction` (str).
  - Component dict fields: `file` (str), `dimensions_xyz` (list[int]), `origin_xyz` (list[int]), `spacing_xyz` (list[float]), `extent_xyz` (list[int]), `bytes` (int), `sha256` (str).

**Parameters:**
- `mode`: 'original', 'crop', or 'partition'. Default: 'partition' [S1, unofficial].
- `components`: Number of partitions. Range: 1–15. Default: 8 [S1, unofficial].
- `spatial_weight`: Weight for kmeans partitioning. Range: [0,1]. Default: 0.5 [S1, unofficial].
- `downsample`: Integer downsampling factor. Range: >= 1. Default: 1 [S1, unofficial].
- `threshold`: Foreground cutoff. Range: [0,255]. If None, uses `background_threshold` [S1, unofficial].
- `cleanup`: 'none', 'connected', 'opening', or 'legacy'. Default: 'none' [S1, unofficial].
- `min_region_voxels`: Minimum voxel count for valid regions. Default: 512 [S1, unofficial].
- `strategy`: 'spatial' or 'kmeans'. Default: 'spatial' [S1, unofficial].
- `seed`: Random seed for kmeans. Default: 42 [S1, unofficial].

**Steps:**
1. **Validate Inputs:** Ensure `mode` is valid, `components` in [1,15], `spatial_weight` in [0,1], `downsample` >= 1, and `threshold` in [0,255] if provided. Raise `ValueError` if output directory is not empty [S1, unofficial].
2. **Read Volume:** Load input volume.
3. **Compute Cutoff:** If `mode` is 'original', cutoff is 0.0. Otherwise, if `threshold` is None, compute `background_threshold(original, seed)`. Else, use `float(threshold)` [S1, unofficial].
4. **Foreground Masking:** Compute foreground mask via `select_foreground(cutoff, cleanup, min_region_voxels)`. Create retained array: `retained = np.where(mask, original, 0).astype(np.uint8)` [S1, unofficial].
5. **Labeling:**
   - If `mode` is 'partition':
     - If `strategy` is 'spatial', compute labels via `spatial_partition(retained, components)`.
     - If `strategy` is 'kmeans', compute labels via `kmeans_partition(retained, components, seed, spatial_weight)` [S1, unofficial].
   - Otherwise (`mode` is 'crop' or 'original'), `labels = (retained > 0).astype(np.uint8)` [S1, unofficial].
6. **Export Components:** Iterate over unique label IDs > 0. For each ID:
   - Compute bounding box `(lo, hi)` from label coordinates.
   - Extract subvolume.
   - Apply mask.
   - Downsample by `downsample` factor along all axes: `data = data[::downsample, ::downsample, ::downsample].copy()` [S1, unofficial].
   - Write to `component_{id}.raw`.
   - Record dimensions, origin, spacing, extent, bytes, and SHA256 hash [S1, unofficial].
7. **Generate Metadata:** Create `metadata.json` with schema version 2, dtype, array/coordinate order, input metadata, parameters, foreground metrics, component records, and reconstruction note [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import hashlib
import json
from pathlib import Path

def volume_partitioning_pipeline(volume, output_dir, mode='partition', components=8,
                                 spatial_weight=0.5, downsample=1, threshold=None,
                                 cleanup='none', min_region_voxels=512,
                                 strategy='spatial', seed=42):
    out = Path(output_dir)
    if out.exists() and any(out.iterdir()):
        raise ValueError('Output directory must be empty')
    out.mkdir(parents=True, exist_ok=True)

    # Validation
    if mode not in ('original', 'crop', 'partition'):
        raise ValueError('Mode must be original, crop, or partition')
    if not (1 <= components <= 15):
        raise ValueError('Components must be 1-15')
    if not (0 <= spatial_weight <= 1):
        raise ValueError('Spatial weight must be in [0,1]')
    if not isinstance(downsample, int) or downsample < 1:
        raise ValueError('Downsample must be integer >= 1')
    if threshold is not None and not (0 <= threshold <= 255):
        raise ValueError('Threshold must be in [0,255]')

    # Compute cutoff
    if mode == 'original':
        cutoff = 0.0
    else:
        if threshold is None:
            cutoff = background_threshold(volume, seed) # [G] Placeholder for specific background threshold logic
        else:
            cutoff = float(threshold)

    # Foreground mask
    mask = select_foreground(cutoff, cleanup, min_region_voxels) # [G] Placeholder for cleanup logic
    retained = np.where(mask, volume, 0).astype(np.uint8)

    # Labeling
    if mode == 'partition':
        if strategy == 'spatial':
            labels = spatial_partition(retained, components) # [G] Placeholder for spatial partitioning
        elif strategy == 'kmeans':
            labels = kmeans_partition(retained, components, seed, spatial_weight) # [G] Placeholder for kmeans partitioning
        else:
            raise ValueError('Unknown strategy')
    else:
        labels = (retained > 0).astype(np.uint8)

    unique_labels = np.unique(labels)
    unique_labels = unique_labels[unique_labels > 0]

    component_records = []
    total_output_bytes = 0

    for label_id in unique_labels:
        # Compute bounding box
        coords = np.argwhere(labels == label_id)
        if coords.size == 0:
            continue
        lo = coords.min(axis=0)
        hi = coords.max(axis=0)

        # Extract subvolume
        # Note: hi is inclusive, so slice end is hi + 1
        subvol = retained[lo[0]:hi[0]+1, lo[1]:hi[1]+1, lo[2]:hi[2]+1]
        
        # Apply mask (re-apply to ensure only foreground is kept, though retained already masked)
        subvol_mask = labels[lo[0]:hi[0]+1, lo[1]:hi[1]+1, lo[2]:hi[2]+1] == label_id
        subvol = np.where(subvol_mask, subvol, 0)

        # Downsample
        if downsample > 1:
            subvol = subvol[::downsample, ::downsample, ::downsample].copy()

        # Write file
        filename = f'component_{label_id}.raw'
        filepath = out / filename
        subvol.tofile(filepath)
        
        # Compute hash
        with open(filepath, 'rb') as f:
            sha256_hash = hashlib.sha256(f.read()).hexdigest()

        # Record metadata
        dims = list(subvol.shape)
        origin = list(lo * downsample) # [G] Adjust origin based on downsampling if needed for reconstruction
        spacing = [1.0, 1.0, 1.0] # [G] Placeholder for actual spacing
        extent = [0, 0, 0] # [G] Placeholder for extent calculation
        
        component_records.append({
            'file': filename,
            'dimensions_xyz': dims,
            'origin_xyz': origin,
            'spacing_xyz': spacing,
            'extent_xyz': extent,
            'bytes': subvol.nbytes,
            'sha256': sha256_hash
        })
        total_output_bytes += subvol.nbytes

    # Generate metadata
    metadata = {
        'schema_version': 2,
        'dtype': 'uint8',
        'array_order': 'ZYX',
        'coordinate_order': 'XYZ',
        'input_file': 'input_volume.raw', # [G] Placeholder for actual input file path
        'input_sha256': 'placeholder', # [G] Placeholder for actual input hash
        'dimensions_xyz': list(volume.shape),
        'original_bytes': volume.nbytes,
        'parameters': {
            'mode': mode,
            'components': components,
            'spatial_weight': spatial_weight,
            'downsample': downsample,
            'threshold': threshold,
            'cleanup': cleanup,
            'min_region_voxels': min_region_voxels,
            'strategy': strategy,
            'seed': seed
        },
        'foreground_metrics': {
            'original_nonzero': int(np.count_nonzero(volume)),
            'retained_nonzero': int(np.count_nonzero(retained))
        },
        'original_nonzero': int(np.count_nonzero(volume)),
        'retained_nonzero': int(np.count_nonzero(retained)),
        'components': component_records,
        'output_bytes': total_output_bytes,
        'processing_seconds': 0.0, # [G] Placeholder for actual time
        'reconstruction': 'Reconstruct by loading components, upsampling if needed, and placing at origin.'
    }

    with open(out / 'metadata.json', 'w') as f:
        json.dump(metadata, f, separators=(',', ':'))
```

## Subtopics

### [Bounded-Memory Large Volume Export](11587-bounded-memory-large-volume-export.md)
Processes large volumes (OME-Zarr or RAW) with bounded memory by streaming chunks, computing intensity baselines, cropping to disjoint aligned bricks, and writing compressed outputs without loading the full volume. It supports public Open SciVis bytes+zstd, unsharded ZYX uint8/uint16 arrays. Raw precision and full resolution are retained. Only values <= cutoff are discarded. Files are streamed; no full volume or full label array exists.

### K-Means Spatial-Intensity Partitioning
Partitions volume into components by clustering foreground voxels based on spatial coordinates (x,y,z) and intensity, using a weighted combination [S1, unofficial].
- **Steps:**
  1. Get foreground indices: `foreground = np.flatnonzero(volume)`. Initialize labels array of zeros with shape `volume.shape`.
  2. Define feature extraction: Unravel indices to z,y,x. Stack x,y,z and intensity values. Cast to float32.
  3. Sample up to 30,000 foreground indices: `sample = rng.choice(foreground, min(30000, foreground.size), replace=False)`.
  4. Fit StandardScaler on sample features.
  5. Define weights: `[spatial_weight, spatial_weight, spatial_weight, 1 - spatial_weight]` for x,y,z,intensity.
  6. Transform sample features, scale by weights, and fit KMeans with `n_clusters=min(components, unique_count)`, `n_init=3`, `random_state=seed`, `max_iter=100`.
  7. Predict labels for all foreground voxels in batches of `batch_size` (100,000). Transform features, scale by weights, predict, add 1 to label (0 is background) [S1, unofficial].
- **Parameters:** `components` (default 8), `spatial_weight` (default 0.5), `batch_size` (default 100,000) [S1, unofficial].

### Greedy Axis-Aligned Spatial Partitioning
Creates a disjoint spatial partition of the volume into axis-aligned bounding boxes using a greedy heuristic that minimizes bounding box volume [S1, unofficial].
- **Steps:**
  1. Compute tight bounding box for the root volume: `root = tight(np.zeros(3, dtype=int), np.array(volume.shape))`. If None, return empty labels.
  2. Initialize leaves list with root: `leaves = [root]`.
  3. While `len(leaves) < components`: Iterate over each box in leaves and each axis (0,1,2).
  4. For each axis, calculate length. Try offsets at 25%, 50%, 75% of length (rounded to int).
  5. Split box into left and right halves at offset. Compute tight bounding boxes for left and right children using `tight()` function.
  6. If both children are valid (not None), calculate score: `(size(box) - size(left) - size(right), size(box) - max(size(left), size(right)), -surface(left) - surface(right))`.
  7. Select the split with the best score. Replace the parent box in leaves with the two children.
  8. Assign labels: For each leaf (identifier starting at 1), set mask to identifier where volume > 0 within the leaf's bounding box [S1, unofficial].
- **Parameters:** `components` (default 8) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the partitioning strategy. K-Means is O(N * K * I) where N is foreground voxels, K is components, I is iterations. Spatial partitioning is O(N) for bounding box computation and O(N * K) for splitting.
- **Memory Complexity:** O(N) for the volume and labels. Bounded-memory variant streams chunks, reducing peak memory usage.
- **Performance:** Not given in the sources.
- **Strengths:** Efficiently reduces data size by cropping and downsampling. Disjoint components simplify rendering and storage.
- **Limitations:** Partitioning quality depends on strategy. K-Means may be sensitive to initialization. Spatial partitioning may create unbalanced partitions.
- **When to use:** When the volume is large and needs to be split into manageable chunks for rendering or storage.
- **When not to use:** When semantic segmentation is required instead of geometric partitioning.

## Background
The Volume Partitioning Pipeline is part of the VolumeSegmenter project by Winson-L [S1, unofficial]. It is designed to preprocess large 3D volumes for efficient handling in visualization and rendering pipelines. The method emphasizes bounded memory usage and disjoint component extraction [S1, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
