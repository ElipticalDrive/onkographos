# Normal-Guided Gaussian Splitting

Also called: Geometry-aware splitting with normal guidance, Normal-guided positioning

## Overview
Normal-Guided Gaussian Splitting is a densification strategy for 3D Gaussian Splatting that determines the initial 3D position of a newly split child Gaussian by projecting a randomly sampled candidate onto the tangent plane of the parent Gaussian, weighted by the local image gradient [S1]. It solves the problem of noisy or misplaced Gaussian splats in weakly textured regions where standard random splitting lacks visual constraints [S1]. Unlike methods relying on true-scaled depth priors from Structure-from-Motion (SfM), this approach avoids errors from unreliable SfM points or scale inconsistencies across frames [S1].

## How it works
**Inputs:**
- Parent Gaussian $G_a$ with mean $\mu_a$ and covariance $\Sigma_a$.
- Reference view $V^{(a)}$ (camera pose and intrinsics).
- Reference image $I_V^{(a)}$.
- Normal map $N_V^{(a)}$ corresponding to the reference view.
- A randomly sampled candidate position $\mu_b^c$ within the elliptical scale of $G_a$.

**Outputs:**
- Optimized child Gaussian mean $\mu_c$.

**Data Structures:**
- $\mu_a \in \mathbb{R}^3$: Parent mean.
- $\mu_b^c \in \mathbb{R}^3$: Randomly sampled candidate mean.
- $N_V^{(a)}(\bar{x}_a) \in \mathbb{R}^3$: Normal vector at projected pixel $\bar{x}_a$.
- $\nabla I_V^{(a)}(\bar{x}_c) \in \mathbb{R}$: Scalar image gradient magnitude at projected pixel $\bar{x}_c$.

**Steps:**
1. **Project Parent:** Project the parent Gaussian $G_a$ to the reference view $V^{(a)}$ to obtain the 2D pixel coordinate $\bar{x}_a$.
2. **Project Candidate:** Project the randomly sampled candidate position $\mu_b^c$ to the reference view $V^{(a)}$ to obtain the 2D pixel coordinate $\bar{x}_c$.
3. **Retrieve Normal:** Retrieve the surface normal vector $N_V^{(a)}(\bar{x}_a)$ from the normal map at pixel $\bar{x}_a$.
4. **Retrieve Gradient:** Retrieve the image gradient magnitude $\nabla I_V^{(a)}(\bar{x}_c)$ from the reference image at pixel $\bar{x}_c$.
5. **Compute Tangent Projection:** Calculate the perpendicular projection $\mu^\perp$ of the candidate $\mu_b^c$ onto the tangent plane of the parent Gaussian. The tangent plane is defined by point $\mu_a$ and normal $N_V^{(a)}(\bar{x}_a)$.
6. **Compute Final Position:** Calculate $\mu_c$ using the weighted combination of the candidate and the tangent projection, weighted by the image gradient.

**Formulas:**
The tangent projection $\mu^\perp$ is computed as:
$$ \mu^\perp = \mu_b^c - ((\mu_b^c - \mu_a) \cdot N_V^{(a)}(\bar{x}_a))N_V^{(a)}(\bar{x}_a) $$
where $N_V^{(a)}(\bar{x}_a)$ is the unit normal vector [S1].

The final child mean $\mu_c$ is computed as:
$$ \mu_c = (\mu_b^c - \mu^\perp)\nabla I_V^{(a)}(\bar{x}_c) + \mu^\perp $$
where $\nabla I_V^{(a)}(\bar{x}_c)$ acts as a weight [S1].

**Parameters:**
- Image gradient $\nabla I$: Serves as a dynamic weight. In weakly textured regions (small gradients), the child stays close to the geometric surface ($\mu_c \approx \mu^\perp$). In fully textured areas (large gradients), the randomness of sampling is preserved ($\mu_c \approx \mu_b^c$) [S1].

