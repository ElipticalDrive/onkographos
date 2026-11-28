# Vertex-level Pruning in Mixed-Primitive Gaussian Splatting

Also called: Mixed-Primitive Vertex Pruning

## Overview
Vertex-level pruning is a refinement mechanism for Mixed-Primitive Gaussian Splatting (MP-GS) that removes redundant vertices from Gaussian triangles and lines during training to reduce storage overhead and simplify the scene representation [S1, unofficial]. Unlike standard 3D Gaussian Splatting which only performs primitive-level opacity pruning, this method dynamically downgrades primitives to lower-vertex forms (ellipses or lines) based on geometric coherence, addressing the "messy" representations that can arise when using mixed primitives [S1, unofficial]. It complements [Attribute Pruning](18516-attribute-pruning.md) by operating on the structural geometry of the splats rather than their scalar attributes.

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of mixed-primitive Gaussians $\{G_i\}$, where each primitive $G_i$ is defined by its vertices $\{\boldsymbol{\mu}_i^1, \boldsymbol{\mu}_i^2, \dots\}$, covariance parameters, opacity, and color.
*   **Output:** A refined set of primitives with reduced vertex counts where geometric redundancy is detected.

**Data Structures:**
Each mixed primitive maintains a list of vertex positions.
*   **Gaussian Triangle:** Stores 3 vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2, \boldsymbol{\mu}_3$.
*   **Gaussian Line:** Stores 2 vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2$.
*   **Gaussian Ellipse (Point):** Stores 1 vertex $\boldsymbol{\mu}_1$.

**Parameters:**
*   $\omega_{dist}$: Threshold for vertex proximity. Default: $0.5$ [S1, unofficial].
*   $\omega_{pear}$: Threshold for collinearity detection. Default: $0.9$ [S1, unofficial].

**Algorithm Steps:**
Vertex-level pruning is triggered concurrently with primitive-level pruning operations [S1, unofficial]. For each primitive undergoing evaluation:

1.  **Check for Gaussian Triangle Primitivity:**
    *   If the primitive is a Gaussian Triangle with vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2, \boldsymbol{\mu}_3$:
        *   Calculate the maximum pairwise L2 distance among the three vertices:
            $$ d_{max} = \max(\|\boldsymbol{\mu}_1 - \boldsymbol{\mu}_2\|_2, \|\boldsymbol{\mu}_2 - \boldsymbol{\mu}_3\|_2, \|\boldsymbol{\mu}_1 - \boldsymbol{\mu}_3\|_2) $$
        *   **Case A (Collapse to Ellipse):** If $d_{max} < \omega_{dist}$, the vertices are considered "close." Prune $\boldsymbol{\mu}_2$ and $\boldsymbol{\mu}_3$. Convert the primitive into a Gaussian Ellipse centered at $\boldsymbol{\mu}_1$.
        *   **Case B (Check Collinearity):** If $d_{max} \ge \omega_{dist}$, check for collinearity. Compute the Pearson correlation coefficient $r$ from the coordinates of the three vertices.
            *   If $|r| > \omega_{pear}$, the vertices are nearly collinear. Convert the primitive into a Gaussian Line.
            *   Otherwise, retain the Gaussian Triangle structure.

2.  **Check for Gaussian Line Primitivity:**
    *   If the primitive is a Gaussian Line with vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2$:
        *   Calculate the L2 distance:
            $$ d = \|\boldsymbol{\mu}_1 - \boldsymbol{\mu}_2\|_2 $$
        *   If $d < \omega_{dist}$, the vertices are "close." Prune $\boldsymbol{\mu}_2$. Convert the primitive into a Gaussian Ellipse centered at $\boldsymbol{\mu}_1$.

