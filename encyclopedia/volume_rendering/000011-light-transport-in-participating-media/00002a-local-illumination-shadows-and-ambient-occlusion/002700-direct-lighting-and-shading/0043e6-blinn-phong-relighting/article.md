# Blinn-Phong Relighting

Also called: Real-time Lighting Adjustment, Semantic Blinn-Phong Relighting

## Overview
Blinn-Phong Relighting is a technique for adjusting the illumination of a scene in real-time by scaling the parameters of the Blinn-Phong shading model. It solves the problem of static lighting in pre-computed or neural scene representations by allowing dynamic modification of ambient, diffuse, and specular contributions without retraining the underlying geometry or appearance model. It differs from Physically-Based Rendering (PBR) Lighting [6815-physically-based-rendering-pbr-lighting.md] by using a simplified, non-physical specular term that is computationally cheaper and easier to tune for artistic or specific simulation effects, such as night-time urban scenes.

## How it works
The method relies on the standard Blinn-Phong shading equation, where the final color is a sum of ambient, diffuse, and specular components. The core mechanism for relighting is the scaling of the model's lighting coefficients during inference or rendering.

**Inputs:**
*   Scene representation (e.g., Gaussian primitives, neural hash encoding) containing albedo, normal, and optionally semantic fields.
*   Lighting parameters: ambient coefficient $k_a$, diffuse coefficient $k_d$, specular coefficient $k_s$, and specular exponent $\beta$ (or $k$).
*   Light positions and directions.
*   Camera position and viewing direction.

**Outputs:**
*   Rendered pixel color.

**Data Structures:**
*   The scene representation stores per-point or per-primitive attributes. In UrbanIR [S2], these include albedo $\mathbf{a}$, normal $\mathbf{n}$, and semantic label $s$. In TexGS-VolVis [S1], primitives store opacity and color attributes that are modulated by lighting coefficients.
*   Lighting parameters are global scalars or per-light vectors.

**Steps:**
1.  **Retrieve Attributes:** For a given ray intersection point $\mathbf{x}(\mathbf{r})$, retrieve the surface normal $\mathbf{n}$, albedo $\mathbf{a}$, and semantic label $s$ from the scene representation.
2.  **Compute Half-Vector:** Calculate the half-vector $\mathbf{h}$ between the view direction $\mathbf{v}$ (from $\mathbf{x}$ to camera) and the light direction $\mathbf{l}$ (from $\mathbf{x}$ to light source).
    $$ \mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|} $$
3.  **Calculate Specular Term:** Compute the specular intensity using the dot product of the normal and the half-vector, raised to the power of the specular exponent $\beta$ (or $k$).
    $$ \text{specular} = (\mathbf{n} \cdot \mathbf{h})^\beta $$
    [G] If $\mathbf{n} \cdot \mathbf{h} < 0$, the specular term is zero.
4.  **Scale Parameters:** Apply the user-defined scaling factors to the base Blinn-Phong coefficients $k_a, k_d, k_s, \beta$. This is the primary relighting step.
    $$ k_a' = k_a \cdot \alpha_a, \quad k_d' = k_d \cdot \alpha_d, \quad k_s' = k_s \cdot \alpha_s, \quad \beta' = \beta \cdot \alpha_\beta $$
    [G] $\alpha$ represents the user-specified scaling factor.
5.  **Compute Final Color:** Combine the components.
    $$ \mathbf{C} = k_a' \mathbf{a} + k_d' (\mathbf{a} \cdot \mathbf{l}) + k_s' \text{specular} $$
    [G] The diffuse term typically uses the Lambertian cosine law $\max(0, \mathbf{n} \cdot \mathbf{l})$.

**Formulas:**
*   **Standard Blinn-Phong:**
    $$ \mathbf{C} = k_a \mathbf{a} + k_d \mathbf{a} (\mathbf{n} \cdot \mathbf{l}) + k_s (\mathbf{n} \cdot \mathbf{h})^\beta $$
