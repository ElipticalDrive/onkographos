# Point Cloud I/O

Also called: PLY Point Cloud I/O, PLY Reader, PLY Writer

## Overview
Point Cloud I/O reads and writes point cloud data in the PLY format, extracting positions, colors, and normals [S1, unofficial]. It focuses on file format parsing and writing rather than spatial mapping [S2, unofficial].

## How it works
**Inputs:** A PLY file (ASCII or binary) containing vertex elements [S1, unofficial].
**Outputs:** Structured arrays for positions, colors, and normals; or a written PLY file [S1, unofficial].

**Data Structures:**
The PLY vertex element layout consists of the following fields [S1, unofficial][S2, unofficial]:
- `x`, `y`, `z`: 32-bit float (`f4`), representing position coordinates.
- `nx`, `ny`, `nz`: 32-bit float (`f4`), representing normal vector components.
- `red`, `green`, `blue`: 8-bit unsigned integer (`u1`), representing color intensity.

**Steps:**
1. Read vertices from the PLY data [S1, unofficial].
2. Stack `x`, `y`, `z` into a position array [S1, unofficial].
3. Stack `red`, `green`, `blue` into a color array and divide by 255.0 [S1, unofficial].
4. Stack `nx`, `ny`, `nz` into a normal array [S1, unofficial].

**Formulas:**
Color normalization during reading: $C_{rgb} = C_{uchar} / 255.0$, where $C_{uchar}$ is the stacked array of RGB values [S1, unofficial].
Color conversion during writing: $C_{uchar} = \text{clip}(C_{rgb} \times 255, 0, 255).astype(\text{uint8})$ [G].

**Parameters:** Not given in the sources.
**Edge cases and pitfalls:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import re

def read_ply_cloud(filepath):
    """
    Reads a PLY file and returns positions, colors (normalized), and normals.
    """
    with open(filepath, 'rb') as f:
        header = b''
        while True:
            line = f.readline()
            header += line
            if line.strip().lower() == b'end_header':
                break
        is_binary = b'binary_little_endian' in header
        match = re.search(rb'element vertex (\d+)', header)
        n = int(match.group(1)) if match else 0

        if is_binary:
            raw = f.read()
            dtype = np.dtype([('x', '<f4'), ('y', '<f4'), ('z', '<f4'),
                              ('nx', '<f4'), ('ny', '<f4'), ('nz', '<f4'),
                              ('red', 'u1'), ('green', 'u1'), ('blue', 'u1')])
            data = np.frombuffer(raw, dtype=dtype)
            positions = np.stack([data['x'], data['y'], data['z']], axis=-1)
            normals = np.stack([data['nx'], data['ny'], data['nz']], axis=-1)
            colors = np.stack([data['red'], data['green'], data['blue']], axis=-1).astype(np.float32) / 255.0
        else:
            lines = header.decode('ascii').splitlines()
            header_end_idx = next(i for i, l in enumerate(lines) if l.strip().lower() == 'end_header')
            ascii_data = ' '.join(lines[header_end_idx+1:])
            tokens = ascii_data.split()
            data = np.array(tokens, dtype=np.float32).reshape(-1, 9)
            positions = data[:, :3]
            normals = data[:, 3:6]
            colors = data[:, 6:9] / 255.0
    return positions, colors, normals

def write_ply_cloud(filepath, positions, colors, normals):
    """
    Writes a PLY file. Colors must be in [0, 1].
    """
    n = len(positions)
    colors_uchar = np.clip(colors * 255.0, 0, 255).astype(np.uint8)
    header = f"ply\nformat binary_little_endian 1.0\nelement vertex {n}\n"
    header += "property float x\nproperty float y\nproperty float z\n"
    header += "property float nx\nproperty float ny\nproperty float nz\n"
    header += "property uchar red\nproperty uchar green\nproperty uchar blue\nend_header\n"
    with open(filepath, 'wb') as f:
        f.write(header.encode('ascii'))
        dtype = np.dtype([('x', '<f4'), ('y', '<f4'), ('z', '<f4'),
                          ('nx', '<f4'), ('ny', '<f4'), ('nz', '<f4'),
                          ('red', 'u1'), ('green', 'u1'), ('blue', 'u1')])
        arr = np.zeros(n, dtype=dtype)
        arr['x'] = positions[:, 0]; arr['y'] = positions[:, 1]; arr['z'] = positions[:, 2]
        arr['nx'] = normals[:, 0]; arr['ny'] = normals[:, 1]; arr['nz'] = normals[:, 2]
        arr['red'] = colors_uchar[:, 0]; arr['green'] = colors_uchar[:, 1]; arr['blue'] = colors_uchar[:, 2]
        f.write(arr.tobytes())
```

## Subtopics

### [PLY Export Format](19138-ply-export-format.md)
Exports a 3D Gaussian cloud (positions, colors, opacities, scales, and rotations) to the PLY file format, enabling visualization in tools like CloudCompare, MeshLab, and Blender [G]. It differs from generic point cloud exports by including the full parametric definition of each Gaussian primitive (opacity, scale, rotation) rather than just position and color, and uses a binary little-endian layout for compactness [G].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Strengths and limitations: Not given in the sources.
When to use: Not given in the sources.

## Background
Origin: Not given in the sources.
History and development: Not given in the sources.
Where it is used: Not given in the sources.

## Sources
[S1] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
[S2] jibo27/3DGS_Hierarchical_Training, https://github.com/jibo27/3DGS_Hierarchical_Training, fetched 2026-10-27 (unofficial)
