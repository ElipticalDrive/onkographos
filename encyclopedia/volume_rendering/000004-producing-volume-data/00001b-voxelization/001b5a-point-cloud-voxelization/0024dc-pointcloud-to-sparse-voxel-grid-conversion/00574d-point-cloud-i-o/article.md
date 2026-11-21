# Point Cloud I/O

Also called: PLY Point Cloud I/O, PLY Reader, PLY Writer

## Overview
Point Cloud I/O handles the serialization and deserialization of point cloud data, specifically extracting positions, colors, and normals from PLY files for voxelization [S1, unofficial]. It serves as the bridge between raw geometric data and the sparse voxel grid conversion process [Parent](3416-pointcloud-to-sparse-voxel-grid-conversion.md). Unlike [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md) or [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md) which perform the spatial mapping, this method focuses strictly on file format parsing and writing [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A PLY file (ASCII or binary) containing vertex elements.
**Outputs:** Structured arrays for positions, colors, and normals; or a written PLY file.

**Data Structures:**
The PLY vertex element layout consists of the following fields [S1, unofficial][S2, unofficial]:
- `x`, `y`, `z`: 32-bit float (`f4`), representing position coordinates.
- `nx`, `ny`, `nz`: 32-bit float (`f4`), representing normal vector components.
- `red`, `green`, `blue`: 8-bit unsigned integer (`u1`), representing color intensity.

**Steps:**

1. **Parse Header:** Read the PLY header to determine format (ASCII/Binary) and count the number of vertices $N$.
2. **Extract Positions:** Read the `x`, `y`, `z` fields. Stack them into a position array $P \in \mathbb{R}^{N \times 3}$.
3. **Extract Normals:** Read the `nx`, `ny`, `nz` fields. Stack them into a normal array $N \in \mathbb{R}^{N \times 3}$ [S1, unofficial].
4. **Extract Colors:** Read the `red`, `green`, `blue` fields.
   - If reading: Normalize the unsigned integers to the range $[0, 1]$ by dividing by 255.0.
     $$ C_{rgb} = \frac{C_{uchar}}{255.0} $$
     where $C_{uchar}$ is the stacked array of RGB values [S1, unofficial].
   - If writing: Clip the normalized color values to $[0, 1]$, multiply by 255, and cast to unsigned 8-bit integers.
     $$ C_{uchar} = \text{clip}(C_{rgb} \times 255, 0, 255).astype(\text{uint8}) $$
     [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def read_ply_cloud(filepath):
    """
    Reads a PLY file and returns positions, colors (normalized), and normals.
    """
    # Implementation depends on PLY library (e.g., plyfile or trimesh)
    # Assuming a generic parser that returns a dictionary of arrays
    data = parse_ply_header_and_data(filepath)
    
    # Extract positions
    x = data['x'].astype(np.float32)
    y = data['y'].astype(np.float32)
    z = data['z'].astype(np.float32)
    positions = np.stack([x, y, z], axis=-1)
    
    # Extract normals
    nx = data['nx'].astype(np.float32)
    ny = data['ny'].astype(np.float32)
    nz = data['nz'].astype(np.float32)
    normals = np.stack([nx, ny, nz], axis=-1)
    
    # Extract and normalize colors
    r = data['red'].astype(np.float32)
    g = data['green'].astype(np.float32)
    b = data['blue'].astype(np.float32)
    colors = np.stack([r, g, b], axis=-1) / 255.0
    
    return positions, colors, normals

def write_ply_cloud(filepath, positions, colors, normals, opacities=None, scales=None, rotations=None):
    """
    Writes a PLY file. Colors must be in [0, 1].
    """
    # Clip and scale colors for storage
    colors_uchar = np.clip(colors * 255.0, 0, 255).astype(np.uint8)
    
    # Prepare data arrays
    x = positions[:, 0]
    y = positions[:, 1]
    z = positions[:, 2]
    
    nx = normals[:, 0]
    ny = normals[:, 1]
    nz = normals[:, 2]
    
    r = colors_uchar[:, 0]
    g = colors_uchar[:, 1]
    b = colors_uchar[:, 2]
    
    # Construct header
    header_lines = [
        "ply",
        "format binary_little_endian 1.0",
        f"element vertex {len(positions)}",
        "property float x",
        "property float y",
        "property float z",
        "property uchar red",
        "property uchar green",
        "property uchar blue"
    ]
    
    # Add optional properties if present
    if opacities is not None:
        header_lines.append("property float opacity")
    if scales is not None:
        header_lines.append("property float scale_0")
        header_lines.append("property float scale_1")
        header_lines.append("property float scale_2")
    if rotations is not None:
        header_lines.append("property float rot_0")
        header_lines.append("property float rot_1")
        header_lines.append("property float rot_2")
        header_lines.append("property float rot_3")
        
    header_lines.append("end_header")
    header = "\n".join(header_lines) + "\n"
    
    # Write binary data
    with open(filepath, 'wb') as f:
        f.write(header.encode('ascii'))
        
        # Pack positions: '<fff' (3 floats)
        pos_bytes = np.packbits([]) # Placeholder for actual packing logic
        # In practice, use numpy struct packing or similar
        # positions.tofile(f) assumes C-order float32
        
        # Pack colors: '<BBB' (3 unsigned chars)
        # colors_uchar.tofile(f)
        
        # Pack optional fields if needed
        # ...
```

## Subtopics

### PLY Export Format
This variant specifically addresses the writing of Gaussian Splatting data (GaussianPoints) to PLY. It extends the standard vertex layout with optional properties for opacity, scale, and rotation to support downstream tools like CloudCompare, MeshLab, or Blender [S3, unofficial]. The header dynamically includes `property float opacity`, `property float scale_0/1/2`, and `property float rot_0/1/2/3` if those attributes exist in the source data [S3, unofficial]. A critical pitfall is the color conversion: internal Gaussian representations often store colors in $[0,1]$, but the PLY format requires `uchar` $[0,255]$. The code must explicitly clip and cast: `np.clip(colors[i] * 255, 0, 255).astype(np.uint8)` [S3, unofficial].

## Operating characteristics
**Time Complexity:** Linear with respect to the number of vertices $O(N)$ for both reading and writing, as each vertex attribute is processed once.
**Memory Complexity:** $O(N)$ to store the raw arrays in memory during I/O.
**Strengths:** Standardized format support allows interoperability with a wide range of 3D tools.
**Limitations:** PLY is a point-based format; it does not inherently store voxel grid topology. It is a precursor step to voxelization, not the voxelization itself.
**When to use:** When initializing a sparse voxel grid from raw scan data or exporting optimized Gaussian models for visualization.

## Background
Originating from the Stanford PLY format specification, this method is widely used in 3D computer graphics for exchanging point cloud data. In the context of modern 3D Gaussian Splatting and volume rendering pipelines, it is used to load initial point clouds for densification and to export trained models for inspection [S1, unofficial][S3, unofficial]. The specific implementation details regarding Gaussian properties (opacity, scale, rotation) in PLY headers reflect adaptations for 3DGS visualization tools [S3, unofficial].

## Sources
[S1] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
[S2] jibo27/3DGS_Hierarchical_Training, https://github.com/jibo27/3DGS_Hierarchical_Training, fetched 2026-10-27 (unofficial)
[S3] Laityperfect7/GaussianScan-Recon, https://github.com/Laityperfect7/GaussianScan-Recon, fetched 2026-10-27 (unofficial)
