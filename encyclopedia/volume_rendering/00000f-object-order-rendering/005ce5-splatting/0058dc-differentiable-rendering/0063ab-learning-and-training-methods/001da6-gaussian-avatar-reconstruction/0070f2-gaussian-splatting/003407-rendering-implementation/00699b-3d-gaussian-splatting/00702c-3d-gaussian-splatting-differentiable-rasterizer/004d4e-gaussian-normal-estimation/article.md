# Gaussian Normal Estimation

Also called: Alpha-composited Gaussian normals, Per-pixel Normal Map Estimation from Depth, Per-pixel normal estimation from Gaussians

## Overview
Gaussian Normal Estimation derives per-pixel surface normal maps directly from the geometry of 3D Gaussians during the differentiable rasterization process, providing geometric supervision without requiring additional learnable parameters [S1][S2, unofficial]. It sets itself apart from sibling methods by treating the shortest axis of the Gaussian covariance matrix as the local surface normal, allowing gradients from normal consistency losses to flow back to the Gaussian rotation and scale parameters [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians, each defined by position, covariance (scale and rotation), opacity $\alpha$, and color [S1][S2, unofficial]. For each pixel, the list of contributing Gaussians $N$ sorted by depth (front-to-back) [Parent].
*   **Outputs:** A per-pixel estimated normal vector $\hat{\boldsymbol{N}} \in \mathbb{R}^3$ in camera space [S1][S2, unofficial].

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores a normalized normal vector $\hat{\boldsymbol{n}}_i \in \mathbb{R}^3$ [S2, unofficial]. This vector represents the direction of the shortest axis of the Gaussian's covariance matrix [S2, unofficial]. In practice, this is derived from the rotation matrix $\mathbf{R}_i$ of the Gaussian, typically corresponding to the third column (or the axis associated with the smallest scale eigenvalue) [G].
*   **Accumulation State:** For each pixel, a running transmittance $T$ (initialized to 1.0) and a running accumulated normal $\hat{\boldsymbol{N}}$ (initialized to $\mathbf{0}$) are maintained during the rasterization pass [S2, unofficial].

**Steps:**
1.  **Normal Extraction:** For each Gaussian $i$, extract the normalized vector $\hat{\boldsymbol{n}}_i$ corresponding to the direction of the shortest scaling axis of its covariance matrix [S2, unofficial].
2.  **Orientation Correction:** At the beginning of training (or per-frame), check the orientation [S1]. If the dot product between the current camera viewing direction $\boldsymbol{v}$ and the Gaussian normal $\hat{\boldsymbol{n}}_i$ is negative ($\boldsymbol{v} \cdot \hat{\boldsymbol{n}}_i < 0$), flip the direction of the normal ($\hat{\boldsymbol{n}}_i \leftarrow -\hat{\boldsymbol{n}}_i$) to ensure consistent outward-facing orientation [S1].
3.  **Camera Space Transformation:** Transform the extracted normal $\hat{\boldsymbol{n}}_i$ from world space to camera space using the current camera rotation matrix $\mathbf{R}_{cam}$ [S1]. Let the transformed normal be $\boldsymbol{n}_i$ [S1].
4.  **Alpha Compositing:** Iterate through the sorted list of Gaussians $i \in N$ contributing to the pixel [S2, unofficial]. For each Gaussian, update the accumulated normal and transmittance:
    *   Compute the weight contribution: $w_i = \alpha_i T_i$, where $\alpha_i$ is the Gaussian's opacity and $T_i$ is the transmittance accumulated from previous (closer) Gaussians [S2, unofficial].
    *   Update the accumulated normal: $\hat{\boldsymbol{N}} \leftarrow \hat{\boldsymbol{N}} + \boldsymbol{n}_i w_i$ [S2, unofficial].
    *   Update the transmittance: $T_{i+1} = T_i (1 - \alpha_i)$ [S2, unofficial].
5.  **Normalization:** After iterating through all contributing Gaussians, normalize the resulting vector: $\hat{\boldsymbol{N}} \leftarrow \frac{\hat{\boldsymbol{N}}}{\|\hat{\boldsymbol{N}}\| + \epsilon}$, where $\epsilon$ is a small constant to prevent division by zero [G].

**Formulas:**
The per-pixel normal estimate $\hat{\boldsymbol{N}}$ is computed by alpha-compositing individual Gaussian normals $\hat{\boldsymbol{n}}_i$ [S1][S2, unofficial]:

$$
\hat{\boldsymbol{N}} = \sum_{i \in N} \boldsymbol{n}_i \alpha_i T_i
$$

Where:
*   $N$ is the set of Gaussians sorted by depth contributing to the pixel [S2, unofficial].
*   $\boldsymbol{n}_i \in \mathbb{R}^3$ is the normalized vector representing the direction of the shortest axis of the $i$-th Gaussian, transformed into camera space [S1].
*   $\alpha_i$ is the opacity of the $i$-th Gaussian [S2, unofficial].
*   $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance from previous Gaussians [S2, unofficial].

**Parameters:**
*   **Orientation Flip Threshold:** The dot product threshold is 0. If $\boldsymbol{v} \cdot \hat{\boldsymbol{n}}_i < 0$, flip [S1].
*   **Normalization Epsilon:** Not given in the sources. [G] A value of $10^{-6}$ is typical.

**Pitfalls:**
*   **Incorrect Normal Orientation:** If normals are not consistently oriented relative to the camera, the alpha-composited result will be noisy or incorrect. This is handled by flipping normals based on the dot product with the viewing direction [S1].
*   **Degenerate Gaussians:** If a Gaussian becomes extremely flat or its scale collapses, the "shortest axis" direction may become unstable. The method assumes Gaussians gradually become flatter and approach a planar shape, making the shortest axis a good normal estimate [S2, unofficial].
*   **Depth Sorting:** The accuracy of the normal estimation depends on the correct front-to-back sorting of Gaussians, as in standard 3D Gaussian Splatting [Parent].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gaussian_normals(gaussians, camera_transform, image_width, image_height):
    """
    Computes per-pixel normal maps from 3D Gaussians.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - pos: (3,) world position
            - cov: (3,3) covariance matrix or (scale, rotation)
            - alpha: scalar opacity
            - color: (3,) color
        camera_transform: (4,4) or (3,3) rotation matrix to transform world to camera space.
        image_width, image_height: int
        
    Returns:
        normals: (H, W, 3) array of estimated normals in camera space.
    """
    import math
    
    # Initialize output
    normals = [[(0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]
    
    # Precompute camera rotation matrix for normal transformation
    R_cam = [row[:3] for row in camera_transform[:3]]
    
    # For each pixel, we need the list of contributing Gaussians sorted by depth
    # This is typically handled by the parent rasterizer's depth sorting step [Parent].
    # A step the sources do not give is the exact projection and sorting logic.
    
    for y in range(image_height):
        for x in range(image_width):
            contributing_gaussians = get_contributing_gaussians(x, y, gaussians, camera_transform)
            
            if not contributing_gaussians:
                continue
                
            accumulated_normal = [0.0, 0.0, 0.0]
            transmittance = 1.0
            
            for g in contributing_gaussians:
                # Step 1: Extract normal from Gaussian geometry
                n_world = get_shortest_axis_normal(g)
                
                # Step 2: Orientation Correction
                # Get camera viewing direction for this pixel (approximate)
                # In practice, this might be done per-Gaussian or per-pixel.
                # S1 says "dot product between the current camera viewing direction and the Gaussian normal"
                # We assume viewing direction is roughly the negative Z axis in camera space,
                # or we transform the normal to camera space first.
                
                # Step 3: Camera Space Transformation
                n_cam = mat_vec_mul(R_cam, n_world)
                
                # Check orientation relative to camera view (assuming camera looks down -Z)
                # Viewing direction in camera space is (0, 0, -1)
                # If dot(n_cam, (0,0,-1)) < 0, it points away from camera?
                # S1 says: "flip ... if dot product between current camera viewing direction and Gaussian normal is negative"
                # Let's assume viewing direction v is (0,0,1) in camera space (looking into scene)
                # or (0,0,-1) depending on convention. Standard OpenGL: camera looks down -Z.
                # If normal points towards camera, dot(n_cam, (0,0,1)) > 0.
                # Let's follow S1 literally: if dot(v, n) < 0, flip.
                v_cam = (0.0, 0.0, 1.0) # Assuming camera looks along +Z in some conventions, or adjust.
                # Actually, standard is camera at origin looking down -Z.
                # Let's use the dot product with the camera's forward vector.
                # If the source doesn't specify the exact vector, we assume a consistent convention.
                # We'll assume v_cam is the direction the camera is looking.
                # If n_cam points opposite to v_cam, dot < 0.
                
                # To be safe and generic, we check if the normal is facing the camera.
                # If the dot product with the view vector is negative, it's facing away.
                # We flip it to face the camera.
                if dot_product(v_cam, n_cam) < 0:
                    n_cam = [-x for x in n_cam]
                    
                # Step 4: Alpha Compositing
                alpha = g.alpha
                weight = alpha * transmittance
                accumulated_normal = [accumulated_normal[k] + n_cam[k] * weight for k in range(3)]
                transmittance *= (1.0 - alpha)
                
            # Step 5: Normalization
            norm_val = math.sqrt(sum(x*x for x in accumulated_normal))
            epsilon = 1e-6
            if norm_val > epsilon:
                normals[y][x] = tuple(x / (norm_val + epsilon) for x in accumulated_normal)
            else:
                normals[y][x] = (0.0, 0.0, 1.0) # Default normal
                
    return normals

def get_shortest_axis_normal(gaussian):
    """
    Extracts the normalized vector of the shortest axis of the Gaussian's covariance.
    """
    # If covariance is given as scale (s) and rotation (R):
    # Cov = R * diag(s^2) * R^T
    # The axes are the columns of R.
    # The shortest axis corresponds to the minimum scale.
    s = gaussian.scale
    R = gaussian.rotation # (3,3) matrix
    
    min_idx = min(range(3), key=lambda i: s[i])
    # The column of R corresponding to the minimum scale
    n_world = [R[i][min_idx] for i in range(3)]
    norm = math.sqrt(sum(x*x for x in n_world))
    return [x / norm for x in n_world]

def mat_vec_mul(M, v):
    return [sum(M[i][j] * v[j] for j in range(3)) for i in range(3)]

def dot_product(u, v):
    return sum(u[i] * v[i] for i in range(3))
```

## Subtopics

### Normal Consistency Loss
Uses the estimated normals $\hat{\boldsymbol{N}}$ to supervise against pseudo-ground truth normals $\tilde{\boldsymbol{N}}$ derived from depth gradients, enforcing local planarity [S2, unofficial].

### Monocular Depth Regularization
Uses off-the-shelf monocular depth estimation networks (e.g., ZoeDepth) to provide dense per-pixel depth priors for scenes without ground truth depth data [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of contributing Gaussians [G]. This is added to the standard rasterization cost [G].
*   **Memory Complexity:** $O(H \times W \times 3)$ for the output normal map [G]. No additional learnable parameters are needed [S1].
*   **Strengths:** Provides geometric supervision without extra parameters; gradients flow directly to Gaussian rotation and scale, improving geometry alignment [S1].
*   **Limitations:** Relies on the assumption that Gaussians become planar; noise in depth maps can affect pseudo-normal quality [S2, unofficial].

## Background
Originated in the context of 3D Gaussian Splatting optimization, where geometric consistency is crucial [S1][S2, unofficial]. DN-Splatter [S1] introduced the method to regularize depth and normal estimates. Gaussian Splatting in Mirrors [S2, unofficial] adapted it for reflection-aware rendering, leveraging the planar nature of Gaussians to estimate normals for mirror plane supervision.

## Sources
[S1] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
[S2] Gaussian Splatting in Mirrors: Reflection-Aware Rendering via Virtual Camera Optimization, https://doi.org/10.48550/arxiv.2410.01614, fetched 2026-10-26 (unofficial)
