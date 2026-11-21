# 3DGS Voxelizer

Also called: 3DGS to OpenUSD PointInstancer Voxelizer

## Overview
The 3DGS Voxelizer converts a 3D Gaussian Splatting (3DGS) scan into colored voxel blocks represented as OpenUSD `PointInstancer` data, enabling visualization and physics collision in NVIDIA Omniverse [S1, unofficial]. It differs from general-purpose voxelizers by specifically handling 3DGS spherical harmonics decoding and SPZ binary formats, and by outputting a USD-native instancer rather than a dense volume texture [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A 3DGS scan file in `.ply` (ASCII or binary) or `.spz` (gzip-compressed binary) format [S1, unofficial].
- Output: An OpenUSD stage containing a `PointInstancer` prim representing the voxelized scene [S1, unofficial].

**Data Structures**
The core intermediate representation is the `PointCloud` dataclass [S1, unofficial]:
- `positions`: `np.ndarray` of shape `(N, 3)`, dtype `float32` [S1, unofficial].
- `colors`: `np.ndarray` of shape `(N, 3)`, dtype `uint8` (optional) [S1, unofficial].
- `opacities`: `np.ndarray` of shape `(N,)`, dtype `float32` (optional, sigmoid-activated logits) [S1, unofficial].
- `is_3dgs`: `bool` indicating if the source is a 3DGS scan [S1, unofficial].

**Parameters**
- `resolution`: The physical size of each voxel block. Smaller values increase detail and block count [S1, unofficial].
- `min_opacity`: Threshold to filter faint splats. Only applicable for `.ply` files [S1, unofficial].
- `density_threshold`: Minimum number of splats required to create a voxel. Default is 1 [S1, unofficial].
- `unit_scale`: Handles unit mismatches (e.g., USD Composer uses centimeters, Isaac Sim uses meters) [S1, unofficial].

**Steps**
1. **Load Point Cloud**: Parse the input file into a `PointCloud` object.
   - For `.ply`: Handle ASCII (limit header reads to 10,000 lines to prevent infinite loops) or binary formats [S1, unofficial].
   - For `.spz`: Ensure the file is gzip-compressed; decode positions from 24-bit fixed-point and colors from scaled DC coefficients [S1, unofficial].
2. **Voxelize**: Compute voxel centers and colors based on resolution, thresholds, and input data using vectorized numpy operations [S1, unofficial].
3. **Author USD**: Create a `PointInstancer` prim in the USD stage using the computed centers, colors, and resolution [S1, unofficial].

**Formulas**
- **Color Extraction (3DGS DC Spherical Harmonics)**:
  $$C = \text{clip}((0.5 + SH\_C0 \cdot dc) \cdot 255.0, 0, 255)$$
  where $SH\_C0 = 0.28209479177387814$ and $dc$ is the stacked DC coefficient vector [S1, unofficial].
- **Opacity Activation**:
  $$O = \frac{1}{1 + e^{-\text{opacity}}}$$
  where `opacity` is the sigmoid-activated logit from the 3DGS data [S1, unofficial].
- **SPZ Color Recovery**:
  $$f\_dc = \frac{dc\_u8 / 255.0 - 0.5}{\_SPZ\_COLOR\_SCALE}$$
  $$C = \text{clip}((0.5 + SH\_C0 \cdot f\_dc) \cdot 255.0, 0, 255)$$
  where $\_SPZ\_COLOR\_SCALE = 0.15$ [S1, unofficial].
- **SPZ Position Decoding**:
  $$pos = \frac{\text{vals}}{2^{frac\_bits}}$$
  where $\text{vals}$ is reconstructed from 3 bytes $b_0, b_1, b_2$ as $b_0 | (b_1 \ll 8) | (b_2 \ll 16)$, adjusted for signedness if $\ge 0x800000$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import gzip
import struct

SH_C0 = 0.28209479177387814
SPZ_COLOR_SCALE = 0.15

def load_pointcloud(filepath):
    """Load a .ply or .spz file into a PointCloud dataclass."""
    if filepath.endswith('.spz'):
        return _load_spz(filepath)
    elif filepath.endswith('.ply'):
        return _load_ply(filepath)
    else:
        raise ValueError(f"Unsupported file format: {filepath}")

def _load_spz(filepath):
    """Decode SPZ format."""
    with gzip.open(filepath, 'rb') as f:
        data = f.read()
    
    # Decode positions (24-bit fixed-point)
    # Assuming 3 bytes per coordinate component, packed as b0, b1, b2
    # This is a simplified reconstruction; actual layout may vary
    num_points = len(data) // (3 * 3) # Assuming 3 components, 3 bytes each
    positions = []
    for i in range(num_points):
        vals = 0
        for j in range(3):
            byte_offset = i * 9 + j * 3
            b0 = data[byte_offset]
            b1 = data[byte_offset + 1]
            b2 = data[byte_offset + 2]
            val = b0 | (b1 << 8) | (b2 << 16)
            if val >= 0x800000:
                val -= 0x1000000
            positions.append(val)
    positions = np.array(positions).reshape(-1, 3).astype(np.float32)
    
    # Decode colors (DC coefficients)
    # Assuming 3 bytes per color component
    colors_u8 = np.frombuffer(data, dtype=np.uint8, offset=num_points * 9)
    # Reshape to (N, 3) for colors
    colors_u8 = colors_u8.reshape(-1, 3)
    dc_u8 = colors_u8.astype(np.float32)
    f_dc = (dc_u8 / 255.0 - 0.5) / SPZ_COLOR_SCALE
    colors = np.clip((0.5 + SH_C0 * f_dc) * 255.0, 0, 255).astype(np.uint8)
    
    return PointCloud(positions=positions, colors=colors, is_3dgs=True)

def _load_ply(filepath):
    """Load PLY format (ASCII or binary)."""
    # Simplified PLY loader; handles ASCII with header limit
    with open(filepath, 'rb') as f:
        header_lines = []
        for _ in range(10000): # Limit to prevent infinite loop
            line = f.readline()
            if line.startswith(b'end_header'):
                break
            header_lines.append(line)
        
        # Parse header to determine format and element count
        # ... (implementation details omitted for brevity) ...
        
        # Read vertex data
        # ... (implementation details omitted for brevity) ...
        
        # Assume positions, colors, opacities are extracted
        positions = np.zeros((0, 3), dtype=np.float32)
        colors = np.zeros((0, 3), dtype=np.uint8)
        opacities = np.zeros((0,), dtype=np.float32)
        
        return PointCloud(positions=positions, colors=colors, opacities=opacities, is_3dgs=True)

def voxelize(pc, resolution, min_opacity=0.0, density_threshold=1):
    """Voxelize a PointCloud into voxel centers and colors."""
    if pc.is_3dgs and pc.opacities is not None:
        # Apply opacity threshold
        opacities = 1.0 / (1.0 + np.exp(-pc.opacities))
        mask = opacities >= min_opacity
        positions = pc.positions[mask]
        colors = pc.colors[mask] if pc.colors is not None else None
    else:
        positions = pc.positions
        colors = pc.colors
    
    # Compute voxel centers
    # Assuming positions are in world units
    voxel_centers = np.floor(positions / resolution) * resolution
    
    # Aggregate by voxel center
    # ... (vectorized aggregation logic) ...
    
    return voxel_centers, colors

def author_voxel_instancer(stage, centers, colors, resolution, unit_scale=1.0):
    """Author a PointInstancer prim in the USD stage."""
    # ... (USD API calls to create PointInstancer) ...
    pass
```

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of points $N$ for loading and basic filtering. Voxelization uses vectorized numpy operations to prevent UI thread stalls with multi-million point splats [S1, unofficial].
- **Memory Complexity**: Proportional to the number of points and the resulting voxel count.
- **Strengths**: Integrates directly with NVIDIA Omniverse for visualization and physics; handles 3DGS-specific formats (SPZ, PLY with SH); undoable commands in Kit [S1, unofficial].
- **Limitations**: Requires Kit SDK 106 or newer; limited to USD output; not a general-purpose volume renderer [S1, unofficial].

## Background
The 3DGS Voxelizer is an unofficial extension for NVIDIA Omniverse, developed by omolism [S1, unofficial]. It was created to bridge the gap between 3D Gaussian Splatting scans and USD-based workflows, enabling use in tools like USD Composer and Isaac Sim [S1, unofficial].

## Sources
[S1] omolism/3dgs-voxelizer-Omni, https://github.com/omolism/3dgs-voxelizer-Omni, fetched 2026-10-27 (unofficial)
