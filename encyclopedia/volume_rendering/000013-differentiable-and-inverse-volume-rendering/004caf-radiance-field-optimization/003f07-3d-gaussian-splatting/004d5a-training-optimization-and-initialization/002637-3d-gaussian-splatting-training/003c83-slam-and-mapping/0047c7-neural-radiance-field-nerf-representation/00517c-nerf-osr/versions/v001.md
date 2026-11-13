# NeRF-OSR

Also called: Neural Radiance Fields for Outdoor Scene Relighting

## Overview
NeRF-OSR is a neural radiance field approach for outdoor scene relighting that learns a neural representation of scene geometry, diffuse albedo, and illumination-dependent shadows from a set of images capturing the same site from different viewpoints and at different times, enabling simultaneous editing of both the scene’s lighting and viewpoint [S1]. Unlike standard NeRF which learns a Lambertian model under fixed lighting, NeRF-OSR decomposes the scene into spatial occupancy, illumination, shadowing, and diffuse albedo reflectance, trained in a self-supervised manner from outdoor data captured from various viewpoints and at different illuminations [S1]. It sets itself apart from siblings like [Scene NeRF](12484-scene-nerf.md) by providing explicit intrinsic decomposition (albedo, geometry, lighting) rather than just view-dependent appearance, allowing for physical relighting and albedo editing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Multiple RGB images of a single scene, shot at different timings (illumination conditions) and from different viewpoints [S1].
*   **Output:** A neural representation of the scene that can render novel views under arbitrary lighting conditions, specified by the user via environment maps or Spherical Harmonics (SH) coefficients [S1].

**Data Structures:**
The method relies on two Multi-Layer Perceptrons (MLPs) to model the scene properties:
1.  **Density MLP:** Takes 3D position $x$ as input and outputs density $\sigma(x)$ [S1].
2.  **Color/Albedo MLP:** Takes 3D position $x$ as input and outputs diffuse albedo $c_{albedo}(x)$ [S1].
*Note: The sources describe the base NeRF structure where density and color are modeled by MLPs. NeRF-OSR modifies the color output to be albedo and combines it with an explicit lighting model.*

**Illumination and Shadow Model:**
The method uses an explicit 2nd-order Spherical Harmonics (SH) illumination model [S1]. The final color is not directly predicted by the MLP but is computed as a function of albedo, geometry, and lighting. The rendering equation is adapted to separate intrinsic properties from lighting [S1]:
$$ C(o, d) = \sum_{i=1}^{N_{depth}} T(t_i) \alpha(\sigma(x_i)\delta_i) c(x_i) $$
where $c(x_i)$ is the color at point $x_i$, derived from the albedo and the illumination model [S1].

**Step-by-Step Method:**

1.  **Data Pre-processing:**
    *   Discard dynamic objects (cars, people, bikes) from the training stage using segmentation masks [S1].
    *   Estimate masks for sky and vegetation and exclude them from evaluation due to highly varying appearance across weather seasons [S1].
    *   Split data into training and testing sets based on recording sessions [S1].

2.  **Ray Sampling:**
    *   For each pixel, cast a ray from camera origin $o$ in direction $d$.
    *   Sample $N_{depth}$ points $\{x_i\}_{i=1}^{N_{depth}}$ along the ray, where $x_i = o + t_i d$ [S1].
    *   **Stratified Sampling:** Depths $\{t_i\}$ are selected using stratified sampling from the uniform distribution, spanning from the near to the far camera plane [S1].
    *   **Hierarchical Sampling:** To capture small details, perform a coarse pass with stratified sampling. Use the densities from the coarse pass for importance sampling in the final pass [S1].

3.  **Neural Network Forward Pass:**
    *   For each sampled point $x_i$, query the Density MLP to get $\sigma(x_i)$ [S1].
    *   Query the Albedo MLP to get $c_{albedo}(x_i)$ [S1].
    *   Compute the final color $c(x_i)$ by combining $c_{albedo}(x_i)$ with the scene illumination (SH coefficients) and shadow term [S1]. The sources state the model is an explicit function of geometry, albedo, shadow, and lighting conditions [S1].

