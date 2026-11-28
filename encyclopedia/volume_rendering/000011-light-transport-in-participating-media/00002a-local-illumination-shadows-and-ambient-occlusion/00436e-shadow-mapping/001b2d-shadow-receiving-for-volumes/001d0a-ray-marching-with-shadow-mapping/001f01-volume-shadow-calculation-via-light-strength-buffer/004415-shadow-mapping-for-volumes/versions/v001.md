# Shadow Mapping for Volumes

Also called: Image-based shadow algorithms

## Overview
Shadow Mapping for Volumes adapts the standard image-based shadow mapping technique to participate in light transport within volumetric data. It solves the problem of determining whether a sample point along a view ray is occluded by the volume itself when viewed from a light source. Unlike methods that compute exact volumetric occlusion integrals, this approach provides a binary shadow decision based on depth comparisons, offering a performance trade-off where speed is prioritized over physically accurate attenuation [S1, unofficial].

## How it works
The method consists of two passes: a shadow map construction pass (preprocessing) and a rendering pass.

**Inputs and Outputs:**
*   **Input:** Volumetric dataset, light source position and direction (for directional lights) or position (for point lights), transfer function (for alpha shadow maps).
*   **Output:** A shadow map texture (depth buffer) and the final rendered volume with shadowed regions.

**Data Structures:**
1.  **Standard Shadow Map:** A 2D texture storing depth values. Each texel stores the distance from the light source to the first surface hit along the light ray.
2.  **Alpha Shadow Map:** A 2D texture storing multiple layers. Each layer consists of a pair: $(d_i, \alpha_i)$, representing depth and transparency. Additionally, control points for piecewise linear functions are stored to reconstruct the attenuation profile [S1, unofficial].

**Steps:**

1.  **Shadow Map Construction (Preprocessing):**
    *   Render the volume from the light source's perspective.
    *   For each pixel in the light-space image, trace a ray through the volume.
    *   Identify the first hit point (entry point) of the ray into the volume.
    *   **Standard:** Store the depth value $d$ of this first hit point in the shadow map.
    *   **Alpha Shadow Maps:**
        *   Save the first key value (depth and transparency) at the first hit point [S1, unofficial].
        *   Evaluate an error function to determine if further sampling is needed.
        *   If the error exceeds a threshold, save further key values (control points) to approximate the piecewise linear function of the volume's opacity profile [S1, unofficial].
        *   Typical error thresholds used are $0.01$ and $0.00005$ [S1, unofficial].

2.  **Rendering (Shadow Lookup):**
    *   For each sample point $P$ along the view ray during volume raycasting:
    *   Project $P$ into light space to obtain coordinates $(u, v)$.
    *   Retrieve the depth value $d_{shadow}$ from the shadow map at $(u, v)$.
    *   Compute the distance $d_{sample}$ from the light source to $P$.
    *   **Binary Decision:** Compare $d_{sample}$ with $d_{shadow}$.
        *   If $d_{sample} > d_{shadow}$, the point is in shadow.
        *   If $d_{sample} \le d_{shadow}$, the point is lit.
    *   Apply a **shadow threshold** to mark the intensity limit. Points determined to be in shadow have their illumination contribution reduced or set to zero based on this threshold [S1, unofficial].
    *   **Filtering:** Apply filtering to the shadow map (e.g., PCF or similar) to reduce aliasing artifacts [S1, unofficial].

**Pitfalls:**
*   Standard shadow maps only support opaque occluders. They cannot represent partial transparency or volumetric attenuation correctly [S1, unofficial].
*   Precision issues may arise from depth buffer resolution limits, leading to shadow acne or peter-panning.
*   The shadow map must be recomputed if the light source position changes or if the transfer function changes significantly, as the first hit points depend on the volume's opacity definition [S1, unofficial].

## Subtopics

### Alpha Shadow Maps
A variant of shadow mapping that supports semi-transparent occluders. It differs from standard shadow mapping by incorporating multiple layers of depth and transparency for each pixel. Instead of a single depth value, it saves control points of piecewise linear functions that approximate the volume's attenuation profile along the light ray. This allows for more accurate shadowing in translucent volumes. Choose this when visual fidelity of shadows in semi-transparent media is required, accepting higher memory and preprocessing costs. Error values of $0.01$ and $0.00005$ are cited as typical thresholds for determining the granularity of these control points [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N_{light\_pixels} \times N_{samples})$, where $N_{light\_pixels}$ is the shadow map resolution and $N_{samples}$ is the number of samples per light ray. Rendering is $O(N_{view\_pixels} \times N_{view\_samples})$ with a constant-time shadow lookup.
*   **Memory Complexity:** $O(W \times H)$ for the shadow map, where $W, H$ are the shadow map dimensions. Alpha shadow maps require more memory per texel due to multiple layers.
*   **Performance:** On an Intel Core2 6600 (2.4GHz) with 2GB RAM and an nVidia GeForce 8800GTX, the method provides real-time or near-real-time performance compared to exact integration methods [S1, unofficial].
*   **Strengths:** Fast, simple to implement, decouples shadow calculation from the main raycasting loop.
*   **Limitations:** Binary shadowing (in standard form) does not account for volumetric attenuation; artifacts from depth precision; requires rebaking for transfer function changes [S1, unofficial].

## Background
The technique was developed by Timo Ropinski (University of Münster) and others (Markus Hadwiger, Patric Ljung, Christof Rezk Salama) as part of advanced illumination techniques for GPU-based volume raycasting. It was presented in the thesis "Advanced Illumination Techniques for GPU-Based Volume Raycasting" in 2008 [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulz.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
