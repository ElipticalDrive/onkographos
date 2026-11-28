# Single-pass vertex-based multi-view PBR

Also called: GPU VS

## Overview
A single-pass rendering technique for multi-view Point-Based Rendering (PBR) that computes parametric coordinates for all target views within the vertex shader and performs a minimal intersection test in the fragment shader. It solves the problem of generating multiple spatial views (e.g., for auto-stereoscopic displays) in a single geometry pass, avoiding the overhead of multiple render-to-texture cycles. It differs from the fragment-based approach [Single-pass vertex-based multi-view PBR](23102-single-pass-vertex-based-multi-view-pbr.md) by shifting the computational burden of ray-splat intersection to the vertex stage, resulting in a conceptually elegant but performance-limited implementation due to hardware architecture constraints.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ points (splat centers and radii) and $V$ target views (e.g., $V=8$ for auto-stereoscopic displays).
*   **Output:** A single frame buffer containing the composite spatial image, or intermediate textures for the $V$ views if compositing is deferred.

**Data Structures:**
*   Point attributes: Position $(x, y, z)$, radius $r$, color/intensity.
*   View parameters: Camera matrices for each of the $V$ views.

**Steps:**
1.  **Vertex Shader Execution:** For each point primitive:
    *   Iterate through all $V$ views.
    *   Transform the point center and calculate the projected bounding box or parametric coordinates for each view.
    *   Compute the ray-splat intersection parameters for all $V$ views.
    *   Store the results in a uniform array or varyings accessible to the fragment shader. Since variable array indices are not supported in older shader models (e.g., OpenGL 2.0 era hardware like GeForce 7800GTX), these indices must be resolved at compile time, requiring unrolled loops or extensive branching logic [S1].
2.  **Rasterization:** The point is rasterized as a primitive (e.g., point sprite or quad) covering the screen area.
3.  **Fragment Shader Execution:**
    *   For each fragment, receive the interpolated data from the vertex shader.
    *   Perform a simple test to determine which view's contribution is valid or required for the current pixel.
    *   Retrieve the pre-computed intersection result for the specific view.
    *   Accumulate color/depth based on the splat kernel.

**Formulas:**
Let $P_i$ be the $i$-th point. For each view $v \in \{1, \dots, V\}$:
1.  Project point center: $C_v = M_v \cdot P_i$, where $M_v$ is the view-projection matrix for view $v$.
2.  Compute parametric coordinates or intersection distance $d_v$.
3.  The fragment shader selects the appropriate $d_v$ based on the fragment's position relative to the splat's bounding box in screen space.

**Parameters:**
*   $V$: Number of views. Typically 8 for auto-stereoscopic displays [S1].

**Edge Cases and Pitfalls:**
*   **Hardware Limitations:** The method relies on the ability to handle large uniform arrays or branching. On hardware with limited vertex shader resources or no dynamic indexing, performance degrades significantly [S1].
*   **Precision:** Calculating all views in the vertex shader may introduce precision issues if the view matrices vary significantly in scale or orientation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
void main() {
    // Pre-compute intersection data for all V views
    for (int v = 0; v < NUM_VIEWS; v++) {
        // Transform point to view space for view v
        vec4 viewPos = viewMatrices[v] * vec4(pointPosition, 1.0);
        
        // Calculate parametric coordinates or intersection distance
        // This involves solving the ray-splat intersection for the current view
        float dist = computeRaySplatIntersection(viewPos, pointRadius);
        
        // Store in a uniform array or varyings
        // Note: Dynamic indexing is not supported, so this loop must be unrolled
        // or handled via extensive branching in the fragment shader
        viewDistances[v] = dist;
    }
    
    // Pass standard position to fragment shader
    gl_Position = projectionMatrix * viewMatrices[0] * vec4(pointPosition, 1.0);
}

# Fragment Shader
void main() {
    // Determine which view corresponds to this fragment
    // This requires a simple test based on fragment coordinates
    int activeView = determineActiveView(gl_FragCoord.xy);
    
    // Retrieve pre-computed distance for the active view
    float dist = viewDistances[activeView];
    
    // Perform simple intersection test
    if (dist < pointRadius) {
        // Compute color contribution
        vec3 color = splatKernel(dist, pointRadius) * pointColor;
        gl_FragColor = vec4(color, 1.0);
    } else {
        discard;
    }
}
```

## Subtopics

### [Multi-view Point Splatting with 1+1 Pass PBR](23120-multi-view-point-splatting-with-1-1-pass-pbr.md)
Extends this method to support auto-stereoscopic displays by grouping points via graph coloring to reduce overdraw and using enlarged image-aligned quads. It differs by requiring a second pass for compositing according to a wavelength-selective filter mask, whereas this method attempts to do it in a single pass.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$ in the vertex shader, where $N$ is the number of points and $V$ is the number of views. The fragment shader work is $O(1)$ per fragment.
*   **Memory Complexity:** $O(V)$ per vertex for storing view-specific data.
*   **Performance:** Significantly slower than the fragment-based approach [Single-pass vertex-based multi-view PBR](23102-single-pass-vertex-based-multi-view-pbr.md). On hardware like the GeForce 7800GTX, it is at least twice as slow as the fragment-based solution [S1].
*   **Strengths:** Conceptually elegant; reduces the number of rendering passes to one.
*   **Limitations:** Poor performance on older hardware due to limited vertex shader resources (e.g., 8 vs 24 fragment shaders) and the inability to use variable array indices, leading to extensive branching and compile-time resolution overhead [S1].
*   **When to use:** When rendering passes are strictly limited to one, and hardware supports efficient vertex shader execution.
*   **When not to use:** When performance is critical and the hardware favors fragment processing or lacks dynamic indexing support.

## Background
Originated in the context of multi-view point-based rendering for auto-stereoscopic displays. The method was proposed as an alternative to multi-pass rendering and fragment-based single-pass approaches. It was evaluated on hardware such as the GeForce 7800GTX [S1].

## Sources
[S1] Multi-view point splatting, https://doi.org/10.1145/1174429.1174479, fetched 2026-11-03
