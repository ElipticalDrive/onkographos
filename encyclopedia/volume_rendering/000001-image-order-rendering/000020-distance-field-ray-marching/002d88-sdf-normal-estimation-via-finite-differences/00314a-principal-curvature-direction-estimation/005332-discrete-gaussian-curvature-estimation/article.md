# Discrete Gaussian Curvature Estimation

Also called: Discrete Gaussian Curvature Classification, Gauss operator, Gaussian curvature operator, Local Extremum Detection

## Overview
This method classifies vertices on a triangular mesh as elliptic (local extrema) or hyperbolic (saddle/blending regions) by evaluating the sign of the discrete Gaussian curvature. It distinguishes itself from continuous curvature estimation by providing a robust Boolean classification based on the angular defect, which is computationally cheaper and less sensitive to noise than second-derivative approximations. It builds on [Principal Curvature Direction Estimation](6492-principal-curvature-direction-estimation.md) by offering a topological classification rather than directional magnitude.

## How it works
**Inputs and Outputs:**
- Input: A triangular mesh defined by vertices $V$ and faces $F$. Specifically, for a target vertex $x_i$, the 1-ring neighborhood (adjacent vertices and faces).
- Output: A Boolean classification `Gauss(x_i)` indicating if the vertex is elliptic or hyperbolic.

**Data Structures:**
- Vertex $x_i$: The center of the local stencil.
- Neighbors $x_j$: Vertices adjacent to $x_i$.
- Faces: Triangles sharing $x_i$ as a vertex.

**Steps:**
1. Identify all faces $f_k$ that contain the vertex $x_i$. Let $N$ be the number of such faces.
2. For each face $f_k$, identify the two edges connected to $x_i$. Let the other two vertices of this face be $x_{j}$ and $x_{j-1}$ (ordered cyclically around $x_i$).
3. Compute the angle $\theta_j$ at vertex $x_i$ within face $f_k$. This is the angle between the vector $\vec{v}_1 = x_j - x_i$ and the vector $\vec{v}_2 = x_{j-1} - x_i$.
   - Formula: $\theta_j = \arccos\left(\frac{\vec{v}_1 \cdot \vec{v}_2}{\|\vec{v}_1\| \|\vec{v}_2\|}\right)$.
4. Sum all inner angles $\theta_j$ for all $N$ faces surrounding $x_i$:
   $$ S = \sum_{j=1}^{N} \theta_j $$
5. Compare the sum $S$ to $2\pi$. The discrete Gaussian curvature sign operator $\kappa_G^{sign}(x_i)$ is defined as:
   $$ \kappa_G^{sign}(x_i) = \text{sign}\left( \sum_{j=1}^{N} \theta_j - 2\pi \right) $$
6. Classify the vertex:
   - If $S < 2\pi$, the vertex is **elliptic** (local extremum).
   - If $S > 2\pi$, the vertex is **hyperbolic** (saddle/blending region).
   - If $S = 2\pi$, the vertex is planar (neither strictly elliptic nor hyperbolic in this binary scheme).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_gaussian_curvature(vertex_i, mesh):
    """
    Classifies a vertex as elliptic or hyperbolic based on discrete Gaussian curvature.
    
    Args:
        vertex_i: The index or ID of the central vertex.
        mesh: A mesh object providing access to vertices and faces.
        
    Returns:
        str: 'elliptic', 'hyperbolic', or 'planar'.
    """
    # Get all faces incident to vertex_i
    incident_faces = mesh.get_faces_containing_vertex(vertex_i)
    
    # Get all unique neighbors of vertex_i
    neighbors = mesh.get_neighbors(vertex_i)
    
    # Map neighbor index to its position in the cyclic order around vertex_i
    # This requires sorting neighbors by angle around the normal or using face connectivity
    # For simplicity, we iterate through faces and extract angles
    
    total_angle_sum = 0.0
    
    for face in incident_faces:
        # Get the three vertices of the face
        v0, v1, v2 = face.vertices
        
        # Identify which vertex is the center (vertex_i)
        # The other two are the neighbors in this face
        if v0 == vertex_i:
            p1, p2 = v1, v2
        elif v1 == vertex_i:
            p1, p2 = v2, v0
        else: # v2 == vertex_i
            p1, p2 = v0, v1
            
        # Compute vectors from center to neighbors
        vec1 = p1 - vertex_i
        vec2 = p2 - vertex_i
        
        # Compute angle between vectors
        dot_product = vec1.dot(vec2)
        norm1 = vec1.length()
        norm2 = vec2.length()
        
        # Guard against division by zero or invalid geometry
        if norm1 == 0 or norm2 == 0:
            continue
            
        # Clamp cosine value to [-1, 1] for numerical stability
        cos_angle = dot_product / (norm1 * norm2)
        cos_angle = max(-1.0, min(1.0, cos_angle))
        
        angle = math.acos(cos_angle)
        total_angle_sum += angle
        
    # Compare sum to 2*pi
    if total_angle_sum < 2 * math.pi:
        return 'elliptic'
    elif total_angle_sum > 2 * math.pi:
        return 'hyperbolic'
    else:
        return 'planar'
```

**Parameters:**
- None. The threshold is fixed at $2\pi$.

**Edge Cases and Pitfalls:**
- **Planar Regions:** If the vertex lies on a flat plane, the sum of angles equals $2\pi$ exactly. The method returns "planar" or is undefined in a strict binary classification. The method relies on the sign of the deviation, ignoring the magnitude [S1].
- **Precision:** Floating-point errors may cause planar vertices to be misclassified. A tolerance epsilon should be used in implementation [G].
- **Boundary Vertices:** Vertices on the boundary of a mesh do not have a closed 1-ring. The sum of angles will be less than $2\pi$. This method assumes an interior vertex or requires special handling for boundaries [G].

## Subtopics

### Surface Type Classification
This is the primary application of the operator, distinguishing between sulci (local minima, elliptic) and gyri (local maxima, elliptic) versus saddle points (hyperbolic). Elliptic regions correspond to local extrema where all neighbors lie on one side of the tangent plane. Hyperbolic regions correspond to saddle points where neighbors lie above and below the best-fitting least-squares plane [S1].

## Operating characteristics
**Time Complexity:** $O(d)$, where $d$ is the degree of the vertex (number of adjacent faces). This is very fast per vertex.
**Memory Complexity:** $O(1)$ auxiliary space per vertex evaluation, assuming the mesh connectivity is accessible.
**Strengths:**
- Robust to noise compared to derivative-based curvature estimates.
- Computationally inexpensive.
- Provides a clear topological classification (extremum vs. blending).
**Limitations:**
- Binary classification loses magnitude information of the curvature.
- Fails to distinguish between minima and maxima without additional gradient information.
- Sensitive to mesh quality (degenerate triangles).
**When to use:** When topological segmentation of surfaces (e.g., brain sulci/gyri) is required and computational efficiency is critical.
**When not to use:** When precise curvature magnitude is needed for shading or physics simulation.

## Background
The method originates from the work of Vivodtzev, Linsen, Bonneau, Hamann, Joy, and Olshausen in their paper "Hierarchical Isosurface Segmentation Based on Discrete Curvature" (2003) [S1]. It was developed to segment isosurfaces into topologically meaningful regions (sulci and gyri) for medical image analysis, specifically for recognizing diseases in brain structures. The approach uses the angular defect as a discrete approximation of Gaussian curvature to classify surface types.

## Sources
[S1] Hierarchical Isosurface Segmentation Based on Discrete Curvature, https://inria.hal.science/inria-00537470v1, fetched 2026-10-26
