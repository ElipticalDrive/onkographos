# Differentiable Radiative Rasterization
## Overview
Differentiable Radiative Rasterization (DRR) is a GPU-friendly rasterization process implemented in CUDA that renders novel X-ray projections from radiative Gaussian point clouds [S1]. It solves the problem of differentiable rendering for X-ray imaging by projecting 3D Gaussians to a 2D detector plane and accumulating radiation intensity per pixel, allowing gradients to flow back to Gaussian parameters (position, covariance, opacity, and radiation intensity) [S1]. Unlike standard optical Gaussian Splatting which uses alpha compositing for color, DRR sums the radiation intensities of intersection points weighted by the Gaussian probability density, tailored for the physics of X-ray attenuation [S1].

## How it works
**Inputs:**
- Radiative Gaussian point cloud $\mathcal{G} = \{G_i(\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, \alpha_i, \mathbf{f}_i) \mid i = 1, \dots, N_p\}$, where $\boldsymbol{\mu}_i \in \mathbb{R}^3$ is the center, $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3\times3}$ is the covariance, $\alpha_i \in \mathbb{R}$ is the opacity, and $\mathbf{f}_i$ represents inherent point features used to fit radiation intensity [S1].
- Extrinsic matrix $\mathbf{M}_{ext} \in \mathbb{R}^{4\times4}$ [S1].
- Intrinsic matrix $\mathbf{M}_{int} \in \mathbb{R}^{4\times3}$ [S1].
- X-ray source distance $L_{SD}$ (distance between X-ray source and detector) [S1].
- X-ray source azimuth angle $\phi$ [S1].

**Outputs:**
- Rendered image $\mathbf{I}$, where each pixel $p$ has an intensity value $I(p) \in \mathbb{R}$ [S1].

**Data Structures:**
- **Gaussian Point Cloud:** A list of $N_p$ Gaussians, each storing $\boldsymbol{\mu}_i$, $\boldsymbol{\Sigma}_i$, $\alpha_i$, and $\mathbf{f}_i$ [S1].
- **Tiles:** The 2D image plane is partitioned into non-overlapping tiles [S1]. Each tile maintains a sorted list of indices of Gaussians that overlap its screen-space bounding box [S1].

**Steps:**

1. **Project Centers to Image Coordinates:**
   Transform the 3D Gaussian center $\boldsymbol{\mu}_i$ to camera coordinates $\mathbf{t}_i$ using homogeneous coordinates and the extrinsic matrix $\mathbf{M}_{ext}$:
   $$ \tilde{\mathbf{t}}_i = \begin{bmatrix} \mathbf{t}_i \\ 1 \end{bmatrix} = \mathbf{M}_{ext} \tilde{\boldsymbol{\mu}}_i = \mathbf{M}_{ext} \begin{bmatrix} \boldsymbol{\mu}_i \\ 1 \end{bmatrix} $$
   where $\tilde{\boldsymbol{\mu}}_i$ and $\tilde{\mathbf{t}}_i$ are homogeneous coordinates [S1].
   Project $\mathbf{t}_i$ to image coordinates $\mathbf{u}_i \in \mathbb{R}^2$ using the intrinsic matrix $\mathbf{M}_{int}$:
   $$ \tilde{\mathbf{u}}_i = \begin{bmatrix} \mathbf{u}_i \\ 1 \end{bmatrix} = \mathbf{M}_{int} \tilde{\mathbf{t}}_i = \mathbf{M}_{int} \begin{bmatrix} \mathbf{t}_i \\ 1 \end{bmatrix} $$
   where $\mathbf{t}_i = (t_x, t_y, t_z)^\top$ [S1].

2. **Transform Covariance:**
   Compute the covariance matrix $\boldsymbol{\Sigma}'_i \in \mathbb{R}^{3\times3}$ in camera coordinates:
   $$ \boldsymbol{\Sigma}'_i = \mathbf{J}_i \mathbf{W}_i \boldsymbol{\Sigma}_i \mathbf{W}_i^\top \mathbf{J}_i^\top $$
   where $\mathbf{W}_i$ is the viewing transformation and $\mathbf{J}_i$ is the Jacobian of the affine approximation of the projective transformation [S1].

   The viewing transformation $\mathbf{W}_i$ is defined as:
   $$ \mathbf{W}_i = \begin{bmatrix} -\sin \phi & \cos \phi & 0 \\ 0 & 0 & -1 \\ -\cos \phi & -\sin \phi & 0 \end{bmatrix} $$
   where $\phi$ is the azimuth angle of the X-ray source [S1].

   The Jacobian $\mathbf{J}_i$ is defined as:
   $$ \mathbf{J}_i = \begin{bmatrix} \frac{L_{SD}}{t_z} & 0 & -\frac{L_{SD} t_y}{t_z^2} \\ 0 & \frac{L_{SD}}{t_z} & -\frac{L_{SD} t_y}{t_z^2} \\ 0 & 0 & 0 \end{bmatrix} $$
   *Note: The source text [S1] contains a formatting error in the matrix layout, listing terms as $-\frac{L_{SD}}{t_z^2} \frac{t_y}{t_z^2}$. The standard projective geometry form $-\frac{L_{SD} t_y}{t_z^2}$ is used here for correctness [G].* Here, $L_{SD}$ is the distance between the X-ray source and detector, and $t_y, t_z$ are components of $\mathbf{t}_i$ [S1].

