# Unbounded inward-facing scene parameterization

Also called: Mip-NeRF 360 parameterization, Scene Contraction, Mip-NeRF 360 Scene Contraction, Contracted space parameterization, Mip-NeRF 360 contraction

## Overview
This method parameterizes unbounded, inward-facing 360-degree scenes by mapping infinite 3D coordinates into a bounded unit sphere (or cuboid), enabling efficient voxel grid allocation and watertight mesh extraction [S1][S2, unofficial][S3]. It sets itself apart from standard NeRF by explicitly warping infinite $t$-space coordinates into a finite $s$-space, ensuring that model capacity is allocated proportional to disparity (inverse distance) for distant content, while maintaining visual fidelity for backgrounds [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D world coordinates $\mathbf{x} \in \mathbb{R}^3$ (and optionally viewing direction $\mathbf{d}$).
- Output: Contracted coordinates $\mathbf{x}' \in [-2, 2]^3$ (or bounded cuboid).

**Data Structures:**
- **Voxel Grid:** A 3D grid allocated in the contracted space. For the standard spherical contraction, a cuboid grid is often used to avoid wasting points [S2, unofficial].
- **Preprocessing State:** Camera centroid and scaling factor derived from input camera poses.

**Preprocessing (Coordinate Alignment):**
Before applying the contraction function, the world coordinates are transformed to align with the camera distribution [S2, unofficial]:
1. **Rotation:** Rotate the world coordinate system to align the first two Principal Component Analysis (PCA) directions of the camera positions with the grid's XY axes. This improves reconstruction quality [S2, unofficial].
2. **Shift:** Translate the world coordinates so that the centroid of the camera positions aligns with the origin [S2, unofficial].
3. **Scaling:** Scale the coordinates such that the cameras' near planes fit within a unit sphere (or unit cube) [S2, unofficial].

**Contraction Function:**
The core operation maps $\mathbf{x}$ to $\mathbf{x}'$. The standard Mip-NeRF 360 parameterization uses an $L_p$ norm-based scaling factor that preserves the direction of the input vector while compressing its magnitude [S2, unofficial].

The general formula for the contraction function $\operatorname{contract}(\mathbf{x})$ is [S1][S2, unofficial][S3]:

$$
\operatorname{contract}(\mathbf{x}) = \begin{cases} 
\mathbf{x} & \|\mathbf{x}\|_p \leq 1 \\ 
\left(1 + b - \frac{b}{\|\mathbf{x}\|_p}\right) \left(\frac{\mathbf{x}}{\|\mathbf{x}\|_p}\right) & \|\mathbf{x}\|_p > 1 
\end{cases}
$$

Where:
- $\|\mathbf{x}\|_p$ is the $L_p$ norm of $\mathbf{x}$.
- $b > 0$ is a hyperparameter controlling the proportion of voxel grid points allocated to the background [S2, unofficial].
- If $b=1$ and $p=2$, this reduces to the specific form mapping to a sphere of radius 2 [S1, S3]:
$$
\operatorname{contract}(\mathbf{x}) = \begin{cases} 
\mathbf{x} & \|\mathbf{x}\| \leq 1 \\ 
\left(2 - \frac{1}{\|\mathbf{x}\|}\right) \left(\frac{\mathbf{x}}{\|\mathbf{x}\|}\right) & \|\mathbf{x}\| > 1 
\end{cases}
$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contract_coordinates(x, p=2, b=1.0):
    """
    Contracts unbounded 3D coordinates into a bounded space.
    
    Args:
        x: numpy array of shape (N, 3) representing 3D coordinates.
        p: The norm to use (2 for Euclidean, inf for L-infinity).
        b: Hyperparameter controlling background voxel proportion.
           For standard Mip-NeRF 360 spherical contraction, b=1.
    
    Returns:
        x_prime: numpy array of shape (N, 3) representing contracted coordinates.
    """
    # Compute the norm of each point
    if p == float('inf'):
        # L-infinity norm: max absolute value across dimensions
        norms = np.max(np.abs(x), axis=1)
    else:
        # L-p norm (typically L2)
        norms = np.linalg.norm(x, axis=1, ord=p)
    
    # Initialize output array
    x_prime = np.zeros_like(x)
    
    # Case 1: Points inside the unit sphere/cube (norm <= 1)
    # These remain unchanged
    inside_mask = norms <= 1.0
    x_prime[inside_mask] = x[inside_mask]
    
    # Case 2: Points outside the unit sphere/cube (norm > 1)
    # These are mapped to the bounded region
    outside_mask = ~inside_mask
    if np.any(outside_mask):
        # Normalize direction
        directions = x[outside_mask] / norms[outside_mask, np.newaxis]
        
        # Compute scaling factor: (1 + b - b / norm)
        # For standard Mip-NeRF 360 (b=1), this becomes (2 - 1/norm)
        scale_factors = 1.0 + b - (b / norms[outside_mask])
        
        # Apply scaling
        x_prime[outside_mask] = directions * scale_factors[:, np.newaxis]
        
    return x_prime

def preprocess_scene(cameras):
    """
    Preprocesses world coordinates based on camera positions.
    
    Args:
        cameras: List of camera poses (position and orientation).
        
    Returns:
        transform_matrix: 4x4 matrix to transform world coordinates.
    """
    # 1. Extract camera positions
    positions = np.array([cam.position for cam in cameras])
    
    # 2. Compute centroid
    centroid = np.mean(positions, axis=0)
    
    # 3. Center coordinates
    centered_positions = positions - centroid
    
    # 4. PCA to find principal directions
    # Compute covariance matrix
    cov = np.cov(centered_positions.T)
    # Get eigenvalues and eigenvectors
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    
    # Sort eigenvectors by eigenvalue (descending)
    # The first two PCA directions are aligned with XY axis
    pca_dirs = eigenvectors[:, np.argsort(eigenvalues)[::-1]][:, :2]
    
    # Construct rotation matrix to align PCA directions with XY
    # Note: Implementation detail for aligning specific axes may vary
    # [G] Assuming standard PCA alignment logic where we rotate the scene
    # so that the major axes of camera distribution align with grid axes.
    rotation_matrix = ... # Construct rotation matrix from pca_dirs
    
    # 5. Scale cameras to fit near planes in unit sphere
    # [G] Determine scale factor s such that max distance of near planes is 1.
    # This requires projecting camera positions onto their viewing directions
    # and finding the minimum distance. For simplicity, we assume a global
    # scale factor derived from the extent of camera positions.
    scale_factor = ... # Computed based on camera near-plane distances
    
    # Construct final transformation matrix
    # T = Scale * Rotation * Translate(-centroid)
    transform_matrix = np.eye(4)
    transform_matrix[:3, :3] = scale_factor * rotation_matrix
    transform_matrix[:3, 3] = -transform_matrix[:3, :3] @ centroid
    
    return transform_matrix
```

**Parameters:**
- $p$: Norm type. $p=2$ (Euclidean) maps to a sphere; $p=\infty$ maps to a cuboid [S2, unofficial].
- $b$: Hyperparameter for background proportion. $b>0$ [S2, unofficial].
- Grid Resolution: Typically $320^3$ for unbounded inward-facing scenes [S2, unofficial].
- Initial Alpha $\alpha_{init}$: $10^{-4}$ [S2, unofficial].
- Voxel Step Size: $0.5$ in contracted space [S2, unofficial].
- TV Loss Weight (Density): $10^{-6}$ [S2, unofficial].
- TV Loss Weight (Feature): $10^{-7}$ [S2, unofficial].
- Distortion Loss Weight: $10^{-2}$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Norm Selection:** Using $p=2$ (Euclidean norm) wastes approximately 50% of grid points because a sphere occupies only a fraction of a cube. Using $p=\infty$ creates a cuboid contracted space, which is more efficient for voxel grid allocation [S2, unofficial].
- **Boundedness:** Standard Occupancy Grids (e.g., from Instant-NGP) only support bounded scenes. Scene contraction is required to extend these grids to unbounded scenes [S4, unofficial].
- **Ray Intersections:** Spherical contraction maps straight lines to curves, making ray-AABB intersections challenging for empty space skipping. Alternative contractions (like Piecewise-Projective) preserve straight lines [S5].

## Subtopics

### [Mip-NeRF 360 Contraction and Skybox Handling](9762-mip-nerf-360-contraction-and-skybox-handling.md)
This method handles unbounded 360-degree scenes by contracting distant geometry into a bounded unit sphere and using a skybox for the remaining background [9762-mip-nerf-360-contraction-and-skybox-handling.md]. It sets itself apart from standard NeRF by explicitly warping infinite t-space coordinates into a finite s-space to enable efficient voxel grid allocation and watertight mesh extraction, while maintaining visual fidelity for distant backgrounds via a skybox representation.

### [Scene Scaling and Contraction](11280-scene-scaling-and-contraction.md)
This method preprocesses 3D scene coordinates to enable efficient training of neural radiance fields on unbounded, inward-facing scenes [11280-scene-scaling-and-contraction.md]. It solves the problem of infinite ray sampling by first scaling the scene to fit a unit cube, then mapping the entire 3D space into a bounded unit sphere. This allows the neural network to learn a compact representation where points at infinity are mapped to the sphere's surface. It differs from other contraction methods by using a specific L2-norm-based scaling factor that preserves the direction of the input vector while compressing its magnitude.

### Forward-facing scene parameterization
A method to handle unbounded forward-facing scenes by warping the frustum to a bounded volume using a dense voxel grid allocation similar to Multiplane Images (MPIs) [S2, unofficial].
- **Steps:** Place $D$ RGB-density images at fixed depths, each with $X \times Z$ resolution. Trace every ray from the first to the $D$-th images with a step size of $s$ image, resulting in $2D - 1$ sampled points if $s = 0.5$ [S2, unofficial].
- **Parameters:** Number of depth layers $D=256$. Voxel resolution per layer $XZ=384^2$. Sampling step size $s=1.0$ layer. TV loss weight for density grid is $10^{-5}$. TV loss weight for feature grid is $10^{-6}$. Distortion loss weight is $10^{-2}$ [S2, unofficial].

### Piecewise-Projective Contraction
A contraction function that maps unbounded space into a finite domain while preserving straight lines (piecewise-linear), allowing for efficient ray-AABB intersection tests for empty space skipping [S5].
- **Definition:** It consists of seven regions (unit cube + 6 outer regions) [S5].
- **Formula:**
$$
\operatorname{contract}_{\pi}(\mathbf{x})_j = \begin{cases} 
x_j & \text{if } \|\mathbf{x}\|_{\infty} \leq 1 \\ 
\frac{x_j}{\|\mathbf{x}\|_{\infty}} & \text{if } x_j \neq \|\mathbf{x}\|_{\infty} > 1, \\ 
\left(2 - \frac{1}{|x_j|}\right) & \text{if } x_j = \|\mathbf{x}\|_{\infty} > 1 
\end{cases}
$$
where $\|\mathbf{x}\|_{\infty} = \max_i |x_i|$ [S5].
- **Pitfall:** Mip-NeRF 360's spherical contraction maps straight lines to curves, making ray-AABB intersections challenging. MERF's contraction preserves straight lines within regions, with discontinuities only at region boundaries [S5].
- **Comparison:** Unlike spherical contraction which maps to a radius-2 ball, this contraction preserves the unit cube and maps other regions via projective transformations. It performs on par with spherical contraction but enables efficient empty space skipping [S5].

## Operating characteristics
- **Time Complexity:** The contraction function itself is $O(1)$ per point. The overhead comes from the preprocessing (PCA) which is $O(N_{cameras})$.
- **Memory Complexity:** The voxel grid size is determined by the contracted space bounds. Using $p=\infty$ (cuboid) is more memory-efficient than $p=2$ (sphere) for the same resolution, as it avoids wasting grid cells in the corners [S2, unofficial].
- **Measured Performance:** On unbounded inward-facing scenes, a grid resolution of $320^3$ with the specified loss weights yields high-quality results [S2, unofficial].
- **Strengths:** Enables the use of bounded data structures (like Occupancy Grids) for unbounded scenes. Allocates higher representational capacity to points near the cameras, which is consistent with perspective projection [S1, S3].
- **Limitations:** Spherical contraction distorts straight lines, complicating ray-AABB intersection tests for empty space skipping [S5]. The choice of norm ($p$) and background parameter ($b$) requires tuning for specific scene geometries [S2, unofficial].

## Background
The scene contraction idea originates from Mip-NeRF 360 [S4, unofficial][S1]. Specifically, it is based on Equation 10 in the Mip-NeRF 360 paper [S4, unofficial]. The method was adapted to warp unbounded 360-degree scenes into a bounded unit sphere volume for voxel grid allocation [S2, unofficial]. It is used in various neural radiance field implementations, including Zip-NeRF [S1], DVGOv2 [S2, unofficial], and NerfAcc [S4, unofficial], to support unbounded scenes with bounded occupancy grids.

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] Improved Direct Voxel Grid Optimization for Radiance Fields Reconstruction, https://doi.org/10.48550/arxiv.2206.05085, fetched 2026-10-26 (unofficial)
[S3] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S4] NerfAcc: A General NeRF Acceleration Toolbox, https://doi.org/10.48550/arxiv.2210.04847, fetched 2026-10-26 (unofficial)
[S5] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
