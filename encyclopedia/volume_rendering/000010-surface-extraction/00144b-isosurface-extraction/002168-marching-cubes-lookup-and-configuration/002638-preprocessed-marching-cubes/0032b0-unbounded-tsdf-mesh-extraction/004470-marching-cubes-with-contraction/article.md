# Marching Cubes with Contraction

Also called: Unbounded TSDF Mesh Extraction with Contraction

## Overview
Extracts a 3D mesh from a Signed Distance Field (SDF) by processing the volume in fixed-size chunks to manage memory, applying the Marching Cubes algorithm locally, and applying an inverse contraction transformation to the final vertices [S1, unofficial]. It sets itself apart from standard Marching Cubes by handling unbounded scenes through space contraction and chunked evaluation, avoiding the need for a single massive global grid [S1, unofficial].

## How it works
**Inputs:**
- `resolution`: Total grid resolution (integer, must be divisible by 512). Default: 512 [S1, unofficial].
- `bounding_box_min`: Minimum coordinates of the bounding box (tuple of 3 floats). Default: (-1.0, -1.0, -1.0) [S1, unofficial].
- `bounding_box_max`: Maximum coordinates of the bounding box (tuple of 3 floats). Default: (1.0, 1.0, 1.0) [S1, unofficial].
- `inv_contraction`: Function to apply inverse contraction to vertices. Default: None [S1, unofficial].
- `max_range`: Maximum range for clipping vertices after contraction (float). Default: 32.0 [S1, unofficial].
- `sdf_function`: A callable that takes a tensor of points $(N, 3)$ and returns SDF values $(N)$ [G].

**Outputs:**
- A combined `trimesh.Trimesh` object containing the extracted mesh [S1, unofficial].

**Data Structures:**
- `xs, ys, zs`: 1D arrays of length $N+1$ defining chunk boundaries, where $N = \text{resolution} // 512$.
- `points`: Tensor of shape $(\text{cropN}^3, 3)$ containing grid coordinates for the current chunk.
- `pts_sdf`: Tensor of shape $(\text{cropN}^3)$ containing SDF values.
- `meshes`: List of `trimesh.Trimesh` objects.

**Steps:**
1. Assert that `resolution` is divisible by 512 [S1, unofficial].
2. Set `resN` = `resolution`, `cropN` = 512, and $N = \text{resN} // \text{cropN}$ [S1, unofficial].
3. Set `grid_min` = `bounding_box_min` and `grid_max` = `bounding_box_max` [S1, unofficial].
4. Create coordinate arrays `xs`, `ys`, `zs` using `np.linspace` from `grid_min` to `grid_max` with $N+1$ points [S1, unofficial].
5. Initialize an empty list `meshes` [G].
6. Loop over $i$ from 0 to $N-1$, $j$ from 0 to $N-1$, and $k$ from 0 to $N-1$:
    a. Define chunk boundaries:
       $$x_{\min} = xs[i], \quad x_{\max} = xs[i+1]$$
       $$y_{\min} = ys[j], \quad y_{\max} = ys[j+1]$$
       $$z_{\min} = zs[k], \quad z_{\max} = zs[k+1]$$
    b. Create 1D coordinate tensors `x`, `y`, `z` using `torch.linspace` from min to max with `cropN` points, moved to CUDA [S1, unofficial].
    c. Create a 3D meshgrid from `x`, `y`, `z` with `indexing='ij'` and stack the flattened coordinates into a `points` tensor of shape $(\text{cropN}^3, 3)$ on CUDA [S1, unofficial].
    d. Define an inner evaluate function that splits `points` into batches of size $256^3$, evaluates the SDF on each batch, and concatenates the results [S1, unofficial].
    e. Reshape `points` to $(\text{cropN}, \text{cropN}, \text{cropN}, 3)$ then to $(-1, 3)$ and evaluate the SDF to get `pts_sdf` [S1, unofficial].
    f. Check if the SDF crosses the isosurface level (assumed to be 0 for standard SDF extraction, or specified by `level`):
       If $\min(z) > \text{level}$ OR $\max(z) < \text{level}$, skip this chunk [S1, unofficial].
    g. Call `measure.marching_cubes` on `z` reshaped to $(\text{cropN}, \text{cropN}, \text{cropN})$ with `level` and spacing calculated from the chunk dimensions [S1, unofficial].
    h. Translate vertices by adding $[x_{\min}, y_{\min}, z_{\min}]$ and create a `trimesh.Trimesh` object, appending it to `meshes` [S1, unofficial].
7. Concatenate all meshes in `meshes` and merge vertices with 6 decimal digits precision [S1, unofficial].
8. If `inv_contraction` is not None:
    a. Convert combined vertices to a CUDA tensor and back to numpy [S1, unofficial].
    b. Apply `inv_contraction` to the vertices [S1, unofficial].
    c. Clip vertices to $[-\text{max\_range}, \text{max\_range}]$ [S1, unofficial].
