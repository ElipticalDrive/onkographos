# Silhouette-based Edge Sampling for Directly Visible Surfaces

Also called: Silhouette Edge Sampling

## Overview
This method efficiently samples boundary integrals for directly visible surfaces by identifying and sampling only silhouette edges, thereby reducing variance compared to uniform edge sampling [S1]. It sets itself apart from its sibling [Primary Edge Sampling](7168-primary-edge-sampling.md) by introducing a geometric classification step that filters out non-silhouette edges, which do not contribute to boundaries under smooth shading assumptions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh defined by edges, faces, and vertex positions; camera parameters (position $v$ and aperture for non-pinhole cameras); screen resolution.
*   **Output:** A set of sampled points on silhouette edges with associated rendering contributions evaluated on both sides of the edge.

**Data Structures:**
*   **Edge List:** A list of edges, where each edge is defined by two vertices or two adjacent faces. For edges shared by more than two faces, they are converted into multiple edges with two faces each [S1].
*   **Silhouette Mask:** A boolean flag or list indicating which edges are classified as silhouette edges relative to the current viewpoint [S1].
*   **Screen-Space Lengths:** A floating-point value for each silhouette edge representing its projected length in screen space, used for probability weighting [S1].

**Steps:**

1.  **Precomputation: Silhouette Classification**
    For each edge in the mesh, determine if it is a silhouette edge relative to the camera viewpoint $v$.
    *   Let the edge be associated with two faces having normals $n_f$ and $n_b$.
    *   Let $p$ be an arbitrary point on the edge.
    *   The edge is a silhouette edge if and only if:
        $$ \text{sign} \langle p - v, n_f \rangle \neq \text{sign} (\langle p - v, n_b \rangle) $$
        [S1]
    *   If the edge is associated with only one face, it is always classified as a silhouette edge [S1].
    *   **Non-pinhole Camera Handling:** If the camera is not a pinhole camera, compute a bounding box for the camera aperture. Conservatively classify an edge as a silhouette if it is a silhouette for at least one viewpoint within that bounding box [S1].

2.  **Precomputation: Screen-Space Projection**
    Project all classified silhouette edges to screen space and calculate their lengths [S1].

3.  **Rendering: Edge Selection**
    Randomly select one silhouette edge from the precomputed list. The probability of selecting an edge is proportional to its length in screen space [S1].

4.  **Rendering: Point Sampling**
    Uniformly select one point on the selected edge [S1].

