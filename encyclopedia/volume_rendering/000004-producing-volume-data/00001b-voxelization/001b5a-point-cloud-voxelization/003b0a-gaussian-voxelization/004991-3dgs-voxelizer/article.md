# 3DGS Voxelizer

Also called: 3DGS to OpenUSD PointInstancer Voxelizer

## Overview
The 3DGS Voxelizer converts a 3D Gaussian Splatting (3DGS) scan, provided as a `.ply` or `.spz` file, into colored voxel blocks represented as OpenUSD `PointInstancer` data [S1, unofficial]. It solves the problem of visualizing and enabling physics collisions for 3DGS scenes in NVIDIA Omniverse (USD Composer, Isaac Sim) by transforming continuous Gaussian primitives into discrete, collision-ready geometry [S1, unofficial]. Unlike general-purpose Gaussian voxelization methods that produce density fields or volume textures, this method specifically targets the creation of instanced cube primitives for real-time rendering and simulation [S1, unofficial].

## How it works
The pipeline loads the source point cloud, decodes Gaussian attributes (position, color, opacity), voxels the data using vectorized operations, and authors the result as a USD `PointInstancer` [S1, unofficial].

### Inputs and Outputs
- **Input**: A 3DGS scan file (`.ply` or `.spz`) containing Gaussian splats.
- **Output**: An OpenUSD stage containing a `PointInstancer` primitive representing the voxelized scene.

### Data Structures
The core data structure for the loaded point cloud is a `PointCloud` dataclass [S1, unofficial]:
- `positions`: `np.ndarray` of shape `(N, 3)`, dtype `float32`.
- `colors`: `np.ndarray` of shape `(N, 3)`, dtype `uint8` (optional, derived from spherical harmonics).
- `opacities`: `np.ndarray` of shape `(N,)`, dtype `float32` (optional, sigmoid-activated logits).
- `is_3dgs`: `bool`, indicating if the data contains 3DGS-specific attributes like spherical harmonics.

### Parameters
- `resolution` (float): The size of each voxel block. Smaller values yield more detail but increase block count and processing time [S1, unofficial].
- `min_opacity` (float): Threshold to filter out faint splats. Only applicable for `.ply` files where opacities are sigmoid-activated logits [S1, unofficial].
- `density_threshold` (int): Minimum number of splats required to create a voxel. Default is 1 [S1, unofficial].
- `unit_scale` (float): Scaling factor to handle unit mismatches between USD Composer (centimeters) and Isaac Sim (meters) [S1, unofficial].

### Steps

1. **Load Point Cloud**: Read the `.ply` or `.spz` file into a `PointCloud` dataclass.
   - For `.ply`: Parse ASCII or binary format. Limit header reads to 10,000 lines to prevent infinite loops if `end_header` is missing [S1, unofficial].
   - For `.spz`: Ensure the file is gzip-compressed; raise an error otherwise [S1, unofficial]. Decode positions from 24-bit fixed-point integers and colors from quantized spherical harmonics coefficients [S1, unofficial].

2. **Decode Attributes**:
   - **Opacity**: If the input is 3DGS, activate the sigmoid logit:
     $$ O = \frac{1}{1 + e^{-\text{opacity}}} $$
   - **Color**: Extract color from spherical harmonics. For standard 3DGS:
     $$ C = \text{clip}\left((0.5 + SH\_C0 \cdot dc) \cdot 255.0, 0, 255\right) $$
     where $SH\_C0 = 0.28209479177387814$ and $dc$ is the stacked DC coefficient vector [S1, unofficial].
   - For `.spz` files, recover the DC coefficient first:
     $$ f\_dc = \frac{dc\_u8 / 255.0 - 0.5}{\_SPZ\_COLOR\_SCALE} $$
     where $\_SPZ\_COLOR\_SCALE = 0.15$, then apply the same color formula [S1, unofficial].

3. **Voxelize**: Compute voxel centers and colors based on the input resolution, `min_opacity`, and `density_threshold`.
   - Filter splats with opacity below `min_opacity`.
   - Assign splats to voxels. If `density_threshold` > 1, only keep voxels with at least that many contributing splats [S1, unofficial].
   - Use vectorized numpy operations to prevent UI thread stalls with multi-million point splats [S1, unofficial].

