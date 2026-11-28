# Differentiable Object Insertion

Also called: Differentiable Object Insertion Composition, Differentiable virtual object insertion, Foreground/background shadow rendering, Object Insertion (Differentiable Rendering)

## Overview
Differentiable Object Insertion composites a virtual 3D object with known geometry and material into a real-world image, ensuring the object is illuminated by estimated scene lighting and casts physically plausible shadows on the background [S1]. It distinguishes itself from standard compositing by rendering both the foreground object's appearance and the background's shadow map as differentiable operations, allowing gradients from the final composited image to backpropagate to the underlying lighting parameters [S1]. This enables the joint optimization of lighting estimation and object insertion for augmented reality applications [S1].

## How it works
**Inputs:**
*   Real input image $I$ with known depth map $D$ and camera intrinsics.
*   Virtual object geometry $M$ (mesh) and material properties $\Theta$ (base color, metallic, roughness, specular).
*   Estimated hybrid lighting representation $L$, which provides radiance $L(x, l)$ for any position $x$ and direction $l$ [S1].

**Outputs:**
*   Composited image $I_{edit}$ containing the inserted object and its shadows on the background [S1].

**Data Structures:**
*   **Lighting Representation $L(x, l)$:** A function returning HDR radiance $\mathbb{R}^3_+$ for a point $x \in \mathbb{R}^3$ and direction $l \in S^2$. It is computed by querying a voxelized scene representation (LVSG) and an environment map $L_{env}$ [S1].
*   **G-Buffer:** Stores intersection location $x$, surface normal $n$, and material properties $\theta$ for rays intersecting the inserted object [S1].
*   **Shadow Ratio Map:** A 2D map storing the ratio $S_p$ for each background pixel $p$ [S1].

**Step 1: Lighting Query Function**
To compute the radiance $L(x, l)$ for a ray starting at $x$ in direction $l$:
1.  Select $K$ equi-spaced locations along the ray.
2.  Use nearest neighbor interpolation to get voxel values $\{\alpha_k, \xi_k\}_{k=1}^K$ from the Light Volume Scene Graph (LVSG) [S1].
3.  Query the environment map intensity $L_{env}(l)$ via bilinear interpolation [S1].
4.  Compute transmittance $\tau_k = \prod_{i=1}^{k-1} (1 - \alpha_i)$ with $\tau_1 = 1$ [S1].
5.  Compute final radiance using volume rendering [S1]:
    $$ L(x, l) = \sum_{k=1}^{K} \tau_{k-1} \alpha_k G(-l; \xi_k) + \tau_K L_{env}(l) \quad (1) $$
    where $G$ is the emission function of the voxel [S1].

**Step 2: Foreground Object Appearance Rendering**
1.  Shoot rays from the camera origin to the scene.
2.  Perform ray-mesh intersection detection against the inserted object $M$ [S1].
3.  For each intersected ray, create a G-buffer storing intersection $x$, normal $n$, and material $\theta$ [S1].
4.  Render the object appearance $I_x$ using Monte-Carlo numerical integration over $N$ sampled lighting directions $l_k$ [S1]:
    $$ I_x = \frac{1}{N} \sum_{k=1}^{N} \frac{f(l_k, v; \theta) L(x, l_k) (n \cdot l_k)_+}{p(l_k)} \quad (2) $$
    where:
    *   $v$ is the viewing direction of the camera ray [S1].
    *   $f$ is the Disney BRDF function [S1].
    *   $p(l_k)$ is the probability density of sampling direction $l_k$ [S1].
    *   $(\cdot)_+$ denotes the positive part [S1].

