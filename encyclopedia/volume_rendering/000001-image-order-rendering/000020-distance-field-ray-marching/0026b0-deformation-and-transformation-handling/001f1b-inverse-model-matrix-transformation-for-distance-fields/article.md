# Inverse Model Matrix Transformation for Distance Fields

Also called: Inverse Transformation for Rotation/Translation, SDF Model Transformations, Transforming SDFs, Transforming SDFs via Inverse Matrix

## Overview
This method enables the application of rigid transformations (rotation, translation, and uniform scaling) to Signed Distance Field (SDF) primitives during ray marching by evaluating the base distance function in the primitive's local coordinate space rather than the world space [S1, S2, unofficial]. It solves the problem of animating or positioning individual scene objects without modifying the underlying geometric definitions or voxel grids [S1, S2, unofficial]. Unlike methods that transform the geometry data itself, this approach transforms the query point using the inverse of the object's model matrix, ensuring the distance metric remains valid for Euclidean distance fields [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{p}$ in world/camera space; a $4 \times 4$ inverse model matrix $M^{-1}$ representing the object's transformation (Translation, Rotation, Scale) [S2, unofficial].
*   **Output:** The signed distance value $d$ at point $\mathbf{p}$ for the transformed object [S1, S2, unofficial].

**Data Structures:**
*   **Inverse Model Matrix ($M^{-1}$):** A $4 \times 4$ homogeneous transformation matrix. In implementation, this is typically passed as a uniform variable to the shader or kernel [S2, unofficial]. The matrix encodes rotation, translation, and potentially uniform scaling [S2, unofficial].

**Steps:**
1.  **Construct Model Matrix:** On the host side (e.g., CPU/scripting environment), construct the model matrix $M$ for the object using its current translation $\mathbf{t}$, rotation quaternion $\mathbf{q}$, and scale $s$ [S2, unofficial].
2.  **Compute Inverse:** Calculate the inverse of the model matrix $M^{-1}$ [S2, unofficial].
3.  **Transform Point:** In the distance evaluation function, transform the world-space query point $\mathbf{p}$ into the local space $\mathbf{q}$ by multiplying $\mathbf{p}$ (homogenized to $\mathbf{p}' = [\mathbf{p}, 1]^T$) by $M^{-1}$ [S2, unofficial]:
    $$ \mathbf{q} = M^{-1} \begin{bmatrix} \mathbf{p} \\ 1 \end{bmatrix} $$
    The resulting local point is $\mathbf{q}_{xyz}$ [S2, unofficial].
4.  **Evaluate Base SDF:** Pass $\mathbf{q}_{xyz}$ to the base, untransformed SDF function $f_{base}$ [S1, S2, unofficial]:
    $$ d = f_{base}(\mathbf{q}_{xyz}) $$
5.  **Handle Scaling:** If the model matrix includes uniform scaling by factor $s$, the distance value must be scaled to preserve the Euclidean distance property [S1, unofficial]:
    $$ d_{final} = s \cdot d $$
    If non-uniform scaling is used, the Euclidean distance property is distorted unless the metric is adjusted, which is generally avoided in standard SDF ray marching [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Host-side (e.g., C# / Python)
def compute_inverse_model_matrix(translation, rotation, scale):
    """
    Constructs the model matrix M and returns its inverse.
    Assumes uniform scale for valid SDF distance preservation.
    """
    # Construct M = T * R * S
    # Note: Order of operations depends on convention (pre/post-multiply).
    # Standard Unity/OpenGL: M = Translation * Rotation * Scale
    M = translate(translation) @ rotate(rotation) @ scale_matrix(scale)
    
    # Compute inverse
    M_inv = inverse(M)
    return M_inv

# Shader-side (e.g., GLSL / HLSL / Shadertoy)
uniform float4x4 _MatTorus_InvModel; # Passed from host

float sdTorus(vec3 p, float2 torus_params) {
    // Base SDF implementation for a torus
    vec2 q = vec2(length(p.xz) - torus_params.x, p.y);
    return length(q) - torus_params.y;
}

float map(float3 p) {
    // Step 3: Transform point to local space
    // Homogeneous coordinate w=1 for translation/rotation
    float4 q_homogeneous = mul(_MatTorus_InvModel, float4(p, 1.0));
    vec3 q_local = q_homogeneous.xyz;
    
    // Step 4: Evaluate base SDF
    float dist = sdTorus(q_local, float2(1.0, 0.2));
    
    // Step 5: Scale distance if uniform scale was applied in M
    // Note: If M included scale 's', M_inv includes 1/s.
    // The distance in local space is correct, but must be scaled back to world space metric.
    // However, if M_inv is pre-computed, the scaling factor 's' must be known separately
    // or extracted from M_inv to scale the final distance value.
    // For uniform scale s, d_world = s * d_local.
    float scale_factor = ...; // Extracted from original M or passed separately
    return dist * scale_factor;
}
```

**Parameters:**
*   `_MatTorus_InvModel` (or generic `M_inv`): A uniform $4 \times 4$ matrix representing the inverse model transformation [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Uniform Scaling:** Applying non-uniform scaling distorts the distance field, breaking the Euclidean distance property. The resulting field is no longer a true SDF, leading to incorrect ray marching steps [S1, unofficial].
*   **Precision:** Computing the inverse matrix on the host side is preferred for performance and precision stability compared to inverting in the shader per-fragment [S2, unofficial].
*   **Homogeneous Coordinates:** Ensure the point is treated as a position (w=1) when multiplying by the transformation matrix, not a direction (w=0), to correctly apply translation [S2, unofficial].

## Subtopics

### [Inverse Viewing Transformations for Volumes](2390-inverse-viewing-transformations-for-volumes.md)
Applies the inverse transform to the entire viewing ray system (camera, lights) rather than individual sample points, reducing complexity for large volumetric datasets.

### [Motion-Compensated Block Warping](4613-motion-compensated-block-warping.md)
Estimates rigid transforms per-block for temporal compression of TSDFs, minimizing prediction error between frames.

### [Mesh Transformation via Embree RTC](5669-mesh-transformation-via-embree-rtc.md)
Uses Embree's ray tracing kernel to handle mesh transformations, distinct from SDF evaluation.

### [Incremental Scaling Algorithm](7792-incremental-scaling-algorithm.md)
Handles scaling in distance fields through incremental adjustments.

### [Per-fragment Deformation Evaluation](9846-per-fragment-deformation-evaluation.md)
Evaluates deformations directly on fragments, potentially using vertex data.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point evaluation, as it involves a single matrix-vector multiplication and one base SDF call [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory for the inverse matrix uniform [S2, unofficial].
*   **Performance:** Highly efficient for rigid transformations. The cost is dominated by the base SDF evaluation and matrix multiplication, which are negligible compared to the ray marching loop [S1, S2, unofficial].
*   **Strengths:** Simple to implement; allows independent animation of scene parts; preserves the static nature of base SDF definitions [S1, S2, unofficial].
*   **Limitations:** Only supports rigid transformations (rotation, translation) and uniform scaling effectively. Non-rigid deformations require different approaches (e.g., [Per-fragment Deformation Evaluation](9846-per-fragment-deformation-evaluation.md)) [S1, S2, unofficial].
*   **When to use:** Ideal for animating rigid objects in SDF scenes [S1, S2, unofficial].
*   **When not to use:** For non-rigid deformations or non-uniform scaling where metric preservation is critical [S1, unofficial].

## Background
The technique originates from the foundational principles of ray marching and Signed Distance Functions, popularized by tutorials such as those by Jamie Wong and Adrian Booth [S1, S2, unofficial]. It is a standard practice in real-time graphics and procedural modeling to decouple geometry definition from scene layout [S1, S2, unofficial].

## Sources
[S1] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