4. **Author USD**: Create a `PointInstancer` primitive on the USD stage.
   - Set the prototype shape to a cube.
   - Apply the `resolution` and `unit_scale` to the transform.
   - Optionally enable physics collision on the prototype [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import gzip
import struct
from dataclasses import dataclass
from typing import Tuple, Optional

# Constants
SH_C0 = 0.28209479177387814
SPZ_COLOR_SCALE = 0.15

@dataclass
class PointCloud:
    positions: np.ndarray  # (N, 3) float32
    colors: Optional[np.ndarray]  # (N, 3) uint8
    opacities: Optional[np.ndarray]  # (N,) float32
    is_3dgs: bool

def load_pointcloud(filepath: str) -> PointCloud:
    """Load PLY or SPZ file into PointCloud dataclass."""
    if filepath.endswith('.spz'):
        return _load_spz(filepath)
    elif filepath.endswith('.ply'):
        return _load_ply(filepath)
    else:
        raise ValueError(f"Unsupported file format: {filepath}")

def _load_spz(filepath: str) -> PointCloud:
    """Load SPZ file (gzip-compressed)."""
    with gzip.open(filepath, 'rb') as f:
        data = f.read()
    
    # Decode positions from 24-bit fixed-point
    # Assuming 3 bytes per coordinate, packed as b0 | (b1 << 8) | (b2 << 16)
    # Adjust for signedness if >= 0x800000
    # This is a simplified representation; actual implementation requires parsing the specific SPZ binary layout
    # For the purpose of this article, we assume a helper function exists for this specific binary format
    positions = _decode_spz_positions(data)
    
    # Decode colors from quantized spherical harmonics
    dc_u8 = _decode_spz_colors(data)
    f_dc = (dc_u8 / 255.0 - 0.5) / SPZ_COLOR_SCALE
    colors = np.clip((0.5 + SH_C0 * f_dc) * 255.0, 0, 255).astype(np.uint8)
    
    return PointCloud(positions=positions, colors=colors, opacities=None, is_3dgs=True)

def _load_ply(filepath: str) -> PointCloud:
    """Load PLY file (ASCII or binary)."""
    # Limit header reads to 10,000 lines to prevent infinite loops
    # Parse vertex element data
    # Extract positions, opacities (logits), and spherical harmonics coefficients
    # ... implementation details omitted for brevity, see source [S1, unofficial] ...
    pass

def decode_opacity(opacity_logits: np.ndarray) -> np.ndarray:
    """Activate sigmoid opacity from 3DGS logits."""
    return 1.0 / (1.0 + np.exp(-opacity_logits))

def decode_color(dc_coefficients: np.ndarray) -> np.ndarray:
    """Extract color from 3DGS DC spherical harmonics."""
    # dc_coefficients shape: (N,) or (N, 1)
    color_float = (0.5 + SH_C0 * dc_coefficients) * 255.0
    return np.clip(color_float, 0, 255).astype(np.uint8)

def voxelize(pc: PointCloud, resolution: float, min_opacity: float = 0.0, density_threshold: int = 1) -> Tuple[np.ndarray, np.ndarray]:
    """
    Voxelize the point cloud.
    Returns centers (M, 3) and colors (M, 3) of the resulting voxels.
    """
    if pc.is_3dgs and pc.opacities is not None:
        opacities = decode_opacity(pc.opacities)
        # Filter by min_opacity
        mask = opacities >= min_opacity
        positions = pc.positions[mask]
        colors = pc.colors[mask] if pc.colors is not None else None
    else:
        positions = pc.positions
        colors = pc.colors
    
    # Vectorized voxelization
    # 1. Compute voxel indices for each position
    voxel_indices = np.floor(positions / resolution).astype(np.int64)
    
    # 2. Aggregate splats into voxels
    # This step involves counting splats per voxel and averaging colors
    # If density_threshold > 1, filter out voxels with count < density_threshold
    # ... vectorized numpy operations to aggregate ...
    
    # Placeholder for the actual vectorized aggregation logic
    # The source emphasizes using numpy to avoid UI stalls
    centers, colors = _aggregate_voxels(voxel_indices, positions, colors, resolution, density_threshold)
    
    return centers, colors

def _aggregate_voxels(voxel_indices, positions, colors, resolution, density_threshold):
    """Aggregate splats into voxels using vectorized numpy."""
    # Get unique voxel indices and their counts
    unique_indices, counts = np.unique(voxel_indices, axis=0, return_counts=True)
    
    # Filter by density_threshold
    valid_mask = counts >= density_threshold
    valid_indices = unique_indices[valid_mask]
    
    # Compute centers (voxel index * resolution + resolution/2)
    centers = valid_indices * resolution + resolution / 2.0
    
    # Compute average colors for each valid voxel
    # This requires mapping each splat to its voxel and averaging
    # ... implementation details omitted ...
    
    return centers, None # Colors would be computed similarly

def author_voxel_instancer(stage, centers: np.ndarray, colors: np.ndarray, resolution: float, unit_scale: float = 1.0, physics: bool = False):
    """Author the voxel instancer in the USD stage."""
    # Create PointInstancer primitive
    # Set prototype to cube
    # Apply transform with unit_scale
    # Enable physics if requested
    # ... USD API calls ...
    pass
```

### Edge Cases and Pitfalls
- **PLY Parsing**: PLY files missing the `end_header` marker can cause infinite loops in ASCII parsing. The implementation limits header reads to 10,000 lines [S1, unofficial].
- **SPZ Compression**: SPZ files must be gzip-compressed. Opening non-gzip files raises an error [S1, unofficial].
- **Unit Mismatch**: USD Composer uses centimeters, while Isaac Sim uses meters. The `unit_scale` parameter handles this mismatch [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The voxelization step uses vectorized numpy operations, which prevents UI thread stalls for multi-million point splats [S1, unofficial]. The complexity is roughly linear with the number of splats $N$ and the number of voxels $M$.
- **Memory Complexity**: Depends on the number of splats and the resulting number of voxels. The `PointCloud` dataclass stores positions, colors, and opacities for all splats [S1, unofficial].
- **Strengths**: Direct integration with NVIDIA Omniverse; supports physics collisions; real-time visualization via USD Composer; handles both `.ply` and `.spz` formats [S1, unofficial].
- **Limitations**: Specific to 3DGS data; requires numpy and pxr libraries; not a general-purpose volume rendering method [S1, unofficial].

## Background
The 3DGS Voxelizer is an unofficial extension for NVIDIA Omniverse, developed by omolism [S1, unofficial]. It was created to bridge the gap between 3D Gaussian Splatting scans and real-time simulation environments like Isaac Sim, allowing for physics interactions with voxelized representations of the splat scenes [S1, unofficial].

## Sources
[S1] omolism/3dgs-voxelizer-Omni, https://github.com/omolism/3dgs-voxelizer-Omni, fetched 2026-10-27 (unofficial)
