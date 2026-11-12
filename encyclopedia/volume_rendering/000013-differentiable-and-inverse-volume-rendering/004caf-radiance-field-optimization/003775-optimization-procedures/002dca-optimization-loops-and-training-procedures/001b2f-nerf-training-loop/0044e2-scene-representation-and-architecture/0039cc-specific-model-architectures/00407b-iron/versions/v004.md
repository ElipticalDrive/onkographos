# IRON

Also called: Inverse Rendering by Optimizing Neural SDFs and Materials

## Overview
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D geometry and spatially varying materials from photometric images captured with a collocated flashlight and moving camera [S1]. It outputs triangle meshes and material textures by optimizing neural Signed Distance Fields (SDFs) and material parameters [S1]. It distinguishes itself from sibling architectures like [AdaNeRF](6195-adanerf.md) or [SinNeRF](6199-sinnerf.md) by employing a hybrid optimization scheme: it first uses volumetric radiance field rendering to recover correct object topology, then refines geometry and disentangles materials from lighting using edge-aware physics-based surface rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of photometric images. These are captured with a static scene, a moving camera, and a flashlight co-located with the camera [S1].
*   **Output:** A triangle mesh and material textures (diffuse albedo, specular albedo, roughness) [S1].

**Assumptions:**
*   Objects are opaque [S1].
*   Input images are captured without ambient light [S1].
*   Shadows and global illumination effects (e.g., interreflections) are ignored for efficiency [S1].

**Data Structures:**
The method uses four Multi-Layer Perceptrons (MLPs) with positional encoding $\gamma(\mathbf{x})$ [S1, Parent].

1.  **Neural SDF $S_{\Theta_s}$**:
    *   **Input:** 3D location $\mathbf{x} \in \mathbb{R}^3$.
    *   **Output:** Tuple $(S, \mathbf{f})$.
    *   $S \in \mathbb{R}$: Signed distance value.
    *   $\mathbf{f} \in \mathbb{R}^{256}$: Local geometric feature descriptor [S1].

