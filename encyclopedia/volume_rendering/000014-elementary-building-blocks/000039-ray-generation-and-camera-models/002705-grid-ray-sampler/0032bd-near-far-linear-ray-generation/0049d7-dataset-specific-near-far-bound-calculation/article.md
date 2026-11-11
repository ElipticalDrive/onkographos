# Dataset-Specific Near/Far Bound Calculation

Also called: Dataset-dependent Near/Far Clipping, Near/Far Bound Calculation per Dataset, Scene Normalization, Scene contraction strategy

## Overview
Computes the near and far clipping planes for ray sampling based on the dataset type and Normalized Device Coordinates (NDC) configuration [S2, unofficial]. It ensures that the ray sampler operates within a geometrically valid range appropriate for the scene's scale and projection model, serving as the parameterization layer for [Near-Far Linear Ray Generation](6841-near-far-linear-ray-generation.md).

## How it works
**Inputs:**
- `dataset_type`: String identifier for the dataset ('llff', 'blender', 'deepvoxels').
- `no_ndc`: Boolean flag. If `True`, disables NDC projection (used for non-forward-facing scenes) [S2, unofficial].
- `bds`: Array of bounding box min/max values derived from camera poses [S2, unofficial].
- `poses`: Array of camera transformation matrices [S2, unofficial].

**Outputs:**
- `near`: Float, the near clipping plane distance.
- `far`: Float, the far clipping plane distance.

**Data Structures:**
- `bds`: Shape `[3]` or `[num_cameras, 3]` containing min/max extents. In LLFF loading, `bds` is rescaled by `bd_factor` (default 0.75) if provided [S2, unofficial].
- `poses`: Shape `[num_cameras, 4, 4]` or `[num_cameras, 3, 4]` representing camera extrinsics.

**Steps:**
1. **LLFF Dataset (`dataset_type == 'llff'`):**
   - If `no_ndc` is `True`:
     - Calculate `near = min(bds) * 0.9` [S2, unofficial].
     - Calculate `far = max(bds) * 1.0` [S2, unofficial].
     - *Note:* The scaling factors `0.9` and `1.0` are explicit bounding box adjustments required when NDC is disabled to prevent clipping artifacts [S2, unofficial].
   - If `no_ndc` is `False` (NDC enabled):
     - Set `near = 0.0` [S2, unofficial].
     - Set `far = 1.0` [S2, unofficial].
     - *Note:* NDC scenes map the scene to a fixed unit cube, making bounds independent of physical scale [S2, unofficial].

2. **Blender Dataset (`dataset_type == 'blender'`):**
   - Set `near = 2.0` [S2, unofficial].
   - Set `far = 6.0` [S2, unofficial].
   - *Note:* These are hardcoded empirical values for the standard Blender synthetic dataset scale [S2, unofficial].

3. **DeepVoxels Dataset (`dataset_type == 'deepvoxels'`):**
   - Compute the mean radius of the camera trajectory:
     $$ \text{hemi\_R} = \frac{1}{N} \sum_{i=1}^{N} \| \text{poses}_i[:, 3] \|_2 $$
     where $\text{poses}_i[:, 3]$ is the translation vector (4th column) of the $i$-th camera pose [S2, unofficial].
   - Calculate `near = hemi_R - 1.0` [S2, unofficial].
   - Calculate `far = hemi_R + 1.0` [S2, unofficial].

4. **Scene Normalization (Unbounded Scenes):**
   - For unbounded scenes (e.g., street scenes in LidaRF), calculate the maximum distance $D_{max}$ between any two camera positions in the training log [S1].
   - Determine sphere diameter $D_{sphere} = D_{max} + 50$ meters [S1].
   - This diameter defines the contraction sphere used to map unbounded space into a bounded domain for the neural field [S1].

**Pitfalls:**
- **LLFF NDC vs. Non-NDC:** Using fixed [0, 1] bounds for LLFF when `no_ndc=True` causes incorrect sampling ranges because the physical scale is not normalized [S2, unofficial]. Conversely, using physical bounds with `no_ndc=False` breaks the NDC assumption [S2, unofficial].
- **Pose Extraction:** Ensure `poses[:, :3, -1]` correctly extracts the translation component from the transformation matrix [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_near_far(dataset_type, no_ndc, bds, poses):
    """
    Compute near and far clipping planes based on dataset type.
    
    Args:
        dataset_type (str): 'llff', 'blender', or 'deepvoxels'.
        no_ndc (bool): If True, disable NDC projection.
        bds (np.ndarray): Bounding box min/max values. Shape (3,) or (N, 3).
        poses (np.ndarray): Camera poses. Shape (N, 4, 4) or (N, 3, 4).
        
    Returns:
        float: near
        float: far
    """
    if dataset_type == 'llff':
        if no_ndc:
            # Explicit bounding box scaling for non-NDC LLFF
            # bds is assumed to be pre-scaled by bd_factor if provided in load_llff_data
            near = np.min(bds) * 0.9
            far = np.max(bds) * 1.0
        else:
            # NDC space is fixed to [0, 1]
            near = 0.0
            far = 1.0
    elif dataset_type == 'blender':
        # Hardcoded values for Blender synthetic dataset
        near = 2.0
        far = 6.0
    elif dataset_type == 'deepvoxels':
        # Extract translation vectors (last column)
        # Assuming poses are (N, 4, 4) or (N, 3, 4)
        translations = poses[:, :3, -1]
        # Compute mean norm of camera positions
        hemi_R = np.mean(np.linalg.norm(translations, axis=-1))
        near = hemi_R - 1.0
        far = hemi_R + 1.0
    else:
        raise ValueError(f"Unsupported dataset_type: {dataset_type}")
        
    return near, far
```

## Operating characteristics
- **Time Complexity:** $O(1)$ for Blender and LLFF (with precomputed `bds`). $O(N)$ for DeepVoxels where $N$ is the number of camera poses, due to the norm calculation.
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Strengths:** Adapts sampling bounds to the specific geometric properties of common NeRF datasets, preventing sampling in empty space or clipping valid geometry.
- **Limitations:** Hardcoded for specific dataset types. Does not generalize to arbitrary unstructured point clouds without scene normalization (contraction) [S1].

## Background
This method originates from the implementation details of early NeRF variants and their derivatives. The LLFF and Blender bounds are standard conventions established in the original NeRF paper and subsequent repositories like [S2, unofficial]. The DeepVoxels approach adapts to spherical camera trajectories. The scene normalization strategy (sphere diameter + 50m) is introduced in LidaRF to handle unbounded street scenes using scene contraction [S1].

## Sources
[S1] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
[S2] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
