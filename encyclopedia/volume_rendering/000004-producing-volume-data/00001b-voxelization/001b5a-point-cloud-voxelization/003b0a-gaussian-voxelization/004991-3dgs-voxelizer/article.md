# 3DGS Voxelizer

Also called: 3DGS to OpenUSD PointInstancer Voxelizer

## Overview
The 3DGS Voxelizer converts a 3D Gaussian Splatting (3DGS) scan, stored as a `.ply` or `.spz` file, into colored voxel blocks represented as OpenUSD `PointInstancer` data [S1, unofficial]. It solves the problem of making 3DGS scenes interactive in physics-enabled environments (such as NVIDIA Isaac Sim) and compatible with standard USD pipelines, which do not natively support Gaussian primitives [S1, unofficial]. Unlike [3D Gaussian Voxelization](7020-3d-gaussian-voxelization.md) which may target generic volume formats, this method specifically targets the `PointInstancer` USD schema for immediate visualization and collision detection in Omniverse [S1, unofficial].

## How it works
The method consists of three phases: loading and decoding the source format, vectorized voxelization, and USD authoring.

### Inputs and Outputs
*   **Input**: A 3DGS scan file (`.ply` or `.spz`) containing positions, spherical harmonic coefficients (for color), and opacity logits.
*   **Output**: An OpenUSD stage containing a `PointInstancer` prim representing the voxelized scene.

### Data Structures
The core intermediate structure is the `PointCloud` dataclass [S1, unofficial]:
*   `positions`: `numpy.ndarray` of shape `(N, 3)`, dtype `float32`.
*   `colors`: `numpy.ndarray` of shape `(N, 3)`, dtype `uint8` (0-255 range).
*   `opacities`: `numpy.ndarray` of shape `(N,)`, dtype `float32` (sigmoid-activated values).
*   `is_3dgs`: `bool`, indicating if the source contains 3DGS-specific attributes (SH coefficients).

### Parameters
*   `resolution`: Float. The physical size of each voxel block. Smaller values increase detail but also increase the number of blocks and processing time [S1, unofficial].
*   `min_opacity`: Float. Threshold to filter faint splats. Applied only to `.ply` files where opacities are sigmoid-activated logits [S1, unofficial].
*   `density_threshold`: Int. Minimum number of splats required to create a voxel. Default is 1 [S1, unofficial].
*   `unit_scale`: Float. Scaling factor to handle unit mismatches between USD Composer (centimeters) and Isaac Sim (meters) [S1, unofficial].

### Steps

1.  **Load Point Cloud**:
    *   If `.spz`: Decompress gzip. Decode positions from 24-bit fixed-point integers. Decode DC spherical harmonic coefficients using `_SPZ_COLOR_SCALE`.
    *   If `.ply`: Parse binary or ASCII format. Handle header termination (limit to 10,000 lines to prevent infinite loops on malformed files) [S1, unofficial].
    *   Construct the `PointCloud` object.

2.  **Decode Attributes**:
    *   **Opacity**: Apply sigmoid activation to logits if not already done:
        $$O = \frac{1}{1 + e^{-\text{opacity}}}$$
    *   **Color**: Extract color from DC spherical harmonic coefficient $dc$ using constant $SH\_C0 = 0.28209479177387814$:
        $$C = \text{clip}\left( (0.5 + SH\_C0 \cdot dc) \cdot 255.0, 0, 255 \right)$$
        For `.spz` files, first recover $dc$:
        $$f\_dc = \frac{dc\_u8 / 255.0 - 0.5}{0.15}$$
        Then apply the color formula above [S1, unofficial].

3.  **Voxelize**:
    *   Use vectorized NumPy operations to map points to voxel centers based on `resolution`.
    *   Filter points where $O < \text{min\_opacity}$.
    *   Aggregate points into voxels. If multiple points map to the same voxel, aggregate their colors and opacities (implementation details for aggregation beyond filtering are not explicitly detailed in the source, but standard weighted averaging is implied by the family principles) [S1, unofficial].
    *   Enforce `density_threshold`: discard voxels with fewer contributing splats than the threshold.

