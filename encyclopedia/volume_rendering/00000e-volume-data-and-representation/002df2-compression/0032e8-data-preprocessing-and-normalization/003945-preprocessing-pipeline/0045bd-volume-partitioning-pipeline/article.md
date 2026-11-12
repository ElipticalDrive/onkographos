# Volume Partitioning Pipeline


## Overview
The Volume Partitioning Pipeline is a data preprocessing method that splits a 3D volume into disjoint foreground regions, optionally partitions them spatially or by intensity, crops them to bounding boxes, downsamples, and exports them as raw files with metadata [S1, unofficial]. It solves the problem of managing large 3D datasets by reducing memory footprint and enabling efficient out-of-core or chunked processing. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) by focusing on intensity-based foreground extraction and spatial partitioning rather than semantic segmentation, and from [3D Seismic Data Compression Pipeline](11727-3d-seismic-data-compression-pipeline.md) by using a generic intensity-threshold approach rather than seismic-specific physics [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume array (typically `uint8` or `uint16`), processing parameters (`mode`, `components`, `threshold`, etc.).
*   **Output:** A directory containing component files (`component_{id}.raw`) and a `metadata.json` file describing the geometry, parameters, and integrity of the exported data.

**Data Structures:**
*   **Metadata JSON:** A JSON object with the following schema:
    *   `schema_version`: Integer (e.g., 2).
    *   `dtype`: String (e.g., 'uint8').
    *   `array_order`: String (e.g., 'ZYX').
    *   `coordinate_order`: String (e.g., 'XYZ').
    *   `input_file`: String (path to original file).
    *   `input_sha256`: String (SHA-256 hash of input).
    *   `dimensions_xyz`: List of 3 integers.
    *   `original_bytes`: Integer.
    *   `parameters`: Dictionary of processing parameters.
    *   `foreground_metrics`: Dictionary containing `original_nonzero`, `retained_nonzero`, etc.
    *   `components`: List of component dictionaries. Each component dict contains:
        *   `file`: String (filename, e.g., 'component_1.raw').
        *   `dimensions_xyz`: List of 3 integers.
        *   `origin_xyz`: List of 3 integers.
        *   `spacing_xyz`: List of 3 floats.
        *   `extent_xyz`: List of 3 integers.
        *   `bytes`: Integer.
        *   `sha256`: String.
    *   `output_bytes`: Integer.
    *   `processing_seconds`: Float.
    *   `reconstruction`: String (note on how to reconstruct the original volume).

**Parameters:**
*   `mode`: Processing mode. Choices: 'original', 'crop', 'partition'. Default: 'partition' [S1, unofficial].
*   `components`: Number of partitions. Range: 1–15. Default: 8 [S1, unofficial].
*   `spatial_weight`: Weight for kmeans partitioning. Range: [0,1]. Default: 0.5 [S1, unofficial].
*   `downsample`: Integer downsampling factor applied to all axes. Range: >= 1. Default: 1 [S1, unofficial].
*   `threshold`: Foreground cutoff value. Range: [0,255]. If None, uses `background_threshold` [S1, unofficial].
*   `cleanup`: Foreground cleanup strategy. Choices: 'none', 'connected', 'opening', 'legacy'. Default: 'none' [S1, unofficial].
*   `min_region_voxels`: Minimum voxel count for valid regions. Default: 512 [S1, unofficial].
*   `strategy`: Partition strategy. Choices: 'spatial', 'kmeans'. Default: 'spatial' [S1, unofficial].
*   `seed`: Random seed for kmeans partitioning. Default: 42 [S1, unofficial].

**Steps:**
1.  **Validate Inputs:** Check that `mode` is one of 'original', 'crop', 'partition'; `components` is between 1 and 15; `spatial_weight` is in [0,1]; `downsample` is an integer >= 1; `threshold` is in [0,255] if provided. Ensure the output directory is empty; raise `ValueError` if `component_*.raw` or `metadata.json` already exists [S1, unofficial].
2.  **Read Volume and Compute Cutoff:** Read the input volume. Compute the cutoff value: 0.0 for 'original' mode; otherwise, use `background_threshold(original, seed)` if `threshold` is None, else use `float(threshold)` [S1, unofficial].
3.  **Compute Foreground Mask:** Compute the foreground mask and metrics using `select_foreground(cutoff, cleanup, min_region_voxels)`. Create a retained array by masking the original: `retained = np.where(mask, original, 0).astype(np.uint8)` [S1, unofficial].
4.  **Partition (if mode is 'partition'):**
    *   If `strategy` is 'spatial', compute labels using `spatial_partition(retained, components)`.
    *   If `strategy` is 'kmeans', compute labels using `kmeans_partition(retained, components, seed, spatial_weight)`.
    *   Otherwise (for 'original' or 'crop' modes), set `labels = (retained > 0).astype(np.uint8)` [S1, unofficial].
5.  **Process Components:** Iterate over unique label IDs > 0. For each ID:
    *   Compute the bounding box `(lo, hi)` from the coordinates of voxels with this label.
    *   Extract the subvolume defined by the bounding box.
    *   Apply the mask for this label.
    *   Downsample by the `downsample` factor along all axes: `data = data[::downsample, ::downsample, ::downsample].copy()` [S1, unofficial].
    *   Write the data to `component_{id}.raw`.
    *   Record dimensions, origin, spacing, extent, byte count, and SHA-256 hash in a component metadata record [S1, unofficial].
6.  **Generate Metadata:** Create `metadata.json` with `schema_version=2`, `dtype='uint8'`, `array_order='ZYX'`, `coordinate_order='XYZ'`, input metadata, parameters, foreground metrics, component records, and a reconstruction note [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import hashlib
import json
import os
from pathlib import Path

def volume_partitioning_pipeline(input_path, output_dir, mode='partition', components=8, spatial_weight=0.5, downsample=1, threshold=None, cleanup='none', min_region_voxels=512, strategy='spatial', seed=42):
    """
    Processes a 3D volume by partitioning foreground regions and exporting them.
    """
    # 1. Validate Inputs
    if mode not in ('original', 'crop', 'partition'):
        raise ValueError("Mode must be 'original', 'crop', or 'partition'")
    if not (1 <= components <= 15):
        raise ValueError("Components must be 1-15")
    if not (0 <= spatial_weight <= 1):
        raise ValueError("Spatial weight must be in [0,1]")
    if not isinstance(downsample, int) or downsample < 1:
        raise ValueError("Downsample must be an integer >= 1")
    if threshold is not None and not (0 <= threshold <= 255):
        raise ValueError("Threshold must be in [0,255]")
    
    out_path = Path(output_dir)
    if out_path.exists() and any(out_path.glob('component_*.raw')) or (out_path / 'metadata.json').exists():
        raise ValueError("Output directory must be empty")
    out_path.mkdir(parents=True, exist_ok=True)

    # 2. Read Volume and Compute Cutoff
    # Assuming input is a numpy array for this example; in practice, read from file
    original = np.load(input_path) # Placeholder for actual reading logic
    if mode == 'original':
        cutoff = 0.0
    else:
        if threshold is None:
            cutoff = background_threshold(original, seed) # [G] Helper function to determine background threshold
        else:
            cutoff = float(threshold)

    # 3. Compute Foreground Mask
    mask, metrics = select_foreground(cutoff, cleanup, min_region_voxels) # [G] Helper function
    retained = np.where(mask, original, 0).astype(np.uint8)

    # 4. Partition
    if mode == 'partition':
        if strategy == 'spatial':
            labels = spatial_partition(retained, components) # [G] Helper function
        elif strategy == 'kmeans':
            labels = kmeans_partition(retained, components, seed, spatial_weight) # [G] Helper function
        else:
            raise ValueError("Unknown strategy")
    else:
        labels = (retained > 0).astype(np.uint8)

    # 5. Process Components
    component_records = []
    unique_labels = np.unique(labels)
    unique_labels = unique_labels[unique_labels > 0] # Exclude background label 0

    for label_id in unique_labels:
        # Compute bounding box
        coords = np.argwhere(labels == label_id)
        if len(coords) == 0:
            continue
        lo = coords.min(axis=0)
        hi = coords.max(axis=0)
        
        # Extract subvolume
        # Note: hi is inclusive, so slice end is hi + 1
        subvolume = retained[lo[0]:hi[0]+1, lo[1]:hi[1]+1, lo[2]:hi[2]+1]
        submask = mask[lo[0]:hi[0]+1, lo[1]:hi[1]+1, lo[2]:hi[2]+1]
        
        # Apply mask
        data = np.where(submask, subvolume, 0).astype(np.uint8)
        
        # Downsample
        if downsample > 1:
            data = data[::downsample, ::downsample, ::downsample].copy()
        
        # Write to file
        filename = f"component_{label_id}.raw"
        filepath = out_path / filename
        data.tofile(filepath)
        
        # Compute hash
        sha256_hash = hashlib.sha256(data.tobytes()).hexdigest()
        
        # Record metadata
        component_records.append({
            "file": filename,
            "dimensions_xyz": list(data.shape),
            "origin_xyz": list(lo),
            "spacing_xyz": [1.0, 1.0, 1.0], # [G] Placeholder for actual spacing
            "extent_xyz": list(data.shape),
            "bytes": data.nbytes,
            "sha256": sha256_hash
        })

    # 6. Generate Metadata
    metadata = {
        "schema_version": 2,
        "dtype": "uint8",
        "array_order": "ZYX",
        "coordinate_order": "XYZ",
        "input_file": str(input_path),
        "input_sha256": hashlib.sha256(original.tobytes()).hexdigest(),
        "dimensions_xyz": list(original.shape),
        "original_bytes": original.nbytes,
        "parameters": {
            "mode": mode,
            "components": components,
            "spatial_weight": spatial_weight,
            "downsample": downsample,
            "threshold": threshold,
            "cleanup": cleanup,
            "min_region_voxels": min_region_voxels,
            "strategy": strategy,
            "seed": seed
        },
        "foreground_metrics": metrics,
        "components": component_records,
        "output_bytes": sum(c["bytes"] for c in component_records),
        "processing_seconds": 0.0, # [G] Placeholder for actual time
        "reconstruction": "Reconstruction requires reassembling components based on origin and extent, upsampling if downsample > 1, and filling background with 0."
    }
    
    with open(out_path / 'metadata.json', 'w') as f:
        json.dump(metadata, f, separators=(',', ':'))

def background_threshold(volume, seed):
    # [G] Placeholder implementation
    return np.percentile(volume[volume > 0], 1) if np.any(volume > 0) else 0

def select_foreground(cutoff, cleanup, min_region_voxels):
    # [G] Placeholder implementation
    mask = np.zeros_like(cutoff, dtype=bool) # [G] Placeholder
    metrics = {"original_nonzero": 0, "retained_nonzero": 0}
    return mask, metrics

def spatial_partition(retained, components):
    # [G] Placeholder implementation
    return np.zeros_like(retained, dtype=np.uint8)

def kmeans_partition(retained, components, seed, spatial_weight):
    # [G] Placeholder implementation
    return np.zeros_like(retained, dtype=np.uint8)
```

## Subtopics

### [Bounded-Memory Large Volume Export](11587-bounded-memory-large-volume-export.md)
Bounded-Memory Large Volume Export processes large 3D volumes (such as OME-Zarr or RAW formats) with bounded memory usage by streaming input chunks, computing intensity baselines, cropping data to disjoint aligned bricks, and writing compressed outputs without loading the full volume into RAM. It differs from sibling methods by explicitly enforcing a memory bound through chunked streaming and packing only foreground voxels into a compact, offset-indexed raw file, discarding background values entirely.

### [K-Means Spatial-Intensity Partitioning](11588-k-means-spatial-intensity-partitioning.md)
K-Means Spatial-Intensity Partitioning splits a 3D volume into a fixed number of components by clustering foreground voxels based on a weighted combination of their spatial coordinates and intensity values. It sets itself apart from sibling methods like Greedy Axis-Aligned Spatial Partitioning by incorporating intensity similarity into the clustering metric, allowing partitions to respect both geometric locality and data content, whereas siblings typically rely solely on spatial boundaries or memory constraints.

### [Greedy Axis-Aligned Spatial Partitioning](11589-greedy-axis-aligned-spatial-partitioning.md)
Greedy Axis-Aligned Spatial Partitioning creates a disjoint spatial partition of a 3D volume into axis-aligned bounding boxes (AABBs) using a greedy heuristic that minimizes the total bounding box volume. It is used in the Volume Partitioning Pipeline to split foreground regions into manageable components for export. It differs from [Slab-projected kd-tree partitioning](11586-slab-projected-kd-tree-partitioning.md) by using a direct greedy box-splitting approach rather than slab-projection slices for out-of-core processing.

### Data Partitioning
Data Partitioning divides display data into N equal sizes according to the sub-display system size [S2]. The parameter N is the number of sub-display modules [S2].

### Feature Plane Splitting
Feature Plane Splitting handles segmented datasets by replacing small blocks of N voxels with a single voxel split by a feature plane into two regions of constant values [S3]. The step involves identifying small blocks of N voxels in segmented datasets, replacing each block with a single voxel, defining a feature plane that splits the voxel into two regions, and assigning constant values to each region [S3]. It has little overhead over storing precomputed gradients and provides minimal geometric information about underlying volume regions that can be effectively exploited for volume rendering [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources. However, the Bounded-Memory variant explicitly streams chunks to avoid loading the full volume into RAM [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Reduces memory footprint by cropping to bounding boxes and downsampling [S1, unofficial]. Allows for out-of-core processing with bounded memory [S1, unofficial]. Provides flexibility in partitioning strategies (spatial vs. kmeans) [S1, unofficial].
*   **Limitations:** Requires an empty output directory [S1, unofficial]. Threshold validation is strict [S1, unofficial]. The reconstruction note is a placeholder and may not fully describe the reconstruction process [S1, unofficial].
*   **When to use:** When dealing with large 3D volumes that need to be processed or rendered in chunks, or when memory is constrained [S1, unofficial].
*   **When not to use:** When the full volume needs to be kept in memory, or when semantic segmentation is required instead of intensity-based partitioning [S1, unofficial].

## Background
The Volume Partitioning Pipeline is described in the context of volume data preprocessing and compression [S1, unofficial]. It is part of a larger pipeline that includes volume segmentation and seismic data compression [S1, unofficial]. The method is used to split 3D volumes into manageable components for export [S1, unofficial]. The Bounded-Memory variant is used for large volumes like OME-Zarr [S1, unofficial]. The Data Partitioning method is used in 3D swept-volume volumetric displays [S2]. The Feature Plane Splitting method is used in compressed GPU-based direct volume rendering [S3].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
[S2] PDF An Improved Design of 3D Swept-Volume Volumetric Display, https://jcomputers.us/vol9/jcp0901-34.pdf, fetched 2026-10-28
[S3] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
