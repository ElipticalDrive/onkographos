# Foveated Rendering

Also called: Foveated Volume Rendering

## Overview
Foveated rendering is a technique for eye-tracked displays that increases performance by reducing the resolution of the rendered image (or hologram) outside the user's tracked foveal position [S1]. It differs from sibling methods like [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md) by relying on biological gaze data rather than temporal frame budgets. It is distinct from [Dynamic Frame Rate Adaptation](8221-dynamic-frame-rate-adaptation.md) as it maintains a constant frame rate by reducing spatial fidelity rather than temporal fidelity.

## How it works
**Inputs:**
- Object point cloud or volume data.
- Eye-tracking data providing the fovea position on the display.
- Display resolution parameters.

**Outputs:**
- A composite image with high resolution in the foveal region and lower resolution in the periphery.

**Data Structures:**
- **Object Points:** A list of all object points $o_i$. Unlike methods that prune points, this method retains all points [S1].
- **Sub-hologram Width ($w$):** A parameter defining the resolution of the calculation grid. This width is variable based on distance from the fovea [S1].
- **Blending Weights:** A transition band mask used to blend high-resolution and low-resolution pixel values [S2].

**Steps:**
1. **Determine Foveal Region:** Identify the center region corresponding to the user's tracked fovea position [S1, S2].
2. **Define Resolution Zones:**
   - **High-Resolution Zone:** A tight projection frustum is defined for the center region to maximize culling efficiency [S2].
   - **Low-Resolution Zone:** For regions outside the fovea, the sub-hologram width $w$ is decreased [S1]. This effectively cuts high frequencies. The calculation speed scales quadratically with the sub-hologram size in point-wise integration methods [S1].
   - **Transition Band:** A band is established between the high-res and low-res zones [S2].
3. **Compute Image Data:**
   - Calculate pixel values for the high-resolution center using all object points [S1, S2].
   - Calculate pixel values for the low-resolution periphery using all object points but with the reduced width $w$ [S1].
4. **Blend:**
   - In the transition band, gradually adjust blending weights for high-res and low-res pixel values to avoid noticeable discontinuities [S2].
5. **Upscale and Composite:**
   - Upscale the low-resolution peripheral data to match the high-resolution center dimensions.
   - Composite the images using NVIDIA Performance Primitives (NPP) and a custom CUDA kernel [S2].

**Formulas:**
- The performance gain is derived from the quadratic scaling of calculation speed with sub-hologram size $w$ in point-wise integration [S1]. If $T$ is time and $w$ is width, $T \propto w^2$. Reducing $w$ outside the fovea reduces $T$ significantly.

**Parameters:**
- **Transition Band Width:** Not given in the sources.
- **Resolution Ratio:** The periphery is sampled at half resolution in the evaluation [S2].

**Edge Cases and Pitfalls:**
- **GPU Performance:** The method has not been found to run at high video rates on current GPUs [S1].
- **Discontinuities:** Abrupt changes in resolution can cause visible artifacts; a transition band is required [S2].
- **Temporal Aliasing:** Including all object points in the calculation helps reduce temporal aliasing compared to methods that remove points [S1].

## Operating characteristics
- **Time Complexity:** Scales quadratically with the sub-hologram size $w$ in the point-wise integration method [S1]. By reducing $w$ in the periphery, total computation time is reduced.
- **Memory Complexity:** Requires storing both high-res and low-res buffers plus blending weights. Not given in the sources.
- **Performance:** Evaluated on a Meta Quest 3 (2064×2208 pixels) with an NVIDIA RTX 4090 [S2]. However, it has not been found to run at high video rates on current GPUs [S1].
- **Strengths:** Reduces temporal aliasing by keeping all object points in the calculation [S1]. Maximizes culling efficiency in the center via a tight frustum [S2].
- **Limitations:** Does not achieve high video rates on current hardware [S1]. Requires eye-tracking hardware.

## Background
**Origin:**
The concept of reducing resolution outside the fovea in eye-tracked displays is attributed to Guenter et al. [2012] [S1].

**History:**
- Hong et al. [2016] performed foveated rendering by reducing the density of object points $o_i$ outside the fovea [S1].
- The method described here differs by including all object points but decreasing the sub-hologram width $w$ [S1].
- Recent implementations utilize 3D Gaussian Splatting frameworks, evaluating performance on VR headsets like the Meta Quest 3 [S2].

**Usage:**
Used in holographic near-eye displays for virtual and augmented reality [S1].

## Subtopics

### Foveated Volume Rendering
A variant that applies foveated resolution reduction to volume data. It differs from point-based foveated rendering by handling volumetric density rather than discrete points. Choose when rendering volumetric scenes with eye-tracking. See [Foveated Volume Rendering](link).

## Sources
[S1] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
[S2] Fast and Robust 3D Gaussian Splatting for Virtual Reality, https://doi.org/10.1145/3681756.3697947, fetched 2026-10-30