*   **UrbanIR Spotlight Diffuse Radiance [S2]:**
    $$ \mathbf{L}^i_{\text{dif}}(\mathbf{r}) = \frac{1}{\|\mathbf{o}^i_L - \mathbf{x}(\mathbf{r})\|^2} \left( l \cdot \mathbf{d}^i_L \right)^k $$
    where $l = \frac{\mathbf{o}_L^i - \mathbf{x}(\mathbf{r})}{\|\mathbf{o}^i_L - \mathbf{x}(\mathbf{r})\|}$.
*   **UrbanIR Night-time Shading [S2]:**
    $$ \mathbf{A}(\mathbf{r}) \left( \sum \mathbf{L}_{\text{dif}}^i \mathbf{D}_i \mathbf{V}_i + \mathbf{L}_{\text{amb}} \right) + \sum_i \mathbf{L}^i_{\text{spec}} $$
    For sky regions $S(\mathbf{r}) \in \text{sky}$, $\mathbf{C}(\mathbf{r}) = \mathbf{L}_{\text{sky}}(\mathbf{r})$.

**Parameters:**
*   $k_a$: Ambient coefficient.
*   $k_d$: Diffuse coefficient.
*   $k_s$: Specular coefficient.
*   $\beta$ or $k$: Specular exponent/shininess. In UrbanIR, $k$ modulates spotlight intensity decay with angle [S2].
*   $\mathbf{o}^i_L$: Center of spotlight $i$ in $\mathbb{R}^3$ [S2].
*   $\mathbf{d}^i_L$: Direction of spotlight $i$ in $\mathbb{R}^3$ [S2].
*   $\gamma$: Specular strength inherited from semantic field for car regions in UrbanIR [S2].

**Edge Cases and Pitfalls:**
*   **Degenerate Normals:** If $\mathbf{n}$ is zero or undefined, the specular term is zero.
*   **Back-facing Surfaces:** If $\mathbf{n} \cdot \mathbf{l} < 0$, the diffuse term should be zero.
*   **Precision:** Scaling parameters can lead to overexposure if not clamped.
*   **Semantic Dependency:** In UrbanIR, specular effects on cars depend on accurate semantic segmentation [S2].

## Subtopics

### UrbanIR Relighting
Simulates night-time scenes by defining headlights and street lights, illuminating the scene with specular and lens flare effects. It uses the Blinn-Phong model for specular reflection on car regions, inheriting the specular strength $\gamma$ from the semantic field. Outdoor relighting adjusts sun position/color and sky color. It uses a spotlight model with center $\mathbf{o}^i_L$ and direction $\mathbf{d}^i_L$, where diffuse radiance decays with distance and angle modulated by constant $k$. Lens flares are approximated using image-based compositing [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for shading, assuming constant-time retrieval of scene attributes. Real-time performance is achievable due to the simplicity of the Blinn-Phong model.
*   **Memory Complexity:** Depends on the scene representation. UrbanIR uses hash tables for geometry and appearance [S2].
*   **Strengths:** Real-time adjustability, low computational cost, ease of implementation.
*   **Limitations:** Non-physical specular term, less accurate than PBR for complex materials.
*   **When to Use:** Real-time applications, night-time simulation, artistic control.
*   **When Not to Use:** Physically accurate rendering, complex material interactions.

## Background
The Blinn-Phong shading model was introduced by James F. Blinn in 1977 [S1, S2]. It is a modification of the Phong reflection model, using the half-vector instead of the reflected view vector for specular calculation. It has been widely used in computer graphics for its balance between visual quality and computational efficiency. Recent applications include TexGS-VolVis for volume visualization editing [S1] and UrbanIR for large-scale urban scene inverse rendering [S2].

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
[S2] UrbanIR: Large-Scale Urban Scene Inverse Rendering from a Single Video, https://doi.org/10.1109/3dv66043.2025.00052, fetched 2026-10-30