4.  **Author USD**:
    *   Create a `PointInstancer` prim in the USD stage.
    *   Set the prototype shape (typically a cube).
    *   Populate the `points` and `colors` arrays using `Vt.Vec3fArray` and `Vt.Vec3fArray` (or equivalent) from the computed centers and colors.
    *   Apply `unit_scale` and optional axis flips via an `Xform` prim.
    *   Optionally enable `UsdPhysics` collision APIs on the prototype [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import gzip
import struct
from dataclasses import dataclass
from pxr import Usd, UsdGeom, UsdPhysics, Vt

SH_C0 = 0.28209479177387814
SPZ_COLOR_SCALE = 0.15

@dataclass
class PointCloud:
    positions: np.ndarray  # (N, 3) float32
    colors: np.ndarray     # (N, 3) uint8
    opacities: np.ndarray  # (N,) float32
    is_3dgs: bool

def load_pointcloud(filepath: str) -> PointCloud:
    """Load PLY or SPZ file into PointCloud."""
    if filepath.endswith('.spz'):
        return _load_spz(filepath)
    else:
        return _load_ply(filepath)

def _load_spz(filepath: str) -> PointCloud:
    with gzip.open(filepath, 'rb') as f:
        data = f.read()
    
    # Decode positions: 24-bit fixed point
    # Reconstruct from 3 bytes b0, b1, b2
    # vals = b0 | (b1 << 8) | (b2 << 16)
    # Adjust for signedness if vals >= 0x800000
    # pos = vals / 2^frac_bits (frac_bits not specified, assumed standard)
    # [G] Implementation detail: frac_bits is typically 8 or similar for SPZ; 
    # exact value not given in sources. Assuming standard 24-bit int decoding.
    
    # Decode DC coefficients
    # dc_u8 is stored as uint8
    # f_dc = (dc_u8 / 255.0 - 0.5) / SPZ_COLOR_SCALE
    # color = clip((0.5 + SH_C0 * f_dc) * 255.0, 0, 255)
    
    # [G] Specific byte offsets and counts for SPZ are not provided in the text.
    # This step requires parsing the specific SPZ binary layout.
    raise NotImplementedError("SPZ binary layout details not fully specified in source.")

def _load_ply(filepath: str) -> PointCloud:
    # Parse PLY header (limit 10,000 lines to avoid infinite loop)
    # Parse vertices
    # Extract positions, opacities (logits), and SH coefficients
    # Decode color using SH_C0
    # Return PointCloud
    raise NotImplementedError("PLY parsing logic depends on specific binary/ascii structure not fully detailed.")

def voxelize(pc: PointCloud, resolution: float, min_opacity: float, density_threshold: int) -> tuple:
    """Voxelize point cloud into centers and colors."""
    # Filter by opacity
    mask = pc.opacities >= min_opacity
    positions = pc.positions[mask]
    colors = pc.colors[mask]
    
    # Compute voxel indices
    # [G] Assuming positions are in world units.
    # voxel_idx = floor(positions / resolution)
    # [G] Exact aggregation logic (e.g., mean color, max opacity) is not specified.
    # Standard approach: group by voxel_idx, aggregate.
    
    # Filter by density
    # [G] Implementation of density threshold filtering is not detailed.
    
    # Return centers and colors
    return positions, colors

def author_voxel_instancer(stage: Usd.Stage, centers: np.ndarray, colors: np.ndarray, resolution: float, unit_scale: float = 1.0) -> UsdGeom.PointInstancer:
    """Create PointInstancer prim in USD stage."""
    # Create root Xform
    root = UsdGeom.Xform.Define(stage, Sdf.Path("/VoxelRoot"))
    
    # Create PointInstancer
    instancer = UsdGeom.PointInstancer.Define(stage, Sdf.Path("/VoxelRoot/Instancer"))
    
    # Set prototype
    # [G] Prototype shape creation details not specified.
    
    # Set points and colors
    points_array = Vt.Vec3fArray.FromNumpy(np.ascontiguousarray(centers * unit_scale, dtype=np.float32))
    colors_array = Vt.Vec3fArray.FromNumpy(np.ascontiguousarray(colors / 255.0, dtype=np.float32))
    
    instancer.CreatePointsAttr().Set(points_array)
    instancer.CreateColorsAttr().Set(colors_array)
    
    # Set resolution as scale or custom attribute
    # [G] Exact mapping of resolution to USD attributes not specified.
    
    return instancer
```

### Edge Cases and Pitfalls
*   **PLY Header**: PLY files lacking an `end_header` marker can cause infinite loops in ASCII parsing. The implementation limits header reads to 10,000 lines [S1, unofficial].
*   **SPZ Compression**: SPZ files must be gzip-compressed. Opening non-gzip files raises an error [S1, unofficial].
*   **Unit Mismatch**: USD Composer uses centimeters, while Isaac Sim uses meters. The `unit_scale` parameter must be adjusted (e.g., to 100) to correct this [S1, unofficial].
*   **Performance**: Non-vectorized approaches stall the UI thread with multi-million point splats. This method uses vectorized NumPy operations to prevent this [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of points $N$ for loading and decoding. Voxelization is vectorized, but the number of blocks increases as $1/\text{resolution}^3$.
*   **Memory Complexity**: Proportional to the number of resulting voxels.
*   **Strengths**: Direct integration with NVIDIA Omniverse and Isaac Sim; supports physics collision; undoable commands in Kit; works outside Omniverse with `usd-core` and `numpy` [S1, unofficial].
*   **Limitations**: Specific to OpenUSD `PointInstancer` schema; requires Kit SDK 106+ for the extension; `.spz` format support is limited to gzip-compressed files [S1, unofficial].
*   **When to use**: When 3DGS scenes need to be visualized or used for collision in Omniverse/Isaac Sim.
*   **When not to use**: For real-time rendering in game engines without USD support, or for high-fidelity volume rendering where Gaussian splatting is preferred over voxel approximation.

## Background
The 3DGS Voxelizer is part of the `3dgs-voxelizer-Omni` extension by omolism [S1, unofficial]. It was developed to bridge the gap between 3D Gaussian Splatting scans and physics-enabled USD environments like NVIDIA Isaac Sim [S1, unofficial]. The pipeline is designed to run within NVIDIA Kit-based applications (USD Composer, Isaac Sim) but can also be used as a standalone Python script with `usd-core` [S1, unofficial].

## Sources
[S1] omolism/3dgs-voxelizer-Omni, https://github.com/omolism/3dgs-voxelizer-Omni, fetched 2026-10-27 (unofficial)
