# IRON

Also called: Inverse Rendering by Optimizing Neural SDFs and Materials

## Overview
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D triangle meshes and spatially varying material textures from photometric images captured with a collocated flashlight and moving camera [S1]. It distinguishes itself from standard NeRF-based methods by outputting explicit mesh-compatible representations (SDFs and material maps) rather than implicit radiance fields, and from direct mesh optimization methods by employing a hybrid two-stage training scheme that first recovers correct topology via volumetric rendering before refining geometry with edge-aware surface rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of photometric images. These are captured assuming opaque objects, no ambient light, and a flashlight co-located with the camera. Shadows and global illumination effects (interreflections) are ignored [S1].
*   **Output:** A triangle mesh and material textures (diffuse albedo, specular albedo, roughness) derived from the optimized neural networks [S1].

**Data Structures:**
The method optimizes four Multi-Layer Perceptrons (MLPs) with positional encoding [S1]:

1.  **Neural SDF ($S_{\Theta_s}$):**
    *   **Input:** 3D location $\mathbf{x}$.
    *   **Output:** Tuple $(S, \mathbf{f})$.
    *   **Layout:** $S$ is the signed distance value. $\mathbf{f}$ is a 256-dimensional local geometric feature descriptor [S1].
2.  **Neural Diffuse Albedo ($\beta_{\Theta_\beta}$):**
    *   **Input:** Location $\mathbf{x}$, surface normal $\mathbf{n}$, and feature descriptor $\mathbf{f}$.
    *   **Output:** Scalar diffuse albedo $\beta$.
    *   **Layout:** In Stage 1, the second normal input is replaced by the view direction $-\mathbf{d}$ to treat $\beta$ as a view-dependent radiance field [S1].
3.  **Neural Specular Albedo ($\kappa_{\Theta_\kappa}$):**
    *   **Input:** Location $\mathbf{x}$, surface normal $\mathbf{n}$, and feature descriptor $\mathbf{f}$.
    *   **Output:** Scalar specular albedo $\kappa$ [S1].
4.  **Neural Roughness ($\alpha_{\Theta_\alpha}$):**
    *   **Input:** Location $\mathbf{x}$, surface normal $\mathbf{n}$, and feature descriptor $\mathbf{f}$.
    *   **Output:** Scalar roughness $\alpha$. Small values indicate shiny surfaces; large values indicate less shiny surfaces [S1].

**Light Model:**
The flashlight is modeled as a point light source. The incident light $L_i$ at surface location $\mathbf{x}$ from light/camera position $\mathbf{o}$ with intensity $L$ is:
$$ L_i(\mathbf{o}; \mathbf{x}) = \frac{L}{\|\mathbf{x} - \mathbf{o}\|_2^2} $$
where $L$ is the scalar light intensity parameter [S1].

**Optimization Steps:**
The method uses a hybrid two-stage optimization scheme [S1]:

1.  **Stage 1: Volumetric Radiance Field Rendering**
    *   Optimize $\Theta_s$ and $\Theta_\beta$.
    *   Treat $\beta_{\Theta_\beta}(\mathbf{x}, \mathbf{n}, -\mathbf{d}, \mathbf{f})$ as a view-dependent neural radiance field by substituting the view direction $-\mathbf{d}$ for the second normal argument.
    *   Perform volumetric rendering of $S_{\Theta_s}$ and $\beta_{\Theta_\beta}$ to recover correct object topology (number and location of holes) [S1].
    *   *Note:* Skipping this stage and optimizing surface rendering from scratch causes divergence or local minima with incorrect topology unless segmentation masks are provided [S1].

2.  **Stage 2: Edge-aware Physics-based Surface Rendering**
    *   Jointly optimize $\Theta_s$, $\Theta_\beta$, $\Theta_\kappa$, $\Theta_\alpha$, and light intensity $L$.
    *   Use the output of Stage 1 as initialization.
    *   Refine geometric details and disentangle materials from lighting using edge-aware surface rendering [S1].
    *   This stage employs the **Edge Sampling Algorithm** (Algorithm 1) to generate unbiased gradient estimates for neural SDFs at depth discontinuities (edges) [S1].

**Edge Sampling Algorithm (Algorithm 1):**
This algorithm locates edge points on the neural SDF to improve gradient accuracy at boundaries [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def locate_edge_points(x_hat, o, K, epsilon, delta):
    """
    Locate edge points for neural SDF optimization.
    
    Args:
        x_hat: Ray-surface intersection point.
        o: Light/Camera position.
        K: Maximum number of steps.
        epsilon: Step size.
        delta: Threshold for edge detection.
        
    Returns:
        An edge point or None (NOT FOUND).
    """
    x_t = x_hat
    for i in range(1, K + 1):
        # Calculate vector from light to current point
        vec = x_t - o
        dist = np.linalg.norm(vec)
        if dist == 0:
            return None # Avoid division by zero
        
        # Normalize vector
        unit_vec = vec / dist
        
        # Get surface normal at x_t from SDF
        # Note: In implementation, S_grad(x_t) provides the normal n_t
        n_t = get_sdf_normal(x_t) 
        
        # Check if the angle between view direction (unit_vec) and normal (n_t)
        # indicates an edge. Specifically, if the dot product is less than delta.
        # The formula in source: ( (x_t - o) / ||x_t - o||_2 )^T n_t < delta
        if np.dot(unit_vec, n_t) < delta:
            return x_t
        else:
            # Project normal onto the plane perpendicular to the view direction
            # and step in that direction
            # Formula: x_t = x_t + epsilon * (n_t - o_xt * (o_xt)^T n_t)
            # where o_xt is the unit vector from light to x_t
            projection = n_t - unit_vec * np.dot(unit_vec, n_t)
            x_t = x_t + epsilon * projection
            
    return None # NOT FOUND
```

**Parameters:**
*   **K:** Maximum steps for edge search. Not given in the sources.
*   **epsilon:** Step size for edge search. Not given in the sources.
*   **delta:** Threshold for edge detection. Not given in the sources.
*   **L:** Light intensity scalar. Optimized during Stage 2 [S1].

**Pitfalls:**
*   **Topology Errors:** If Stage 1 is skipped, optimization frequently diverges or gets stuck in local minima with incorrect topology (e.g., wrong number of holes) unless object segmentation masks are provided [S1].
*   **Assumptions:** The method assumes opaque objects, no ambient light, and ignores shadows and global illumination. Applying it to scenes with significant ambient light or transparency will fail [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** IRON achieves significantly better inverse rendering quality compared to prior works for photometric images [S1].
*   **Strengths:** Produces deployable triangle meshes and material textures; handles topology recovery better than direct surface optimization; improves reconstruction quality around edges via the novel sampling algorithm [S1].
*   **Limitations:** Restricted to photometric capture setups (collocated flashlight); ignores shadows and global illumination; assumes opaque objects [S1].
*   **When to use:** When high-quality mesh and material output is required from photometric image sequences, and the scene consists of opaque objects under controlled lighting.
*   **When not to use:** For scenes with transparent/translucent objects, significant ambient light, or complex global illumination effects [S1].

## Background
IRON was proposed by Kai Zhang, Fujun Luan, Zhengqi Li, and Noah Snavely from Cornell University, Adobe Research, and Google Research [S1]. It was published in CVPR 2022 [S1]. The work addresses the third capture scenario of inverse rendering: static scene, static environmental lighting (specifically a collocated flashlight), and moving camera [S1]. It builds on prior works using neural SDFs [30, 34, 39] and neural radiance fields [23, 34, 38], extending them with a hybrid optimization scheme and edge-aware sampling for neural representations [S1].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