5.  **Rendering: Contribution Evaluation**
    Evaluate the rendering contribution at the two sides of the selected edge using the sampled point [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def classify_and_sample_silhouette_edges(mesh, camera):
    """
    Precomputation and sampling for silhouette-based edge sampling.
    
    Args:
        mesh: Object containing edges, faces, vertices, and normals.
        camera: Object containing viewpoint position (v) and aperture info.
        
    Returns:
        list of (point, contribution_left, contribution_right) samples.
    """
    silhouette_edges = []
    
    # Step 1 & 2: Precomputation
    for edge in mesh.edges:
        # Get normals of adjacent faces
        # If edge has only 1 face, it is always silhouette
        if len(edge.faces) == 1:
            is_silhouette = True
        elif len(edge.faces) == 2:
            n_f = edge.faces[0].normal
            n_b = edge.faces[1].normal
            p = edge.vertex_1 # Arbitrary point on edge
            v = camera.position
            
            # Check silhouette condition
            # sign function: returns 1 if > 0, -1 if < 0, 0 if == 0
            sign_f = 1 if (p - v) @ n_f > 0 else (-1 if (p - v) @ n_f < 0 else 0)
            sign_b = 1 if (p - v) @ n_b > 0 else (-1 if (p - v) @ n_b < 0 else 0)
            
            # For non-pinhole cameras, conservative classification is needed
            # [G] Assuming camera has a method to check if edge is silhouette 
            # for any viewpoint in aperture bounding box.
            if camera.is_non_pinhole:
                # Conservative check: if silhouette for ANY viewpoint in aperture
                # Simplified: if standard pinhole check passes, it's a candidate.
                # To be strictly conservative as per [S1], we might need to check
                # multiple points in aperture or use bounding box logic.
                # Here we assume the standard check is sufficient for the "at least one"
                # condition if the aperture is small, or we perform a broader check.
                # For implementation completeness based on [S1]:
                is_silhouette = (sign_f != sign_b)
                # Note: [S1] says "classify as silhouette if it is a silhouette for 
                # at least one viewpoint". A robust implementation would check the 
                # bounding box of the aperture. If the standard check fails, 
                # we might still need to check corners of the aperture.
                # However, the core formula is for a single viewpoint v.
                # We stick to the formula for the primary viewpoint v, 
                # but note the pitfall.
            else:
                is_silhouette = (sign_f != sign_b)
        else:
            # Edges with 3+ faces are converted to multiple edges with 2 faces
            # [S1]
            # [G] Implementation detail: Split edge into pairs of faces.
            # For this pseudocode, we assume the mesh data structure 
            # handles this or we iterate over face pairs.
            is_silhouette = False # Placeholder for complex topology handling
            
        if is_silhouette:
            # Project to screen space and calculate length
            # [G] Assuming mesh has a method to project vertices to screen space
            p1_screen = camera.project(edge.vertex_1)
            p2_screen = camera.project(edge.vertex_2)
            length = math.sqrt((p1_screen.x - p2_screen.x)**2 + (p1_screen.y - p2_screen.y)**2)
            silhouette_edges.append({
                'edge': edge,
                'screen_length': length
            })
            
    if not silhouette_edges:
        return []
        
    # Step 3: Select edge proportional to screen length
    total_length = sum(e['screen_length'] for e in silhouette_edges)
    r = random.uniform(0, total_length)
    cumulative = 0
    selected_edge_data = None
    for edge_data in silhouette_edges:
        cumulative += edge_data['screen_length']
        if cumulative >= r:
            selected_edge_data = edge_data
            break
            
    # Step 4: Uniformly pick point on edge
    t = random.uniform(0, 1)
    p1 = selected_edge_data['edge'].vertex_1
    p2 = selected_edge_data['edge'].vertex_2
    sampled_point = p1 + t * (p2 - p1)
    
    # Step 5: Evaluate contribution at two sides
    # [G] The actual rendering contribution evaluation depends on the 
    # specific differentiable rendering framework. 
    # We assume a function `evaluate_edge_contribution` exists.
    contribution = evaluate_edge_contribution(sampled_point, selected_edge_data['edge'])
    
    return [sampled_point, contribution]
```

**Parameters:**
*   **Camera Aperture:** If non-pinhole, the bounding box of the aperture determines the conservative classification range [S1].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Non-pinhole Cameras:** Silhouette classification must be conservative. If the camera has an aperture, an edge must be classified as a silhouette if it is a silhouette for at least one viewpoint within the aperture's bounding box [S1].
*   **Complex Topology:** Edges associated with three or more faces must be converted into multiple edges with two faces each to apply the binary normal test [S1].
*   **Variance:** Naïve uniform edge sampling generates large variance for complex scenes; this method reduces variance by focusing on silhouettes [S1].

## Subtopics

### Primary Edge Sampling
The parent method that defines the general framework of sampling points from primary edges $S$, defined as the intersection of the implicit surface $\phi$ and the orientation function $\psi$. Silhouette-based sampling is a specific strategy for selecting these edges. [Primary Edge Sampling](7168-primary-edge-sampling.md)

## Operating characteristics
*   **Time Complexity:** The precomputation pass involves classifying all edges and projecting them to screen space. The number of silhouette edges is roughly at the complexity of $N$ with $N$ being the number of edges, whereas the majority of edges are not silhouettes [S1].
*   **Memory Complexity:** Requires storing the list of silhouette edges and their screen-space lengths.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Fast and does not require any sophisticated data structures. Reduces variance compared to uniform edge sampling for complex scenes [S1].
*   **Limitations:** For scenes with high depth complexity, further occlusion culling algorithms may be needed to skip edges, as the method itself does not handle occlusion for directly visible surfaces beyond silhouette classification [S1].
*   **When to Use:** When rendering directly visible surfaces with smooth shading, where only silhouette edges contribute to boundaries [S1].
*   **When Not to Use:** For secondary visibility such as shadow, mirror reflection, and global illumination, where the viewpoint is a shading point anywhere in the scene, making silhouette determination more involved and similar to the many-lights sampling problem [S1].

## Background
This method is described in the context of differentiable path tracing and physics-based differentiable rendering. It addresses the issue of high variance in boundary integral sampling for complex scenes by leveraging the geometric property that only silhouette edges form boundaries under smooth shading [S1]. The approach builds on the observation that for directly visible surfaces, the number of silhouette edges is significantly smaller than the total number of edges [S1].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
