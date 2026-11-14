# IRON

Also called: Inverse Rendering by Optimizing Neural SDFs and Materials

## Overview
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D geometry and spatially varying materials from photometric images (captured with a collocated flashlight and moving camera) [S1]. It outputs triangle meshes and material textures suitable for standard graphics pipelines [S1]. Unlike methods that optimize surface representations directly, IRON employs a hybrid optimization scheme: it first uses volumetric radiance field rendering to recover correct object topology, then refines geometry and disentangles materials from lighting using edge-aware physics-based surface rendering [S1]. This two-stage approach prevents divergence and local minima associated with direct surface optimization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of photometric images captured with a flashlight co-located with a moving camera [S1].
*   **Output:** A triangle mesh and material textures (diffuse albedo, specular albedo, roughness) [S1].

**Data Structures:**
The method uses four Multi-Layer Perceptrons (MLPs) with positional encoding [S1].
1.  **Neural SDF ($S_\Theta$):**
    *   **Input:** 3D location $\mathbf{x}$.
    *   **Output:** A tuple $(S, \mathbf{f})$, where $S$ is the signed distance field value and $\mathbf{f}$ is a 256-dimensional local geometric feature descriptor [S1].
2.  **Neural Diffuse Albedo ($\beta_\Theta$):**
    *   **Input:** 3D location $\mathbf{x}$, surface normal $\mathbf{n}$, view direction $\mathbf{v}$ (or normal $\mathbf{n}$ in Stage 2), and feature descriptor $\mathbf{f}$ [S1].
    *   **Output:** Diffuse albedo $\beta$ [S1].
3.  **Neural Specular Albedo ($\kappa_\Theta$):**
    *   **Input:** 3D location $\mathbf{x}$, surface normal $\mathbf{n}$, and feature descriptor $\mathbf{f}$ [S1].
    *   **Output:** Specular albedo $\kappa$ [S1].
4.  **Neural Roughness ($\alpha_\Theta$):**
    *   **Input:** 3D location $\mathbf{x}$, surface normal $\mathbf{n}$, and feature descriptor $\mathbf{f}$ [S1].
    *   **Output:** Specular roughness $\alpha$ [S1]. Small values indicate shiny surfaces; large values indicate less shiny surfaces [S1].

**Optimization Steps:**
The optimization is performed in two stages [S1].
1.  **Stage 1: Volumetric Radiance Field Rendering** [S1]. This stage initializes the topology [S1]. Treat the neural diffuse albedo network $\beta_\Theta(\mathbf{x}, \mathbf{n}, \mathbf{n}, \mathbf{f})$ as a view-dependent neural radiance field by substituting the view direction $-\mathbf{d}$ for the second $\mathbf{n}$ input [S1]. Perform volumetric rendering of the neural SDF $S_\Theta$ and the modified colors $\beta_\Theta(\mathbf{x}, \mathbf{n}, -\mathbf{d}, \mathbf{f})$ [S1]. Optimize the weights $\Theta_s$ and $\Theta_\beta$ [S1].
2.  **Stage 2: Edge-aware Physics-based Surface Rendering** [S1]. This stage refines geometry and disentangles materials from lighting [S1]. Jointly optimize the weights $\Theta_s, \Theta_\beta, \Theta_\kappa, \Theta_\alpha$ and the light intensity $L$ [S1]. Use the neural SDF $S_\Theta$ to define the surface [S1]. Use the neural material networks ($\beta_\Theta, \kappa_\Theta, \alpha_\Theta$) to define surface properties [S1]. Apply the novel edge sampling algorithm (Algorithm 1) to generate unbiased gradient estimates for the neural SDF around edges (depth discontinuities) [S1]. Render using physics-based surface rendering with the point light model [S1].

**Formulas:**
*Light Model:* The flashlight is modeled as a point light source [S1]. The incoming light radiance $L_i$ at surface location $\mathbf{x}$ from light position $\mathbf{o}$ with intensity $L$ is:
$$
L_i(\mathbf{o}; \mathbf{x}) = \frac{L}{\|\mathbf{x} - \mathbf{o}\|^2}
$$
where $L$ is light intensity, $\mathbf{x}$ is surface location, and $\mathbf{o}$ is the light/camera position [S1].

**Parameters:**
*   Light intensity $L$ (scalar) [S1].
*   Flashlight position $\mathbf{o}$ (co-located with camera) [S1].
*   Edge sampling hyperparameters: maximum steps $K$, step size $\epsilon$, threshold $\delta$ [S1].

**Edge Cases and Pitfalls:**
*   If Stage 1 is skipped and surface rendering is optimized from scratch, optimization frequently diverges unless object segmentation masks are provided, and often gets stuck in local minima with incorrect topology [S1].
*   The method assumes opaque objects, no ambient light, and ignores shadows and global illumination effects for efficiency [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def locate_edge_points(x_hat, o, S_Theta, K, epsilon, delta):
    """
    Locate edge points on a neural SDF for unbiased gradient estimation.
    Input: ray-surface intersection x_hat, light/camera position o,
           neural SDF network S_Theta, max steps K, step size epsilon, threshold delta.
    Output: edge point x_t or None (NOT FOUND).
    """
    x_t = x_hat
    for i in range(1, K + 1):
        # Compute surface normal at current point via gradient of SDF
        n_t = gradient(S_Theta, x_t)
        # Compute normalized vector from point to light/camera
        v = x_t - o
        v_hat = v / norm(v, 2)
        # Check if point is near an edge (dot product of view direction and normal)
        if dot(v_hat, n_t) < delta:
            return x_t
        # Update position along surface tangent to find edge
        x_t = x_t + epsilon * (n_t - v_hat * dot(v_hat, n_t))
    return None
```

## Operating characteristics
*   **Strengths:** Produces high-quality inverse rendering results with correct topology and disentangled materials/lighting [S1]. The hybrid optimization avoids the divergence issues common in direct surface optimization [S1]. The output is compatible with standard mesh-based graphics pipelines [S1].
*   **Limitations:** Assumes opaque objects, no ambient light, and ignores shadows and global illumination [S1]. Performance depends on the quality of photometric image capture [S1].
*   **When to use:** When reconstructing shape and spatially varying materials from photometric images (flashlight + moving camera) is required, and mesh/texture output is desired [S1].
*   **When not to use:** For scenes with transparent/translucent objects, significant ambient light, or complex global illumination effects [S1].

## Background
IRON was proposed by Kai Zhang, Fujun Luan, Zhengqi Li, and Noah Snavely from Cornell University, Adobe Research, and Google Research [S1]. It was published in CVPR 2022 [S1]. The method builds upon prior work in neural implicit representations (SDFs) and differentiable rendering, addressing the challenge of optimizing neural SDFs for inverse rendering by combining the robustness of volumetric rendering for topology with the precision of surface rendering for details [S1]. It draws inspiration from mesh-based differentiable rendering techniques, specifically regarding edge sampling [S1].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
