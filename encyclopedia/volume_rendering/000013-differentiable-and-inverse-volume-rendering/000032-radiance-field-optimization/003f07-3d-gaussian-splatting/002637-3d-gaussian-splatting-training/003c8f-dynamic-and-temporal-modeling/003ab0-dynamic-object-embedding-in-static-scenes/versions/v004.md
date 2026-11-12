# Dynamic Object Embedding in Static Scenes

Also called: Lightweight Dynamic NeRF Modeling via Local Deformation Field, Local deformation field model

## Overview
This method integrates dynamic advertising objects into static Neural Radiance Field (NeRF) scenes by ensuring perspective consistency through spatial alignment and joint rendering with alpha blending [S1]. It addresses the high computational cost of traditional dynamic NeRFs (e.g., DyNeRF), which require over 100M parameters and independent models per time step, by using a static NeRF for the background and a lightweight local deformation field for dynamic regions [S1]. Unlike siblings that model the entire 4D volume, this approach isolates motion to specific objects, reducing parameter count by 95% [S1].

## How it works
The method decomposes the scene into a static background and a dynamic foreground object, aligning them spatially and rendering them jointly.

**Inputs and Outputs:**
*   **Inputs:** Sparse video frames of the static scene (exhibition stand + background), sparse video frames of the dynamic object (e.g., rotating watch), and four corresponding marking points (e.g., stand corners) in both views for alignment.
*   **Outputs:** A rendered frame at time $t$ combining the static scene and the deformed dynamic object.

**Data Structures:**
1.  **Static NeRF Model:** A standard NeRF representation (MLP or grid-based) with fixed parameters. It does not accept time $t$ as an input. It maps 3D coordinates $(x, y, z)$ to color $C$ and density $\sigma$.
2.  **Local Deformation Field:** A 3-layer Multi-Layer Perceptron (MLP).
    *   **Architecture:** Input dimension 3 (spatial coordinates $x$) + 1 (time $t$) = 4. Hidden layers: 2 layers with 128 units each. Output dimension: 3 (displacement vector $\Delta x$).
    *   **Parameters:** Approximately 5 million parameters [S1].
3.  **Alignment Data:** A set of 4 pairs of 3D points $\{(p_{static}^{(i)}, p_{dynamic}^{(i)})\}_{i=1}^4$ used for coordinate transformation.

**Numbered Steps:**

1.  **Static Scene Reconstruction:** Train a static NeRF model on the background scene using sparse input reconstruction. The model parameters are fixed and do not include a time dimension [S1].
2.  **Dynamic Object Modeling:** Train the local deformation field $\Delta(x, t)$ on the dynamic object's video frames. This field predicts the displacement of a point $x$ at time $t$ [S1].
3.  **Spatial Alignment:**
    *   Select four marking points in the static scene and the corresponding four points in the dynamic object's coordinate system.
    *   Use the RANSAC algorithm to compute the rigid transformation (rotation and translation) that aligns the dynamic object's coordinate system with the static scene's coordinate system.
    *   Goal: Ensure perspective consistency with an alignment error of less than 1 pixel [S1].
4.  **Joint Rendering (at time $t$):**
    *   For a ray passing through the dynamic object's region, calculate the deformed position $x_{deformed}$ using the local deformation field:
        $$ x_{deformed} = x + \Delta(x, t) $$
        where $x$ is the original 3D coordinate along the ray, and $\Delta(x, t)$ is the displacement vector predicted by the 3-layer MLP [S1].
    *   Query the **Static NeRF** at $x_{deformed}$ to obtain the color $C_{dyn}$ and density $\sigma_{dyn}$ of the dynamic object. Note: The static NeRF is queried here to ensure the dynamic object's appearance is consistent with the static scene's lighting and background context, effectively embedding the object [S1].
    *   Query the **Static NeRF** at the original position $x$ (or along the ray behind the object) to obtain the background color $C_{bg}$ and density $\sigma_{bg}$.
    *   Fuse the dynamic object with the static background using alpha blending. The final color $C_{final}$ is calculated as:
        $$ C_{final} = \alpha_{obj} C_{dyn} + (1 - \alpha_{obj}) C_{bg} $$
        where $\alpha_{obj}$ is the alpha value derived from the dynamic object's density $\sigma_{dyn}$ and the integration step size. The source specifies a blending factor of 0.95 for the dynamic object, implying $\alpha_{obj} = 0.95$ or a weighted combination where the dynamic object dominates [S1].

**Formulas:**
*   **Deformation:** $F(x, t) = F_{static}(x + \Delta(x, t))$ [S1]. This equation indicates that the radiance field of the dynamic element is evaluated by warping the input coordinate through the deformation field before querying the static field.
*   **Temporal Consistency Loss:** To avoid visual discontinuity (color jumps, light/shadow changes), a temporal consistency loss is applied during training. The specific formula is not provided in the text, but it aims for SSIM $\ge 0.92$ [S1].

**Parameters:**
*   **Deformation Field MLP:** 3 layers, hidden dimension 128 [S1].
*   **Total Parameters:** ~5M for the deformation field [S1].
*   **Blending Factor:** 0.95 for the dynamic object in the final frame fusion [S1].
*   **Alignment Error:** < 1 pixel [S1].

**Edge Cases and Pitfalls:**
*   **Alignment Error:** If the RANSAC alignment fails or has high error (>1 pixel), perspective consistency is lost, causing visual artifacts where the dynamic object appears detached or misaligned with the static background [S1].
*   **Sparse Input:** The method relies on sparse input reconstruction. Insufficient frames (e.g., fewer than 20-30 for the dynamic object) may lead to poor deformation field training and temporal flickering [S1].
*   **Occlusion:** The method assumes the dynamic object is embedded in the static scene. If the dynamic object occludes significant parts of the static scene, the static NeRF query at $x_{deformed}$ might not correctly represent the background behind the object if the static NeRF was not trained to handle such occlusions or if the deformation pushes points into unobserved regions.

## Operating characteristics
*   **Time Complexity:** Rendering is comparable to standard NeRF but requires an additional MLP evaluation for the deformation field $\Delta(x, t)$ for points in the dynamic region. The deformation field is small (5M params), so this overhead is minimal compared to full dynamic NeRFs [S1].
*   **Memory Complexity:** Significantly lower than traditional dynamic NeRFs. By using a static NeRF for the background and a small deformation field for the dynamic part, the total parameter count is reduced by 95% compared to DyNeRF (100M+ params) [S1].
*   **Performance:** On sparse input (e.g., 20-30 frames), the method achieves a PSNR increase of $2.3 \pm 0.2$ dB over 10 trials compared to baseline sparse reconstruction methods [S1]. Temporal consistency is maintained with SSIM $\ge 0.92$ [S1].
*   **Strengths:** Low parameter count, suitable for VR devices due to reduced computational load, perspective consistency through spatial alignment [S1].
*   **Limitations:** Only models dynamic regions (up to 20% of the scene). Requires manual or semi-automatic selection of marking points for alignment. Not suitable for fully dynamic scenes where the entire volume changes [S1].

## Background
This method was proposed in the context of VR advertising, specifically for embedding dynamic advertising elements (like rotating watches) into static exhibition stand scenes. It was developed to overcome the limitations of traditional dynamic NeRFs (like DyNeRF) which are too computationally expensive for real-time VR rendering. The approach was published in *Informatica* in 2026 [S1].

## Sources
[S1] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
