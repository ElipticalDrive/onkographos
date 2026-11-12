# Octree Sampled SDF Dataset


## Overview
A dataset class that generates signed-distance function (SDF) samples from a sparse octree acceleration structure constructed from a single mesh shape [S1, unofficial]. It limits sampling to occupied octree cells, supports equal distribution of samples per cell, and allows online resampling via a cached narrowband pool [S1, unofficial]. It optionally includes RGB data derived from mesh textures [S1, unofficial].

## How it works
**Inputs:**
- `occupancy_struct` (OctreeAS): An octree acceleration structure initialized from a mesh [S1, unofficial].
- `sample_mode` (List[str]): Sampling scheme. Supports `'tracer'` (on surface), `'near'` (near surface), `'rand'` (uniform in occupied grid cells) [S1, unofficial]. Default: `['rand', 'rand', 'near', 'near', 'trace']` [S1, unofficial].
- `num_samples` (int): Number of data points in the working set. Default: 100,000 [S1, unofficial].
- `samples_per_voxel` (int): Samples generated per octree grid cell for the preliminary pool. Default: 32 [S1, unofficial].
- `sample_tex` (bool): Whether to include RGB from mesh textures [S1, unofficial].

**Outputs:**
- A dataset of points with associated SDF values (and optionally RGB) suitable for neural field training [S1, unofficial].

**Data Structures:**
- `blas`: The OctreeAS object containing `points` (coordinates), `pyramid` (hierarchy), and `extent` (metadata) [S1, unofficial].
- `data_pool`: A cached pool of narrowband values used for online resampling [S1, unofficial].
- `data`: The current working set of samples [S1, unofficial].

**Steps:**
1.  **Extract Geometry and Level:** Retrieve vertices, faces, and the maximum octree level from the acceleration structure [S1, unofficial].
    $$
    \text{vertices} = \text{blas.extent['vertices']}, \quad \text{faces} = \text{blas.extent['faces']}, \quad \text{level} = \text{blas.max\_level}
    $$
2.  **Compute Voxel Corners:** Determine the corner coordinates for the target octree level [S1, unofficial].
    $$
    \text{corners} = \text{spc\_ops.unbatched\_get\_level\_points}(\text{blas.points}, \text{blas.pyramid}, \text{level})
    $$
3.  **Generate Initial Points:** Based on `sample_mode`, generate point sets:
    -   For `'near'` modes: Sample points near the surface with variance inversely proportional to the level [S1, unofficial].
        $$
        \text{variance} = \frac{1}{2^{\text{level}}}
        $$
    -   For `'trace'` modes: Sample points directly on the surface [S1, unofficial].
    -   For `'rand'` modes: Uniformly in occupied cells [S1, unofficial].
4.  **Filter by Narrowband:** Concatenate all generated points and query the acceleration structure to keep only those within the narrowband (occupied cells) [S1, unofficial].
    $$
    \text{query\_results} = \text{blas.query}(\text{pts}, 0)
    $$
    $$
    \text{pts} = \text{pts}[\text{query\_results.pidx} > -1]
    $$
5.  **Compute SDF and RGB:**
    -   If `sample_tex` is True, compute RGB using texture mapping [S1, unofficial].
    -   Compute SDF distances [S1, unofficial].
