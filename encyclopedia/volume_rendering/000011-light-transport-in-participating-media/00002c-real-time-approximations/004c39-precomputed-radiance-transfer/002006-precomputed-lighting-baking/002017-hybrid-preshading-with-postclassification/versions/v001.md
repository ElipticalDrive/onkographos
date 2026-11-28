# Hybrid Preshading with Postclassification

Also called: Hybrid Classification and Shading

## Overview
Hybrid Preshading with Postclassification is a real-time volume rendering optimization that decouples classification from shading to balance visual fidelity and performance [S1]. Unlike pure preclassification/preshading which risks artifacts without opacity-weighted interpolation, or pure postclassification/postshading which is computationally expensive, this method performs classification after interpolation (postclassification) but shading before interpolation (preshading) [S1]. It builds on [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md) by precomputing lighting contributions for a discrete set of surface normals, allowing the renderer to apply shading via texture lookup during the raymarching step while retaining the visual benefits of post-classification [S1].

## How it works
**Inputs:**
- Volume dataset (scalar field).
- Transfer function (mapping scalar values to color and opacity).
- Lighting model (e.g., Phong) with light positions and material properties.
- Number of quantized space directions ($N_{dir}$), typically 8192 [S1].

**Outputs:**
- Rendered image with shaded volume data.

**Data Structures:**
1. **Volume Grid:** Standard 3D array of scalar values.
2. **Gradient Grid:** 3D array of 3D vectors ($\nabla V$), precomputed from the volume grid.
3. **Preshaded Reflectance Map (PRM):** A 3D texture or lookup table indexed by:
   - Quantized direction index ($d \in [0, N_{dir}-1]$).
   - Scalar value (or classification result).
   - Stores the precomputed color contribution (reflectance) for that direction and value.
4. **Direction Index Grid:** 3D array storing an integer index ($0 \dots N_{dir}-1$) for each voxel, corresponding to the quantized space direction of the precomputed gradient at that location.

**Steps:**

1. **Precompute Gradients:**
   Calculate the volume gradient $\nabla V(x,y,z)$ for every voxel in the volume grid using finite differences or central differences [S1].

2. **Quantize Directions:**
   Define a set of $N_{dir}$ unit vectors representing space directions. A common choice is $N_{dir} = 8192$ [S1]. Map each precomputed gradient vector to the nearest index in this set. Store this index in the Direction Index Grid for each voxel [S1].

3. **Generate Preshaded Reflectance Map:**
   For each quantized direction $d_i$ in the set:
   a. Iterate through all possible scalar values (or classification bins) $v$.
   b. Compute the shading color $C_{shade}$ for direction $d_i$ and value $v$ using the chosen lighting model (e.g., Phong) and transfer function [S1].
   c. Store $C_{shade}$ in the PRM at index $(d_i, v)$ [S1].
   *Note: This step is performed once before rendering [S1].*

4. **Raymarching with Hybrid Shading:**
   For each pixel, cast a ray through the volume. For each sampling step along the ray:
   a. **Interpolation:** Trilinearly interpolate the scalar value $v$ and the Direction Index $idx$ from the 8 surrounding voxels of the current sampling location [S1].
   b. **Classification:** Use the interpolated scalar value $v$ to determine the optical properties (opacity $\alpha$, base color $C_{base}$) via the transfer function [S1]. This is the "postclassification" step.
   c. **Shading Lookup:** Use the interpolated Direction Index $idx$ to look up the precomputed shading color $C_{shade}$ from the PRM, indexed by $idx$ and $v$ (or the classification result) [S1].
   d. **Composite:** Combine $C_{base}$ and $C_{shade}$ to get the final sample color $C_{sample}$ [S1].
   e. **Accumulate:** Perform standard volume compositing (e.g., front-to-back) using $C_{sample}$ and the interpolated opacity $\alpha$ [S1].

**Formulas:**
- Gradient: $\nabla V = \left[ \frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z} \right]$ [S1].
- Direction Index: $idx = \arg \min_{j} \| \frac{\nabla V}{\|\nabla V\|} - \vec{d}_j \|$ where $\vec{d}_j$ is the $j$-th quantized direction [S1].
- Shading Lookup: $C_{shade} = \text{PRM}[idx, v]$ [S1].
- Sample Color: $C_{sample} = \text{Combine}(C_{base}(v), C_{shade})$ [S1].

**Parameters:**
- $N_{dir} = 8192$ [S1].
- Sampling rate $\Delta s = 0.5$ (distance between interpolations), allowing 2-4 samplings per cell [S1].
- Image resolution: $512 \times 512$ pixels [S1].
- Pixel subdivisions and precomputed rays: $16 \times 16$ [S1].

**Edge Cases and Pitfalls:**
- **Gradient Zero:** If $\nabla V = 0$, the direction index is undefined. [G] Assign a default direction (e.g., $(0,0,1)$) or handle as a special case in the quantization step.
- **Quantization Artifacts:** Discrete directions may cause directional aliasing. [S1] The large number of directions (8192) mitigates this.
- **Artifact Claim:** Contrary to Mueller et al. [24] who reported artifacts with preshading/postclassification, this implementation shows no visible artifacts [S1].

## Subtopics

### [Fast Two-Pass Shading](2311-fast-two-pass-shading.md)
Decouples classification/illumination from geometric filtering via a P-buffer, improving performance by 5-10%.

### [Congradient Shading](2341-congradient-shading.md)
Integrates shading directly into the projection pipeline using local gradients, combining geometric and contextual shading.

### [Fake Phong Illumination via Normal Quantization](5846-fake-phong-illumination-via-normal-quantization.md)
Replaces per-voxel Phong calculations with a pre-computed color lookup table indexed by quantized normals.

## Operating characteristics
- **Time Complexity:** Precomputation is $O(N_{voxels} + N_{dir} \cdot N_{values})$. Rendering is $O(N_{pixels} \cdot N_{steps})$ with constant-time shading lookup.
- **Memory Complexity:** Requires storage for Volume Grid, Gradient Grid, Direction Index Grid, and PRM. PRM size is $N_{dir} \times N_{values} \times 3$ floats.
- **Performance:** High efficiency for interactive rendering. Benchmarks performed on MMX/SSE/3DNow! SIMD processors [S1].
- **Strengths:** Avoids artifacts associated with pure preclassification/preshading (without opacity-weighted interpolation) and is faster than pure postclassification/postshading [S1].
- **Limitations:** Requires precomputation of gradients and PRM. Directional aliasing possible if $N_{dir}$ is too small [S1].

## Background
This method is described in "A new object-order ray-casting algorithm" [S1]. It addresses the trade-off between the speed of preclassification/preshading and the visual quality of postclassification/postshading [S1]. The authors argue against the artifact claims of Mueller et al. [24] for this specific implementation [S1].

## Sources
[S1] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
