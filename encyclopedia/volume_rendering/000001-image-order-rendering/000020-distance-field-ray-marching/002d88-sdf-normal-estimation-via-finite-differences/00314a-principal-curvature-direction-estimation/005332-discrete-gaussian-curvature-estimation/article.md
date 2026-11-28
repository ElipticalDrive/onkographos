# Discrete Gaussian Curvature Estimation

Also called: Discrete Gaussian Curvature Classification, Gauss operator, Gaussian curvature operator, Local Extremum Detection

## Overview
This method classifies surface vertices as elliptic (local extrema) or hyperbolic (saddle/blending regions) by evaluating the sign of the discrete Gaussian curvature [S1].

## How it works
**Inputs and Outputs:**
- Input: A vertex $x_i$ on a triangular mesh and its 1-ring neighborhood (adjacent faces and vertices) [S1].
- Output: A Boolean classification `Gauss(x_i)` indicating elliptic or hyperbolic type [S1].

**Data Structures:**
- Vertex $x_i$: The center of the local stencil [S1].
- Neighbors $x_j$: Vertices adjacent to $x_i$ [S1].
- Faces: Triangles sharing $x_i$ as a vertex [S1].

**Steps:**
1. Identify all faces $f_k$ that contain the vertex $x_i$ [S1].
2. For each face $f_k$, identify the two edges connected to $x_i$ and the other two vertices $x_j$ and $x_{j-1}$ ordered cyclically around $x_i$ [S1].
3. Compute the inner angle $\theta_j$ at vertex $x_i$ within face $f_k$, defined as the angle between vectors $\vec{v}_1 = x_j - x_i$ and $\vec{v}_2 = x_{j-1} - x_i$ [S1].
4. Sum all inner angles $\theta_j$ for all $N_i$ adjacent faces surrounding $x_i$:
   $$ S = \sum_{j=1}^{N_i} \theta_j $$
5. Compute the sign operator:
   $$ \kappa_G^{sign}(x_i) = \text{sign}\left( \sum_{j=1}^{N_i} \theta_j - 2\pi \right) $$
6. Classify the vertex: elliptic if $S < 2\pi$, hyperbolic if $S > 2\pi$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def classify_gaussian_curvature(x_i, vertices, faces):
    """
    Classifies a vertex as elliptic or hyperbolic based on discrete Gaussian curvature.
    
    Args:
        x_i: 3D coordinate tuple of the central vertex.
        vertices: List of 3D coordinate tuples for all mesh vertices.
        faces: List of tuples, each containing three vertex indices forming a triangle.
        
    Returns:
        str: 'elliptic', 'hyperbolic', or 'planar'.
    """
    # 1. Identify all faces containing vertex_i
    incident_faces = [f for f in faces if x_i in f]
    
    total_angle_sum = 0.0
    
    # 2-3. For each face, compute the inner angle at vertex_i
    for f in incident_faces:
        # Find the two other vertices in this face
        p1_idx, p2_idx = [v for v in f if v != x_i]
        p1 = vertices[p1_idx]
        p2 = vertices[p2_idx]
        
        # Vectors from center to neighbors
        v1 = (p1[0] - x_i[0], p1[1] - x_i[1], p1[2] - x_i[2])
        v2 = (p2[0] - x_i[0], p2[1] - x_i[1], p2[2] - x_i[2])
        
        # Dot product and norms
        dot = v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
        norm1 = math.sqrt(v1[0]**2 + v1[1]**2 + v1[2]**2)
        norm2 = math.sqrt(v2[0]**2 + v2[1]**2 + v2[2]**2)
        
        if norm1 == 0 or norm2 == 0:
            continue
            
        # Clamp cosine for numerical stability
        cos_angle = dot / (norm1 * norm2)
        cos_angle = max(-1.0, min(1.0, cos_angle))
        
        # 3. Compute angle
        angle = math.acos(cos_angle)
        total_angle_sum += angle
        
    # 4-6. Sum and classify
    if total_angle_sum < 2 * math.pi:
        return 'elliptic'
    elif total_angle_sum > 2 * math.pi:
        return 'hyperbolic'
    else:
        return 'planar'
```

**Parameters:**
- The threshold is fixed at $2\pi$ [S1].

**Edge Cases and Pitfalls:**
- **Planar Regions:** If the vertex lies on a flat planar region, the angles sum exactly to $2\pi$, which is neither strictly extremum nor blending in this binary scheme [S1]. The method relies on the sign of the deviation, ignoring the magnitude [S1].
- **Precision:** Floating-point errors may cause planar vertices to be misclassified; a tolerance epsilon should be used in implementation [G].
- **Boundary Vertices:** Not given in the sources.

## Subtopics

### Surface Type Classification
This is the primary application of the operator, distinguishing between sulci (local minima, elliptic) and gyri (local maxima, elliptic) versus saddle points (hyperbolic) [S1]. Elliptic regions correspond to local extrema where all neighbors lie on one side of the tangent plane [S1]. Hyperbolic regions correspond to saddle points where neighbors lie above and below the best-fitting least-squares plane [S1].

## Operating characteristics
**Time Complexity:** $O(d)$, where $d$ is the number of adjacent faces, due to summing $d$ angles [S1].
**Memory Complexity:** $O(1)$ auxiliary space per vertex evaluation, assuming 1-ring access [G].
**Strengths:** Provides a Boolean classification distinguishing local extrema from blending regions [S1].
**Limitations:** Binary classification loses curvature magnitude information [S1].
**When to use:** Not given in the sources.
**When not to use:** Not given in the sources.

## Background
The method originates from the work of Vivodtzev, Linsen, Bonneau, Hamann, Joy, and Olshausen in their paper "Hierarchical Isosurface Segmentation Based on Discrete Curvature" (2003) [S1]. It was developed to segment isosurfaces into topologically meaningful regions (sulci and gyri) for medical image analysis, specifically for recognizing diseases in brain structures [S1].

## Sources
[S1] Hierarchical Isosurface Segmentation Based on Discrete Curvature, https://inria.hal.science/inria-00537470v1, fetched 2026-10-26
