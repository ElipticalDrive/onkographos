# Vertex-level Pruning in Mixed-Primitive Gaussian Splatting

Also called: Mixed-Primitive Vertex Pruning

## Overview
Vertex-level pruning is a refinement mechanism for Mixed-Primitive Gaussian Splatting (MP-GS) that removes redundant vertices from Gaussian triangles and lines during training to reduce storage overhead and simplify the representation [S1, unofficial]. Unlike standard Gaussian Splatting which only performs primitive-level opacity pruning, this method dynamically downgrades the geometric complexity of primitives (e.g., triangle to line or ellipse) based on vertex proximity and collinearity [S1, unofficial]. It is chosen when memory efficiency and representation cleanliness are prioritized over the maximum possible geometric fidelity of complex primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of mixed-primitive Gaussians (Gaussian triangles, Gaussian lines, Gaussian ellipses) with their current vertex positions $\mu_i$, covariance parameters, and opacity $\omega$.
- **Output:** The same set of primitives, with some primitives converted to lower-vertex-count primitives (e.g., triangle $\to$ line, triangle/line $\to$ ellipse) and redundant vertices removed.

**Data Structures:**
Each primitive stores its vertices. A Gaussian triangle has 3 vertices ($\mu_1, \mu_2, \mu_3$). A Gaussian line has 2 vertices ($\mu_1, \mu_2$). A Gaussian ellipse has 1 vertex ($\mu_1$).

**Parameters:**
- $\omega_{dist}$: Threshold for vertex proximity. Default: 0.5 [S1, unofficial].
- $\omega_{pear}$: Threshold for collinearity. Default: 0.9 [S1, unofficial].

**Steps:**
1. **Trigger:** Vertex-level pruning is executed whenever primitive-level pruning is performed during the training loop [S1, unofficial].
2. **Proximity Check:** For any set of vertices in a primitive, calculate the maximum pairwise L2 distance. If this distance is less than $\omega_{dist}$, the vertices are considered "close" [S1, unofficial].
3. **Collinearity Check:** For a Gaussian triangle, compute the Pearson correlation coefficient of the three vertex positions. If the absolute value exceeds $\omega_{pear}$, the vertices are considered "nearly collinear" [S1, unofficial].
4. **Pruning Actions:**
   - **Case 1 (Triangle, Close Vertices):** If a Gaussian triangle's three vertices are all close to each other (max pairwise L2 < $\omega_{dist}$), prune vertices $\mu_2$ and $\mu_3$. Convert the primitive into a Gaussian ellipse centered at $\mu_1$ [S1, unofficial].
   - **Case 2 (Triangle, Collinear Vertices):** If a Gaussian triangle's vertices are not close but are nearly collinear (Pearson correlation > $\omega_{pear}$), convert the primitive into a Gaussian line [S1, unofficial].
   - **Case 3 (Line, Close Vertices):** If a Gaussian line's two vertices are close (L2 distance < $\omega_{dist}$), prune vertex $\mu_2$. Convert the primitive into a Gaussian ellipse centered at $\mu_1$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_level_pruning(primitive, omega_dist=0.5, omega_pear=0.9):
    """
    Prunes vertices from a mixed-primitive Gaussian based on proximity and collinearity.
    primitive: object with type ('triangle', 'line', 'ellipse') and vertices (list of 3D vectors).
    Returns: updated primitive.
    """
    if primitive.type == 'triangle':
        mu1, mu2, mu3 = primitive.vertices
        # Check if all vertices are close
        d12 = l2_distance(mu1, mu2)
        d13 = l2_distance(mu1, mu3)
        d23 = l2_distance(mu2, mu3)
        max_dist = max(d12, d13, d23)
        
        if max_dist < omega_dist:
            # Case 1: Convert triangle to ellipse
            primitive.type = 'ellipse'
            primitive.vertices = [mu1]
            return primitive
            
        # Check if vertices are nearly collinear
        # Compute Pearson correlation coefficient for the 3 points
        # Treat coordinates as a 3x3 matrix where rows are vertices
        # Or flatten to 1D arrays for correlation calculation
        x = [mu1.x, mu2.x, mu3.x]
        y = [mu1.y, mu2.y, mu3.y]
        z = [mu1.z, mu2.z, mu3.z]
        
        # Calculate Pearson correlation for each dimension pair or overall?
        # The source says "Pearson correlation coefficient computed from these vertices".
        # Typically this implies checking if the points lie on a line in 3D space.
        # A robust way is to check if the covariance matrix has rank 1, or use correlation of coordinates.
        # We assume a scalar metric derived from the 3 points.
        # Let's use the correlation of the x-coordinates against the index, or similar.
        # However, a simpler interpretation often used in such contexts is the correlation between 
        # the projection of points onto the principal axis. 
        # Given the ambiguity, we implement a standard Pearson correlation on the flattened coordinates 
        # or assume a specific implementation detail not fully specified. 
        # We will use a placeholder for the specific correlation calculation method [G].
        # [G] A common approach is to compute the correlation between the coordinates of the points 
        # when sorted by their projection onto the vector (mu2-mu1). 
        # For simplicity, we check if the absolute Pearson correlation of the x, y, z coordinates 
        # across the 3 points exceeds the threshold. Note: Pearson correlation of 3 points is 
        # ill-defined if variance is 0. We assume non-degenerate inputs.
        
        corr_x = pearson_correlation([mu1.x, mu2.x, mu3.x], [1, 2, 3]) # Dummy index
        corr_y = pearson_correlation([mu1.y, mu2.y, mu3.y], [1, 2, 3])
        corr_z = pearson_correlation([mu1.z, mu2.z, mu3.z], [1, 2, 3])
        
        # The source says "absolute value of the Pearson correlation coefficient".
        # It likely refers to a single scalar metric. Let's assume it checks if the points 
        # are collinear by checking if the correlation of their positions is high.
        # We will use the maximum absolute correlation among dimensions as a proxy [G].
        max_corr = max(abs(corr_x), abs(corr_y), abs(corr_z))
        
        if max_corr > omega_pear:
            # Case 2: Convert triangle to line
            primitive.type = 'line'
            primitive.vertices = [mu1, mu2] # Keep first two
            return primitive
            
    elif primitive.type == 'line':
        mu1, mu2 = primitive.vertices
        dist = l2_distance(mu1, mu2)
        
        if dist < omega_dist:
            # Case 3: Convert line to ellipse
            primitive.type = 'ellipse'
            primitive.vertices = [mu1]
            return primitive
            
    return primitive

