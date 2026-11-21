# 4D Tetrahedron Slicing (Vertex Shader)

Also called: 4D Tetrahedron Clipping via Vertex Shader, CodeParade Method

## Overview
A GPU-based method to compute the 3D cross-section of a 4D tetrahedron against a slicing hyperplane using the Vertex Shader [S1, unofficial]. It outputs 0, 1, or 2 triangles representing the intersection [S1, unofficial]. It sets itself apart from its sibling [4D Tetrahedron Slicing (Geometry Shader)](19630-4d-tetrahedron-slicing-geometry-shader.md) by avoiding adjacency primitives and data replication, instead replicating vertex data four times to drive parallel computation in the Vertex Shader, at the cost of increased conditional branching [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A structure containing the four 4D vertices of a tetrahedron. To facilitate processing, the vertex data is replicated four times, with an appended index (0–3) identifying which vertex is currently being processed [S1, unofficial].
*   **Output:** 0, 1, or 2 triangles (0 to 6 vertices) representing the intersection polygon [S1, unofficial].

**Data Structures:**
The input to the Vertex Shader is a stream of vertices. Each entry in the stream corresponds to one of the four vertices of a specific tetrahedron. The structure typically includes:
*   `vec4 position`: The 4D coordinates of the vertex.
*   `int id`: An integer index (0, 1, 2, or 3) indicating the vertex's role within the tetrahedron [S1, unofficial].

**Steps:**
1.  **Hyperplane Evaluation:** For each vertex passed to the Vertex Shader, substitute its 4D coordinates into the slicing hyperplane equation $H(\mathbf{v}) = \mathbf{n} \cdot \mathbf{v} + d = 0$ to determine its signed distance or position relative to the plane [S1, unofficial].
2.  **Topology Determination:** Determine the intersection shape based on the signs of the four vertices. Since the Vertex Shader processes vertices individually, it must determine the global configuration (1 vs 3 or 2 vs 2) to know what to output. This is typically done by using a pre-computed lookup table (texture) indexed by the vertex ID and the configuration of signs, or by passing the configuration as a uniform/attribute [S1, unofficial].
    *   If all four vertices are on one side of the plane: Output nothing (empty set) [S1, unofficial].
    *   If the vertices are split 1 vs 3: Output a triangle [S1, unofficial].
    *   If the vertices are split 2 vs 2: Output a quadrilateral, which is decomposed into two triangles [S1, unofficial].
3.  **Intersection Calculation:** Calculate the exact 3D coordinates of the intersection points on the edges of the tetrahedron. For an edge connecting vertices $\mathbf{v}_i$ and $\mathbf{v}_j$ with signed distances $d_i$ and $d_j$, the intersection point $\mathbf{p}$ is found by linear interpolation:
    $$ \mathbf{p} = \frac{d_j \mathbf{v}_i - d_i \mathbf{v}_j}{d_j - d_i} $$
    The resulting 3D coordinates are derived by projecting or slicing the 4D point onto the 3D hyperplane [S1, unofficial].
4.  **Output Generation:** Emit the calculated vertices as triangles. The Vertex Shader must emit the correct number of vertices (3 for a triangle, 6 for two triangles) based on the topology determined in Step 2 [S1, unofficial].

**Pitfalls:**
*   **Branching:** The algorithm involves many conditional branches based on vertex signs to determine topology and output vertices. This hurts GPU parallel efficiency [S1, unofficial].
*   **Mitigation:** Use a pre-computed lookup table (texture) to determine the output topology and avoid conditional branches in the shader [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Input
struct VertexInput {
    vec4 position; // 4D vertex coordinates
    int id;        // Vertex ID (0-3)
};

// Uniforms
uniform vec4 planeNormal; // Normal of the slicing hyperplane
uniform float planeDist;  // Distance offset of the slicing hyperplane
uniform sampler2D topologyLookup; // Lookup table for topology based on vertex ID and configuration

// Helper to compute signed distance
float signedDistance(vec4 v) {
    return dot(planeNormal, v) + planeDist;
}

// Main Vertex Shader
void main() {
    vec4 v = input.position;
    int id = input.id;
    
    // Compute signed distance for this vertex
    float d = signedDistance(v);
    
    // Determine topology configuration
    // In a real implementation, the configuration (signs of all 4 vertices) 
    // would be passed or computed via a lookup based on the batch ID.
    // For this pseudocode, we assume a function getTopology(id, signs) exists.
    // The lookup table mitigates branching.
    
    // Example: If topology is 'triangle' (1 vs 3)
    if (topology == TRIANGLE) {
        // Calculate intersection points on edges
        // Edges connected to the single vertex on one side
        vec3 p1 = interpolateIntersection(v, neighbor1, d, neighbor1Dist);
        vec3 p2 = interpolateIntersection(v, neighbor2, d, neighbor2Dist);
        vec3 p3 = interpolateIntersection(v, neighbor3, d, neighbor3Dist);
        
        // Output triangle vertices
        emitVertex(p1);
        emitVertex(p2);
        emitVertex(p3);
    }
    // Example: If topology is 'quad' (2 vs 2)
    else if (topology == QUAD) {
        // Calculate intersection points on the 4 edges crossing the plane
        vec3 p1 = interpolateIntersection(v, neighbor1, d, neighbor1Dist);
        vec3 p2 = interpolateIntersection(v, neighbor2, d, neighbor2Dist);
        vec3 p3 = interpolateIntersection(v, neighbor3, d, neighbor3Dist);
        vec3 p4 = interpolateIntersection(v, neighbor4, d, neighbor4Dist);
        
        // Output two triangles
        emitVertex(p1);
        emitVertex(p2);
        emitVertex(p3);
        
        emitVertex(p1);
        emitVertex(p3);
        emitVertex(p4);
    }
    // If topology is 'empty', output nothing
}

vec3 interpolateIntersection(vec4 v1, vec4 v2, float d1, float d2) {
    // Linear interpolation to find intersection point
    float t = d1 / (d1 - d2);
    vec4 p = mix(v1, v2, t);
    // Project to 3D by removing the slicing dimension or using the hyperplane equation
    return projectTo3D(p);
}
```

## Subtopics

### 4D Rasterization Pipeline Adaptation
A technique to render 4D cross-sections on standard 3D GPUs by adapting the coordinate system [S1, unofficial]. It uses a 5D homogeneous coordinate system conceptually, but implements it using 4D vectors and a separate float to handle depth and perspective [S1, unofficial]. It is chosen when rendering on hardware that lacks native 5D support [S1, unofficial]. Steps: Transform 4D vertices to 5D homogeneous coordinates. Since `vec5` is unavailable, store the 5th component (depth) in a separate float variable and the first 4 components in a `vec4` [S1, unofficial]. If the slicing plane is perpendicular to a coordinate axis, discard the coordinate corresponding to that axis (as it is constant for the slice). Reduce the homogeneous vector to 4D [S1, unofficial]. Pass the resulting 4D vector to the Vertex Shader. The first 3 components are treated as 3D screen position, the 4th component is used for depth testing (Z-buffer), and the 4th component is also used for perspective division (w-component) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra, $O(N)$, where $N$ is the number of tetrahedra [S1, unofficial].
*   **Memory Complexity:** Low, as it does not require storing adjacency information or complex data structures beyond the vertex buffer [S1, unofficial].
*   **Performance:** Efficient on GPUs with strong vertex processing capabilities, but may suffer from branch divergence due to the conditional logic required for topology determination [S1, unofficial].
*   **Strengths:** Simple implementation, no need for Geometry Shader support, compatible with WebGL and WebGPU which lack Geometry Shader support [S1, unofficial].
*   **Limitations:** Data replication (4x vertex data) increases memory bandwidth usage. Branching can reduce parallel efficiency [S1, unofficial].
*   **When to use:** When targeting platforms without Geometry Shader support (e.g., WebGL, WebGPU) or when simplicity of implementation is prioritized over memory efficiency [S1, unofficial].
*   **When not to use:** When memory bandwidth is a critical bottleneck or when the scene complexity makes data replication prohibitive [S1, unofficial].

## Background
Originated from the work of YouTuber CodeParade in the game "4D Golf" [S1, unofficial]. It is part of a broader effort in 4D computer graphics to render 4D objects by slicing them into 3D cross-sections [S1, unofficial]. The method was developed to overcome the limitations of CPU-based slicing and the lack of native 4D rendering support in standard 3D GPUs [S1, unofficial].

## Sources
[S1] 四维计算机图形学：渲染篇, https://wxyhly.github.io/archives/cg4d/, fetched 2026-10-30 (unofficial)
