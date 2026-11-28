# Gaussian Voxelization

Also called: 3D Gaussian Voxelization, 3D Gaussian to Voxel Grid Conversion, Gaussian to Voxel Grid Conversion, Voxelization of 3D Gaussians, Voxelize Gaussians

## Overview
Gaussian Voxelization converts continuous 3D Gaussian primitives into discrete voxel grid representations for efficient neural processing, collision detection, or physics simulation [S1, S2, unofficial][S5]. Unlike sibling methods that process raw point clouds or mesh surfaces, this family explicitly handles the probabilistic nature of Gaussians, using their covariance and opacity to determine voxel occupancy, feature aggregation, or spatial subdivision [S2, unofficial][S5].

## How it works
**Inputs and Outputs:**
- Inputs: 3D Gaussians $G_k$ defined by mean $\mu_k$, scale $s_k$, rotation $q_k$, opacity $\alpha_k$, and feature vector $F_k$ [S5]. Voxel grid configuration: dimensions $(nVoxel_x, nVoxel_y, nVoxel_z)$, physical extents $(sVoxel_x, sVoxel_y, sVoxel_z)$, center $(center_x, center_y, center_z)$, and `scale_modifier` [S2, unofficial].
- Outputs: Voxel grid features/values, per-Gaussian bounding radii, and inverse index mappings [S1, S2, unofficial].

**Data Structures:**
- `GaussianVoxelizationSettings`: NamedTuple containing `scale_modifier`, `nVoxel_*`, `sVoxel_*`, `center_*`, `prefiltered`, `debug` [S3, unofficial].
- `conic_opacity`: Float array of size $P \times 7$, storing inverse covariance elements followed by opacity: $[\text{inv}_a, \text{inv}_b, \text{inv}_c, \text{inv}_d, \text{inv}_e, \text{inv}_f, \text{opacity}]$ [S2, unofficial].
- `points_xyz_vol`: Float array of size $P \times 3$, storing voxel-space coordinates of each Gaussian center [S2, unofficial].
- `radii_x`, `radii_y`, `radii_z`: Integer arrays of size $P$, storing bounding box radius in voxel units [S2, unofficial].
- `tiles_touched`: `uint32_t` array of size $P$, storing voxel/tile intersection counts [S2, unofficial].

**Steps:**
1. Compute voxel spacing: $dVoxel_x = sVoxel_x / nVoxel_x$, $dVoxel_y = sVoxel_y / nVoxel_y$, $dVoxel_z = sVoxel_z / nVoxel_z$ [S2, unofficial].
2. Transform covariance to voxel space: $\mathbf{\Sigma}_{vox} = \mathbf{M} \mathbf{\Sigma}_{world} \mathbf{M}^T$, where $\mathbf{M} = \text{diag}(1/dVoxel_x, 1/dVoxel_y, 1/dVoxel_z)$ [S2, unofficial].
3. Compute bounding radius in voxels: $r_* = \lceil 3 \cdot \max(s_x, s_y, s_z) / dVoxel_* \rceil$ [S2, unofficial].
4. Transform center to voxel space: $p_{vol} = ((p_{orig} - \mathbf{c} + \mathbf{s}/2) / \mathbf{dVoxel})$, where $\mathbf{c}$ is grid center and $\mathbf{s}$ is grid size [S2, unofficial].
5. Check boundary: if $p_{vol} + \mathbf{r} < 0$ or $p_{vol} - \mathbf{r} > \mathbf{nVoxel}$, skip the Gaussian [S2, unofficial].
6. Compute bounding box $[\mathbf{min}, \mathbf{max}]$. If volume is zero, skip [S2, unofficial].
7. Compute inverse covariance (conic matrix) elements via cofactor expansion. If $\det = 0$, skip [S2, unofficial].
8. Store conic elements and opacity in `conic_opacity` [S2, unofficial].
9. Splat Depositing & Fusion: For each Gaussian, identify the nearest 8 voxels $\{\mathbf{M}_k\}$ [S5]. Compute weight $w_{jk} = \mathcal{K}(\mu_k - \mathbf{x}_j) / \sum_{j' \in \{\mathbf{M}_k\}} \mathcal{K}(\mu_k - \mathbf{x}_{j'})$ [S5]. Aggregate voxel attribute $\mathbf{E}_j = \frac{1}{\mathbf{W}_j} \sum_{k} w_{jk} \alpha_k [\mathbf{F}_k; \mathbf{G}_k]$, where $\mathbf{W}_j = \sum_{k} w_{jk} \alpha_k$ [S5].
10. Sparsification: Downsample high-resolution voxels by factor $[d, h, w]$ [S5]. Sort coarse voxels by weight $W_i$, retain top 20%, cull corresponding fine voxels [S5].

