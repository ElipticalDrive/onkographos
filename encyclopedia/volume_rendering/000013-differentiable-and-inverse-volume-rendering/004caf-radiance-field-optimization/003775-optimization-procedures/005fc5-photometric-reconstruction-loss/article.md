# Photometric Reconstruction Loss

Also called: 2D re-rendering loss, Geometric Photometric Loss, Lrec, Lrgb, Multi-view Photometric Loss, Reprojection Loss

## Overview
Photometric Reconstruction Loss minimizes the error between rendered pixel colors and input training images during joint optimization of scene representation and camera parameters [S3, unofficial]. It distinguishes itself from standard pixel-wise losses by explicitly mapping 3D sample coordinates from a canonical ray to camera-specific rays via rigid transformations, ensuring geometric consistency across views [S3, unofficial]. Unlike sibling methods that may rely solely on intensity matching without explicit 3D warping, this method enforces that the radiance field evaluated at the transformed 3D position matches the observed image color [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $L$: Number of input 2D images [S3, unofficial].
    *   $I = \{I_1, \dots, I_L\}$: Set of input images, where $I_{iu} \in \mathbb{R}^3$ is the RGB color of pixel $u$ in image $i$ [S3, unofficial].
    *   $P = \{P_1, \dots, P_L\}$: Camera poses, parameterized as elements of the $se(3)$ Lie algebra, representing rigid 3D transformations [S3, unofficial].
    *   $F_\sigma, F_c$: Radiance field functions representing density and color, respectively [S3, unofficial].
    *   $U$: Set of all possible 2D coordinates in the input images [S3, unofficial].
*   **Outputs:**
    *   Scalar loss value $L_{rec}$ representing the photometric error [S3, unofficial].

**Data Structures:**
*   **Camera Pose ($P_i$):** Represented as a vector in $se(3)$ Lie algebra [S3, unofficial].
*   **Sample Coordinate ($s(\vec{0}, \vec{d_u})$):** A 3D point on the canonical ray (origin $\vec{0}$, direction $\vec{d_u}$) corresponding to pixel $u$ [S3, unofficial].
*   **Warping Function ($W_{3d}$):** A function $W_{3d}: \mathbb{R}^3 \to \mathbb{R}^3$ that performs rigid 3D transformation parameterized by $P \in se(3)$ [S3, unofficial].

**Steps:**
1.  For each image $i$ from $1$ to $L$:
2.  For each pixel coordinate $u$ in the set $U$:
3.  Determine the 3D sample coordinate in the canonical ray space: $s_{canon} = s(\vec{0}, \vec{d_u})$, where $\vec{d_u}$ is the ray direction for pixel $u$ [S3, unofficial].
4.  Warp the canonical sample coordinate to the camera ray space using the camera pose $P_i$:
    $$ s_{cam} = W_{3d}(P_i, s_{canon}) $$
    This maps the sample 3D coordinate from the canonical ray into the 3D sample coordinate of the camera ray with pose $P_i$ [S3, unofficial].
5.  Evaluate the radiance field at the warped coordinate to get the rendered color:
    $$ \hat{C}_{iu} = V(F_\sigma, F_c, s_{cam}) $$
    where $V$ is the volume rendering operator [S3, unofficial].
6.  Compute the difference between the rendered color and the ground truth image color $I_{iu}$:
    $$ \text{error}_{iu} = \| \hat{C}_{iu} - I_{iu} \| $$
7.  Sum the errors over all pixels and images to get the total loss:
    $$ L_{rec}(F_\sigma, F_c) = \sum_{i=1}^{L} \sum_{u \in U} \| V(F_\sigma, F_c, W_{3d}(P_i, s(\vec{0}, \vec{d_u}))) - I_{iu} \| $$
    [S3, unofficial]

**Formulas:**
The core photometric reconstruction loss is defined as [S3, unofficial]:
$$ L_{rec}(F_\sigma, F_c) = \sum_{i=1}^{L} \sum_{u \in U} \| V(F_\sigma, F_c, W_{3d}(P_i, s(\vec{0}, \vec{d_u}))) - I_{iu} \| $$

In joint optimization scenarios where camera poses $P$ are also unknown and optimized, the objective extends to [S3, unofficial]:
$$ L_{joint}(F_\sigma, F_c, P) = \sum_{i=1}^{L} \sum_{u \in U} \| V(F_\sigma, F_c, W_{3d}(P_i, s(\vec{0}, \vec{d_u}))) - I_{iu} \| $$

**Parameters:**
*   $P_i$: Camera pose parameterized in $se(3)$ Lie algebra [S3, unofficial].
*   Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Ill-posedness:** The reconstruction problem is ill-posed and suffers from shape-radiance ambiguity [S3, unofficial].
*   **Joint Optimization:** When jointly estimating camera poses and scene representation, the problem becomes even more ill-defined [S3, unofficial].
*   **Precision:** The warping function $W_{3d}$ must correctly handle the $se(3)$ parameterization to avoid geometric misalignment [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def photometric_reconstruction_loss(F_sigma, F_c, images, poses, intrinsics, U):
    """
    Computes the photometric reconstruction loss L_rec.
    
    Args:
        F_sigma: Density field query function.
        F_c: Color field query function.
        images: List of L images, where images[i][u] is the RGB value of pixel u in image i.
        poses: List of L camera poses, where poses[i] is the se(3) parameter for image i.
        intrinsics: Camera intrinsic matrix K.
        U: List of 2D pixel coordinates (u, v) to evaluate.
        
    Returns:
        Total photometric loss.
    """
    total_loss = 0.0
    L = len(images)
    
    for i in range(L):
        P_i = poses[i]
        image_i = images[i]
        
        for u_coord in U:
            # Ground truth color
            I_iu = image_i[u_coord]
            
            # 1. Compute ray direction from pixel coordinate and intrinsics [G]
            d_u = compute_ray_direction(u_coord, intrinsics)
            
            # 2. Sample a 3D point on the canonical ray (origin 0, direction d_u) [G]
            s_canon = sample_on_canonical_ray(d_u, depth=1.0)
            
            # 3. Warp canonical sample to camera ray space using pose P_i [S3, unofficial]
            s_cam = apply_se3_transform(P_i, s_canon)
            
            # 4. Evaluate radiance field at warped position [S3, unofficial]
            rendered_color = evaluate_radiance_field(F_sigma, F_c, s_cam)
            
            # 5. Compute L2 norm difference and accumulate
            error = compute_l2_norm(rendered_color - I_iu)
            total_loss += error
            
    return total_loss

# Self-contained helper definitions
def compute_ray_direction(pixel_uv, K):
    # Standard pinhole camera projection inversion [G]
    u, v = pixel_uv
    fx, fy, cx, cy = K[0,0], K[1,1], K[0,2], K[1,2]
    x = (u - cx) / fx
    y = (v - cy) / fy
    z = 1.0
    return normalize([x, y, z])

def sample_on_canonical_ray(direction, depth=1.0):
    # Returns a 3D point along the canonical ray [G]
    return [direction[0] * depth, direction[1] * depth, direction[2] * depth]

def apply_se3_transform(se3_params, point):
    # Applies rigid transformation from se(3) parameterization [S3, unofficial]
    R = se3_params['R']
    t = se3_params['t']
    x, y, z = point
    return [
        R[0,0]*x + R[0,1]*y + R[0,2]*z + t[0],
        R[1,0]*x + R[1,1]*y + R[1,2]*z + t[1],
        R[2,0]*x + R[2,1]*y + R[2,2]*z + t[2]
    ]

def evaluate_radiance_field(F_sigma, F_c, point):
    # Queries density and color, applies standard emission-absorption model [G]
    sigma = F_sigma(point)
    color = F_c(point)
    # Simplified single-sample rendering for demonstration
    return color

def compute_l2_norm(vec):
    return (vec[0]**2 + vec[1]**2 + vec[2]**2)**0.5

def normalize(vec):
    mag = (vec[0]**2 + vec[1]**2 + vec[2]**2)**0.5
    return [v/mag for v in vec]
```

## Subtopics

### [Photometric Loss (RawNeRF Variant)](9563-photometric-loss-rawnerf-variant.md)
Specialized for NeRFs using a "gradient trick" to reduce bias in Monte Carlo estimators.

### [Photometric Loss](11090-photometric-loss.md)
Combines $\ell_1$ reconstruction with D-SSIM for perceptual structure.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enforces geometric consistency across views by explicitly warping 3D points [S3, unofficial].
*   **Limitations:** The problem is ill-posed and suffers from shape-radiance ambiguity [S3, unofficial]. Joint optimization is highly ill-defined without additional constraints [S3, unofficial].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method originates from the need to jointly optimize camera poses and neural radiance fields [S3, unofficial]. The use of $se(3)$ Lie algebra for camera pose parameterization is a common technique in robotics [S3, unofficial]. The formulation addresses the ill-posed nature of shape-radiance ambiguity [S3, unofficial].

## Sources
[S1] StegaNeRF: Embedding Invisible Information within Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.00047, fetched 2026-10-26
[S2] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
[S3] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S4] PVA: Pixel-aligned Volumetric Avatars, https://doi.org/10.48550/arxiv.2101.02697, fetched 2026-10-30 (unofficial)
