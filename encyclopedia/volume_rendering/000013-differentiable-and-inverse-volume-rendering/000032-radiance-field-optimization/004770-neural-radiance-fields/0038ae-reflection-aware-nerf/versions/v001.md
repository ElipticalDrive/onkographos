# Reflection-aware NeRF

Also called: RA-NeRF

## Overview
Reflection-aware NeRF is a neural rendering method that jointly models planar surfaces and explicitly casts reflected rays to capture the source of high-frequency reflections, preventing false geometry duplication along the primary ray [S1]. It differs from methods like NeRFReN and MS-NeRF, which decompose the scene into independent components and may cause false geometries, by using a single radiance field and ensuring that if an actual object is observed along the reflected ray, no false duplicate is created along the primary ray at the same depth [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view images of a scene containing planar reflections (e.g., windows, mirrors).
*   **Output:** A radiance field model (InstantNGP-based) capable of rendering reflection-aware images and accurate geometry.

**Data Structures:**
*   **Plane Parameterization:** Each planar surface is defined by $s = (p, n, u, w, h)$ [S1].
    *   $p$: Plane center (3D point).
    *   $n$: Plane normal (unit vector).
    *   $u$: Normalized up vector defining the y-axis direction of the plane.
    *   $w, h$: Dimensions of the plane.
*   **Radiance Field:** Uses the InstantNGP architecture (hash grid + MLP) to predict volume density $\sigma$ and color $\mathbf{c}$ [S1].
*   **Attenuation Field:** A separate NGP predicts an attenuation factor to model Fresnel effects and compositing [S1].

**Algorithm Steps:**

1.  **Plane Annotation and Parameterization:**
    *   Train an initial InstantNGP model and render depth maps for the input views [S1].
    *   Use Segment Anything (SAM) to segment planar regions (e.g., windows) in 2D images [S1].
    *   Unproject the four dilated corners of the segmented regions into 3D space using the rendered depth maps [S1].
    *   Perform a least-squares fit to these 3D points to initialize the plane parameters $s = (p, n, u, w, h)$ [S1].
    *   *Note:* If multiple planes intersect a ray, select the intersection with the nearest plane [S1].

2.  **Ray Casting and Reflection Handling:**
    *   For a primary ray $\mathbf{r} = (\mathbf{x}, \mathbf{d})$, check for intersection with any annotated plane [S1].
    *   If no intersection occurs, perform standard volume rendering using the NeRF model to obtain color $\mathbf{C}(\mathbf{r})$ [S1].
    *   If an intersection occurs at point $\mathbf{x}$ with normal $\mathbf{n}$:
        *   Calculate the reflected direction $\mathbf{d}'$ [S1]:
            $$
            \mathbf{d}' = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n}) \mathbf{n}
            $$
        *   Define the reflected ray $\mathbf{r}' = (\mathbf{x}, \mathbf{d}')$ [S1].
        *   Query the same NeRF model along $\mathbf{r}'$ to obtain the volume density $\sigma$ and color $\mathbf{c}$ for the source of the reflection [S1].
        *   Perform volume rendering along $\mathbf{r}'$ to obtain the reflection color $\mathbf{C}(\mathbf{r}')$ [S1].

3.  **Color Composition:**
    *   The final color is composed of the primary rendering $\mathbf{C}(\mathbf{r})$ and the reflected rendering $\mathbf{C}(\mathbf{r}')$ modulated by an attenuation field [S1].
    *   The attenuation field is optimized to compensate for non-linear RGB space (post-tone mapping), which invalidates direct application of Fresnel equations [S1].
    *   Let $\alpha$ be the attenuation factor predicted by the attenuation NGP. The composed color $\mathbf{C}_{comp}(\mathbf{r})$ is calculated as:
        $$
        \mathbf{C}_{comp}(\mathbf{r}) = (1 - \alpha) \mathbf{C}(\mathbf{r}) + \alpha \mathbf{C}(\mathbf{r}')
        $$
        *Note: The exact weighting formula is not explicitly provided in the sources, but the attenuation field models both actual attenuation (Fresnel) and compositing [S1]. The loss function uses $\mathbf{C}_{comp}$.*

4.  **Training Schedule:**
    *   **Phase 1 (0–50K iterations):** Jointly optimize plane geometry $s$, the radiance field, and the attenuation field using photometric loss [S1]:
        $$
        \mathcal{L}_{photo} = \| \mathbf{C}_{comp}(\mathbf{r}) - \mathbf{C}_{gt}(\mathbf{r}) \|^2
        $$
    *   **Phase 2 (50K–100K iterations):** Fix $s$. Add sparse edge regularization loss $\mathcal{L}_{edge}$ with weight 0.5 [S1]:
        $$
        \mathcal{L} = \mathcal{L}_{photo} + 0.5 \mathcal{L}_{edge}
        $$
    *   **Phase 3 (100K–200K iterations):** Fix the attenuation field. Gradually decay the weight of $\mathcal{L}_{edge}$ to restore detail [S1].

**Parameters:**
*   **Hardware:** Single NVIDIA V100 GPU [S1].
*   **Network:** InstantNGP (hash grid + MLP) for density/color; additional NGP for attenuation [S1].
*   **Scene Parameterization:** Contraction parameterization [S1].
*   **Iterations:** 150K total (50K joint + 50K regularization + 100K decay) [S1].

**Edge Cases and Pitfalls:**
*   **Non-linear Color Space:** Fresnel equations cannot be directly applied because RGB values are in non-linear space after tone mapping [S1]. The method solves this by optimizing an adaptable attenuation field instead of using physical Fresnel equations [S1].
*   **False Geometry:** Without explicit reflection handling, reflections are often modeled as solid objects at the wrong depth (e.g., a shelf reflected in a window appearing as a real shelf) [S1]. This method prevents that by explaining the reflection via the reflected ray [S1].

## Operating characteristics
*   **Time Complexity:** The method adds approximately 2 hours to the 5 hours required for vanilla InstantNGP to complete 150K iterations on a single V100 GPU [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Eliminates false geometry duplication caused by reflections; handles non-linear color spaces via learned attenuation; uses a single radiance field for simplicity [S1].
*   **Limitations:** Requires plane annotation (via SAM and depth unprojection); performance depends on the accuracy of plane detection; may require significant training time for joint optimization [S1].
*   **When to use:** Scenes with prominent planar reflections (windows, mirrors) where standard NeRF methods produce false geometry [S1].
*   **When not to use:** Scenes without significant reflections or with non-planar reflective surfaces (though the method focuses on planar surfaces) [S1].

## Background
*   **Origin:** Introduced in "Planar Reflection-Aware Neural Radiance Fields" by Chen Gao, Yipeng Wang, Changil Kim, Jia-Bin Huang, and Johannes Kopf [S1].
*   **History:** The method addresses limitations in self-supervised decomposition methods like NeRFReN and MS-NeRF, which can lead to suboptimal decomposition and persistent false geometries [S1]. It builds on InstantNGP and SAM for plane detection [S1].
*   **Usage:** Evaluated on the "Office" dataset, a real-world 360-degree dataset containing pronounced reflections [S1].

## Sources
[S1] Planar Reflection-Aware Neural Radiance Fields, https://doi.org/10.1145/3680528.3687560, fetched 2026-10-30