**Formulas:**
- Covariance transformation: $\mathbf{\Sigma}_{vox} = \mathbf{M} \mathbf{\Sigma}_{world} \mathbf{M}^T$ [S2, unofficial].
- Conic matrix inverse: $\text{inv}_a = (\hat{d}\hat{f} - \hat{e}^2)/\det$, $\text{inv}_b = (\hat{c}\hat{e} - \hat{b}\hat{f})/\det$, $\text{inv}_c = (\hat{b}\hat{e} - \hat{c}\hat{d})/\det$, $\text{inv}_d = (\hat{a}\hat{f} - \hat{c}^2)/\det$, $\text{inv}_e = (\hat{b}\hat{c} - \hat{a}\hat{e})/\det$, $\text{inv}_f = (\hat{a}\hat{d} - \hat{b}^2)/\det$, where $\det = \hat{a}\hat{d}\hat{f} + 2\hat{b}\hat{c}\hat{e} - \hat{a}\hat{e}^2 - \hat{f}\hat{b}^2 - \hat{d}\hat{c}^2$ [S2, unofficial].
- Weight calculation: $w_{jk} = \frac{\mathcal{K}(\mu_k - \mathbf{x}_j)}{\sum_{j' \in \{\mathbf{M}_k\}} \mathcal{K}(\mu_k - \mathbf{x}_{j'})}$ for $j \in \{\mathbf{M}_k\}$, else $0$ [S5].
- Voxel attribute aggregation: $\mathbf{E}_j = \frac{1}{\mathbf{W}_j} \sum_{k} w_{jk} \alpha_k [\mathbf{F}_k; \mathbf{G}_k]$, $\mathbf{W}_j = \sum_{k} w_{jk} \alpha_k$ [S5].

**Parameters:**
- `scale_modifier`: float multiplier applied to Gaussian scales before covariance computation [S2, unofficial].
- `grid_size`: float defining the physical size of each voxel cell [S1, unofficial].
- `nVoxel_*`, `sVoxel_*`, `center_*`: int/float dimensions, extents, and center coordinates of the target voxel grid [S2, unofficial].
- `prefiltered`: bool flag indicating whether Gaussians have already been filtered [S4, unofficial].
- `debug`: bool flag for saving debug snapshots [S3, unofficial].
- `SAMPLE_SHAPE`: Literal["volume", "tri-plane", "tri-line", "center"], default "center", controls geometric sampling shape [S1, unofficial].

