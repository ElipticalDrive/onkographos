# Rasterize-then-splat (RtS)

## Overview
Rasterize-then-splat (RtS) is a method for differentiable rendering of 3D surfaces that supports both explicit and implicit representations, providing derivatives at occlusion boundaries while remaining fast and simple to implement [S1]. It differs from alternatives like Soft Rasterizer by avoiding costly closest-point queries or mesh processing, and from Nvdiffrast by requiring no specific GPU hardware or custom derivative code, making its differentiable cost independent of mesh size [S1].

## How it works
The method consists of three sequential steps: rasterization of the surface, shading the surface samples, and multi-layer splatting [S1]. All derivatives are produced using automatic differentiation on the forward rendering computation [S1].

**Inputs and Outputs**
*   **Input:** Scene parameters $\theta$ (e.g., camera pose, light parameters, surface definition parameters).
*   **Output:** Final rendered image $S$.

**Data Structures**
1.  **Parameter Buffers ($U_k$):** Per-layer screen-space parameter buffers. For triangle meshes, these consist of:
    *   $T_k$: Per-pixel triangle indices.
    *   $B_k$: Per-pixel perspective-correct barycentric coordinates of the pixel inside the corresponding triangle.
    *   These are computed via conventional Z-buffer graphics processing using depth peeling to retrieve multiple intersections per pixel [S1].
2.  **G-buffers ($G_k$):** Interpolated shape attributes (positions, normals) derived from $U_k$.
3.  **Splat Colors ($C_k$):** Shaded colors computed via a deferred shading function.
4.  **Screen-space Positions ($P_k$):** Projected positions of the splats.

**Steps**
1.  **Sampling (Non-differentiable):** Scene parameters $\theta$ are passed through a sampling function $U$, yielding per-layer screen-space parameter buffers $U_k$. This step is non-differentiable and computed only in the forward pass [S1].
2.  **Evaluation (Differentiable):** An evaluation function $G$ uses the non-differentiable parameters $U_k$ to interpolate shape attributes (such as positions and normals) into G-buffers $G_k$. Automatic differentiation is applied to $G$ and downstream functions [S1].
3.  **Shading and Projection (Differentiable):** The evaluated attributes are combined to compute splat colors $C_k$ via a deferred shading function $\mathcal{C}$ and corresponding screen-space positions $P_k$ via a projection function $\mathcal{P}$ [S1].
4.  **Splatting (Differentiable):** Function $\mathcal{S}$ splats the shaded colors $C_k$ at the corresponding pixel locations $P_k$ using a 3x3 kernel to produce the final result $S$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_then_splat(scene_params, camera, surface_def):
    """
    Implements the RtS method.
    scene_params: Parameters theta (e.g., lights, material props).
    camera: Camera intrinsics and extrinsics.
    surface_def: Explicit mesh or implicit function definition.
    """
    
    # Step 1: Rasterization (Non-differentiable sampling)
    # Use conventional Z-buffer with depth peeling to get multiple layers.
    # For each layer k:
    #   U_k = SamplingFunction(scene_params, camera, surface_def)
    #   Returns:
    #     T_k: Per-pixel triangle indices (or implicit surface hit IDs)
    #     B_k: Per-pixel perspective-correct barycentric coordinates
    
    # Step 2: Evaluation (Differentiable interpolation)
    # G_k = EvaluationFunction(U_k)
    # Interpolate shape attributes (positions, normals) from surface_def
    # using T_k and B_k.
    # Let G_k contain:
    #   Pos_k: World-space positions of the hit points
    #   Norm_k: World-space normals at the hit points
    
    # Step 3: Shading and Projection (Differentiable)
    # Compute screen-space positions P_k
    P_k = Project(Pos_k, camera)
    
    # Compute splat colors C_k via deferred shading
    # C_k = ShadingFunction(scene_params, Norm_k, Pos_k, ...)
    
    # Step 4: Multi-layer Splatting (Differentiable)
    # S = SplattingFunction(C_k, P_k)
    # Splat colors using a 3x3 kernel at positions P_k.
    # Accumulate contributions to the final image S.
    
    return S
```

**Parameters**
*   **Splatting Kernel Size:** 3x3 [S1].
*   **Depth Peeling:** Used to retrieve multiple intersections per pixel for multi-layer splatting [S1].

**Edge Cases and Pitfalls**
*   **Occlusion Boundaries:** RtS provides derivatives at occlusion boundaries, a feature not always present in standard rasterization [S1].
*   **Implicit Surfaces:** The method supports isosurfaces extracted from implicit surface definitions (e.g., neural radiance fields), avoiding issues with under- or over-sampling of splats by resampling at each frame [S1].

## Operating characteristics
*   **Time Complexity:** The cost of the differentiable sections is independent of the size of the mesh [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Fast and simple to implement; requires no differentiable meshing or rasterization steps; efficient for large 3D models; applicable to isosurfaces from implicit definitions; no custom derivative code needed [S1].
*   **Limitations:** Relies on non-differentiable rasterization for sampling, which may limit gradient quality in specific discontinuous scenarios compared to fully differentiable approaches, though it handles occlusion boundaries well [S1].
*   **When to use:** When rendering large explicit meshes or implicit surfaces (like NeRF isosurfaces) where implementation simplicity and hardware independence are prioritized over the specific optimizations of Nvdiffrast [S1].
*   **When not to use:** When specific GPU hardware optimizations (like those in Nvdiffrast) are required for maximum performance, or when the scene geometry is too sparse for effective splatting [S1].

## Background
RtS was introduced by Zhoutong Zhang in the paper "Differentiable Surface Rendering via Non-Differentiable Sampling" [S1]. It was presented at ICCV 2021 [S1]. The method was demonstrated for implicit-, mesh-, and parametric-surface-based inverse rendering and neural-network training applications, including the first efficient, differentiable rendering of an isosurface extracted from a neural radiance field (NeRF) [S1]. It builds on surface splatting techniques [42] and differentiable visibility methods [34], but distinguishes itself by using a true surface representation and resampling splats per frame [S1].

## Sources
[S1] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
