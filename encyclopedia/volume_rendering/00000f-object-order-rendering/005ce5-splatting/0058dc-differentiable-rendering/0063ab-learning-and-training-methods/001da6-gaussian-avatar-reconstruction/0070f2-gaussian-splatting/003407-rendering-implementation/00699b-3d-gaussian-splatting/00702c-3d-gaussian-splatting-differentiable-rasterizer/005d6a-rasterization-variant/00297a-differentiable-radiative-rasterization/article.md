# Differentiable Radiative Rasterization


## Overview
Differentiable Radiative Rasterization (DRR) is a GPU-friendly rasterization process implemented in CUDA that renders novel X-ray projections from radiative Gaussian point clouds [S1]. It solves the problem of differentiable volume rendering for X-ray imaging by treating the object as a collection of 3D Gaussians with radiation intensity attributes, allowing gradients to flow from the rendered projection back to the Gaussian parameters [S1]. Unlike standard alpha-blending methods used in visible light [Parent](17388-rasterization-variant.md), DRR computes pixel intensity by summing weighted radiation contributions from intersection points along the ray, where the weight is determined by the Gaussian probability density at that intersection [S1].

## How it works
DRR takes a set of 3D Gaussian point clouds and camera parameters (intrinsic and extrinsic matrices) as input, and outputs a 2D intensity image $\mathbf{I}$ [S1]. The method projects 3D Gaussians to 2D, assigns them to tiles, sorts them by depth, and blends their radiation contributions [S1].

### Inputs and Outputs
*   **Input:**
    *   $\mathcal{G} = \{G_i(\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, \alpha_i, \mathbf{f}_i) \mid i = 1, \dots, N_p\}$: A set of $N_p$ radiative Gaussian point clouds. Each $G_i$ has a center $\boldsymbol{\mu}_i \in \mathbb{R}^3$, a 3D covariance $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3\times3}$, an opacity $\alpha_i \in \mathbb{R}$, and an inherent point feature vector $\mathbf{f}_i$ [S1].
    *   $\mathbf{M}_{ext} \in \mathbb{R}^{4\times4}$: The extrinsic matrix transforming world coordinates to camera coordinates [S1].
    *   $\mathbf{M}_{int} \in \mathbb{R}^{4\times3}$: The intrinsic matrix mapping camera coordinates to image coordinates [S1].
    *   $L_{SD}$: The distance between the X-ray source and the detector [S1].
    *   $\phi$: The azimuth angle of the X-ray source [S1].
*   **Output:**
    *   $\mathbf{I}$: The rendered 2D image, where each pixel $p$ has an intensity value $I(p)$ [S1].

### Data Structures
*   **Gaussian Point Cloud:** Defined by the tuple $(\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, \alpha_i, \mathbf{f}_i)$ [S1].
*   **Tile:** Non-overlapping regions of the 2D image plane. Gaussians are assigned to tiles based on their 2D projection bounds [S1].
*   **Ordered Points:** For each pixel, a list of intersection points $\mathbf{x}_j$ from overlapping Gaussians, sorted by distance to the detector [S1].

### Steps

1.  **Project 3D Gaussian Centers:**
    Transform the 3D center $\boldsymbol{\mu}_i$ to camera coordinates $\mathbf{t}_i$ and then to image coordinates $\mathbf{u}_i$ using homogeneous coordinates [S1]:
    $$ \tilde{\mathbf{t}}_i = \mathbf{M}_{ext} \tilde{\boldsymbol{\mu}}_i = \mathbf{M}_{ext} \begin{bmatrix} \boldsymbol{\mu}_i \\ 1 \end{bmatrix} $$
    $$ \tilde{\mathbf{u}}_i = \mathbf{M}_{int} \tilde{\mathbf{t}}_i = \mathbf{M}_{int} \begin{bmatrix} \mathbf{t}_i \\ 1 \end{bmatrix} $$
    where $\tilde{\mathbf{t}}_i = [\mathbf{t}_i; 1]$, $\tilde{\boldsymbol{\mu}}_i = [\boldsymbol{\mu}_i; 1]$, and $\tilde{\mathbf{u}}_i = [\mathbf{u}_i; 1]$ [S1]. Here $\mathbf{t}_i = (t_x, t_y, t_z) \in \mathbb{R}^3$ and $\mathbf{u}_i \in \mathbb{R}^2$ [S1].