9. Return the combined mesh [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import trimesh

def marching_cubes_with_contraction(
    resolution=512,
    bounding_box_min=(-1.0, -1.0, -1.0),
    bounding_box_max=(1.0, 1.0, 1.0),
    inv_contraction=None,
    max_range=32.0,
    sdf_function=None,
    level=0.0
):
    assert resolution % 512 == 0, "Resolution must be divisible by 512"
    
    resN = resolution
    cropN = 512
    N = resN // cropN
    
    grid_min = np.array(bounding_box_min)
    grid_max = np.array(bounding_box_max)
    
    xs = np.linspace(grid_min[0], grid_max[0], N + 1)
    ys = np.linspace(grid_min[1], grid_max[1], N + 1)
    zs = np.linspace(grid_min[2], grid_max[2], N + 1)
    
    meshes = []
    
    for i in range(N):
        for j in range(N):
            for k in range(N):
                x_min, x_max = xs[i], xs[i+1]
                y_min, y_max = ys[j], ys[j+1]
                z_min, z_max = zs[k], zs[k+1]
                
                x = torch.linspace(x_min, x_max, cropN, device='cuda')
                y = torch.linspace(y_min, y_max, cropN, device='cuda')
                z = torch.linspace(z_min, z_max, cropN, device='cuda')
                
                X, Y, Z = torch.meshgrid(x, y, z, indexing='ij')
                points = torch.stack([X.flatten(), Y.flatten(), Z.flatten()], dim=-1)
                
                # Evaluate SDF in batches of 256^3
                pts_sdf = []
                batch_size = 256**3
                for start in range(0, points.shape[0], batch_size):
                    end = min(start + batch_size, points.shape[0])
                    batch_points = points[start:end]
                    # sdf_function should return a tensor of shape (end-start,)
                    sdf_vals = sdf_function(batch_points)
                    pts_sdf.append(sdf_vals)
                pts_sdf = torch.cat(pts_sdf, dim=0)
                
                # Check if isosurface exists in this chunk
                z_vals = pts_sdf.reshape(cropN, cropN, cropN)
                min_z = z_vals.min()
                max_z = z_vals.max()
                
                if not (min_z > level and max_z < level):
                    # Marching Cubes
                    verts, faces, normals, values = measure.marching_cubes(
                        z_vals.cpu().numpy(),
                        level=level,
                        spacing=(
                            (x_max - x_min) / (cropN - 1),
                            (y_max - y_min) / (cropN - 1),
                            (z_max - z_min) / (cropN - 1)
                        )
                    )
                    
                    # Translate vertices to global coordinates
                    verts += np.array([x_min, y_min, z_min])
                    
                    mesh = trimesh.Trimesh(vertices=verts, faces=faces, vertex_normals=normals)
                    meshes.append(mesh)
    
    if not meshes:
        return trimesh.Trimesh()
        
    combined_mesh = trimesh.util.concatenate(meshes)
    # Merge vertices with 6 decimal digits precision
    combined_mesh.merge_vertices(merge_tex=True, merge_norm=True, round_ndigits=6)
    
    if inv_contraction is not None:
        verts = torch.tensor(combined_mesh.vertices, device='cuda', dtype=torch.float32)
        verts = inv_contraction(verts)
        combined_mesh.vertices = verts.cpu().numpy()
        
        # Clip vertices
        combined_mesh.vertices = np.clip(combined_mesh.vertices, -max_range, max_range)
        
    return combined_mesh
```

**Parameters:**
- `resolution`: Integer, default 512 [S1, unofficial].
- `bounding_box_min`: Tuple of 3 floats, default (-1.0, -1.0, -1.0) [S1, unofficial].
- `bounding_box_max`: Tuple of 3 floats, default (1.0, 1.0, 1.0) [S1, unofficial].
- `return_mesh`: Boolean flag, unused in logic shown [S1, unofficial].
- `simplify_mesh`: Boolean flag, unused in logic shown [S1, unofficial].
- `inv_contraction`: Function, default None [S1, unofficial].
- `max_range`: Float, default 32.0 [S1, unofficial].

**Pitfalls:**
- Resolution must be divisible by 512; otherwise, an assertion error occurs [S1, unofficial].
- If SDF values in a chunk are entirely above or below the level, no mesh is generated for that chunk, potentially leaving holes in the final mesh [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(R^3)$ where $R$ is the resolution, due to evaluating the SDF at every grid point. Chunking does not change asymptotic complexity but allows parallelization or out-of-core processing.
- **Memory Complexity:** $O(\text{cropN}^3)$ per chunk, where cropN is 512. This is significantly lower than storing a full $R^3$ grid if $R \gg 512$.
- **Strengths:** Handles unbounded scenes by contracting space and processing in chunks. Memory efficient for high-resolution grids.
- **Limitations:** Requires resolution to be a multiple of 512. Potential for holes if the SDF does not cross the isosurface level in a chunk. Relies on `trimesh` for mesh merging.
- **When to use:** When extracting meshes from large, unbounded SDF volumes where memory is a constraint.
- **When not to use:** When the resolution is not divisible by 512 or when the SDF is known to be sparse and a sparse representation would be more efficient.

## Background
This method is part of the VGVGGT framework, which integrates volume rendering with geometric extraction [S1, unofficial]. It builds on [Unbounded TSDF Mesh Extraction](6832-unbounded-tsdf-mesh-extraction.md) by adding the contraction mechanism and chunked marching cubes implementation.

## Sources
[S1] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
