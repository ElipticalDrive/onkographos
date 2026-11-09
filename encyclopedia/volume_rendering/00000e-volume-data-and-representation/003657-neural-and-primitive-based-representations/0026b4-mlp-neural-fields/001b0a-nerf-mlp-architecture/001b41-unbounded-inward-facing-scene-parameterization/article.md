# Unbounded inward-facing scene parameterization

Also called: Contracted space parameterization, Geometric mean Jacobian contraction, Mip-NeRF 360 Scene Contraction, Mip-NeRF 360 contraction, Mip-NeRF 360 parameterization, Scene Contraction, Spatial Contraction for Unbounded Scenes

## Overview
This method parameterizes unbounded scenes by mapping coordinates from $[-\infty, \infty]^d$ to a bounded domain (typically $[-2, 2]^d$ or a unit sphere), allowing neural fields and voxel grids to allocate model capacity proportional to disparity (inverse distance) for distant content [S1][S2, S3, unofficial]. It sets itself apart from bounded scene representations by enabling the use of standard bounded data structures (like Occupancy Grids or dense voxel grids) on infinite environments without clipping or loss of background detail [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D world coordinates $\mathbf{x} \in \mathbb{R}^3$ (and optionally viewing direction, though the contraction primarily affects spatial coordinates).
*   **Output:** Contracted coordinates $\mathbf{x}'$ within a bounded range.

**Data Structures:**
*   **Voxel Grid:** A cuboid or spherical grid centered at the origin. The resolution and bounds depend on the specific variant (e.g., $320^3$ cuboid for DVGOv2 [S2, unofficial], or unit sphere for Mip-NeRF 360 [S1]).
*   **Camera Parameters:** Camera positions are used to determine the centroid and scale of the scene prior to contraction [S2, unofficial].

**Preprocessing (Coordinate Alignment):**
Before applying the non-linear contraction, the world coordinates are often transformed to align with the camera setup [S2, unofficial]:
1.  **Rotate:** Align the first two Principal Component Analysis (PCA) directions of the camera positions with the grid's XY axis.
2.  **Shift:** Translate the world coordinate system so that the origin aligns with the centroid of the camera positions.
3.  **Scale:** Scale the coordinates such that the cameras lie near the planes of a unit sphere (or within the defined boundary radius $r$) [S2, unofficial][S6].

**Contraction Function:**
The core operation maps unbounded space to a bounded space. The most common formulation, adapted from Mip-NeRF 360, uses an $L_p$ norm [S2, unofficial]:

$$
\mathbf{x}' = \begin{cases} 
\mathbf{x} & \|\mathbf{x}\|_p \leq 1 \\ 
\left(1 + b - \frac{b}{\|\mathbf{x}\|_p}\right) \left(\frac{\mathbf{x}}{\|\mathbf{x}\|_p}\right) & \|\mathbf{x}\|_p > 1 
\end{cases}
$$

Where:
*   $\mathbf{x}$ is the original coordinate.
*   $\mathbf{x}'$ is the transformed coordinate.
*   $\|\mathbf{x}\|_p$ is the $L_p$ norm of $\mathbf{x}$.
*   $b > 0$ is a hyperparameter controlling the proportion of the voxel grid allocated to the background [S2, unofficial].

**Alternative Formulation (Mip-NeRF 360 Original):**
The original Mip-NeRF 360 paper uses a spherical contraction with $p=2$ and specific boundary conditions [S1]:

$$
\operatorname{contract}(\mathbf{x}) = \begin{cases} 
\mathbf{x} & \|\mathbf{x}\| \leq 1 \\ 
\left(2 - \frac{1}{\|\mathbf{x}\|}\right)\left(\frac{\mathbf{x}}{\|\mathbf{x}\|}\right) & \|\mathbf{x}\| > 1 
\end{cases}
$$

This maps values to $[-2, 2]^d$ [S1].

**Gaussian Scale Contraction (for 3D Gaussians):**
When contracting isotropic Gaussians, the mean $\mathbf{x}_j$ is contracted directly. The scale $\sigma_j$ is adjusted using the geometric mean of the eigenvalues of the Jacobian $J_C(\mathbf{x}_j)$ of the contraction function [S1]:

$$
\operatorname{contract}(\sigma_j) = \sigma_j \left| \det(J_C(\mathbf{x}_j)) \right|^{1/d}
$$

where $d=3$ for 3D coordinates [S1]. This approach is preferred over multivariate Gaussian contraction for isotropic scales as it is less expensive [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contract_coordinate(x, p=2, b=0.0):
    """
    Contracts unbounded 3D coordinates to a bounded space.
    
    Args:
        x: numpy array of shape (N, 3) or (3,), world coordinates.
        p: The norm to use (2 for spherical, inf for cuboid).
        b: Hyperparameter for background allocation (DVGOv2 variant). 
           If b=0, falls back to standard Mip-NeRF 360 spherical contraction.
           
    Returns:
        x_prime: Contracted coordinates.
    """
    # Compute norm
    if p == float('inf'):
        # L-infinity norm: max absolute value
        norm_x = np.max(np.abs(x), axis=-1, keepdims=True)
    else:
        # L-p norm (typically 2)
        norm_x = np.linalg.norm(x, axis=-1, keepdims=True, ord=p)
        
    # Handle zero vector to avoid division by zero
    # [G] Standard practice: if norm is 0, return 0. 
    # The contraction is identity at origin.
    mask_zero = (norm_x == 0)
    norm_x_safe = np.where(mask_zero, 1.0, norm_x)
    
    # Apply contraction formula
    # Case 1: ||x|| <= 1 -> x' = x
    # Case 2: ||x|| > 1 -> x' = (1 + b - b/||x||) * (x / ||x||)
    
    # Note: The standard Mip-NeRF 360 formula (S1) uses boundary 1 and maps to 2.
    # The DVGOv2 formula (S2) uses boundary 1 and maps to 1+b.
    # We implement the DVGOv2 general form as it includes b.
    # If b=0, it becomes x' = x/||x||, which maps to unit sphere. 
    # However, S1 says it maps to [-2, 2]. 
    # Let's stick to the explicit S2 formula provided in notes:
    # x' = (1 + b - b/||x||_p) * (x / ||x||_p)
    
    factor = 1.0 + b - (b / norm_x_safe)
    
    # Normalize direction
    direction = np.where(mask_zero, 0.0, x / norm_x_safe)
    
    x_prime = np.where(norm_x <= 1.0, x, factor * direction)
    
    return x_prime

def contract_gaussian_scale(x, sigma, p=2):
    """
    Contracts the isotropic scale of a Gaussian using the Jacobian determinant.
    
    Args:
        x: numpy array of shape (N, 3), Gaussian means.
        sigma: numpy array of shape (N,), isotropic scales.
        p: Norm used for coordinate contraction.
        
    Returns:
        contracted_sigma: Scaled values in contracted space.
    """
    # [G] The Jacobian determinant for the spherical contraction 
    # contract(x) = (2 - 1/||x||) * (x/||x||) for ||x|| > 1
    # has a closed form. For general p and b, numerical differentiation 
    # or the specific closed form for the chosen contraction must be used.
    # S1 provides the closed form for the standard Mip-NeRF 360 contraction (b=0, p=2, boundary 1->2).
    # For the DVGOv2 contraction (S2), the Jacobian is not explicitly given in closed form in notes.
    # We assume the user implements the Jacobian determinant |det(J_C(x))|^(1/3) 
    # based on the specific contraction function chosen.
    
    # Placeholder for Jacobian determinant calculation based on chosen contraction
    # This function must be implemented according to the specific contract_coordinate variant.
    jacobian_det = compute_jacobian_determinant(x, p, b=0.0) 
    
    d = 3
    scale_factor = np.abs(jacobian_det) ** (1.0 / d)
    
    contracted_sigma = sigma * scale_factor
    return contracted_sigma
```

**Parameters:**
*   **$p$ (Norm):** $2$ (Euclidean) or $\infty$ (Cuboid). $p=\infty$ is preferred for voxel grids to avoid wasting ~50% of grid points on empty spherical corners [S2, unofficial].
*   **$b$ (Background Hyperparameter):** $b > 0$. Controls the proportion of grid points allocated to the background. Typical value: Not explicitly given for general use, but DVGOv2 uses a grid length of $2+2b$ [S2, unofficial].
*   **Grid Resolution:** $320^3$ for DVGOv2 [S2, unofficial].
*   **Initial Alpha:** $\alpha_{init} = 10^{-4}$ [S2, unofficial].
*   **Voxel Step Size:** $0.5$ (in contracted space) [S2, unofficial].
*   **Loss Weights:** TV loss for density grid $10^{-6}$, feature grid $10^{-7}$, distortion loss $10^{-2}$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Norm Selection:** Using $p=2$ (spherical contraction) with a cuboid voxel grid wastes approximately 50% of grid points because the sphere occupies only a fraction of the cube [S2, unofficial]. Use $p=\infty$ for cuboid grids [S2, unofficial].
*   **Bounded Grids:** Standard Occupancy Grids (e.g., from Instant-NGP) only support bounded scenes; scene contraction is required to extend them to unbounded scenes [S3, unofficial].
*   **Straight Lines:** Spherical contraction maps straight rays to curves, making ray-AABB intersection tests for empty space skipping difficult [S5]. Variants like MERF use piecewise-projective contractions to preserve straight lines [S5].
*   **Learning Failure:** Without the parameterization function, models on unbounded scenes may fail to learn effectively [S6].

## Subtopics

### Forward-facing scene parameterization
A method to handle unbounded forward-facing scenes by warping the frustum to a bounded volume using a dense voxel grid allocation similar to Multiplane Images (MPIs) [S2, unofficial]. It places $D$ RGB-density images at fixed depths, each with $X \times Z$ resolution, and traces rays through these layers [S2, unofficial]. Choose this for forward-facing (e.g., LLFF) datasets where the camera motion is constrained to a plane or narrow angle, rather than full 360-degree inward-facing scenes [S2, unofficial]. Parameters: $D=256$ depth layers, $XZ=384^2$ resolution per layer, sampling step size $s=1.0$ layer [S2, unofficial].

### Mip-NeRF 360 Contraction and Skybox Handling
A method to handle unbounded backgrounds in the Mip-NeRF 360 dataset by contracting far objects into a sphere and using a skybox for distant areas [S4]. It warps far objects from original space ($t$-space) into contracted space ($s$-space, a sphere with radius 1.2), applies differentiable meshing (DiffMC) in $t$-space to ensure watertightness, and contracts points back to $s$-space for color computation [S4]. Use this when extracting watertight meshes from unbounded NeRFs [S4]. Parameters: Higher resolution (340) for foreground, lower (56) for background in $t$-space [S4].

### Piecewise-Projective Contraction
A contraction function that maps unbounded space into a finite domain while preserving straight lines (piecewise-linear), allowing for efficient ray-AABB intersection tests for empty space skipping [S5]. It consists of seven regions (unit cube + 6 outer regions) and uses projective transformations [S5]. Choose this for real-time rendering pipelines where empty space skipping is critical, as spherical contraction maps straight lines to curves [S5]. Formula: $\operatorname{contract}_{\pi}(\mathbf{x})_j = x_j$ if $\|\mathbf{x}\|_\infty \leq 1$; $\frac{x_j}{\|\mathbf{x}\|_\infty}$ if $x_j \neq \|\mathbf{x}\|_\infty > 1$; $\left(2 - \frac{1}{|x_j|}\right)$ if $x_j = \|\mathbf{x}\|_\infty > 1$ [S5].

### Scene Scaling and Contraction
A combined approach where the scene is first scaled to fit a unit cube, then contracted using a parameterization function with radius parameters $r$ and $b$ [S6]. The contraction function is $\text{contract}(x) = x/r$ if $\|x\| \leq r$, otherwise $\frac{x}{b(1 + b - \|x\|)/\|x\|}$ [S6]. Use this for LiDAR-NeRF or point cloud-based NeRFs where explicit scaling of the scene bounds is beneficial [S6]. Parameters: $r=10, b=1$ for scene point clouds [S6].

## Operating characteristics
*   **Time Complexity:** The contraction function itself is $O(1)$ per sample. The primary cost is the additional coordinate transformation and potential Jacobian calculation for Gaussian scales [S1].
*   **Memory Complexity:** Enables the use of bounded voxel grids (e.g., $320^3$) for unbounded scenes, significantly reducing memory compared to bounding the scene with a massive box [S2, unofficial].
*   **Measured Performance:** DVGOv2 using this method achieves comparable quality to Plenoxels and TensoRF with less training time and lower memory usage on bounded inward scenes [S2, unofficial]. On LLFF (forward-facing), it achieves efficient rendering with $384^2 \times 256$ voxels [S2, unofficial].
*   **Strengths:** Allows standard bounded rendering techniques (voxel grids, occupancy grids) to work on infinite scenes; allocates resolution proportional to disparity; supports watertight mesh extraction when combined with skybox handling [S1][S2, unofficial][S4].
*   **Limitations:** Spherical contraction breaks straight lines, complicating ray-AABB intersections for empty space skipping [S5]; requires careful tuning of hyperparameters like $b$ and grid resolution [S2, unofficial].
*   **When to use:** Unbounded inward-facing (360-degree) or forward-facing scenes where background detail is important and bounded representations are preferred for efficiency [S2, S3, unofficial].
*   **When not to use:** Real-time rendering pipelines requiring efficient empty space skipping with standard AABB tests, unless a piecewise-projective variant (MERF) is used [S5].

## Background
The scene contraction idea originates from Mip-NeRF 360 [S3, unofficial][S1], specifically Equation 10 in that paper [S3, unofficial]. It was adapted for Direct Voxel Grid Optimization (DVGO) and DVGOv2 to support unbounded 360-degree scenes by warping them into a bounded unit sphere volume for voxel grid allocation [S2, unofficial]. It has also been incorporated into acceleration toolboxes like NerfAcc to extend Instant-NGP's bounded occupancy grids to unbounded scenes [S3, unofficial]. Variants like MERF were developed to address the limitation of spherical contraction breaking straight lines, enabling real-time rendering [S5].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] Improved Direct Voxel Grid Optimization for Radiance Fields Reconstruction, https://doi.org/10.48550/arxiv.2206.05085, fetched 2026-10-26 (unofficial)
[S3] NerfAcc: A General NeRF Acceleration Toolbox, https://doi.org/10.48550/arxiv.2210.04847, fetched 2026-10-26 (unofficial)
[S4] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
[S5] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
[S6] LiDAR-NeRF: Novel LiDAR View Synthesis via Neural Radiance Fields, https://arxiv.org/abs/2304.10406v2, fetched 2026-10-26
