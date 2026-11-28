# PLY Export Format

## Overview
Exports a 3D Gaussian cloud (positions, colors, opacities, scales, and rotations) to the PLY file format, enabling visualization in tools like CloudCompare, MeshLab, and Blender. It differs from generic point cloud exports by including the full parametric definition of each Gaussian primitive (opacity, scale, rotation) rather than just position and color, and uses a binary little-endian layout for compactness.

## How it works
**Inputs:** A Gaussian cloud object containing arrays for positions ($N \times 3$), colors ($N \times 3$ in $[0,1]$), and optional arrays for opacities ($N \times 1$), scales ($N \times 3$), and rotations ($N \times 4$, quaternions).
**Outputs:** A binary PLY file.

**Data Structures:**
The file consists of an ASCII header followed by binary vertex data.
- **Header:** ASCII text defining the format and properties.
- **Vertex Data:** Binary records for each of the $N$ Gaussians.

**Steps:**
1. **Write Header:**
   - Write the string `"ply\n"`.
   - Write the string `"format binary_little_endian 1.0\n"`.
   - Write the string `"element vertex %d\n"` where `%d` is the number of Gaussians $N$.
   - Write the position properties:
     - `"property float x\n"`
     - `"property float y\n"`
     - `"property float z\n"`
   - Write the color properties:
     - `"property uchar red\n"`
     - `"property uchar green\n"`
     - `"property uchar blue\n"`
   - If opacities are present, write:
     - `"property float opacity\n"`
   - If scales are present, write:
     - `"property float scale_0\n"`
     - `"property float scale_1\n"`
     - `"property float scale_2\n"`
   - If rotations are present, write:
     - `"property float rot_0\n"`
     - `"property float rot_1\n"`
     - `"property float rot_2\n"`
     - `"property float rot_3\n"`
   - Write the string `"end_header\n"`.

2. **Write Vertex Data:**
   For each Gaussian $i$ from $0$ to $N-1$:
   - **Position:** Write `positions[i]` as three little-endian floats (`<fff`).
   - **Color:**
     - Scale `colors[i]` by 255.
     - Clip values to the range $[0, 255]$.
     - Cast to unsigned 8-bit integers (`uint8`).
     - Write as three unsigned chars (`<BBB`).
   - **Opacity (if present):** Write `opacities[i]` as one little-endian float (`<f`).
   - **Scale (if present):** Write `scales[i]` as three little-endian floats (`<fff`).
   - **Rotation (if present):** Write `rotations[i]` as four little-endian floats (`<ffff`).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import numpy as np

def export_ply(filename, positions, colors, opacities=None, scales=None, rotations=None):
    """
    Export Gaussian cloud to PLY format.
    
    Args:
        filename (str): Output file path.
        positions (np.ndarray): Shape (N, 3), float32.
        colors (np.ndarray): Shape (N, 3), float32 in [0, 1].
        opacities (np.ndarray, optional): Shape (N, 1), float32.
        scales (np.ndarray, optional): Shape (N, 3), float32.
        rotations (np.ndarray, optional): Shape (N, 4), float32 (quaternions).
    """
    N = positions.shape[0]
    
    # Build header
    header_lines = [
        "ply",
        "format binary_little_endian 1.0",
        f"element vertex {N}",
        "property float x",
        "property float y",
        "property float z",
        "property uchar red",
        "property uchar green",
        "property uchar blue"
    ]
    
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
    
    # Prepare binary data
    # Positions: 3 floats
    pos_data = positions.astype(np.float32).tobytes()
    
    # Colors: 3 unsigned chars, clipped and scaled
    # Clip to [0, 255] and cast to uint8
    colors_uint8 = np.clip(colors * 255.0, 0, 255).astype(np.uint8)
    color_data = colors_uint8.tobytes()
    
    # Opacity: 1 float
    opacity_data = b""
    if opacities is not None:
        opacity_data = opacities.astype(np.float32).tobytes()
        
    # Scale: 3 floats
    scale_data = b""
    if scales is not None:
        scale_data = scales.astype(np.float32).tobytes()
        
    # Rotation: 4 floats
    rotation_data = b""
    if rotations is not None:
        rotation_data = rotations.astype(np.float32).tobytes()
        
    # Write file
    with open(filename, 'wb') as f:
        f.write(header.encode('ascii'))
        f.write(pos_data)
        f.write(color_data)
        f.write(opacity_data)
        f.write(scale_data)
        f.write(rotation_data)
```

**Parameters:**
- `positions`: Required. $N \times 3$ array of Gaussian centers.
- `colors`: Required. $N \times 3$ array of RGB values in $[0, 1]$.
- `opacities`: Optional. $N \times 1$ array of scalar opacity values.
- `scales`: Optional. $N \times 3$ array of scale factors.
- `rotations`: Optional. $N \times 4$ array of quaternion components $(w, x, y, z)$ or $(x, y, z, w)$ depending on convention; the source uses `rot_0` to `rot_3` without specifying order, but standard PLY export for 3DGS typically follows the internal storage order [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Color Clipping:** Colors must be clipped to $[0, 255]$ after scaling by 255 to prevent overflow when casting to `uint8` [S1, S1, unofficial].
- **Optional Fields:** Opacity, scale, and rotation properties are only included in the header and data if the corresponding flags (`has_opacities`, `has_scales`, `has_rotations`) are true [S1, S1, unofficial].
- **Endianness:** The format is explicitly `binary_little_endian` [S1, S1, unofficial].

## Subtopics

### PLY File Writer for Gaussian Clouds
Writes a Gaussian cloud to a PLY file format, including positions, RGB colors, opacity, scale, and rotation parameters. It is identical to the main method but explicitly emphasizes the conditional writing of optional fields (opacity, scale, rotation) based on availability flags. It is chosen when the Gaussian cloud contains full parametric data and needs to be preserved in a standard visualization format. Link: [PLY Export Format](15927-ply-export-format.md).

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the number of Gaussians. The operation involves a single pass to write the header and a single pass to write the binary data.
**Memory Complexity:** $O(N)$ for buffering the binary data. The header is small and constant size relative to $N$.
**Performance:** Writing is fast as it involves simple memory copies and minimal computation (color clipping). The binary format is compact, reducing I/O time compared to ASCII.
**Strengths:** Standard compatibility with 3D viewers; compact binary storage; preserves full Gaussian parameters for downstream processing.
**Limitations:** Not a streaming format; requires loading all data into memory. The PLY format does not natively support the hierarchical structure of sparse voxel grids, making it suitable for point-based visualization rather than efficient volume rendering storage.

## Background
The PLY (Polygon File Format) is a common format for 3D geometric data. The specific export method for 3D Gaussian Splatting clouds was popularized by implementations such as GaussianScan-Recon, which adapted the standard PLY structure to include Gaussian-specific parameters (opacity, scale, rotation) alongside position and color [S1, unofficial]. This allows tools like MeshLab and CloudCompare to visualize the Gaussians as points with additional attributes, facilitating debugging and inspection of the reconstruction.

## Sources
[S1] Laityperfect7/GaussianScan-Recon, https://github.com/Laityperfect7/GaussianScan-Recon, fetched 2026-10-27 (unofficial)