**Pitfalls:**
- Singular covariance ($\det = 0$) causes division by zero; handled by early return [S2, unofficial].
- Gaussians completely outside the voxel grid bounds are skipped to prevent out-of-bounds memory access [S2, unofficial].
- Quaternion normalization is assumed; non-normalized inputs produce incorrect rotation matrices [S4, unofficial].
- `scales`/`rotations` and `cov3D_precomp` are mutually exclusive; the code raises an exception if both or neither are provided [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def voxelize_gaussians(gaussians, settings):
    """
    Converts 3D Gaussians to a voxel grid representation.
    gaussians: list of dicts with keys 'mean', 'scale', 'rotation', 'opacity', 'features'
    settings: dict with keys 'nVoxel_x', 'nVoxel_y', 'nVoxel_z', 'sVoxel_x', 'sVoxel_y', 'sVoxel_z', 'center_x', 'center_y', 'center_z', 'scale_modifier'
    """
    nV = (settings['nVoxel_x'], settings['nVoxel_y'], settings['nVoxel_z'])
    sV = (settings['sVoxel_x'], settings['sVoxel_y'], settings['sVoxel_z'])
    cV = (settings['center_x'], settings['center_y'], settings['center_z'])
    mod = settings['scale_modifier']

    dV = (sV[0]/nV[0], sV[1]/nV[1], sV[2]/nV[2])
    M = np.diag([1.0/dV[0], 1.0/dV[1], 1.0/dV[2]])

    conic_opacity = []
    points_xyz_vol = []
    radii = []
    tiles_touched = []

    for g in gaussians:
        mu = np.array(g['mean'])
        s = np.array(g['scale']) * mod
        q = np.array(g['rotation']) # assumed unit quaternion
        alpha = g['opacity']
        F = np.array(g['features'])

        # Rotation matrix from quaternion
        r, x, y, z = q[0], q[1], q[2], q[3]
        R = np.array([
            [1-2*(y**2+z**2), 2*(x*y-r*z), 2*(x*z+r*y)],
            [2*(x*y+r*z), 1-2*(x**2+z**2), 2*(y*z-r*x)],
            [2*(x*z-r*y), 2*(y*z+r*x), 1-2*(x**2+y**2)]
        ])

        # World covariance Sigma = M^T * M where M = S * R
        S = np.diag(s)
        Sigma_world = (S @ R).T @ (S @ R)
        cov = np.array([Sigma_world[0,0], Sigma_world[0,1], Sigma_world[0,2],
                        Sigma_world[1,1], Sigma_world[1,2], Sigma_world[2,2]])

        # Transform to voxel space
        Sigma_vox = M @ Sigma_world @ M.T
        a, b, c, d, e, f = Sigma_vox[0,0], Sigma_vox[0,1], Sigma_vox[0,2], \
                           Sigma_vox[1,1], Sigma_vox[1,2], Sigma_vox[2,2]

        det = a*d*f + 2*b*c*e - a*e**2 - f*b**2 - d*c**2
        if det == 0.0:
            continue

        inv_a = (d*f - e**2) / det
        inv_b = (c*e - b*f) / det
        inv_c = (b*e - c*d) / det
        inv_d = (a*f - c**2) / det
        inv_e = (b*c - a*e) / det
        inv_f = (a*d - b**2) / det

        conic_opacity.append([inv_a, inv_b, inv_c, inv_d, inv_e, inv_f, alpha])

        # Voxel space center
        p_vol = ((mu - np.array(cV) + np.array(sV)/2) / np.array(dV))
        points_xyz_vol.append(p_vol)

        # Bounding radius
        r_x = math.ceil(3 * s[0] / dV[0])
        r_y = math.ceil(3 * s[1] / dV[1])
        r_z = math.ceil(3 * s[2] / dV[2])
        radii.append((r_x, r_y, r_z))

        # Boundary check
        if (p_vol + np.array([r_x, r_y, r_z]) < 0).any() or \
           (p_vol - np.array([r_x, r_y, r_z]) > np.array(nV)).any():
            continue

        tiles_touched.append(1)

    return conic_opacity, points_xyz_vol, radii, tiles_touched
```

## Subtopics

### [Point Cloud Voxelization](1000-point-cloud-voxelization.md)
Converts an unstructured set of 3D points into a discrete volumetric grid representation by mapping each point to a specific voxel index based on spatial coordinates. It solves the problem of organizing irregular point data into a structured format suitable for downstream rendering, reconstruction, or spatial analysis. Unlike Hard Voxelization, which often implies binary occupancy or simple binning for deep learning pipelines, this method explicitly supports feature aggregation (e.g., mean, max) and sparse storage formats to handle varying point densities and attribute dimensions. It differs from Ray Model Voxelization by directly binning observed points rather than inferring occupancy from sensor rays.

### [Point Cloud to Gaussian Converter](1788-point-cloud-to-gaussian-converter.md)
The Point Cloud to Gaussian Converter transforms a sparse 3D point cloud into a set of Gaussian ellipsoids for volume rendering. It initializes Gaussian Splatting attributes from raw geometric data by determining spatial variance based on point proximity. Unlike the 3D Gaussian Voxelization method which aggregates existing splats, or the Point-to-Gaussian Generation method which uses generative models, this converter uses a deterministic proximity-based variance function.

### [Voxel Format Generation](6914-voxel-format-generation.md)
Voxel Format Generation converts 3D Gaussian Splats into a sparse voxel octree optimized for collision detection, outputting metadata (`.voxel.json`) and binary data (`.voxel.bin`). It distinguishes itself from general-purpose voxelizers by integrating scene topology analysis (interior vs. exterior) and generating watertight collision meshes directly from the voxel grid, rather than just visual representation.

### [De-voxelization](8432-de-voxelization.md)
De-voxelization aggregates network predictions computed on a sparse set of unique voxels back to the original dense voxel grid, and subsequently reduces these per-voxel predictions into per-Gaussian predictions. It serves as the inverse operation to 3D Gaussian Voxelization, enabling efficient processing by 3D networks that operate on voxelized representations while preserving the native Gaussian structure for final rendering.

### [GaGS](11809-gags.md)
GaGS is a method for learning spatial-temporal deformation of 3D Gaussians in dynamic scenes by voxelizing Gaussian distributions and extracting geometry-aware features via sparse convolutions. It sets itself apart from sibling methods by explicitly combining positional features (via MLP on point clouds) with local geometric features (via 3D U-Net on voxels) to predict deformation, rather than relying solely on implicit deformation fields or 4D Gaussian formulations.

### [3DGS Voxelizer](12530-3dgs-voxelizer.md)
The 3DGS Voxelizer converts a 3D Gaussian Splatting (3DGS) scan into colored voxel blocks represented as OpenUSD `PointInstancer` data, enabling visualization and physics collision in NVIDIA Omniverse. It differs from general-purpose voxelizers by specifically handling 3DGS spherical harmonics decoding and SPZ binary formats, and by outputting a USD-native instancer rather than a dense volume texture.

### [Gaussian to Voxel Conversion via Grid/Kernel](22706-gaussian-to-voxel-conversion-via-grid-kernel.md)
This family converts continuous 3D Gaussian primitives into discrete voxel grid representations for efficient processing by 3D networks or differentiable renderers. It solves the problem of mapping continuous spatial distributions to a fixed-resolution grid while preserving geometric and photometric information. It differs from Point Cloud Voxelization by using analytical Gaussian kernels rather than simple occupancy or nearest-neighbor assignment, and from De-voxelization by operating in the forward direction (continuous to discrete).

### Sparse Voxel Transformer
A Transformer network that processes coarse voxel features as tokens to fuse information in a canonical 3D space and produce refined splats [S5]. Tokenization treats each coarse voxel feature as a token and reshapes them to 1D vectors [S5]. Replication processes latent features $O^c$ from coarse voxel grids to their corresponding fine voxel grids to produce fine-level latent features $O_j$ [S5]. The output dimension is $G'_j \in \mathbb{R}^l$ where $l$ is the number of rendering parameters [S5].

### GVKF Voxel Grid Management
A dynamic voxel management strategy for open scene reconstruction [S6]. Hide Gaussians with opacity less than 0 during each iteration [S6]. Subdivide qualifying voxels using an octree method with a maximum recursion depth of 3 to cap the total number of Gaussians [S6]. Evaluate voxel subdivision and reclamation every 500 iterations [S6]. Initial voxel size for downsampling the Gaussian point cloud is 0.01 [S6]. Four MLPs are shared across all voxel grids in a scene to decode different Gaussian attributes from corresponding voxels [S6]. The gradient threshold for voxel subdivision is $2 \times 10^{-4}$; voxels with an average gradient exceeding this value are subdivided [S6]. The feature vector $F$ per voxel has dimension 32. It stores relative coordinates for up to 10 Gaussian points per voxel grid, indicating a maximum of 10 Gaussians generated per voxel grid [S6]. Sparse viewpoint coverage areas lead to irregular Gaussian distribution, causing uneven surfaces and geometric inaccuracy [S6].

## Operating characteristics
Time complexity is $O(P)$ where $P$ is the number of Gaussians, as each Gaussian undergoes constant-time matrix operations and bounding box calculations [S2, unofficial]. Memory usage scales with the voxel grid resolution and the number of Gaussians, with sparsification reducing token counts to fewer than 10K for transformer processing [S5]. Strengths include efficient neural processing, collision detection compatibility, and differentiable rendering support [S1, S2, unofficial][S5]. Limitations include sensitivity to sparse viewpoint coverage, which causes uneven surfaces and geometric inaccuracy, and the requirement for pre-normalized quaternions [S4, unofficial][S6]. Use this method when preparing Gaussian data for grid-based neural processing, physics simulation, or collision detection [S1, S2, unofficial][S5]. Avoid it when real-time 2D rasterization is sufficient, as voxelization maps Gaussians to a 3D grid for volume queries rather than projecting them to a 2D image plane [S2, unofficial].

## Background
The concept of voxelizing Gaussian primitives has emerged alongside the rise of 3D Gaussian Splatting (3DGS) for novel view synthesis [G]. While 3DGS operates directly with splats, downstream applications like physics simulation, collision detection, and neural field processing often require discrete voxel representations [S2, unofficial][S5]. Methods such as GVKF (Gaussian Voxel Kernel Functions) have been proposed to efficiently reconstruct surfaces in open scenes by managing voxel grids dynamically based on gradient thresholds and opacity [S6]. Other approaches focus on format conversion for specific engines like USD/Omniverse or for integrating Gaussians into transformer-based architectures [S1, unofficial][S5].

## Sources
[S1] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
[S2] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
[S3] yuyouxixi/x2-gaussian, https://github.com/yuyouxixi/x2-gaussian, fetched 2026-10-26 (unofficial)
[S4] yulunwu0108/Perturbed-Gaussian-Ensemble, https://github.com/yulunwu0108/Perturbed-Gaussian-Ensemble, fetched 2026-10-26 (unofficial)
[S5] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S6] GVKF: Gaussian Voxel Kernel Functions for Highly Efficient Surface Reconstruction in Open Scenes, https://doi.org/10.52202/079017-3328, fetched 2026-10-26
