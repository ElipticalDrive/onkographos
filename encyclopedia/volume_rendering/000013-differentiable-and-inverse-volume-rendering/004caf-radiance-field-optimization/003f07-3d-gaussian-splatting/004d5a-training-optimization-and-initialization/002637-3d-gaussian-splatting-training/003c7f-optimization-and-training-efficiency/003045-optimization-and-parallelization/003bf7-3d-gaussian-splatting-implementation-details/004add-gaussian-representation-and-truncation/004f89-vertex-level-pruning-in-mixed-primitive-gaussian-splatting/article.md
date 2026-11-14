# Vertex-level Pruning in Mixed-Primitive Gaussian Splatting

Also called: Mixed-Primitive Vertex Pruning

## Overview
Vertex-level pruning is a geometric simplification mechanism applied to mixed-primitive Gaussian Splatting (MP-GS) representations during the training optimization loop [S1, unofficial]. It removes redundant vertices from Gaussian triangles and lines to refine the surface representation and reduce storage overhead, operating in conjunction with standard primitive-level opacity pruning [S1, unofficial]. Unlike sibling methods that focus on opacity thresholds or voxel-based culling, this method specifically addresses the topological inefficiency of higher-order primitives (triangles/lines) when their defining vertices become geometrically degenerate [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of mixed-primitive Gaussians, where each primitive is defined by a set of vertex positions $\{\mu_i\}$ and associated covariance/opacity parameters.
*   **Output:** A refined set of primitives where degenerate vertices have been removed or the primitive type has been downgraded (e.g., triangle to line, line to ellipse).

**Data Structures:**
The method operates on the vertex coordinates $\mu$ of the primitives.
*   **Gaussian Triangle:** Defined by three vertices $\mu_1, \mu_2, \mu_3$.
*   **Gaussian Line:** Defined by two vertices $\mu_1, \mu_2$.
*   **Gaussian Ellipse:** Defined by a single center vertex $\mu_1$ (effectively a point-centered primitive with 2D covariance).

**Parameters:**
*   $\omega_{dist}$: The distance threshold for determining if vertices are "close". Default value: $0.5$ [S1, unofficial].
*   $\omega_{pear}$: The correlation threshold for determining if vertices are "collinear". Default value: $0.9$ [S1, unofficial].

**Algorithm Steps:**
Vertex-level pruning is triggered synchronously with primitive-level pruning operations [S1, unofficial]. The logic proceeds as follows for each active primitive:

1.  **Triangle to Ellipse Conversion:**
    For a Gaussian triangle with vertices $\mu_1, \mu_2, \mu_3$:
    *   Calculate the maximum pairwise L2 distance among the three vertices:
        $$ d_{max} = \max(\|\mu_1 - \mu_2\|_2, \|\mu_2 - \mu_3\|_2, \|\mu_1 - \mu_3\|_2) $$
    *   If $d_{max} < \omega_{dist}$, the vertices are considered too close. Prune vertices $\mu_2$ and $\mu_3$. Convert the primitive into a Gaussian ellipse centered at $\mu_1$ [S1, unofficial].

2.  **Triangle to Line Conversion:**
    For a Gaussian triangle with vertices $\mu_1, \mu_2, \mu_3$:
    *   If the condition in Step 1 is not met (vertices are not close), check for collinearity.
    *   Compute the Pearson correlation coefficient $r$ from the coordinates of the three vertices.
    *   If $|r| > \omega_{pear}$, the vertices are considered nearly collinear. Convert the Gaussian triangle into a Gaussian line [S1, unofficial].

3.  **Line to Ellipse Conversion:**
    For a Gaussian line with vertices $\mu_1, \mu_2$:
    *   Calculate the L2 distance:
        $$ d = \|\mu_1 - \mu_2\|_2 $$
    *   If $d < \omega_{dist}$, the vertices are considered too close. Prune vertex $\mu_2$. Reduce the primitive to a Gaussian ellipse centered at $\mu_1$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_level_pruning(primitive, omega_dist=0.5, omega_pear=0.9):
    """
    Prunes vertices from a mixed-primitive Gaussian based on geometric degeneracy.
    
    Args:
        primitive: An object containing 'type' ('triangle' or 'line') 
                   and 'vertices' (list of numpy arrays for mu).
        omega_dist: Threshold for vertex proximity.
        omega_pear: Threshold for collinearity.
        
    Returns:
        primitive: The potentially modified primitive.
    """
    verts = primitive.vertices
    p_type = primitive.type
    
    if p_type == 'triangle':
        # Ensure we have 3 vertices
        if len(verts) != 3:
            return primitive
            
        m1, m2, m3 = verts[0], verts[1], verts[2]
        
        # Step 1: Check if vertices are close
        d12 = np.linalg.norm(m1 - m2)
        d23 = np.linalg.norm(m2 - m3)
        d13 = np.linalg.norm(m1 - m3)
        max_dist = max(d12, d23, d13)
        
        if max_dist < omega_dist:
            # Prune m2 and m3, convert to ellipse at m1
            primitive.type = 'ellipse'
            primitive.vertices = [m1]
            return primitive
            
        # Step 2: Check if vertices are collinear
        # Compute Pearson correlation coefficient from vertex coordinates
        # Vertices are 3D points; we flatten them or compute correlation across dimensions
        # Standard approach: stack coordinates and compute correlation matrix or 
        # fit a line and check residuals. The source specifies "Pearson correlation 
        # coefficient computed from these vertices".
        # Assuming this implies checking the correlation of the coordinate vectors.
        coords = np.array([m1, m2, m3]) # Shape (3, 3)
        
        # Calculate Pearson correlation. 
        # Note: With only 3 points, standard Pearson correlation on the flattened 
        # array or per-dimension might be unstable. The source implies a single 
        # scalar metric. We assume the implementation computes the correlation 
        # of the positions along the principal axis or similar geometric proxy.
        # For implementation completeness, we use the correlation of the 
        # concatenated coordinate vectors.
        r = np.corrcoef(coords.flatten())[0, 1] # Placeholder for exact correlation logic
        
        if abs(r) > omega_pear:
            # Convert to line
            primitive.type = 'line'
            # Keep two vertices, typically the endpoints of the line segment
            # The source does not specify which two to keep, but logically 
            # it becomes a line primitive.
            primitive.vertices = [m1, m3] # Example selection
            return primitive
            
    elif p_type == 'line':
        # Ensure we have 2 vertices
        if len(verts) != 2:
            return primitive
            
        m1, m2 = verts[0], verts[1]
        
        # Step 3: Check if vertices are close
        dist = np.linalg.norm(m1 - m2)
        
        if dist < omega_dist:
            # Prune m2, convert to ellipse at m1
            primitive.type = 'ellipse'
            primitive.vertices = [m1]
            return primitive
            
    return primitive
```

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** If a triangle collapses to a point, it becomes an ellipse. If it collapses to a line, it becomes a line primitive. The order of checks (distance first, then collinearity) ensures that point-like triangles are handled before line-like ones [S1, unofficial].
*   **Collinearity Metric:** The use of Pearson correlation for 3D points requires careful implementation. With only 3 points, the correlation coefficient might be sensitive to the ordering or scaling of coordinates. The source does not specify the exact vectorization of the correlation calculation [S1, unofficial].
*   **Interaction with Densification:** This pruning mechanism works alongside the original densification (splitting/cloning) mechanisms. Pruning does not permanently eliminate primitives from the scene, only simplifies their vertex count [S1, unofficial].

## Operating characteristics
*   **Storage Reduction:** The vertex pruning mechanism reduces the average storage size required to store vertex coordinates by 24% compared to the variant without pruning [S1, unofficial].
*   **Performance:** The pruning step is lightweight, involving only distance and correlation calculations on a small number of vertices per primitive. It is executed during the training loop's pruning phase [S1, unofficial].
*   **Limitations:** The method relies on fixed thresholds ($\omega_{dist}, \omega_{pear}$) which may not adapt well to scenes with varying scales without normalization. It does not address primitive-level redundancy (handled by opacity pruning) [S1, unofficial].

## Background
Vertex-level pruning was introduced in the context of Mixed-Primitive Gaussian Splatting (MP-GS) for surface reconstruction [S1, unofficial]. The authors observed that while standard 3D Gaussian Splatting uses only point-centered primitives, MP-GS utilizes primitives with varying numbers of vertices (triangles, lines). During learning, some areas of an object do not require the complexity of Gaussian triangles, and simpler primitives (lines or ellipses) are sufficient. This mechanism was added to eliminate unnecessary vertices and refine the representation into less messy structures [S1, unofficial].

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
