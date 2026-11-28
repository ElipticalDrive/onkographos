# Relightable 3D Gaussian

Also called: Relightable 3D Gaussian Splatting

## Overview
Relightable 3D Gaussian is a method that extends 3D Gaussian Splatting to support dynamic lighting and physically-based materials by assigning per-Gaussian BRDF parameters and precomputed visibility for shadows [S1, unofficial]. It differs from standard 3D Gaussian Splatting by enabling relighting capabilities, setting it apart from siblings like [3D Gaussian Splatting with BRDF Embedding](1174-3d-gaussian-splatting-with-brdf-embedding.md) which may use different embedding strategies, and [Inverse Rendering with Gaussians (GS-IR / GIR)](6153-inverse-rendering-with-gaussians-gs-ir-gir.md) which focuses on inverse rendering with split-sum approximations [S1, unofficial].

## How it works
**Inputs:** A set of 3D Gaussians (position, covariance, opacity, color) and target lighting conditions (environment map or directional lights).
**Outputs:** Relit images with shadows and material reflections.

**Data Structures:**
Each Gaussian $i$ stores:
- Base attributes: Position $\mathbf{p}_i$, Covariance $\Sigma_i$, Opacity $\alpha_i$, Base Color $\mathbf{c}_i$.
- BRDF attributes: Albedo $\mathbf{k}_d$, Roughness $\rho$, Normal $\mathbf{n}$ [S1, unofficial].
- Visibility: Precomputed visibility factor $v_i$ for shadow casting [S1, unofficial].

**Steps:**
1. **Initialization:** Assign initial BRDF parameters ($\mathbf{k}_d, \rho, \mathbf{n}$) to each Gaussian. Initialize visibility $v_i$ based on scene geometry or default to 1.0 if no occlusion is considered [S1, unofficial].
2. **Visibility Precomputation:** Compute $v_i$ for each Gaussian to account for self-occlusion and shadows. This is typically done via ray casting or depth-based occlusion checks during training or preprocessing [S1, unofficial].
3. **Rendering:** For each pixel, splat the Gaussians in screen space. For each Gaussian, compute the BRDF contribution using the assigned parameters and the current lighting. Apply the visibility factor $v_i$ to modulate the contribution, effectively casting shadows [S1, unofficial].
4. **Compositing:** Accumulate the contributions of all Gaussians, respecting opacity and depth order, to produce the final pixel color [S1, unofficial].

**Formulas:**
The color contribution $C_i$ of a Gaussian $i$ is modulated by its visibility $v_i$:
$$ C_i = v_i \cdot \text{BRDF}(\mathbf{k}_d, \rho, \mathbf{n}, \mathbf{L}, \mathbf{V}) \cdot \alpha_i $$
where $\mathbf{L}$ is the light direction and $\mathbf{V}$ is the view direction [S1, unofficial].

**Parameters:**
- Albedo $\mathbf{k}_d$: Typically in $[0, 1]^3$.
- Roughness $\rho$: Typically in $[0, 1]$.
- Normal $\mathbf{n}$: Unit vector.
- Visibility $v_i$: Typically in $[0, 1]$.

**Edge Cases:**
- Degenerate Gaussians with zero opacity should be skipped.
- Precision issues in visibility computation may lead to artifacts; robust occlusion checks are recommended [G].

## Subtopics

### R3DG
R3DG is a variant that combines BRDF parameters with ambient occlusion baked per Gaussian [S1, unofficial]. It differs from the base method by explicitly baking ambient occlusion into the Gaussian attributes, potentially simplifying the lighting computation at the cost of flexibility for dynamic lighting. Choose R3DG when static ambient occlusion is sufficient and performance is critical [S1, unofficial].

## Operating characteristics
**Time Complexity:** Rendering time is similar to standard 3D Gaussian Splatting, with additional overhead for BRDF evaluation and visibility lookup [S1, unofficial].
**Memory Complexity:** Increased memory usage due to storage of BRDF parameters and visibility factors per Gaussian [S1, unofficial].
**Performance:** Specific measured performance data is not given in the sources.
**Strengths:** Enables dynamic relighting and realistic materials [S1, unofficial].
**Limitations:** Requires additional training or preprocessing for BRDF and visibility parameters; may suffer from artifacts if visibility is not accurately computed [G].
**When to Use:** When dynamic lighting and physically-based materials are required [S1, unofficial].
**When Not to Use:** For static lighting scenarios where standard 3D Gaussian Splatting is sufficient [S1, unofficial].

## Background
Relightable 3D Gaussian was introduced by Gao et al. in ECCV'24 [S1, unofficial]. It builds on the foundation of 3D Gaussian Splatting to address the need for dynamic lighting and material representation in Gaussian-based rendering [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
