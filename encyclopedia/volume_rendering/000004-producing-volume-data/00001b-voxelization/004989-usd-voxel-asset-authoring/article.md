# USD Voxel Asset Authoring

Also called: OpenUSD PointInstancer Voxelizer, danciVoxelizer

## Overview
USD Voxel Asset Authoring exports discrete voxel data as a USD asset using `UsdGeom.PointInstancer` to represent individual voxel instances [S1, unofficial]. It persists voxelized point cloud data into a standardized, metadata-rich format that supports live updates, collision detection, and metadata persistence [S1, unofficial]. It differs from direct array-based voxelization methods by leveraging USD's instancing capabilities to store sparse voxel positions and attributes efficiently, rather than dense grid arrays [S1, unofficial].

## How it works
The method takes a point cloud (positions, colors, opacities) and a set of generation parameters, voxelizes the points into a grid, and authors a USD stage containing a `PointInstancer` prim where each instance represents a voxel [S1, unofficial].

### Inputs and Outputs
- **Inputs**:
  - Point cloud data: Positions $(N, 3)$, Colors $(N, 3)$, Opacities $(N)$ [S1, unofficial].
  - Parameters: `resolution`, `min_opacity`, `density_threshold`, `origin`, `shape` (cube/sphere), `flip_axes`, `unit_scale` [S1, unofficial].
  - Source file: `.ply` or `.spz` (Niantic SPZ format) [S1, unofficial].
- **Outputs**:
  - A USD file (`.usd` or `.usda`) containing the voxel asset [S1, unofficial].

### Data Structures
1.  **Point Cloud**:
    -   `positions`: `np.float32` array of shape $(N, 3)$ [S1, unofficial].
    -   `colors`: `np.uint8` array of shape $(N, 3)$, values in $[0, 255]$ [S1, unofficial].
    -   `opacities`: `np.float32` array of shape $(N)$, values in $[0, 1]$ [S1, unofficial].
2.  **Voxel Grid**:
    -   `centers`: `np.float32` array of shape $(M, 3)$, voxel centers [S1, unofficial].
    -   `avg_colors`: `np.float32` array of shape $(M, 3)$, values in $[0, 1]$ [S1, unofficial].
    -   `origin`: `np.float32` array of shape $(3)$, grid origin [S1, unofficial].
3.  **USD Stage**:
    -   Root Xform with custom metadata [S1, unofficial].
    -   `UsdGeom.PointInstancer` prim with proto indices, positions, scales, and `displayColor` primvar [S1, unofficial].
    -   Prototype meshes (Cube or Sphere) [S1, unofficial].
    -   Material binding [S1, unofficial].

### Steps

1.  **Load Point Cloud**:
    -   If `.ply`: Parse vertices. If `f_dc_0` exists, recover RGB from spherical harmonics: $C = \text{clip}((0.5 + SH\_C0 \cdot dc) \cdot 255.0, 0, 255)$ [S1, unofficial]. If `opacity` exists, apply sigmoid: $O = 1.0 / (1.0 + e^{-opacity})$ [S1, unofficial].
    -   If `.spz`: Parse gzip-compressed binary. Read 16-byte header (magic `0x5053474E`, version, $N$, SH degree, frac bits) [S1, unofficial]. Read positions as 24-bit signed fixed-point: $P = \text{vals} / 2^{frac\_bits}$ [S1, unofficial]. Read alphas as u8 normalized to $[0, 1]$ [S1, unofficial]. Read colors as u8 SH DC: $C = \text{clip}((0.5 + SH\_C0 \cdot f\_dc) \cdot 255.0, 0, 255)$ where $f\_dc = (dc\_u8 / 255.0 - 0.5) / 0.15$ [S1, unofficial].