2.  **Transform Covariance:**
    Compute the covariance matrix $\boldsymbol{\Sigma}'_i \in \mathbb{R}^{3\times3}$ in camera coordinates [S1]:
    $$ \boldsymbol{\Sigma}'_i = \mathbf{J}_i \mathbf{W}_i \boldsymbol{\Sigma}_i \mathbf{W}_i^\top \mathbf{J}_i^\top $$
    where $\mathbf{J}_i$ is the Jacobian of the affine approximation of the projective transformation, and $\mathbf{W}_i$ is the viewing transformation [S1].

    The Jacobian $\mathbf{J}_i$ is defined as [S1]:
    $$ \mathbf{J}_i = \begin{bmatrix} \frac{L_{SD}}{t_z} & 0 & -\frac{L_{SD}}{t_z^2} \frac{t_y}{t_z^2} \\ 0 & \frac{L_{SD}}{t_z} & -\frac{L_{SD}}{t_z^2} \frac{t_y}{t_z^2} \\ 0 & 0 & 0 \end{bmatrix} $$
    *Note: The source text contains a formatting error in the matrix layout (missing column separators in the first row), but the terms are explicitly listed as $\frac{L_{SD}}{t_z}$, $0$, and $-\frac{L_{SD}}{t_z^2} \frac{t_y}{t_z^2}$ for the first row [S1].*

    The viewing transformation $\mathbf{W}_i$ is defined as [S1]:
    $$ \mathbf{W}_i = \begin{bmatrix} -\sin \phi & \cos \phi & 0 \\ 0 & 0 & -1 \\ -\cos \phi & -\sin \phi & 0 \end{bmatrix} $$
    where $\phi$ is the azimuth angle of the source [S1].

3.  **Obtain 2D Covariance:**
    Derive the 2D covariance matrix $\boldsymbol{\Sigma}''_i \in \mathbb{R}^{2\times2}$ by skipping the third row and third column of $\boldsymbol{\Sigma}'_i$ [S1].

4.  **Tile Assignment:**
    Partition the 2D image plane into non-overlapping tiles. Assign each 3D Gaussian $G_i$ to the tiles that its 2D projection $(\mathbf{u}_i, \boldsymbol{\Sigma}''_i)$ overlaps [S1].

5.  **Sorting:**
    Within each tile, sort the assigned 3D Gaussians by their distances to the 2D detector (i.e., by $t_z$) [S1].