**Step 3: Background Shadow Map Rendering**
1.  For each background scene pixel $p$, compute its 3D location $x_s$ from the depth map $D$ [S1].
2.  Compute pre-insertion lighting distribution $\{L(x_s, l_k)\}_{k=1}^{N_s}$ by uniformly selecting $N_s$ light directions on the upper hemisphere [S1].
3.  Compute post-insertion lighting distribution $\{L'(x_s, l_k)\}_{k=1}^{N_s}$:
    *   Perform ray-mesh query for rays $(x_s, l_k)$ against the inserted object $M$ [S1].
    *   If the ray is occluded by the object, set $L'(x_s, l_k) = I_a$ (ambient value, empirically $0.1$) [S1].
    *   Otherwise, $L'(x_s, l_k)$ remains the original radiance [S1].
4.  Compute the shadow ratio $S_p$. The general form is [S1]:
    $$ S_p = \frac{I'_p}{I_p} = \frac{\sum_{k=1}^{N_s} f_{scene}(x_s, l_k, v, \theta) L'(x_s, l_k) (n \cdot l_k)_+}{\sum_{k=1}^{N_s} f_{scene}(x_s, l_k, v, \theta) L(x_s, l_k) (n \cdot l_k)_+} \quad (3) $$
5.  Simplify $S_p$ by assuming the scene surface is Lambertian with constant diffuse albedo $f_{scene} = f_d$ and normal $n$ pointing upward. The BRDF term $f_d$ cancels out [S1]:
    $$ S_p = \frac{\sum_{k=1}^{N_s} L'(x_s, l_k) (n \cdot l_k)_+}{\sum_{k=1}^{N_s} L(x_s, l_k) (n \cdot l_k)_+} \quad (4) $$
6.  Compute the shadowed background intensity $I'_p = S_p I_p$ [S1].

**Step 4: Composition**
Composite the foreground object $I_{object}$ (rendered in Step 2), the alpha mask $M$, the input image $I$, and the shadow map $I_{shadow}$ (derived from $I'$) into the final result [S1]:
$$ I_{edit} = M I_{object} + (1 - M) I - I_{shadow} \quad (5) $$
Note: The term $I_{shadow}$ in Eq. 5 represents the shadow cast *onto* the background, effectively subtracting the darkened portion from the original background or adding the shadow layer. The source defines $I_{shadow}$ as the ratio map result applied to $I$, but the composition formula subtracts it from the background term $(1-M)I$. Specifically, the background becomes $(1-M)I - I_{shadow}$ where $I_{shadow}$ is the darkening effect [S1].

**Step 5: Gradient Propagation**
The process is differentiable with respect to lighting $L$ [S1]:
*   Foreground gradients: $\frac{\partial I_x}{\partial L}$ via Eq. 2 [S1].
*   Background gradients: $\frac{\partial I'_p}{\partial L} = \frac{\partial S_p}{\partial L} I_p$, where $S_p$ is differentiable via Eq. 4 [S1].
*   This allows backpropagation from the loss on $I_{edit}$ to the lighting parameters [S1].

**Parameters and Settings:**
*   $N$: Number of sampled lighting directions for foreground. During training, exactly 5000 rays are sampled per instance [S1].
*   $N_s$: Number of sampled light directions for background shadows. During training, 450 rays per pixel are sampled [S1].
*   Shadow Map Resolution: During training, 160x90 resolution is used to conserve memory [S1]. After training, higher resolution is affordable [S1].
*   $I_a$: Ambient lighting intensity, set to 0.1 [S1].
*   BRDF: Simplified Disney BRDF using base color $c_{base}$, metallic $m$, roughness $r$, and specular $s$ [S1].

**Edge Cases and Pitfalls:**
*   **Depth Accuracy:** The method relies on accurate depth maps $D$ to reconstruct 3D scene points $x_s$ for shadow casting. Errors in depth lead to incorrect shadow placement [S1].
*   **Lambertian Assumption:** The simplification of the shadow ratio (Eq. 4) assumes Lambertian surfaces. This may introduce errors on specular or complex background materials [S1].
*   **Occlusion Handling:** If a ray is occluded, the intensity is clamped to a constant ambient value $I_a$. This is an approximation; physically, the occluded light should be zero or scattered, but $I_a=0.1$ provides a stable empirical baseline [S1].
*   **Training vs. Inference:** Training uses uniform sampling and low-resolution shadow maps for efficiency. Inference can switch to importance sampling and higher resolutions for realism [S1].

## Subtopics

### [Data Augmentation with Virtual Object Insertion](19387-data-augmentation-with-virtual-object-insertion.md)
This method utilizes differentiable object insertion to automatically generate synthetic training data for downstream 3D object detection tasks by inserting diverse 3D assets into real-world images. It distinguishes itself from category-specific image manipulation methods by predicting physics-based scene lighting, making it agnostic to object class and enabling the insertion of rarely observed categories.

## Operating characteristics
*   **Time Complexity:** Depends on the number of sampled rays $N$ (foreground) and $N_s$ (background). Training uses fixed low counts (5000 rays/instance, 450 rays/pixel) for efficiency [S1]. Inference can increase these counts for higher quality [S1].
*   **Memory Complexity:** Proportional to the resolution of the shadow map and the number of sampled directions. Training uses a small 160x90 shadow map to save memory [S1].
*   **Performance:** The lighting estimation network takes 180ms per image on a TITAN V GPU [S1]. The differentiable rendering module adds overhead due to ray-mesh intersection and Monte-Carlo integration but remains feasible for real-time or near-real-time AR applications [S1].
*   **Strengths:** Enables end-to-end differentiable optimization of lighting and object insertion; handles complex shadows and lighting interactions; agnostic to object class [S1].
*   **Limitations:** Requires accurate depth maps and camera intrinsics; assumes Lambertian background for simplified shadow ratios; performance is sensitive to the number of samples [S1].
*   **When to use:** Augmented Reality insertion, synthetic data generation for 3D detection, lighting-aware image editing [S1].
*   **When not to use:** Scenes with highly specular backgrounds where the Lambertian assumption fails significantly; scenes without reliable depth information [S1].

## Background
The method was introduced in "Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion" by Wang et al. [S1]. It addresses the challenge of realistic object insertion in augmented reality by jointly estimating lighting and rendering shadows in a differentiable manner. The work builds on neural light field estimation and deferred rendering techniques [S1]. It is used in the context of outdoor street scene analysis and AR applications, trained on datasets like nuScenes and HoliCity [S1].

## Sources
[S1] Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion, https://doi.org/10.1007/978-3-031-20086-1_22, fetched 2026-10-30