2.  **Neural Diffuse Albedo $\beta_{\Theta_\beta}$**:
    *   **Input:** Tuple $(\mathbf{x}, \mathbf{n}, \mathbf{n}', \mathbf{f})$.
        *   $\mathbf{x} \in \mathbb{R}^3$: 3D location.
        *   $\mathbf{n} \in \mathbb{R}^3$: Surface normal.
        *   $\mathbf{n}' \in \mathbb{R}^3$: Second normal vector (used as view direction $-\mathbf{d}$ in Stage 1, or surface normal in Stage 2).
        *   $\mathbf{f} \in \mathbb{R}^{256}$: Feature descriptor from $S_{\Theta_s}$.
    *   **Output:** $\beta \in \mathbb{R}^3$: Diffuse albedo color [S1].

3.  **Neural Specular Albedo $\kappa_{\Theta_\kappa}$**:
    *   **Input:** Tuple $(\mathbf{x}, \mathbf{n}, \mathbf{f})$.
    *   **Output:** $\kappa \in \mathbb{R}^3$: Spatially-varying specular albedo [S1].

4.  **Neural Roughness $\alpha_{\Theta_\alpha}$**:
    *   **Input:** Tuple $(\mathbf{x}, \mathbf{n}, \mathbf{f})$.
    *   **Output:** $\alpha \in \mathbb{R}$: Spatially-varying specular roughness. Small values indicate shiny surfaces; large values indicate less shiny surfaces [S1].

**Light Model:**
The flashlight is modeled as a point light source [S1]. The incident light intensity $L_i$ at surface location $\mathbf{x}$ from light/camera position $\mathbf{o}$ is:
$$
L_i(\mathbf{o}; \mathbf{x}) = \frac{L}{\|\mathbf{x} - \mathbf{o}\|^2}
$$
where $L$ is the scalar light intensity parameter [S1].

**Optimization Procedure:**
The method uses a two-stage hybrid optimization scheme to optimize MLP weights $\Theta_s, \Theta_\beta, \Theta_\kappa, \Theta_\alpha$ and light intensity $L$ [S1].

**Stage 1: Volumetric Radiance Field Rendering**
*   **Goal:** Recover correct object topology (number and location of holes) [S1].
*   **Method:** Treat $\beta_{\Theta_\beta}$ as a view-dependent neural radiance field. Substitute the view direction $-\mathbf{d}$ for the second normal $\mathbf{n}'$ in the input to $\beta_{\Theta_\beta}$.
*   **Rendering:** Perform volumetric rendering of $S_{\Theta_s}$ and colors $\beta_{\Theta_\beta}(\mathbf{x}, \mathbf{n}, -\mathbf{d}, \mathbf{f})$ [S1].
*   **Note:** Skipping this stage and optimizing surface rendering from scratch causes divergence or convergence to local minima with incorrect topology, unless segmentation masks are provided [S1].

**Stage 2: Edge-aware Physics-based Surface Rendering**
*   **Goal:** Refine geometric details and factorize materials from lighting [S1].
*   **Method:** Jointly optimize $S_{\Theta_s}$, $\beta_{\Theta_\beta}$, $\kappa_{\Theta_\kappa}$, $\alpha_{\Theta_\alpha}$, and $L$ using physics-based surface rendering [S1].
*   **Edge Sampling:** To handle depth discontinuities, a novel edge sampling algorithm is used to generate unbiased gradient estimates [S1].

**Edge Sampling Algorithm:**
This algorithm locates edge points on the neural SDF surface [S1].

**Algorithm 1: Locate Edge Points**
*   **Input:** Ray-surface intersection point $\hat{\mathbf{x}}$.
*   **Output:** An edge point $\mathbf{x}_t$ or `NOT FOUND`.
*   **Hyperparameters:**
    *   $K$: Maximum number of steps.
    *   $\epsilon$: Step size.
    *   $\delta$: Threshold for edge detection.
*   **Steps:**
    1.  Initialize $\mathbf{x}_t \leftarrow \hat{\mathbf{x}}$.
    2.  For $i$ from 1 to $K$:
        3.  Compute surface normal $\mathbf{n}_t$ at $\mathbf{x}_t$ (via gradient of $S_{\Theta_s}$).
        4.  Compute vector from light/camera to point: $\mathbf{v} = \mathbf{x}_t - \mathbf{o}$.
        5.  Normalize: $\hat{\mathbf{v}} = \mathbf{v} / \|\mathbf{v}\|_2$.
        6.  Check condition: If $\hat{\mathbf{v}}^T \mathbf{n}_t < \delta$:
            7.  Return $\mathbf{x}_t$.
        8.  Else:
            9.  Update position: $\mathbf{x}_t \leftarrow \mathbf{x}_t + \epsilon \cdot (\mathbf{n}_t - \hat{\mathbf{v}} (\hat{\mathbf{v}}^T \mathbf{n}_t))$.
                *   *Note:* The term $(\mathbf{n}_t - \hat{\mathbf{v}} (\hat{\mathbf{v}}^T \mathbf{n}_t))$ projects the normal onto the plane perpendicular to the view direction, effectively moving the sample along the surface tangent towards the edge [S1].
    10. Return `NOT FOUND` if the loop completes without finding an edge [S1].

**Parameters:**
*   Positional encoding frequencies: As defined in [Parent].
*   Feature dimension: 256 [S1].
*   Light intensity $L$: Optimized scalar [S1].

**Pitfalls:**
*   **Topology Errors:** If Stage 1 is skipped, the optimization frequently diverges or gets stuck in local minima with incorrect topology (e.g., wrong number of holes) [S1].
*   **Ambiguity:** Without the collocated flashlight assumption, shadows and global illumination would complicate the light model, but these are ignored in this method [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** IRON achieves significantly better inverse rendering quality compared to prior works [S1].
*   **Strengths:**
    *   Recovers correct topology via volumetric initialization [S1].
    *   Produces deployable mesh and texture outputs [S1].
    *   Edge sampling improves reconstruction quality around depth discontinuities [S1].
*   **Limitations:**
    *   Restricted to opaque objects [S1].
    *   Restricted to photometric images with collocated flashlight and no ambient light [S1].
    *   Ignores shadows and global illumination [S1].

## Background
IRON was proposed by Kai Zhang, Fujun Luan, Zhengqi Li, and Noah Snavely from Cornell University, Adobe Research, and Google Research [S1]. It was published in 2022 [S1]. The method addresses the problem of inverse rendering for photometric images, a capture scenario where a flashlight is co-located with a moving camera [S1]. It builds on prior works using neural SDFs [30, 34, 39] and neural radiance fields [23, 34, 38] [S1].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