2.  **Voxelize**:
    -   Filter points: Keep if $opacity \ge min\_opacity$ [S1, unofficial].
    -   Compute grid indices: $G = \lfloor (P - origin) / resolution \rfloor$ [S1, unofficial].
    -   Shift grid to non-negative: $G' = G - \min(G)$ [S1, unofficial].
    -   Compute dimensions: $D = \max(G') + 1$ [S1, unofficial].
    -   Hash to unique keys: $K = (G'_x \cdot D_y + G'_y) \cdot D_z + G'_z$ [S1, unofficial].
    -   Group by key: Compute sum of colors and count per unique key [S1, unofficial].
    -   Average colors: $C_{avg} = \text{color\_sum} / count$ [S1, unofficial].
    -   Filter density: Keep if $count \ge density\_threshold$ [S1, unofficial].
    -   Reconstruct centers: $C_{center} = (G' + 0.5) \cdot resolution + origin$ [S1, unofficial].

3.  **Author USD Stage**:
    -   Create root Xform at `root_path` [S1, unofficial].
    -   Set metadata: `customDataByKey` with key `danciVoxelizer` and value `dict(params)` [S1, unofficial].
    -   Apply root transform: Set scale on `XformCommonAPI`. If `flip_axes[i]` is true, scale component is $-unit\_scale$, else $unit\_scale$ [S1, unofficial].
    -   Create prototype:
        -   If `shape` is cube: Define unit cube mesh (half-extent 1, 24 vertices, 6 faces, faceVarying normals, no subdivision) [S1, unofficial].
        -   If `shape` is sphere: Define icosphere mesh (12 vertices, 20 faces, Catmull-Clark subdivision) [S1, unofficial].
    -   Create material: `UsdPreviewSurface` with roughness 0.6, metallic 0.0, `diffuseColor` connected to `UsdPrimvarReader_float3` reading `displayColor` [S1, unofficial].
    -   Create PointInstancer:
        -   Set `protoIndices` to $[0, 0, \dots, 0]$ (length $M$) [S1, unofficial].
        -   Set `positions` to voxel centers [S1, unofficial].
        -   Set `scales` to $half \cdot [1, 1, 1]$ (where $half = resolution / 2$) [S1, unofficial].
        -   Set `displayColor` primvar to averaged colors [S1, unofficial].
        -   Set `extent` to $\min(centers) \pm half$ and $\max(centers) \pm half$ [S1, unofficial].
    -   Set prototype collision: If enabled, apply `UsdPhysics.CollisionAPI` and `UsdPhysics.MeshCollisionAPI` with approximation `boundingCube` to prototype [S1, unofficial].
    -   Flatten stage, copy subtree to new anonymous layer rooted at `/name`, set `defaultPrim`, `upAxis`, `metersPerUnit`, and export [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import struct
import gzip
from typing import Tuple, Dict, Any, Optional

SH_C0 = 0.28209479177387814
_SPZ_MAGIC = 0x5053474E
_SPZ_COLOR_SCALE = 0.15
METADATA_KEY = 'danciVoxelizer'

class PointCloud:
    def __init__(self, positions: np.ndarray, colors: Optional[np.ndarray], opacities: Optional[np.ndarray], is_3dgs: bool = False):
        self.positions = positions
        self.colors = colors
        self.opacities = opacities
        self.is_3dgs = is_3dgs

def load_ply(filepath: str) -> PointCloud:
    # Parse PLY header and vertex data
    # Extract positions (x,y,z), colors (r,g,b), opacities
    # If f_dc_0 exists, recover RGB: colors = clip((0.5 + SH_C0 * dc) * 255.0, 0, 255)
    # If opacity exists, apply sigmoid: opacities = 1.0 / (1.0 + exp(-opacity))
    # Return PointCloud object
    raise NotImplementedError("Detailed PLY parsing omitted for brevity; see [S1, unofficial]")

def load_spz(filepath: str) -> PointCloud:
    with gzip.open(filepath, "rb") as f:
        header = f.read(16)
        if len(header) < 16:
            raise ValueError("SPZ file too small")
        magic, version, n, sh_degree, frac_bits, _, _ = struct.unpack("<IIIBBBB", header)
        if magic != _SPZ_MAGIC or version not in (2, 3) or n == 0:
            raise ValueError(f"Invalid SPZ file (magic 0x{magic:08X}, version {version}, n={n})")
        data = f.read()
        # Positions: 24-bit signed fixed-point
        b = np.frombuffer(data, dtype=np.uint8, count=n * 9).reshape(n * 3, 3)
        vals = (b[:, 0].astype(np.int32) | (b[:, 1].astype(np.int32) << 8) | (b[:, 2].astype(np.int32) << 16))
        vals = np.where(vals >= 0x800000, vals - 0x1000000, vals)
        positions = (vals.astype(np.float32) / float(1 << frac_bits)).reshape(n, 3)
        # Alphas: u8
        opacities = (np.frombuffer(data, dtype=np.uint8, count=n, offset=n * 9).astype(np.float32) / 255.0)
        # Colors: u8 SH DC
        dc_u8 = np.frombuffer(data, dtype=np.uint8, count=n * 3, offset=n * 10).reshape(n, 3)
        f_dc = (dc_u8.astype(np.float32) / 255.0 - 0.5) / _SPZ_COLOR_SCALE
        colors = np.clip((0.5 + SH_C0 * f_dc) * 255.0, 0, 255).astype(np.uint8)
        return PointCloud(positions=positions, colors=colors, opacities=opacities, is_3dgs=True)

def load_pointcloud(filepath: str) -> PointCloud:
    if filepath.lower().endswith(".spz"):
        return load_spz(filepath)
    return load_ply(filepath)

def voxelize(pc: PointCloud, resolution: float, min_opacity: float = 0.0, density_threshold: int = 1, origin: Optional[np.ndarray] = None) -> Tuple[np.ndarray, np.ndarray, np.ndarray]:
    positions, colors, opacities = pc.positions, pc.colors, pc.opacities
    if opacities is not None and min_opacity > 0.0:
        keep = opacities >= min_opacity
        positions, colors = positions[keep], colors[keep] if colors is not None else None
    if len(positions) == 0:
        raise ValueError("Every point was removed by the opacity filter")
    if colors is None:
        z = positions[:, 2]
        t = (z - z.min()) / max(float(np.ptp(z)), 1e-8)
        colors = np.repeat((t[:, None] * 255).astype(np.uint8), 3, axis=1)
    if origin is None:
        origin = positions.min(axis=0)
    origin = np.asarray(origin, dtype=np.float32)
    grid = np.floor((positions - origin) / resolution).astype(np.int64)
    grid_offset = grid.min(axis=0)
    grid -= grid_offset
    dims = grid.max(axis=0) + 1
    keys = (grid[:, 0] * dims[1] + grid[:, 1]) * dims[2] + grid[:, 2]
    order = np.argsort(keys)
    keys_sorted = keys[order]
    unique_keys, start, counts = np.unique(keys_sorted, return_index=True, return_counts=True)
    csum = np.cumsum(colors[order].astype(np.float64), axis=0)
    csum = np.vstack([np.zeros(3), csum])
    color_sums = csum[start + counts] - csum[start]
    avg_colors = (color_sums / counts[:, None]).astype(np.float32) / 255.0
    if density_threshold > 1:
        keep = counts >= density_threshold
        unique_keys, avg_colors = unique_keys[keep], avg_colors[keep]
    gz = unique_keys % dims[2]
    gy = (unique_keys // dims[2]) % dims[1]
    gx = unique_keys // (dims[1] * dims[2])
    cells = np.column_stack([gx, gy, gz]) + grid_offset
    centers = (cells.astype(np.float32) + 0.5) * resolution + origin
    return centers, avg_colors, origin

def author_usd(centers: np.ndarray, avg_colors: np.ndarray, origin: np.ndarray, params: Dict[str, Any], out_path: str):
    stage = {}
    root_path = params.get("root_path", "/voxels")
    root = {"path": root_path, "type": "Xform"}
    metadata_key = "danciVoxelizer"
    root["customDataByKey"][metadata_key] = dict(params)
    unit_scale = params.get("unit_scale", 1.0)
    flip_axes = params.get("flip_axes", [False, False, False])
    scale = [-unit_scale if flip_axes[i] else unit_scale for i in range(3)]
    root["XformCommonAPI"]["scale"] = scale
    shape = params.get("shape", "cube")
    proto_path = root_path + "/prototype"
    if shape == "cube":
        pts = [(-1,-1,-1), (-1,1,-1), (1,1,-1), (1,-1,-1), (-1,-1,1), (1,-1,1), (1,1,1), (-1,1,1), (-1,-1,-1), (1,-1,-1), (1,-1,1), (-1,-1,1), (-1,1,-1), (-1,1,1), (1,1,1), (1,1,-1), (-1,-1,-1), (-1,-1,1), (-1,1,1), (-1,1,-1), (1,-1,-1), (1,1,-1), (1,1,1), (1,1,-1)]
        normals = [(0,0,-1)]*4 + [(0,0,1)]*4 + [(0,-1,0)]*4 + [(0,1,0)]*4 + [(-1,0,0)]*4 + [(1,0,0)]*4
        proto = {"path": proto_path, "type": "Mesh", "points": pts, "faceVertexCounts": [4]*6, "faceVertexIndices": list(range(24)), "normals": normals, "normalsInterpolation": "faceVarying", "subdivisionScheme": "none"}
    else:
        proto = {"path": proto_path, "type": "Mesh", "vertices": 12, "faces": 20, "subdivisionScheme": "catmull_clark"}
    mat_path = root_path + "/material"
    shader = {"id": "UsdPreviewSurface", "inputs": {"roughness": 0.6, "metallic": 0.0, "diffuseColor": {"source": "DisplayColorReader", "output": "result"}}}
    reader = {"id": "UsdPrimvarReader_float3", "inputs": {"varname": "displayColor"}}
    material = {"path": mat_path, "surfaceOutput": {"source": "PreviewSurface", "output": "surface"}}
    instancer_path = root_path + "/instancer"
    n = len(centers)
    half = params.get("resolution", 1.0) / 2.0
    instancer = {"path": instancer_path, "protoIndices": [0]*n, "positions": centers, "scales": np.full((n, 3), half), "displayColor": avg_colors, "extent": [np.min(centers, axis=0) - half, np.max(centers, axis=0) + half]}
    if params.get("enable_collision", False):
        instancer["proto_prim"]["CollisionAPI"] = {"approximation": "boundingCube"}
    else:
        if "CollisionAPI" in instancer["proto_prim"]: del instancer["proto_prim"]["CollisionAPI"]
    flatten(stage)
    copy_subtree(stage, root_path, "/name")
    set_default_prim(stage, "/name")
    set_up_axis(stage, "Y")
    set_meters_per_unit(stage, 1.0)
    export(stage, out_path)
```

### Parameters
-   `resolution`: Voxel grid cell size. Typical values depend on scene scale [S1, unofficial].
-   `min_opacity`: Minimum opacity threshold for point inclusion. Default 0.0 [S1, unofficial].
-   `density_threshold`: Minimum number of points per voxel to keep it. Default 1 [S1, unofficial].
-   `origin`: Grid origin. Defaults to point cloud minimum corner [S1, unofficial].
-   `shape`: Prototype shape ('cube' or 'sphere') [S1, unofficial].
-   `flip_axes`: List of booleans indicating axis flipping [S1, unofficial].
-   `unit_scale`: Scaling factor for root transform [S1, unofficial].
-   `METADATA_KEY`: String 'danciVoxelizer' [S1, unofficial].

### Edge Cases and Pitfalls
-   **Empty Cloud**: If all points are filtered by opacity, raise `ValueError` [S1, unofficial].
-   **Big-Endian PLY**: Not supported [S1, unofficial].
-   **SPZ Version**: Only versions 2 and 3 are supported [S1, unofficial].
-   **Grid Drift**: Use a fixed `origin` for sequences to prevent shimmering [S1, unofficial].

## Subtopics

### [Point Cloud Voxelization](1000-point-cloud-voxelization.md)
General method for voxelizing point clouds. Choose when you need a generic voxelization approach without USD-specific output.

### [VoxelGrid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md)
Voxelizes meshes by filling interior. Choose when your source is a mesh, not a point cloud.

### [Sparse and Hierarchical Structures](4391-sparse-and-hierarchical-structures.md)
Discusses sparse data structures for voxels. Choose when memory efficiency for large sparse volumes is critical.

### [Volumetric Attribute and Density Formation](6304-volumetric-attribute-and-density-formation.md)
Discusses how to form density and attributes in volumes. Choose when you need to understand the underlying density representation.

## Operating characteristics
-   **Time Complexity**: Voxelization is $O(N \log N)$ due to sorting keys, where $N$ is the number of points [S1, unofficial]. USD authoring is $O(M)$ where $M$ is the number of voxels [S1, unofficial].
-   **Memory Complexity**: $O(N)$ for point cloud storage, $O(M)$ for voxel storage [S1, unofficial].
-   **Performance**: Depends on point cloud size and resolution. Vectorized NumPy operations are used for efficiency [S1, unofficial].
-   **Strengths**: Standardized USD output, supports metadata, collision, and instancing. Round-trips with NVIDIA's reader [S1, unofficial].
-   **Limitations**: Only supports sparse voxel representation via instancing. Not suitable for dense volume rendering without additional processing. Limited to `.ply` and `.spz` input formats [S1, unofficial].
-   **When to Use**: When you need to export voxelized 3DGS or point cloud data to USD for use in USD-based pipelines (e.g., Omniverse) [S1, unofficial].
-   **When Not to Use**: When you need dense volume data for ray marching or when your source is a mesh [S1, unofficial].

## Background
The method is part of the `3dgs-voxelizer-Omni` tool, an unofficial implementation for converting 3D Gaussian Splatting data into USD voxel assets [S1, unofficial]. It was developed to facilitate the use of 3DGS in USD-based environments, leveraging USD's PointInstancer for efficient sparse voxel representation [S1, unofficial]. The tool ensures compatibility with NVIDIA's reader for round-tripping [S1, unofficial].

## Sources
[S1] omolism/3dgs-voxelizer-Omni, https://github.com/omolism/3dgs-voxelizer-Omni, fetched 2026-10-27 (unofficial)