def l2_distance(p1, p2):
    return sqrt((p1.x - p2.x)**2 + (p1.y - p2.y)**2 + (p1.z - p2.z)**2)

def pearson_correlation(x, y):
    # Standard Pearson correlation formula
    n = len(x)
    mean_x = sum(x) / n
    mean_y = sum(y) / n
    num = sum((xi - mean_x) * (yi - mean_y) for xi, yi in zip(x, y))
    den_x = sqrt(sum((xi - mean_x)**2 for xi in x))
    den_y = sqrt(sum((yi - mean_y)**2 for yi in y))
    if den_x == 0 or den_y == 0:
        return 0.0
    return num / (den_x * den_y)
```

**Edge Cases and Pitfalls:**
- **Degenerate Input:** If vertices are identical, L2 distance is 0, triggering ellipse conversion. If variance in coordinates is 0, Pearson correlation is undefined; handle with a guard [G].
- **Boundary Conditions:** The thresholds $\omega_{dist}$ and $\omega_{pear}$ are sensitive. Too low $\omega_{dist}$ may prune too aggressively; too high may fail to prune. [S1, unofficial]

## Operating characteristics
- **Storage Reduction:** Reduces average storage size for vertex coordinates by 24% compared to the variant without pruning [S1, unofficial].
- **Performance:** No significant rendering performance impact reported; primarily affects training memory and final model size [S1, unofficial].
- **Strengths:** Simplifies representation, reduces memory footprint, removes "messy" redundant vertices [S1, unofficial].
- **Limitations:** May reduce geometric fidelity if primitives are pruned too aggressively; requires careful tuning of thresholds [S1, unofficial].
- **When to Use:** When memory is constrained or when the scene contains many degenerate or redundant primitives (e.g., flat surfaces represented by triangles) [S1, unofficial].

## Background
Vertex-level pruning was introduced in the Mixed-Primitive-based Gaussian Splatting (MP-GS) method for surface reconstruction [S1, unofficial]. It addresses the limitation of standard Gaussian Splatting, which only uses point-centered primitives, by allowing triangles and lines, but recognizing that these can become redundant during training [S1, unofficial]. The method was developed to refine the representation by eliminating unnecessary vertices, complementing the original densification mechanism [S1, unofficial].

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