3.  **Preservation of Complexity:**
    *   This pruning does not reduce the entire scene to point-centered ellipses because the standard Gaussian Splatting densification mechanisms (cloning and splitting) continue to operate, potentially re-introducing higher-order primitives where detail is required [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_level_pruning(primitive, omega_dist=0.5, omega_pear=0.9):
    """
    Refines a mixed-primitive Gaussian by reducing vertex count based on geometry.
    primitive: object with type ('triangle', 'line', 'ellipse') and vertices [mu1, mu2, ...]
    """
    if primitive.type == 'triangle':
        mu1, mu2, mu3 = primitive.vertices
        # Calculate pairwise distances
        d12 = l2_norm(mu1 - mu2)
        d23 = l2_norm(mu2 - mu3)
        d13 = l2_norm(mu1 - mu3)
        max_dist = max(d12, d23, d13)
        
        if max_dist < omega_dist:
            # Vertices are too close; collapse to ellipse
            primitive.type = 'ellipse'
            primitive.vertices = [mu1]
            # Note: Covariance and other attributes are inherited/adjusted to center mu1
        else:
            # Vertices are spread out; check for collinearity
            # Compute Pearson correlation coefficient from vertex coordinates
            # Let X be the matrix of coordinates [mu1, mu2, mu3]
            r = pearson_correlation(X)
            if abs(r) > omega_pear:
                # Vertices are collinear; convert to line
                primitive.type = 'line'
                primitive.vertices = [mu1, mu2] # Keep first two as representative
            # Else: remain triangle
             
    elif primitive.type == 'line':
        mu1, mu2 = primitive.vertices
        dist = l2_norm(mu1 - mu2)
        
        if dist < omega_dist:
            # Vertices are too close; collapse to ellipse
            primitive.type = 'ellipse'
            primitive.vertices = [mu1]
            
    return primitive

def l2_norm(v):
    return sqrt(sum(x**2 for x in v))

def pearson_correlation(points):
    """
    Computes the absolute Pearson correlation coefficient for a set of 3 points in 3D.
    Simplified implementation for the specific context of MP-GS.
    """
    # Extract x, y, z coordinates
    xs = [p[0] for p in points]
    ys = [p[1] for p in points]
    zs = [p[2] for p in points]
    
    # Calculate means
    mean_x = sum(xs) / 3
    mean_y = sum(ys) / 3
    mean_z = sum(zs) / 3
    
    # Calculate covariance and variances
    # Cov(x,y) = sum((xi - mean_x)(yi - mean_y)) / (n-1)
    cov_xy = sum((xs[i] - mean_x) * (ys[i] - mean_y) for i in range(3)) / 2
    cov_xz = sum((xs[i] - mean_x) * (zs[i] - mean_z) for i in range(3)) / 2
    cov_yz = sum((ys[i] - mean_y) * (zs[i] - mean_z) for i in range(3)) / 2
    
    var_x = sum((x - mean_x)**2 for x in xs) / 2
    var_y = sum((y - mean_y)**2 for y in ys) / 2
    var_z = sum((z - mean_z)**2 for z in zs) / 2
    
    # Correlation coefficients
    r_xy = cov_xy / (sqrt(var_x) * sqrt(var_y)) if var_x > 0 and var_y > 0 else 0
    r_xz = cov_xz / (sqrt(var_x) * sqrt(var_z)) if var_x > 0 and var_z > 0 else 0
    r_yz = cov_yz / (sqrt(var_y) * sqrt(var_z)) if var_y > 0 and var_z > 0 else 0
    
    # The source specifies "the Pearson correlation coefficient" (singular).
    # In 3D collinearity checks, this often refers to the maximum correlation or a specific projection.
    # Given the ambiguity in the source text regarding the exact scalar derivation from 3D points,
    # we assume the implementation uses the maximum absolute correlation among pairs or a principal component analysis result.
    # For strict adherence to "absolute value ... exceeds omega_pear", we take the max.
    return max(abs(r_xy), abs(r_xz), abs(r_yz))
```

**Edge Cases and Pitfalls:**
*   **Precision:** The L2 distance threshold $\omega_{dist}=0.5$ is absolute. In scenes with varying scales, this may require normalization [S1, unofficial].
*   **Collinearity Ambiguity:** The source cites "the Pearson correlation coefficient" for 3D points without specifying the exact reduction method (e.g., PCA-based or pairwise max). The pseudocode assumes the maximum pairwise correlation as a robust proxy for collinearity [G].
*   **Interaction with Densification:** Pruning must not interfere with the cloning/splitting logic. If a primitive is pruned to an ellipse, subsequent splits must correctly handle the single-vertex representation [S1, unofficial].

## Operating characteristics
*   **Storage Reduction:** The vertex pruning mechanism reduces the average storage size required to store vertex coordinates by 24% compared to the variant without pruning [S1, unofficial].
*   **Performance Impact:** The source notes that the framework with vertex pruning performs better (lower Chamfer distance) than the variant without it, suggesting improved surface reconstruction quality [S1, unofficial].
*   **Complexity:** The pruning check is $O(1)$ per primitive per training step, as it involves only a few distance and correlation calculations.
*   **Limitations:** The method relies on heuristic thresholds ($\omega_{dist}, \omega_{pear}$) which may not generalize perfectly across all scene geometries without tuning.

## Background
Vertex-level pruning was introduced in the context of Mixed-Primitive Gaussian Splatting (MP-GS), an extension of standard 3D Gaussian Splatting that uses triangles and lines in addition to point-centered ellipses to better represent surfaces [S1, unofficial]. Standard 3D Gaussian Splatting only performs primitive-level pruning based on opacity [S1, unofficial]. MP-GS authors observed that mixed primitives could become "messy" with redundant vertices, necessitating this additional geometric simplification step during training [S1, unofficial].

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
