# Compositional Rendering

Also called: GO-NeRF Compositional Rendering, GO-NeRF Rendering

## Overview
Compositional Rendering is a method for integrating generated 3D objects into an existing neural radiance field (NeRF) scene by decoupling the rendering of the object and the background scene. It solves the problem of occlusion and seamless blending in virtual content creation by rendering the object into a 3D-aware opacity map and compositing it with a pre-rendered background image in image space. It differs from standard direct volume ray casting [19-direct-volume-ray-casting.md] by splitting the integration domain into two distinct components (object vs. scene) and using depth comparisons to resolve occlusions, rather than integrating all geometry into a single ray pass.

## How it works
**Inputs:**
*   A pre-trained background scene NeRF, $F(\theta_s)$.
*   A pre-rendered set of RGB-D images for the background scene, $\{(S_0, D_0), \dots, (S_n, D_n)\}$, corresponding to camera viewpoints $\{C_0, \dots, C_n\}$.
*   An object NeRF, $F(\theta_o)$, defined within a 3D bounding box.
*   A target camera viewpoint $v \in \{C_0, \dots, C_n\}$.

**Outputs:**
*   Final composited image $I_v$.
*   Object RGB image $G_v$.
*   Object opacity map $O_v$.

**Data Structures:**
*   **Scene Pre-render:** A list of tuples $(S_i, D_i)$ where $S_i$ is an RGB image and $D_i$ is a depth image (distance from camera to surface).
*   **Object NeRF:** A neural network $F(\theta_o)$ that takes a 3D coordinate $q$ and viewing direction $d$ as input and outputs volume density $\delta$ and RGB color $c$.
*   **Bounding Box:** A 3D axis-aligned box defining the spatial extent of the object.

**Parameters:**
*   $K$: Number of query points sampled along the ray inside the bounding box.
*   $\Delta_k$: Distance between adjacent query points $q_k$ and $q_{k+1}$ along the ray.

**Algorithm:**

1.  **Pre-render Background (Offline):**
    For each predefined camera viewpoint $C_i$, render the background scene NeRF $F(\theta_s)$ to generate an RGB image $S_i$ and a depth image $D_i$. Store these as the sequence $\{(S_i, D_i)\}$. [S1]

2.  **Ray Casting for Object:**
    For the target viewpoint $v$, define a ray $r_i$ for each pixel $i$. Sample $K$ query points $q_k$ along $r_i$ that lie within the object's 3D bounding box. [S1]

3.  **Object Rendering (Volume Integration):**
    For each query point $q_k$, query the object NeRF $F(\theta_o)$ to obtain volume density $\delta_k$ and RGB color $c_k$. Compute the transmittance $\tau_k$ and the accumulated color $G_v(i)$ and opacity $O_v(i)$ using the standard volume rendering equations:

    $$ \tau_k = \exp\left(-\sum_{t=1}^{k-1} \delta_t \Delta_t\right) $$

    $$ G_v(i) = \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)) c_k $$

    $$ O_v(i) = \begin{cases} \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)) & \text{if } r_i \in \text{box} \text{ and } D_{\text{box}}(i) \le D_v(i) \\ 0 & \text{otherwise} \end{cases} $$

    Where:
    *   $\delta_t$ is the volume density at query point $q_t$.
    *   $\Delta_t$ is the distance between adjacent query points.
    *   $D_{\text{box}}(i)$ is the depth of the object's bounding box intersection (or the depth of the rendered object geometry) for pixel $i$.
    *   $D_v(i)$ is the depth of the pre-rendered background scene for pixel $i$.
    *   The condition $D_{\text{box}}(i) \le D_v(i)$ ensures that the object is in front of the background. If the object is behind the background or the ray does not intersect the box, the opacity is set to 0. [S1]

4.  **Compositing:**
    Combine the rendered object $G_v$ and the pre-rendered background $S_v$ using the opacity map $O_v$:

    $$ I_v = G_v \cdot O_v + S_v \cdot (1 - O_v) $$

    This blends the object into the scene, preserving the background where the object is transparent or absent. [S1]

**Pitfalls:**
*   **Occlusion Handling:** If the ray does not intersect the 3D bounding box ($r_i \notin \text{box}$) or the generated object is occluded by foreground scene elements ($D_{\text{box}}(i) > D_v(i)$), the opacity $O_v(i)$ must be explicitly set to 0. Failure to do so will result in the object incorrectly appearing behind the background or rendering artifacts at the box boundaries. [S1]
*   **Depth Consistency:** The depth values $D_{\text{box}}(i)$ and $D_v(i)$ must be measured in the same coordinate system (typically camera-space Z-depth) for the comparison to be valid.

## Operating characteristics
*   **Time Complexity:** The method reduces computational load during optimization by pre-rendering the background scene once. Object rendering requires standard volume integration over $K$ samples per ray. Compositing is $O(1)$ per pixel.
*   **Memory Complexity:** Requires storing the pre-rendered RGB-D images for all predefined viewpoints. This scales with the number of viewpoints $n$ and image resolution.
*   **Strengths:** Decouples object generation from scene rendering, allowing for plug-and-play integration. Handles occlusions effectively by leveraging 3D depth information during the opacity calculation.
*   **Limitations:** Restricted to predefined camera viewpoints used for pre-rendering. Not suitable for arbitrary novel view synthesis outside the pre-rendered set without re-computing the background.
*   **When to use:** Virtual reality content creation where objects need to be inserted into a static or pre-trained NeRF scene.
*   **When not to use:** Dynamic scenes where the background changes, or when arbitrary novel views are required without pre-rendering.

## Background
Compositional Rendering was introduced in the context of GO-NeRF (Generating Objects in Neural Radiance Fields) for virtual reality content creation. It addresses the challenge of seamlessly integrating generated 3D objects into existing neural radiance fields. The method leverages the flexibility of decoupling rendering and the precision of 3D-aware opacity maps to handle occlusions. It is designed to be a plug-and-play solution compatible with various scene pre-training methods. [S1]

## Sources
[S1] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