6.  **Intensity Blending:**
    For each pixel $p$, identify the set $\mathcal{N}$ of ordered points overlapping the pixel in the corresponding tile [S1]. The intensity $I(p)$ is calculated as [S1]:
    $$ I(p) = \sum_{j \in \mathcal{N}} \mathbf{i}_j $$
    where $\mathbf{i}_j$ is the radiation intensity of the $j$-th intersection point $\mathbf{x}_j$ [S1]. The contribution $\mathbf{i}_j$ is weighted by the probability term $\sigma_j$ [S1]:
    $$ \sigma_j = \alpha_j P(\mathbf{x}_j | \boldsymbol{\mu}_j, \boldsymbol{\Sigma}_j) $$
    The probability value $P$ is defined as [S1]:
    $$ P(\mathbf{x}|\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu}_i)^\top \boldsymbol{\Sigma}_i^{-1}(\mathbf{x} - \boldsymbol{\mu}_i)\right) $$
    where $\mathbf{x} \in \mathbb{R}^3$ is the 3D point position [S1]. The intersection point $\mathbf{x}_j$ is the $j$-th intersection of the X-ray landing on pixel $p$ with the Gaussian point clouds in 3D space [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def differentiable_radiative_rasterization(
    gaussians: list, 
    M_ext: np.ndarray, 
    M_int: np.ndarray, 
    L_SD: float, 
    phi: float, 
    image_width: int, 
    image_height: int, 
    tile_size: int
) -> np.ndarray:
    """
    Renders an X-ray projection from radiative Gaussian point clouds.
    
    Args:
        gaussians: List of dicts with keys 'mu' (3,), 'Sigma' (3,3), 'alpha' (float), 'f' (vector).
        M_ext: 4x4 extrinsic matrix.
        M_int: 4x3 intrinsic matrix.
        L_SD: Distance between X-ray source and detector.
        phi: Azimuth angle of the source.
        image_width, image_height: Dimensions of the output image.
        tile_size: Size of rasterization tiles.
        
    Returns:
        I: Rendered image (H, W).
    """
    # Initialize output image
    I = np.zeros((image_height, image_height))
    
    # Precompute viewing transformation W
    W = np.array([
        [-math.sin(phi), math.cos(phi), 0],
        [0, 0, -1],
        [-math.cos(phi), -math.sin(phi), 0]
    ])
    
    # Step 1 & 2: Project centers and transform covariances
    projected_gaussians = []
    for i, g in enumerate(gaussians):
        mu = g['mu']
        Sigma = g['Sigma']
        alpha = g['alpha']
        f = g['f']
        
        # Homogeneous coordinates
        mu_h = np.append(mu, 1.0)
        
        # Transform to camera coordinates
        t_h = M_ext @ mu_h
        t = t_h[:3]
        t_z = t[2]
        
        # Project to image coordinates
        u_h = M_int @ t_h
        u = u_h[:2]
        
        # Compute Jacobian J
        # Note: Source defines J with specific terms. 
        # Row 1: L_SD/t_z, 0, -(L_SD/t_z^2)*(t_y/t_z^2)
        # Row 2: 0, L_SD/t_z, -(L_SD/t_z^2)*(t_y/t_z^2)
        # Row 3: 0, 0, 0
        term = -(L_SD / (t_z**2)) * (t[1] / (t_z**2))
        J = np.array([
            [L_SD / t_z, 0, term],
            [0, L_SD / t_z, term],
            [0, 0, 0]
        ])
        
        # Transform covariance
        Sigma_prime = J @ W @ Sigma @ W.T @ J.T
        
        # Get 2D covariance by skipping 3rd row/col
        Sigma_double_prime = Sigma_prime[:2, :2]
        
        projected_gaussians.append({
            'index': i,
            'u': u,
            'Sigma_2d': Sigma_double_prime,
            't_z': t_z,
            'mu': mu,
            'Sigma': Sigma,
            'alpha': alpha,
            'f': f
        })
        
    # Step 4: Tile Assignment
    # Create tiles
    num_tiles_x = math.ceil(image_width / tile_size)
    num_tiles_y = math.ceil(image_height / tile_size)
    tiles = {}
    
    for g in projected_gaussians:
        u = g['u']
        Sigma_2d = g['Sigma_2d']
        
        # Determine bounding box in pixels
        # Approximate bounding box using 2 sigma
        eigenvalues = np.linalg.eigvals(Sigma_2d)
        max_sigma = math.sqrt(max(eigenvalues)) * 2.0
        
        min_x = int(u[0] - max_sigma)
        max_x = int(u[0] + max_sigma)
        min_y = int(u[1] - max_sigma)
        max_y = int(u[1] + max_sigma)
        
        # Clip to image bounds
        min_x = max(0, min_x)
        max_x = min(image_width, max_x)
        min_y = max(0, min_y)
        max_y = min(image_height, max_y)
        
        # Assign to tiles
        for ty in range(math.ceil(min_y / tile_size), math.ceil(max_y / tile_size) + 1):
            for tx in range(math.ceil(min_x / tile_size), math.ceil(max_x / tile_size) + 1):
                tile_key = (ty, tx)
                if tile_key not in tiles:
                    tiles[tile_key] = []
                tiles[tile_key].append(g)
                
    # Step 5: Sorting and Step 6: Blending
    for (ty, tx), g_list in tiles.items():
        # Sort by distance to detector (t_z)
        g_list.sort(key=lambda g: g['t_z'])
        
        # Define tile pixel range
        y_start = ty * tile_size
        y_end = min((ty + 1) * tile_size, image_height)
        x_start = tx * tile_size
        x_end = min((tx + 1) * tile_size, image_width)
        
        for y in range(y_start, y_end):
            for x in range(x_start, x_end):
                pixel_coords = np.array([x, y])
                
                # Find intersections with Gaussians in this tile
                # Note: In a real implementation, this involves ray-Gaussian intersection
                # The source defines x_j as the intersection point.
                # For simplicity in pseudocode, we assume we can compute intersection points.
                intersections = []
                
                for g in g_list:
                    # Compute intersection of ray through (x,y) with Gaussian g
                    # Ray direction is roughly along camera Z, but for X-ray, 
                    # the source is at a specific pose. 
                    # The source text implies x_j is the intersection of the X-ray 
                    # landing on pixel p with the Gaussian.
                    # We approximate the ray origin and direction based on camera model.
                    # However, the source does not provide the exact ray origin/direction 
                    # calculation for the intersection point x_j, only the weighting.
                    # [G] Assuming a standard pinhole camera model for ray generation 
                    # to find x_j, as the source focuses on the weighting formula.
                    
                    # Ray from camera center through pixel (x,y)
                    # This is a simplification; X-ray geometry might differ.
                    # The source mentions "X-ray landing on pixel p".
                    # We will compute the point on the ray closest to the Gaussian center 
                    # or the actual intersection if possible.
                    # Given the complexity and lack of explicit ray origin in source,
                    # we use the Gaussian center as a proxy for x_j if intersection is not provided.
                    # But the formula uses P(x_j | mu, Sigma).
                    # Let's assume x_j is the point on the ray closest to mu_j.
                    
                    # Ray: R(t) = O + t*D
                    # O = camera center, D = direction to pixel
                    # This part is not explicitly detailed in the source for X-ray specific geometry.
                    # We will use the Gaussian center mu_j as x_j for the probability calculation 
                    # if the intersection is not explicitly computable from the given text.
                    # However, the source says "x_j is the j-th intersection 3D point".
                    # Without explicit ray origin, we cannot compute x_j precisely.
                    # [G] We will assume the intersection point x_j is approximated by the 
                    # projection of the Gaussian center onto the ray, or simply use mu_j 
                    # if the Gaussian is small.
                    
                    # To be safe and follow the source strictly:
                    # The source defines sigma_j = alpha_j * P(x_j | mu_j, Sigma_j).
                    # It does not define how x_j is found.
                    # We will assume x_j is the point on the ray closest to mu_j.
                    
                    # Ray origin (camera center)
                    # M_ext is world to camera. Inverse is camera to world.
                    # Camera center in world is -M_ext[:3,:3].T @ M_ext[:3,3]
                    # But we are in camera coordinates for t_z.
                    # Let's work in camera coordinates.
                    # Ray origin in camera coords is (0,0,0).
                    # Ray direction is inverse(M_int[:2,:2]) @ (x - M_int[:2,2]) normalized?
                    # M_int maps 3D to 2D.
                    # u = M_int @ t
                    # t = M_int^{-1} @ u_h
                    # This is complex. 
                    # Let's assume the ray direction D is derived from pixel (x,y).
                    
                    # Simplified: Use mu_j as x_j for the probability calculation.
                    # This is an approximation.
                    x_j = g['mu']
                    
                    # Compute probability
                    diff = x_j - g['mu']
                    # Sigma_inv might be singular or ill-conditioned.
                    # [G] Add small epsilon to diagonal for stability.
                    Sigma_inv = np.linalg.inv(g['Sigma'] + 1e-6 * np.eye(3))
                    prob = math.exp(-0.5 * diff.T @ Sigma_inv @ diff)
                    
                    sigma_j = g['alpha'] * prob
                    
                    # Radiation intensity i_j
                    # Source says i_j is radiation intensity of x_j.
                    # It also mentions "inner product between a learnable vector ... and basis weights".
                    # This part is not fully detailed in the provided notes for i_j calculation.
                    # [G] Assuming i_j is a constant or derived from f_j.
                    # The source says "fit the radiation intensity of a 3D point".
                    # We will assume i_j is a scalar value associated with the Gaussian.
                    # Since the source doesn't give the exact formula for i_j, 
                    # we will use a placeholder or assume it's part of the Gaussian attributes.
                    # Let's assume i_j is stored in the Gaussian or computed from f_j.
                    # For the purpose of this pseudocode, we assume i_j is a scalar attribute.
                    i_j = g.get('intensity', 1.0) # [G] Default intensity
                    
                    intersections.append({
                        'sigma': sigma_j,
                        'i': i_j
                    })
                
                # Blend
                # I(p) = sum_{j in N} i_j
                # But the source shows a table with sigma_j and product terms.
                # The formula is I(p) = sum_{j in N} i_j.
                # The table shows the weight for i_j is sigma_j * product_{k=1}^{j-1} (1 - sigma_k).
                # This is standard alpha compositing.
                
                total_intensity = 0.0
                transmittance = 1.0
                for inter in intersections:
                    sigma = inter['sigma']
                    i_val = inter['i']
                    
                    # Contribution
                    contribution = i_val * sigma * transmittance
                    total_intensity += contribution
                    
                    # Update transmittance
                    transmittance *= (1 - sigma)
                    
                I[y, x] = total_intensity
                
    return I
```

### Parameters
*   $L_{SD}$: Distance between the X-ray source and detector [S1].
*   $\phi$: Azimuth angle of the X-ray source [S1].
*   $\mathbf{M}_{ext}$: Extrinsic matrix [S1].
*   $\mathbf{M}_{int}$: Intrinsic matrix [S1].

### Edge Cases and Pitfalls
*   **Singular Covariance:** The inversion of $\boldsymbol{\Sigma}_i$ in the probability function $P$ may be unstable if the covariance is singular or near-singular. A small epsilon should be added to the diagonal for numerical stability [G].
*   **Matrix Formatting:** The source text for $\mathbf{J}_i$ contains a formatting error with missing column separators in the first row. The implementation must correctly parse the three terms provided: $\frac{L_{SD}}{t_z}$, $0$, and $-\frac{L_{SD}}{t_z^2} \frac{t_y}{t_z^2}$ [S1].
*   **Intersection Point Calculation:** The source defines $\mathbf{x}_j$ as the intersection point but does not provide the explicit algorithm for computing the ray-Gaussian intersection for X-ray geometry. The implementation must derive this based on the camera model and X-ray source position [G].

## Operating characteristics
*   **Time Complexity:** The rasterization process is $O(N_p)$ for projection and sorting, and $O(P \cdot T)$ for blending, where $P$ is the number of pixels and $T$ is the average number of Gaussians per tile [S1]. The CUDA implementation is designed to be GPU-friendly [S1].
*   **Memory Complexity:** Requires storage for the Gaussian point clouds and intermediate projected data structures [S1].
*   **Performance:** The forward and backward processes of the X-Gaussian model (which uses DRR) are much faster than those of the original 3DGS due to the avoidance of complex spherical harmonic (SH) function computations [S1].
*   **Strengths:** Enables differentiable rendering for X-ray novel view synthesis; reduces training time via the ACUI initialization strategy [S1].
*   **Limitations:** Specific to X-ray imaging geometry; relies on the accuracy of the intrinsic and extrinsic matrices derived from the X-ray scanner parameters [S1].

## Background
Differentiable Radiative Rasterization (DRR) was introduced in the context of "Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis" [S1]. It is part of a pipeline that includes a Radiative Gaussian Point Cloud Model and an Angle-pose Cuboid Uniform Initialization (ACUI) strategy [S1]. The method was developed to address the need for efficient and differentiable rendering in X-ray imaging, allowing for the optimization of Gaussian point clouds from X-ray projections [S1].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
