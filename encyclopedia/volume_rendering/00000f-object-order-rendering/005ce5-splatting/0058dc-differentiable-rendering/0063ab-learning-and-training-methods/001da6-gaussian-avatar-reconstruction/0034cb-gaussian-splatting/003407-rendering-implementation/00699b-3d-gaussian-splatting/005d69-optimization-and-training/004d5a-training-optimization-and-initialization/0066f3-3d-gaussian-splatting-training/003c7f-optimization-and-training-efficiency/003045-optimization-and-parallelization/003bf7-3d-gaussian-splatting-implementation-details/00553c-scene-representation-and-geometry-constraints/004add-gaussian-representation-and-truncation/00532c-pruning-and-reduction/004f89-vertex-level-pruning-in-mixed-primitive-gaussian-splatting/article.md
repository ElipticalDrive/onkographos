# Vertex-level Pruning in Mixed-Primitive Gaussian Splatting

Also called: Mixed-Primitive Vertex Pruning

## Overview
Vertex-level pruning is a geometric simplification mechanism applied during the training of Mixed-Primitive Gaussian Splatting (MP-GS) to remove redundant vertices from Gaussian triangles and lines, converting them into lower-order primitives (ellipses or lines) [S1, unofficial]. It addresses the issue of "messy" representations where high-order primitives persist in regions that can be adequately modeled by simpler shapes, thereby refining the surface representation and reducing storage overhead [S1, unofficial]. Unlike standard 3D Gaussian Splatting which only performs primitive-level opacity pruning, this method operates on the vertex level within mixed-primitive frameworks [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of mixed-primitive Gaussians $\{G_i\}$, where each primitive is defined by a set of vertices $\{\boldsymbol{\mu}_j\}$, covariance parameters, opacity $\alpha_i$, and color $\mathbf{c}_i$.
*   **Output:** A refined set of primitives with reduced vertex counts where geometric redundancy is detected.

**Data Structures:**
Each mixed primitive stores its vertices explicitly. For a Gaussian triangle, three vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2, \boldsymbol{\mu}_3$ are stored. For a Gaussian line, two vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2$ are stored. The primitive type (triangle, line, ellipse) is tracked.

**Parameters:**
*   $\omega_{dist}$: The threshold for determining if vertices are "close". Default value: $0.5$ [S1, unofficial].
*   $\omega_{pear}$: The threshold for determining if Gaussian triangle vertices are collinear. Default value: $0.9$ [S1, unofficial].

**Algorithm Steps:**
Vertex-level pruning is triggered whenever primitive-level pruning is performed during the learning procedure [S1, unofficial]. The process evaluates the geometric configuration of each primitive:

1.  **Check for Gaussian Triangle Collapse to Ellipse:**
    For a Gaussian triangle primitive with vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2, \boldsymbol{\mu}_3$:
    *   Calculate the maximum pairwise L2 distance among the three vertices:
        $$ d_{max} = \max(\|\boldsymbol{\mu}_1 - \boldsymbol{\mu}_2\|_2, \|\boldsymbol{\mu}_2 - \boldsymbol{\mu}_3\|_2, \|\boldsymbol{\mu}_3 - \boldsymbol{\mu}_1\|_2) $$
    *   If $d_{max} < \omega_{dist}$, the vertices are considered close. Prune vertices $\boldsymbol{\mu}_2$ and $\boldsymbol{\mu}_3$. Convert the primitive into a Gaussian ellipse centered at $\boldsymbol{\mu}_1$ [S1, unofficial].

2.  **Check for Gaussian Triangle Degeneration to Line:**
    For a Gaussian triangle primitive where the condition in Step 1 is not met (vertices are not close):
    *   Compute the Pearson correlation coefficient from the coordinates of the three vertices. Let the vertices be represented as a matrix $V$ where rows are vertices. The correlation is computed across the dimensions [S1, unofficial].
    *   If the absolute value of the Pearson correlation coefficient exceeds $\omega_{pear}$, the vertices are considered nearly collinear. Convert the primitive from a Gaussian triangle to a Gaussian line [S1, unofficial].

3.  **Check for Gaussian Line Collapse to Ellipse:**
    For a Gaussian line primitive with vertices $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2$:
    *   Calculate the L2 distance:
        $$ d = \|\boldsymbol{\mu}_1 - \boldsymbol{\mu}_2\|_2 $$
    *   If $d < \omega_{dist}$, the vertices are considered close. Prune vertex $\boldsymbol{\mu}_2$. Reduce the primitive to a Gaussian ellipse centered at $\boldsymbol{\mu}_1$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_level_pruning(primitive, omega_dist=0.5, omega_pear=0.9):
    """
    Prunes vertices from a mixed-primitive Gaussian based on geometric redundancy.
    
    Args:
        primitive: An object containing:
            - type: 'triangle', 'line', or 'ellipse'
            - vertices: List of numpy arrays [mu1, mu2, ...]
            - covariance: Covariance matrix or parameters
            - opacity: Scalar alpha
            - color: Color vector
        omega_dist: Threshold for vertex proximity (default 0.5)
        omega_pear: Threshold for collinearity (default 0.9)
        
    Returns:
        Updated primitive object
    """
    if primitive.type == 'triangle':
        mu1, mu2, mu3 = primitive.vertices[0], primitive.vertices[1], primitive.vertices[2]
        
        # Step 1: Check if vertices are close
        d12 = np.linalg.norm(mu1 - mu2)
        d23 = np.linalg.norm(mu2 - mu3)
        d31 = np.linalg.norm(mu3 - mu1)
        max_dist = max(d12, d23, d31)
        
        if max_dist < omega_dist:
            # Prune mu2 and mu3, convert to ellipse at mu1
            primitive.type = 'ellipse'
            primitive.vertices = [mu1]
            # Note: Covariance must be preserved or recomputed for the ellipse
            return primitive
            
        # Step 2: Check if vertices are collinear
        # Form matrix V with vertices as rows
        V = np.vstack([mu1, mu2, mu3])
        # Compute Pearson correlation coefficient
        # Using numpy corrcoef which returns a matrix; take the off-diagonal element
        corr_matrix = np.corrcoef(V.T)
        # Handle potential NaNs if variance is zero
        if np.isnan(corr_matrix[0, 1]):
            pearson_val = 0.0
        else:
            pearson_val = abs(corr_matrix[0, 1])
            
        if pearson_val > omega_pear:
            # Convert triangle to line
            primitive.type = 'line'
            # Keep mu1 and mu2 as the line endpoints
            primitive.vertices = [mu1, mu2]
            # Note: Covariance must be adjusted for line representation
            return primitive
            
    elif primitive.type == 'line':
        mu1, mu2 = primitive.vertices[0], primitive.vertices[1]
        
        # Step 3: Check if line vertices are close
        dist = np.linalg.norm(mu1 - mu2)
        
        if dist < omega_dist:
            # Prune mu2, reduce to ellipse at mu1
            primitive.type = 'ellipse'
            primitive.vertices = [mu1]
            return primitive
            
    return primitive
```

**Edge Cases and Pitfalls:**
*   **Covariance Preservation:** When converting a triangle to a line or ellipse, or a line to an ellipse, the covariance matrix $\boldsymbol{\Sigma}$ must be appropriately transformed or re-initialized to maintain the visual appearance of the primitive. The source notes do not specify the exact mathematical transformation for $\boldsymbol{\Sigma}$ during these conversions [S1, unofficial].
*   **Collinearity Calculation:** The Pearson correlation coefficient calculation assumes the vertices are treated as variables. If vertices are identical or degenerate in a way that causes zero variance, the correlation is undefined (NaN). The pseudocode above handles this by defaulting to 0.0 [G].
*   **Interaction with Densification:** Vertex pruning does not reduce the representation to only point-centered ellipses because the original densification mechanism (cloning and splitting) continues to operate alongside pruning [S1, unofficial].

## Operating characteristics
*   **Storage Reduction:** The vertex pruning mechanism reduces the average storage size required to store vertex coordinates by 24% compared to the variant without pruning [S1, unofficial].
*   **Performance:** The method adds a lightweight geometric check during the pruning phase. It does not significantly increase rendering time as it only modifies the data structure during training updates.
*   **Quality:** Ablation studies show that the framework with vertex pruning performs better in surface reconstruction metrics (e.g., Chamfer distance) compared to the variant without it [S1, unofficial].

## Background
Vertex-level pruning was introduced in the context of Mixed-Primitive Gaussian Splatting (MP-GS), a method that extends standard 3D Gaussian Splatting by using Gaussian triangles and lines in addition to point-centered Gaussians [S1, unofficial]. The method was proposed to handle the increased complexity of mixed primitives, where standard opacity-based pruning is insufficient to remove geometrically redundant vertices [S1, unofficial].

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