4.  **Volume Rendering:**
    *   Compute transmittance $T(t_i) = \exp(-\sum_{j=1}^{N_{depth}-1} \sigma(x_j)\delta_j)$ [S1].
    *   Compute opacity $\alpha(y) = 1 - \exp(-y)$ [S1].
    *   Compute ray color: $C(o, d) = \sum_{i=1}^{N_{depth}} T(t_i) \alpha(\sigma(x_i)\delta_i) c(x_i)$ [S1].
    *   Here, $\delta_i = t_{i+1} - t_i$ [S1].

5.  **Optimization:**
    *   Train the MLPs in a self-supervised manner using the observed ground-truth per-pixel colors [S1].
    *   Supervise the rendered pixel colors of both the coarse and fine passes with the ground-truth colors [S1].
    *   Estimate per-image illumination (SH coefficients) during training [S1].

6.  **Geometry Extraction (Optional for Real-time):**
    *   Extract geometry and albedo from the learned model as a mesh using Marching Cubes [16] at resolution $1000^3$ voxels [S1].
    *   Use the extracted mesh and albedo in an interactive renderer (e.g., OpenGL) for real-time VR rendering [S1].
    *   Lighting model for real-time: $C_{interactive} = C_{ambient} + s \odot C_{sun} \max\{0, D_{sun} \cdot N\}$, where $C_{ambient}$ is ambient color, $s$ is 0/1 based on shadow map occlusion, $C_{sun}$ is sun color, $D_{sun}$ is sun direction, and $N$ is mesh normal [S1].

**Parameters:**
*   $N_{depth}$: Number of samples per ray. Not given in the sources.
*   Marching Cubes Resolution: $1000^3$ voxels [S1].
*   SH Order: 2nd-order [S1].

**Edge Cases and Pitfalls:**
*   **Geometry Inaccuracies:** Tiny bumps in learned geometry lead to significant changes in normals and illumination errors. The model may smooth geometry to achieve more accurate lighting, resulting in some blur [S1].
*   **Dynamic Objects:** Must be masked out; otherwise, they corrupt the static scene representation [S1].
*   **Sky/Vegetation:** Highly variable appearance makes evaluation difficult; masks should be used to exclude these regions from metrics [S1].
*   **Standard NeRF Limitation:** Standard NeRF formulation learns a Lambertian model under fixed lighting because $c(x)$ is defined only through an MLP without encoding lighting [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   **Site 1:** PSNR 19.86 (d/s) and 18.72 (full) [S1].
    *   **Site 2:** PSNR 15.83 (d/s) and 15.43 (full) [S1].
    *   **Site 3:** PSNR 17.38 (d/s) and 16.65 (full) [S1].
    *   Outperforms Yu et al. [46] and Philip et al. [27] on all sites [S1].
*   **Real-time Demo:** Runs in real-time on Intel i7-4770 CPU, Nvidia GeForce GTX 970 (4GB VRAM) GPU, Oculus Rift S HMD. System RAM usage below 3 GB [S1].
*   **Strengths:** Simultaneous novel view and novel lighting generation; explicit intrinsic decomposition (albedo, geometry, lighting); ability to edit albedo and shadows; handles uncontrolled outdoor settings.
*   **Limitations:** Geometry inaccuracies can cause lighting errors; requires posed images with varying illumination; sky and vegetation are difficult to evaluate due to seasonal changes.
*   **When to use:** When simultaneous viewpoint and lighting editing of outdoor scenes is required, and intrinsic decomposition is needed.
*   **When not to use:** When the scene contains significant dynamic objects that cannot be masked, or when only static lighting editing of a single view is needed (simpler methods may suffice).

## Background
NeRF-OSR was introduced by Viktor Rudnev, Mohamed Elgharib, William Smith, Lingjie Liu, Vladislav Golyanik, and Christian Theobalt from MPI for Informatics, SIC, and University of York [S1]. It was published in arXiv:2112.05140v2 on 21 Jul 2022 [S1]. It is the first neural radiance fields approach for outdoor scene relighting, enabling the learning of scene intrinsics (geometry, albedo, illumination, shadows) from uncontrolled outdoor data [S1]. It builds on the original NeRF representation [12096-neural-radiance-field-nerf-representation.md] but extends it with explicit lighting and shadow models [S1].

## Sources
[S1] NeRF for Outdoor Scene Relighting, https://doi.org/10.1007/978-3-031-19787-1_35, fetched 2026-10-26
