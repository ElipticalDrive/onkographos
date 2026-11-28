# Layer Separation Heuristic

Also called: Feature Peeling

## Overview
The Layer Separation Heuristic is a method for determining split points in multi-layer volume ray casting to preserve depth cues and handle view-dependent lighting in transparent volumes. It solves the problem of identifying the "main visual effect" along a ray to decide where to transition from high-quality raycasting to lower-cost image warping. It sets itself apart from sibling methods like [Ray-Layer Buffer Generation via Depth Peeling](9474-ray-layer-buffer-generation-via-depth-peeling.md) by using optical properties (opacity/color impact) rather than geometric depth or surface normals to define layer boundaries.

## How it works
The method operates on a per-ray basis within the context of [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md). It identifies a specific sample point along the ray that represents the dominant visual contribution, which serves as a candidate for layer separation.

**Inputs:**
- Ray segment defined by entry and exit points in the volume.
- Volume data and transfer function.
- Current accumulated opacity and color along the ray.

**Outputs:**
- A separation depth (sample index or physical distance) along the ray.

**Data Structures:**
- The ray is discretized into $N$ sample points.
- For each sample point $i$, the following values are tracked:
  - $o_i$: Voxel opacity at sample $i$ (after classification).
  - $c_i$: Voxel color contribution at sample $i$.
  - $A_i$: Accumulated opacity from the ray origin to sample $i$.
  - $R_i$: Remaining opacity (transmittance) from sample $i$ to the ray exit, calculated as $R_i = 1 - A_{i+1}$ (assuming standard front-to-back accumulation where $A_{i+1}$ includes sample $i$'s contribution to the accumulated term for the next step, or more precisely $R_i = \prod_{j=i+1}^{N} (1-o_j)$).

**Steps:**
1. **Evaluate Criteria:** For each sample point $i$ along the ray, calculate two metrics:
   - **Maximum Opacity:** Identify the point where $o_i$ is maximal.
   - **Maximum Color Impact:** Identify the point where the product of the voxel's opacity and the remaining opacity along the ray segment is maximal. The impact $I_i$ is defined as:
     $$ I_i = o_i \times R_i $$
     where $R_i$ is the transmittance through the volume *after* sample $i$.
2. **Select Main Effect:** Choose the sample point that maximizes either the opacity or the color impact. The source notes that the **Maximum Color Impact** method is generally preferred as it better captures light attenuation effects that may occur in the middle of the ray interval, rather than just at the surface.
3. **Handle Discrete Maxima:** If the maximum spans several consecutive sample points (a plateau), select the median of these sample points as the separation depth.
4. **Layer Assignment:** The ray is split at this depth. Segments before this point are typically processed with higher fidelity (raycasting), while segments after may be processed with image warping or lower resolution, depending on the distribution strategy.

**Parameters:**
- **Criterion Selection:** Binary choice between "Max Opacity" and "Max Color Impact". Default/Recommended: Max Color Impact.
- **Layer Count:** Fixed in advance. The heuristic does not dynamically adjust the number of layers.

**Edge Cases and Pitfalls:**
- **Fixed Layer Count:** The heuristic requires a pre-allocated number of layers. If the volume's features are not distributed evenly relative to the camera (e.g., too many features close to the camera, few far away), the heuristic may "waste" layers on over-represented near features, leaving distant features underrepresented. This leads to suboptimal visual quality.
- **Insufficient Layers:** If the total number of layers is too low, the heuristic cannot capture all significant visual effects, leading to artifacts.
- **Overly Sensitive Heuristic:** A heuristic that is too sensitive to noise may create unnecessary layer splits, degrading performance without improving quality.

## Subtopics

### Distribution by Depth Layer
A workload distribution strategy that exploits the observation that close structures contribute more to the final image. It assigns high-quality raycasting to the first $n$ depth layers (close structures) and multi-frame rate image warping to the remaining layers.
- **Mechanism:** Raycasting of close layers is performed on a fast GPU, while back layers are raycasted on a slow GPU. The number of raycasted layers $n$ controls the quality/performance tradeoff.
- **Adaptive Control:** The system can automatically adjust $n$ based on the current frame rate and a desired minimum frame rate. If the frame rate falls below the minimum, $n$ is decreased immediately. Periodically, the system renders with $n+1$ layers to check if improved quality is possible.
- **Performance:** Evaluation on a rotated scene showed that reconstruction error converges to the reference image as more layers are raycasted. Performance (FPS) degrades linearly as more layers are raycasted. The benefit of warping more than 6 layers vanishes.

## Operating characteristics
- **Visual Quality:** Preserves depth cues better than single layered image warping. It outperforms single surface approaches for transparent transfer functions (e.g., MANIX dataset). For opaque transfer functions revealing bones (e.g., VIX dataset), the quality difference is less significant due to less inner structure.
- **Performance:** The number of raycasted layers directly impacts frame rate. Performance degrades linearly with the number of raycasted layers.
- **Limitations:** The heuristic is fixed in advance. It cannot adapt to dynamic changes in feature distribution within the volume. If layers are not distributed well inside the volume, visual quality suffers.
- **When to Use:** Transparent volumes where depth cues and view-dependent lighting are critical. When a fixed number of layers can be reasonably allocated to cover the volume's depth range.
- **When Not to Use:** Opaque volumes with minimal internal structure where the quality gain is marginal. Scenarios requiring dynamic adaptation of layer count to varying feature density.

## Background
The Layer Separation Heuristic, also known as Feature Peeling, was introduced by S. Hauswiesner, D. Kalkofen, and D. Schmalstieg in their work on Multi-Frame Rate Volume Rendering. It addresses the limitations of single-image warping by representing the inner structure of an object as multiple partial images, called layers. This approach solves the problem of preserving depth cues and handling view-dependent lighting in transparent volumes, which cannot be shaded correctly using a single map of normal vectors. The method groups volume elements by distance from the camera and similar opacity to ensure consistent shading.

## Sources
[S1] Multi-Frame Rate Volume Rendering, https://doi.org/10.2312/egpgv.egpgv10.019-026, fetched 2026-10-29
