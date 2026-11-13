# Vertex-level Pruning in Mixed-Primitive Gaussian Splatting

Also called: Mixed-Primitive Vertex Pruning

## Overview
Vertex-level pruning is a reduction mechanism for mixed-primitive Gaussian Splatting that removes redundant vertices from Gaussian triangles and lines during training to refine the geometric representation [S1, unofficial]. It operates in conjunction with standard primitive-level opacity pruning, converting high-vertex primitives into lower-vertex equivalents (ellipses or lines) when their spatial configuration becomes degenerate [S1, unofficial]. This method distinguishes itself from sibling truncation techniques by targeting the internal vertex structure of mixed primitives rather than discarding entire primitives or modifying the alpha blending function [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of mixed-primitive Gaussians, where each primitive is defined by a set of vertices (means $\mu$), a covariance matrix $\Sigma$ (scale $s$, rotation $r$), opacity $\omega$, and spherical harmonic coefficients.
*   **Output:** A refined set of primitives with reduced vertex counts where appropriate.

**Data Structures:**
Each mixed primitive maintains a list of vertex positions. For a Gaussian triangle, this list contains three vertices: $\mu_1, \mu_2, \mu_3$. For a Gaussian line, it contains two vertices: $\mu_1, \mu_2$.

**Parameters:**
*   $\omega_{dist}$: The threshold for determining if vertices are "close". Default value: 0.5 [S1, unofficial].
*   $\omega_{pear}$: The threshold for determining if Gaussian triangle vertices are collinear. Default value: 0.9 [S1, unofficial] [G].

**Algorithm:**
Vertex-level pruning is triggered synchronously with primitive-level pruning operations [S1, unofficial]. The process evaluates the spatial relationship between vertices within each primitive and applies one of three reduction rules:

1.  **Triangle to Ellipse (Collapse):** If a Gaussian triangle's three vertices are all close to each other, the primitive is converted into a Gaussian ellipse centered at $\mu_1$. Vertices $\mu_2$ and $\mu_3$ are pruned.
2.  **Triangle to Line (Flatten):** If a Gaussian triangle's vertices are not close but are nearly collinear, the primitive is converted into a Gaussian line.
3.  **Line to Ellipse (Collapse):** If a Gaussian line's two vertices are close to each other, the primitive is converted into a Gaussian ellipse centered at $\mu_1$. Vertex $\mu_2$ is pruned.

**Definitions:**
*   **Close Vertices:** A set of vertices is considered close if their maximum pairwise L2 distance is below $\omega_{dist}$ [S1, unofficial].
*   **Nearly Collinear Vertices:** For a triangle with vertices $\mu_1, \mu_2, \mu_3$, they are nearly collinear if the absolute value of the Pearson correlation coefficient computed from the vertices exceeds $\omega_{pear}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_level_pruning(primitive, omega_dist=0.5, omega_pear=0.9):
    """
    Prunes vertices from a mixed-primitive Gaussian based on spatial configuration.
    Called during the primitive-level pruning phase of training.
    """
    p_type = primitive.type
    vertices = primitive.vertices  # List of mean positions
    
    if p_type == "triangle":
        mu1, mu2, mu3 = vertices[0], vertices[1], vertices[2]
        
        # Check if all three vertices are close
        max_pairwise_dist = max(
            l2_distance(mu1, mu2),
            l2_distance(mu2, mu3),
            l2_distance(mu1, mu3)
        )
        
        if max_pairwise_dist < omega_dist:
            # Rule 1: Collapse triangle to ellipse
            primitive.type = "ellipse"
            primitive.vertices = [mu1]
            # Covariance and other attributes are retained/adjusted for the ellipse
            return primitive
            
        # Check if vertices are nearly collinear
        if is_nearly_collinear(mu1, mu2, mu3, omega_pear):
            # Rule 2: Flatten triangle to line
            primitive.type = "line"
            # Retain all three vertices? Or reduce to two?
            # The source states "convert... into a Gaussian line".
            # A line primitive typically has 2 vertices.
            # We keep mu1 and mu2 (or the two most distant) to define the line.
            # However, the source doesn't specify which 2 to keep.
            # Standard practice: keep the endpoints.
            # We assume the line is defined by the first two vertices for simplicity,
            # or we recompute the line basis from the collinear points.
            # Given the ambiguity, we retain the structure but change type.
            # Note: The source implies a structural change.
            primitive.vertices = [mu1, mu2] 
            return primitive
            
    elif p_type == "line":
        mu1, mu2 = vertices[0], vertices[1]
        
        # Check if the two vertices are close
        if l2_distance(mu1, mu2) < omega_dist:
            # Rule 3: Collapse line to ellipse
            primitive.type = "ellipse"
            primitive.vertices = [mu1]
            return primitive
            
    return primitive

def l2_distance(v1, v2):
    return sqrt(sum((a - b) ** 2 for a, b in zip(v1, v2)))

def is_nearly_collinear(mu1, mu2, mu3, threshold):
    """
    Computes the absolute Pearson correlation coefficient for the 3D points.
    Treats the x, y, z coordinates as separate variables against the index.
    """
    # Extract coordinates
    xs = [mu1[0], mu2[0], mu3[0]]
    ys = [mu1[1], mu2[1], mu3[1]]
    zs = [mu1[2], mu2[2], mu3[2]]
    
    # Pearson correlation requires variance. 
    # With only 3 points, we compute correlation between the index (0,1,2) 
    # and the coordinates.
    indices = [0, 1, 2]
    
    # Helper to compute Pearson r
    def pearson(x, y):
        n = len(x)
        mean_x = sum(x) / n
        mean_y = sum(y) / n
        
        num = sum((xi - mean_x) * (yi - mean_y) for xi, yi in zip(x, y))
        den_x = sqrt(sum((xi - mean_x) ** 2 for xi in x))
        den_y = sqrt(sum((yi - mean_y) ** 2 for yi in y))
        
        if den_x == 0 or den_y == 0:
            return 0.0
            
        return num / (den_x * den_y)
        
    r_x = pearson(indices, xs)
    r_y = pearson(indices, ys)
    r_z = pearson(indices, zs)
    
    # The source says "Pearson correlation coefficient computed from these vertices".
    # In 3D, this is ambiguous. It likely refers to the correlation of the 
    # projection onto the principal component, or simply the max correlation 
    # of coordinates. Given the threshold 0.9, it implies a strong linear fit.
    # A common interpretation for 3D collinearity using Pearson is checking 
    # if the points lie on a line. 
    # We take the maximum absolute correlation across dimensions as a proxy,
    # or the correlation of the data with its mean projection.
    # However, a simpler interpretation often used in such contexts is the 
    # correlation of the sorted coordinates or the fit to a line.
    # Let's assume the source implies the correlation of the 3 points 
    # considered as a multivariate set, which reduces to checking if 
    # the variance is concentrated in one direction.
    # For implementation simplicity and adherence to "Pearson... computed from vertices":
    # We check if the correlation between the index and any coordinate is high.
    # Or, more robustly, we check the correlation of the points with their centroid.
    
    # Re-reading: "absolute value of the Pearson correlation coefficient computed from these vertices"
    # This phrasing is singular. It might refer to the correlation of the 
    # 3D point cloud with a line. 
    # Let's use the max absolute correlation of the coordinates with the index 
    # as a conservative estimate, or the correlation of the first two coordinates.
    # Actually, for 3 points, if they are collinear, the correlation between 
    # any two coordinates (if the line is not axis-aligned) will be 1 or -1.
    # If the line is axis aligned, one coordinate is constant (variance 0).
    
    # We will use the maximum absolute Pearson correlation between any pair of 
    # coordinate dimensions (x-y, y-z, x-z).
    r_xy = pearson(xs, ys)
    r_yz = pearson(ys, zs)
    r_xz = pearson(xs, zs)
    
    max_r = max(abs(r_xy), abs(r_yz), abs(r_xz))
    
    return max_r > threshold
```

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** If vertices are extremely close, numerical instability in covariance calculation may occur when converting to an ellipse. [G] Ensure the covariance matrix is regularized or reset to a default isotropic scale when collapsing to an ellipse.
*   **Collinearity Threshold:** The choice of $\omega_{pear}$ (0.9) is critical. Too high, and valid thin triangles are not converted to lines, leaving redundant vertices. Too low, and valid triangles are incorrectly flattened, losing surface detail.
*   **Order of Operations:** Vertex pruning must occur *after* primitive-level opacity pruning but *before* the next densification step to ensure that reduced primitives can still participate in splitting/clone operations if needed [S1, unofficial].

## Operating characteristics
*   **Storage Efficiency:** The vertex pruning mechanism reduces the average storage size required to store vertex coordinates by 24% compared to the variant without pruning [S1, unofficial].
*   **Performance:** The method adds minimal computational overhead during training as it relies on simple distance and correlation checks. It does not affect rendering speed directly, as the rendering pipeline handles ellipses and lines similarly to triangles in terms of rasterization cost (though triangle rasterization may be slightly more complex than ellipse/line).
*   **Quality:** Ablation studies show that removing vertex pruning leads to "messier" representations and lower reconstruction quality (higher Chamfer distance) [S1, unofficial].

## Background
This method is part of the Mixed-Primitive Gaussian Splatting (MP-GS) framework, which extends standard 3D Gaussian Splatting by using primitives with varying numbers of vertices (triangles, lines, ellipses) to better represent surfaces [S1, unofficial]. The vertex pruning mechanism was introduced to handle the redundancy that arises when high-vertex primitives become unnecessary in smooth or linear regions of the object [S1, unofficial].

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