**Edge Cases and Pitfalls:**
- **Weakly Textured Regions:** Standard splitting may place splats incorrectly due to lack of visual details. This method handles it by enforcing proximity to the parent's geometric surface via the normal vector [S1].
- **Scale Inconsistency:** Unlike methods using true-scaled depth priors (e.g., DAv2 alignment), this method does not require true scale, avoiding errors from unreliable SfM points or scale inconsistency across frames [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normal_guided_splitting(parent_mean, candidate_mean, normal_map, image, camera_intrinsics, camera_pose):
    """
    Computes the optimized child mean for a split Gaussian.
    
    Args:
        parent_mean: 3D vector, mean of parent Gaussian G_a.
        candidate_mean: 3D vector, randomly sampled mean within parent scale.
        normal_map: 2D array or function, returns normal vector at pixel.
        image: 2D array or function, returns image intensity/gradient at pixel.
        camera_intrinsics: Camera intrinsic matrix.
        camera_pose: Camera extrinsic matrix (world to camera).
        
    Returns:
        child_mean: 3D vector, optimized mean mu_c.
    """
    # Step 1: Project parent to reference view
    # Homogeneous coordinates
    parent_h = np.append(parent_mean, 1.0)
    # Transform to camera space
    parent_cam = camera_pose @ parent_h
    # Project to 2D pixel coordinates
    x_bar_a = project_to_pixel(parent_cam[:3], camera_intrinsics)
    
    # Step 2: Project candidate to reference view
    candidate_h = np.append(candidate_mean, 1.0)
    candidate_cam = camera_pose @ candidate_h
    x_bar_c = project_to_pixel(candidate_cam[:3], camera_intrinsics)
    
    # Step 3: Retrieve normal at parent's projected pixel
    # Normal map provides normal in camera space or world space? 
    # Source implies N_V^(a)(x_bar_a) is used directly in dot product with 3D vectors.
    # Assuming normal map returns normal in the same coordinate system as the 3D points (likely camera space or world space aligned).
    # For implementation, ensure coordinate consistency.
    normal_vec = normal_map(x_bar_a)
    
    # Step 4: Retrieve image gradient at candidate's projected pixel
    # Gradient magnitude
    grad_mag = compute_gradient_magnitude(image, x_bar_c)
    
    # Step 5: Compute perpendicular projection mu_perp
    # mu_perp = mu_b^c - ((mu_b^c - mu_a) . N) * N
    diff = candidate_mean - parent_mean
    projection_scalar = np.dot(diff, normal_vec)
    mu_perp = candidate_mean - projection_scalar * normal_vec
    
    # Step 6: Compute final child mean mu_c
    # mu_c = (mu_b^c - mu_perp) * grad + mu_perp
    child_mean = (candidate_mean - mu_perp) * grad_mag + mu_perp
    
    return child_mean

def project_to_pixel(point_cam, K):
    """Projects a 3D point in camera space to 2D pixel coordinates."""
    x = point_cam[0] / point_cam[2]
    y = point_cam[1] / point_cam[2]
    u = K[0, 0] * x + K[0, 2]
    v = K[1, 1] * y + K[1, 2]
    return np.array([u, v])

def compute_gradient_magnitude(image, pixel):
    """Computes the magnitude of the image gradient at a pixel."""
    # Simple Sobel or finite difference approximation
    # Placeholder for specific gradient computation method
    # In practice, use Sobel filters on the image
    pass
```

## Subtopics

### [Tuning-Free Model-Size Control for 3D Gaussian Splatting](18917-tuning-free-model-size-control-for-3d-gaussian-splatting.md)
A method for controlling model size without manual tuning. Choose when model size is the primary constraint.

### [PSCD-GS](18934-pscd-gs.md)
A densification strategy using point cloud sampling and density control. Choose when point cloud priors are available and effective.

## Operating characteristics
- **Time Complexity:** Adds overhead for normal map lookup and gradient computation per split operation.
- **Memory Complexity:** Requires storage/access to normal maps and image gradients.
- **Performance:** Produces cleaner, more reasonably and accurately distributed splats, offering superior rendering quality (SSIM, PSNR, LPIPS) compared to methods relying on true-scaled depth priors [S1].
- **Strengths:** Effective in weakly textured regions; avoids errors from unreliable SfM points.
- **Limitations:** Relies on the availability and accuracy of normal maps and image gradients.

## Background
Originated in the GeoTexDensifier paper, which combines texture-aware densification and geometry-aware splitting strategies [S1]. It was developed to address the limitations of standard random splitting in 3D Gaussian Splatting, particularly in areas with low visual detail.

## Sources
[S1] GeoTexDensifier: Geometry-Texture-Aware Densification for High-Quality Photorealistic 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3644697, fetched 2026-10-26