6.  **Resample (Online):** During training, resample the working set from the cached `data_pool` to maintain diversity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def init_octree_sdf_dataset(occupancy_struct, sample_mode=['rand', 'rand', 'near', 'near', 'trace'], 
                            num_samples=100000, samples_per_voxel=32, sample_tex=True):
    """
    Initialize the Octree Sampled SDF Dataset.
    
    Args:
        occupancy_struct (OctreeAS): The sparse octree acceleration structure.
        sample_mode (list): List of sampling strategies.
        num_samples (int): Size of the working set.
        samples_per_voxel (int): Samples per voxel for initial pool.
        sample_tex (bool): Include RGB.
        
    Returns:
        dict: The initialized dataset object containing 'data' and 'data_pool'.
    """
    # 1. Extract mesh geometry and octree level
    vertices = occupancy_struct.extent['vertices']
    faces = occupancy_struct.extent['faces']
    level = occupancy_struct.max_level
    
    # 2. Compute voxel corners for the target level
    # Note: spc_ops is assumed to be available in the environment
    corners = spc_ops.unbatched_get_level_points(occupancy_struct.points, occupancy_struct.pyramid, level)
    
    # 3. Generate points based on sample_mode
    all_pts = []
    num_pts_per_mode = num_samples // len(sample_mode)
    
    for mode in sample_mode:
        if mode == 'near':
            # Generate near-surface samples
            # Variance decreases as level increases (finer grid)
            variance = 1.0 / (2 ** level)
            pts = mesh_ops.sample_near_surface(
                vertices.cuda(), 
                faces.cuda(), 
                num_pts_per_mode, 
                variance=variance
            ).cpu()
            all_pts.append(pts)
        elif mode == 'trace':
            # Generate surface-traced samples
            pts, _ = mesh_ops.sample_surface(vertices.cuda(), faces.cuda(), num_pts_per_mode)
            all_pts.append(pts.cpu())
        elif mode == 'rand':
            # Uniform sampling in occupied grid cells (implementation detail omitted in source)
            # Placeholder for uniform random generation within active octree cells
            pts = generate_uniform_occupied_samples(occupancy_struct, num_pts_per_mode)
            all_pts.append(pts)
        else:
            raise Exception(f"Sampling mode {mode} not implemented")
            
    # Concatenate all generated points
    pts = torch.cat(all_pts, dim=0).cuda()
    
    # 4. Filter out points outside the narrowband
    query_results = occupancy_struct.query(pts, 0)
    valid_mask = query_results.pidx > -1
    pts = pts[valid_mask]
    
    # 5. Compute distances and optionally RGB
    if sample_tex:
        rgb, hit_pts, d = mesh_ops.closest_tex(
            vertices, faces, 
            occupancy_struct.texv, occupancy_struct.texf, 
            occupancy_struct.mats, pts
        )
    else:
        d = mesh_ops.compute_sdf(vertices, faces, pts)
        rgb = None # Or default color
        
    # 6. Initialize the working set and pool
    # The source indicates resampling from a pool. We initialize the pool with the initial samples.
    data_pool = {
        'pts': pts.clone().detach(),
        'sdf': d.clone().detach(),
        'rgb': rgb.clone().detach() if rgb is not None else None
    }
    
    # Initial working set is a subset or the full set depending on num_samples
    # Source implies 'num_samples' is the working set size, and pool is cached.
    # We truncate or sample from the pool for the initial 'data'
    pool_size = data_pool['pts'].shape[0]
    if pool_size > num_samples:
        idx = torch.randperm(pool_size, device='cuda')[:num_samples]
        data = {k: v[idx.to(v.device)] for k, v in data_pool.items() if v is not None}
    else:
        data = data_pool
        
    return {
        'data': data,
        'data_pool': data_pool,
        'num_samples': num_samples
    }

def resample_working_set(dataset_obj):
    """
    Resample the working set from the cached pool.
    """
    data_pool = dataset_obj['data_pool']
    num_samples = dataset_obj['num_samples']
    
    # Get pool size
    pool_size = data_pool['pts'].shape[0]
    
    # Random permutation
    idx = torch.randperm(pool_size - 1, device='cuda') # Source uses pool_size - 1
    
    # Truncate to num_samples if needed
    if num_samples < pool_size:
        idx = idx[:num_samples]
        
    # Resample
    new_data = {k: v[idx.to(v.device)] for k, v in data_pool.items() if v is not None}
    
    dataset_obj['data'] = new_data
    return dataset_obj
```

**Parameters:**
- `occupancy_struct`: Must be an `OctreeAS` initialized from a mesh. If not, `supports_blas` returns `False` and `validate` raises `RuntimeError` [S1, unofficial].
- `sample_mode`: List of strings. Unrecognized strings raise `Exception(f"Sampling mode {mode} not implemented")` [S1, unofficial].

## Operating characteristics
- **Memory:** Depends on `num_samples` and `samples_per_voxel`. The pool size is determined by the number of occupied cells times `samples_per_voxel` [S1, unofficial].
- **Performance:** Sampling is executed on mesh faces and limited to occupied octree cells, making it efficient for sparse representations [S1, unofficial].
- **Strengths:** Supports online resampling, which helps in training neural fields by preventing overfitting to specific surface regions [S1, unofficial].
- **Limitations:** Limited to single mesh shapes [S1, unofficial].

## Background
This method is part of the NVIDIA Kaolin-Wisp library, specifically designed for neural field training with sparse octree representations [S1, unofficial].

## Sources
[S1] NVIDIAGameWorks/kaolin-wisp, https://github.com/NVIDIAGameWorks/kaolin-wisp, fetched 2026-10-26 (unofficial)
