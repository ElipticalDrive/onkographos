# Mesh Extraction via Poisson Reconstruction

Also called: Poisson Mesh Extraction, Poisson Surface Reconstruction

## Overview
This method extracts a textured 3D mesh from an optimized 3D Gaussian Splatting scene by converting rendered 2D depth and normal maps into a 3D oriented point cloud, which is then reconstructed into a manifold mesh using Poisson surface reconstruction. It differs from direct Gaussian-to-mesh approaches (like [Gaussian Opacity Fields (GOF) Mesh Extraction](3692-gaussian-opacity-fields-gof-mesh-extraction.md)) by relying on explicit geometric priors derived from rendered views rather than implicit density fields or signed distance functions. It is chosen when high-fidelity surface topology is required from a radiance field that lacks explicit geometric regularization.

## How it works
**Inputs:**
*   Optimized set of $M$ 3D Gaussians $\mathcal{G} = \{G_i\}_{i=1}^M$, where each $G_i$ has position $\mathbf{p}_i$, rotation $\mathbf{R}_i$, scale $\mathbf{S}_i$, and color/feature attributes.
*   Camera poses and intrinsics for the training views.
*   Pre-trained normal estimation model (e.g., Omnidata) if explicit normal regularization was not used during training [S1].

**Outputs:**
*   A textured triangular mesh $\mathcal{M}$.

**Data Structures:**
*   **Rendered Depth Map:** A 2D array $D \in \mathbb{R}^{H \times W}$ storing depth values per pixel.
*   **Rendered Normal Map:** A 2D array $N \in \mathbb{R}^{H \times W \times 3}$ storing surface normals in camera space.
*   **Oriented Point Cloud:** A list of points $P = \{(\mathbf{x}_k, \mathbf{n}_k, \mathbf{c}_k)\}$, where $\mathbf{x}_k$ is the 3D position, $\mathbf{n}_k$ is the surface normal, and $\mathbf{c}_k$ is the color.

**Steps:**

1.  **Render Depth Maps:**
    For each training view, render a depth map using the median depth of the splats projected onto each pixel. This avoids bias from large Gaussians [S2, unofficial].
    *   Project each Gaussian $G_i$ to 2D using the projected covariance $\mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).
    *   For each pixel, collect depths of overlapping splats and compute the median.

2.  **Derive Normal Maps:**
    Convert the rendered depth maps into 3D space to compute corresponding normal maps.
    *   Back-project pixel coordinates using the depth values to obtain 3D points in camera space.
    *   Compute normals via finite differences or by using a pre-trained normal estimator (e.g., Omnidata) on the rendered RGB images to provide monocular normal cues [S1].

3.  **Generate Oriented Point Cloud:**
    Create a dense set of 3D points with orientation and color.
    *   For each pixel in the training views, back-project the depth value to 3D space to get position $\mathbf{x}$.
    *   Assign the corresponding normal $\mathbf{n}$ from the normal map.
    *   Assign the color $\mathbf{c}$ from the rendered RGB image.
    *   Filter or sample points to ensure uniform density. The sources specify extracting a total of 2 million points for the reconstruction [S1].

4.  **Poisson Surface Reconstruction:**
    Input the oriented point cloud into the Poisson surface reconstruction algorithm.
    *   The algorithm solves for a scalar function whose gradient field best fits the oriented points.
    *   Extract the mesh as the zero-level set of this function.
    *   **Parameters:**
        *   Depth level: 9 [S1].
        *   Number of points: 2 million [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_mesh(gaussians, cameras, num_points_target=2_000_000, depth_level=9):
    """
    Extracts a mesh from 3D Gaussians using Poisson Reconstruction.
    """
    oriented_points = []
    
    # Step 1 & 2: Render and derive depth/normal maps
    for camera in cameras:
        # Render depth map using median depth of splats
        depth_map = render_median_depth(gaussians, camera)
        
        # Render RGB image for color and normal estimation
        rgb_image = render_rgb(gaussians, camera)
        
        # Derive normal map from depth or pre-trained model
        # Option A: From depth (finite differences)
        normal_map = compute_normals_from_depth(depth_map, camera.intrinsics)
        
        # Option B: From pre-trained model (e.g., Omnidata) [S1]
        # normal_map = pretrained_normal_estimator(rgb_image)
        
        # Step 3: Back-project to 3D point cloud
        for y in range(height):
            for x in range(width):
                depth_val = depth_map[y, x]
                if depth_val is valid:
                    # Back-project pixel (x,y) with depth to 3D
                    point_3d = back_project(camera, x, y, depth_val)
                    normal_3d = normal_map[y, x]
                    color = rgb_image[y, x]
                    oriented_points.append((point_3d, normal_3d, color))
    
    # Sample points to target count if necessary
    if len(oriented_points) > num_points_target:
        oriented_points = sample_points(oriented_points, num_points_target)
    
    # Step 4: Poisson Reconstruction
    mesh = poisson_reconstruction(oriented_points, depth_level=depth_level)
    
    return mesh
```

**Edge Cases and Pitfalls:**
*   **Depth Ambiguity:** Using mean depth can bias the surface towards the center of large Gaussians. Median depth is preferred [S2, unofficial].
*   **Normal Consistency:** If normals are estimated from RGB (monocular), they may be inconsistent across views. Explicit depth-derived normals or regularization during training (e.g., DN-Splatter) improve consistency [S1].
*   **Point Density:** Insufficient points lead to noisy meshes; excessive points increase reconstruction time. 2 million points is a typical target [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by rendering $V$ views and the Poisson reconstruction step, which is $O(N \log N)$ for $N$ points. Rendering is $O(M \cdot V \cdot \text{pixels})$.
*   **Memory Complexity:** $O(M + V \cdot H \cdot W)$ for storing Gaussians and intermediate maps. The point cloud is $O(N)$.
*   **Strengths:** Produces watertight, manifold meshes with texture. Works well with regularized Gaussians (e.g., DN-Splatter) where positions are aligned to the surface [S1].
*   **Limitations:** Requires multiple views for dense point clouds. Sensitive to noise in depth/normal maps. Does not preserve internal structures or transparency.
*   **When to use:** When a textured, watertight mesh is needed from a 3DGS scene, especially if the Gaussians have been regularized to lie on the surface.
*   **When not to use:** For scenes with significant transparency, internal structures, or when real-time extraction is required.

## Background
Poisson surface reconstruction was originally introduced by Kazhdan, Bolitho, and Hoppe [S1, ref 18]. In the context of 3D Gaussian Splatting, it is used as a post-processing step to extract explicit geometry from the implicit representation. Methods like DN-Splatter [S1] and AtomGS [S2, unofficial] utilize this approach to evaluate mesh quality. DN-Splatter specifically uses depth and normal priors during training to ensure the Gaussians are well-distributed and aligned with the surface, facilitating better Poisson reconstruction [S1].

## Sources
[S1] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