3. **Obtain 2D Covariance:**
   Derive the 2D covariance matrix $\boldsymbol{\Sigma}''_i \in \mathbb{R}^{2\times2}$ by skipping the third row and column of $\boldsymbol{\Sigma}'_i$ [S1].

4. **Tile Assignment:**
   Partition the 2D image plane into non-overlapping tiles [S1]. Assign each 3D Gaussian $i$ to the tiles whose bounding boxes overlap with the 2D projection defined by $(\mathbf{u}_i, \boldsymbol{\Sigma}''_i)$ [S1].

5. **Sorting:**
   Sort the 3D Gaussians within each tile by their distance to the 2D detector (i.e., by $t_z$) [S1].

6. **Intensity Calculation:**
   For each pixel $p$, identify the set $\mathcal{N}$ of ordered points (Gaussians) overlapping the pixel in the corresponding tile [S1].
   The probability value of the $i$-th Gaussian distribution at a 3D point position $\mathbf{x} \in \mathbb{R}^3$ is:
   $$ P(\mathbf{x}|\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu}_i)^\top \boldsymbol{\Sigma}_i^{-1}(\mathbf{x} - \boldsymbol{\mu}_i)\right) $$
   [S1].
   The radiation intensity contribution $\mathbf{i}_j$ of the $j$-th intersection point $\mathbf{x}_j$ is derived from the Gaussian's opacity $\alpha_j$, the probability $P(\mathbf{x}_j | \boldsymbol{\mu}_j, \boldsymbol{\Sigma}_j)$, and the inherent point features $\mathbf{f}_j$ [S1].
   The total intensity $I(p)$ at pixel $p$ is the sum of contributions from all overlapping Gaussians:
   $$ I(p) = \sum_{j \in \mathcal{N}} \mathbf{i}_j $$
   where $\mathbf{i}_j$ is the radiation intensity of the $j$-th intersection point $\mathbf{x}_j$ [S1].
   The source defines a term $\sigma_j = \alpha_j P(\mathbf{x}_j | \boldsymbol{\mu}_j, \boldsymbol{\Sigma}_j)$ in the context of the blending table, but the final intensity formula is a direct sum $\sum \mathbf{i}_j$, not the alpha-compositing product form used in standard 3DGS [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def differentiable_radiative_rasterization(gaussians, M_ext, M_int, L_SD, phi, image_width, image_height, tile_size):
    """
    Renders X-ray projections from radiative Gaussian point clouds.
    
    Args:
        gaussians: List of dicts, each containing:
            'mu': (3,) array, center position
            'Sigma': (3,3) array, covariance matrix
            'alpha': float, opacity
            'f': array, inherent point features
        M_ext: (4,4) array, extrinsic matrix
        M_int: (4,3) array, intrinsic matrix
        L_SD: float, distance between X-ray source and detector
        phi: float, azimuth angle of the source
        image_width: int
        image_height: int
        tile_size: int, size of each tile in pixels
        
    Returns:
        I: (image_height, image_width) array, rendered intensities
    """
    N = len(gaussians)
    
    # Step 1: Project centers to image coordinates
    mu_arr = np.array([g['mu'] for g in gaussians]) # (N, 3)
    mu_homo = np.hstack([mu_arr, np.ones((N, 1))])  # (N, 4)
    t_homo = mu_homo @ M_ext.T                      # (N, 4)
    t = t_homo[:, :3]                               # (N, 3)
    
    # Project to image coordinates
    u_homo = t @ M_int.T                            # (N, 4)
    u = u_homo[:, :2]                               # (N, 2), image coordinates
    
    # Step 2: Transform Covariance
    sin_phi, cos_phi = math.sin(phi), math.cos(phi)
    W = np.array([
        [-sin_phi, cos_phi, 0],
        [0, 0, -1],
        [-cos_phi, -sin_phi, 0]
    ])
    
    Sigma_prime_list = []
    for i in range(N):
        Sigma_i = gaussians[i]['Sigma']
        ty, tz = t[i, 1], t[i, 2]
        
        # Guard: discard Gaussians behind the detector plane
        if tz <= 0:
            continue
            
        # Compute J_i (corrected from source's formatting error)
        J = np.array([
            [L_SD / tz, 0, -L_SD * ty / (tz ** 2)],
            [0, L_SD / tz, -L_SD * ty / (tz ** 2)],
            [0, 0, 0]
        ])
        
        # Sigma_prime = J @ W @ Sigma @ W.T @ J.T
        Sigma_prime_list.append(J @ W @ Sigma_i @ W.T @ J.T)
        
    # Step 3: Obtain 2D Covariance
    Sigma_2d_list = [Sp[:2, :2] for Sp in Sigma_prime_list]
        
    # Step 4: Tile Assignment
    num_tiles_x = int(math.ceil(image_width / tile_size))
    num_tiles_y = int(math.ceil(image_height / tile_size))
    tiles = [[[] for _ in range(num_tiles_x)] for _ in range(num_tiles_y)]
    
    for i in range(len(Sigma_2d_list)):
        ux, uy = u[i]
        Sigma_2d_i = Sigma_2d_list[i]
        
        # Determine bounding box in pixels (approx 2 sigma)
        eigvals = np.linalg.eigvalsh(Sigma_2d_i)
        max_std = math.sqrt(max(eigvals)) * 2.0
        
        min_x, max_x = int(ux - max_std), int(ux + max_std)
        min_y, max_y = int(uy - max_std), int(uy + max_std)
        
        tile_x_min, tile_x_max = max(0, min_x // tile_size), min(num_tiles_x - 1, max_x // tile_size)
        tile_y_min, tile_y_max = max(0, min_y // tile_size), min(num_tiles_y - 1, max_y // tile_size)
        
        for tile_y in range(tile_y_min, tile_y_max + 1):
            for tile_x in range(tile_x_min, tile_x_max + 1):
                tiles[tile_y][tile_x].append(i)
                
    # Step 5: Sorting
    for tile_y in range(num_tiles_y):
        for tile_x in range(num_tiles_x):
            tiles[tile_y][tile_x].sort(key=lambda idx: t[idx, 2])
            
    # Step 6: Intensity Calculation
    I = np.zeros((image_height, image_width))
    
    for tile_y in range(num_tiles_y):
        for tile_x in range(num_tiles_x):
            tile_gaussians = tiles[tile_y][tile_x]
            
            y_start, y_end = tile_y * tile_size, min((tile_y + 1) * tile_size, image_height)
            x_start, x_end = tile_x * tile_size, min((tile_x + 1) * tile_size, image_width)
            
            for y in range(y_start, y_end):
                for x in range(x_start, x_end):
                    total_intensity = 0.0
                    for idx in tile_gaussians:
                        v = np.array([x - u[idx, 0], y - u[idx, 1]])
                        
                        # Check if pixel is inside the 2D Gaussian ellipse
                        try:
                            Sigma_inv = np.linalg.inv(Sigma_2d_list[idx] + 1e-6 * np.eye(2))
                            dist_sq = v @ Sigma_inv @ v
                        except np.linalg.LinAlgError:
                            dist_sq = float('inf')
                            
                        if dist_sq <= 4.0: # 2 sigma threshold
                            # Probability weight sigma_j = alpha_j * P(x_j | mu_j, Sigma_j)
                            # x_j is the intersection point; evaluated at projected pixel center as proxy [G]
                            P_2d = math.exp(-0.5 * dist_sq)
                            i_j = gaussians[idx]['alpha'] * P_2d
                            total_intensity += i_j
                    I[y, x] = total_intensity
                    
    return I
```

**Edge Cases and Pitfalls:**
- **Precision:** The Jacobian calculation involves division by $t_z$ and $t_z^2$. If $t_z$ is close to zero, numerical instability occurs. Gaussians with $t_z \le 0$ (behind the camera/source) should be discarded [G].
- **Boundaries:** The bounding box calculation for tile assignment must be clamped to the image boundaries [G].
- **Degenerate Input:** If $\boldsymbol{\Sigma}_i$ is singular, the inverse in the probability calculation will fail. Regularization or checking for positive definiteness is required [G].
- **X-ray Geometry:** The definition of the intersection point $\mathbf{x}_j$ is not explicitly detailed in the source beyond being the "intersection 3D point of the X-ray landing on pixel p" [S1]. Implementers must define the ray equation from the X-ray source through the pixel to find $\mathbf{x}_j$ for the probability evaluation [S1].

## Operating characteristics
- **Time Complexity:** $O(N_p \log N_p)$ for sorting, plus linear tile assignment and pixel blending [S1].
- **Memory Complexity:** $O(N_p)$ to store Gaussian parameters and tile lists [S1].
- **Measured Performance:** The forward and backward processes of the X-Gaussian model (which uses DRR) are much faster than those of the original 3DGS due to the avoidance of complex spherical harmonic (SH) function computations [S1].
- **Strengths:** GPU-friendly CUDA implementation; fast rendering for X-ray novel view synthesis; differentiable for optimization of Gaussian parameters [S1].
- **Limitations:** Specific to X-ray imaging geometry; requires accurate knowledge of $L_{SD}$ and $\phi$; relies on the ACUI initialization strategy for camera calibration and Gaussian placement [S1].
- **When to use:** When rendering novel X-ray projections from a radiative Gaussian point cloud [S1].
- **When not to use:** For standard optical RGB rendering; when X-ray source parameters are unknown [S1].

## Background
Differentiable Radiative Rasterization (DRR) was introduced in the paper "Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis" [S1]. It is part of the X-Gaussian framework, which adapts 3D Gaussian Splatting for X-ray imaging [S1]. The method was developed to enable efficient training and rendering of X-ray projections without running Structure-from-Motion (SfM) algorithms, using a custom Angle-pose Cuboid Uniform Initialization (ACUI) strategy [S1].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
