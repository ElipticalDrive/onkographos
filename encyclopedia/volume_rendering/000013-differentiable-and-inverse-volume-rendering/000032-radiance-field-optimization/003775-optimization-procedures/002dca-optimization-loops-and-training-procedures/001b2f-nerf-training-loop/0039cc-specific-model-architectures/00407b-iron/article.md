# IRON

Also called: Inverse Rendering by Optimizing Neural SDFs and Materials

## Overview
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D triangle meshes and spatially varying material textures from photometric images captured with a collocated flashlight and moving camera [S1, unofficial]. It distinguishes itself from sibling methods by employing a hybrid optimization scheme for neural Signed Distance Fields (SDFs): it first uses volumetric radiance field rendering to recover correct object topology, then refines geometry and disentangles materials from lighting using edge-aware physics-based surface rendering [S1, unofficial]. This two-stage approach avoids the local minima and topological errors common in direct surface optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of photometric images. Assumptions: opaque objects, no ambient light, flashlight co-located with the camera, shadows and global illumination (interreflections) are ignored [S1, unofficial].
*   **Output:** A triangle mesh and material textures (diffuse albedo, specular albedo, roughness) [S1, unofficial].

**Data Structures:**
The method optimizes four Multi-Layer Perceptrons (MLPs) with positional encoding [S1, unofficial]:
1.  **Neural SDF $S_\Theta_s$**: Input $\mathbf{x} \in \mathbb{R}^3$. Output: SDF value $S \in \mathbb{R}$ and a 256D local geometric feature descriptor $\mathbf{f} \in \mathbb{R}^{256}$ [S1, unofficial].
2.  **Neural Diffuse Albedo $\beta_\Theta_\beta$**: Input $(\mathbf{x}, \mathbf{n}, \mathbf{n}, \mathbf{f})$. Output: Diffuse albedo $\beta \in \mathbb{R}^3$. In Stage 1, the second normal $\mathbf{n}$ is replaced by view direction $-\mathbf{d}$ [S1, unofficial].
3.  **Neural Specular Albedo $\kappa_\Theta_\kappa$**: Input $(\mathbf{x}, \mathbf{n}, \mathbf{f})$. Output: Specular albedo $\kappa \in \mathbb{R}^3$ [S1, unofficial].
4.  **Neural Roughness $\alpha_\Theta_\alpha$**: Input $(\mathbf{x}, \mathbf{n}, \mathbf{f})$. Output: Roughness $\alpha \in \mathbb{R}$ (small values = shiny, large values = matte) [S1, unofficial].
5.  **Light Intensity**: Scalar $L \in \mathbb{R}$ [S1, unofficial].

**Optimization Steps:**

**Stage 1: Volumetric Radiance Field Rendering**
Goal: Recover correct object topology (number and location of holes) [S1, unofficial].
1.  Treat $\beta_\Theta_\beta$ as a view-dependent neural radiance field by substituting the view direction $-\mathbf{d}$ for the second normal input: $\beta_\Theta_\beta(\mathbf{x}, \mathbf{n}, -\mathbf{d}, \mathbf{f})$ [S1, unofficial].
2.  Perform standard volumetric rendering of the neural SDF $S_\Theta_s$ and colors $\beta_\Theta_\beta$ [S1, unofficial].
3.  Optimize $\Theta_s$ and $\Theta_\beta$ [S1, unofficial].
*Note:* Skipping this stage causes optimization to diverge or get stuck in local minima with incorrect topology unless segmentation masks are provided [S1, unofficial].

**Stage 2: Edge-aware Physics-based Surface Rendering**
Goal: Refine geometric details and factorize materials from lighting [S1, unofficial].
1.  Jointly optimize $\Theta_s, \Theta_\beta, \Theta_\kappa, \Theta_\alpha$ and $L$ [S1, unofficial].
2.  Use the edge sampling algorithm (Algorithm 1) to generate unbiased gradient estimates for the neural SDF around edges (depth discontinuities) [S1, unofficial].
3.  Render using physics-based surface shading with the estimated materials and light [S1, unofficial].

**Edge Sampling Algorithm (Algorithm 1):**
This algorithm locates edge points on the neural SDF to compute accurate gradients [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def locate_edge_points(x_hat, o, K, epsilon, delta):
    """
    Locate edge points for neural SDF gradient estimation.
    
    Args:
        x_hat: Ray-surface intersection point.
        o: Light/Camera position (vector).
        K: Maximum number of steps.
        epsilon: Step size.
        delta: Threshold for edge detection.
        
    Returns:
        Edge point xt or "NOT FOUND".
    """
    xt = x_hat
    for i in range(1, K + 1):
        # Calculate vector from light to current point
        vec = xt - o
        # Normalize vector
        vec_norm = vec / np.linalg.norm(vec)
        # Get surface normal at xt (computed via SDF gradient)
        nt = compute_sdf_normal(xt)
        
        # Check if the angle between the light direction and normal exceeds threshold
        # Dot product of normalized light direction and normal
        if np.dot(vec_norm, nt) < delta:
            return xt
        else:
            # Project normal onto the plane perpendicular to the light direction
            # and step in that direction
            # o_xt is the unit vector from o to xt (same as vec_norm)
            o_xt = vec_norm
            # Projection of nt onto the plane perpendicular to o_xt
            projection = nt - np.dot(o_xt, nt) * o_xt
            xt = xt + epsilon * projection
            
    return "NOT FOUND"
```

**Light Model:**
The flashlight is modeled as a point light source [S1, unofficial]:
$$
L_i(\mathbf{o}; \mathbf{x}) = \frac{L}{\|\mathbf{x} - \mathbf{o}\|^2}
$$
Where $L$ is light intensity, $\mathbf{x}$ is surface location, and $\mathbf{o}$ is light/camera position [S1, unofficial].

**Parameters:**
*   $K$: Max steps for edge search (Not given in the sources).
*   $\epsilon$: Step size for edge search (Not given in the sources).
*   $\delta$: Threshold for edge detection (Not given in the sources).

**Pitfalls:**
*   **Topology Errors:** Optimizing surface rendering from scratch without Stage 1 initialization frequently diverges or gets stuck in local minima with incorrect topology [S1, unofficial].
*   **Ambient Light:** The method assumes no ambient light. If present, results may be inaccurate [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Produces high-quality 3D content in the format of triangle meshes and material textures; handles spatially varying materials; robust topology recovery via hybrid optimization [S1, unofficial].
*   **Limitations:** Requires specific capture setup (collocated flashlight and camera); ignores shadows and global illumination; limited to opaque objects [S1, unofficial].
*   **When to use:** When high-fidelity mesh and material reconstruction is needed from photometric images with controlled lighting [S1, unofficial].
*   **When not to use:** For scenes with transparent objects, significant ambient light, or complex global illumination effects [S1, unofficial].

## Background
IRON was proposed by Kai Zhang, Fujun Luan, Zhengqi Li, and Noah Snavely from Cornell University, Adobe Research, and Google Research [S1, unofficial]. It was published in CVPR 2022 [S1, unofficial]. The method builds on prior work in neural inverse rendering and differentiable rendering, specifically addressing the challenges of optimizing neural SDFs for inverse rendering by combining volumetric and surface-based optimization strategies [S1, unofficial].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26 (unofficial)
