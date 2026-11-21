# Hierarchical Iso-Surface Extraction
Also called: KdTreeExtraction

## Overview
Hierarchical Iso-Surface Extraction is an adaptive algorithm that identifies active cells containing an isosurface within a scalar field represented by an Implicit Neural Representation (INR) by traversing a k-d tree hierarchy. It differs from uniform sampling methods by querying the INR only at the corners of active cells, significantly reducing computational cost, and differs from standard k-d tree extraction by using uncertainty propagation to prune non-active regions before subdivision.

## How it works
**Inputs:**
*   $f_\theta$: An INR mapping $\mathbb{R}^3 \rightarrow \mathbb{R}$ representing the scalar field.
*   $x_l, x_u \in \mathbb{R}^3$: The lower and upper bounds of the domain (bounding box).
*   $c$: The target iso-value.
*   $t$: Hyperparameter controlling the confidence interval width.
*   $D_{max}$: Maximum depth of the k-d tree.

**Outputs:**
*   $A$: A set of active cells, where each cell is defined by its lower and upper bounds $(x_l, x_u)$.

**Data Structures:**
*   **PAF (Probabilistic Affine Form):** A vector representing a region as an affine combination of uniformly distributed random variables. A 3D box centered at $x_c$ with half-widths $x_1, y_1, z_1$ is represented by $\hat{X} = x_c + V \epsilon$, where $V$ is a diagonal matrix of orthogonal axes and $\epsilon$ represents normalized random variables.
*   **Active Cell:** A leaf node in the k-d tree where the iso-value $c$ is within the estimated confidence interval and the maximum depth is reached.

**Parameters:**
*   $t$: Controls the confidence interval $[\mu - t\sigma, \mu + t\sigma]$. Larger $t$ yields more conservative bounds (fewer active cells); smaller $t$ yields tighter bounds (more active cells). [S1]
*   $D_{max}$: Must be a multiple of 3 to ensure active cells are cubes, which is required for standard marching cubes implementations. [S1]

**Steps:**
1.  **Calculate Node Representation:** Compute the node center $x_c = (x_l + x_u)/2$. Compute the diagonal matrix $V$ for orthogonal axes, where the diagonal elements are $\frac{1}{\sqrt{3}}(x_u - x_c)$. Construct the PAF vector $\hat{X} = x_c + V \epsilon$. [S1]
2.  **Uncertainty Propagation:** Propagate $\hat{X}$ through the INR $f_\theta$ to obtain the output PAF $\hat{Y}$. This involves sequentially applying linear and activation functions to the PAF. [S1]
3.  **Estimate Distribution:** Apply the Central Limit Theorem (CLT) to $\hat{Y}$ to estimate the mean $\mu$ and standard deviation $\sigma$ of the scalar values within the node region, assuming a Gaussian distribution. [S1]
4.  **Check Confidence Interval:** If $c < \mu - t\sigma$ or $c > \mu + t\sigma$, the node is non-active. Return the empty set $\emptyset$. [S1]
5.  **Check Depth:** If the current depth equals $D_{max}$, the node is an active cell. Return the set $\{(x_l, x_u)\}$. [S1]
6.  **Subdivide:** If not at max depth, split the bounding box along the widest dimension into $x_l'$ and $x_u'$. Recursively call the algorithm on both children and return the union of their results: $A_{left} \cup A_{right}$. [S1]
7.  **Mesh Extraction:** After collecting all active cells, query the INR for scalar values at the 8 grid corners of each active cell. Apply the marching cubes algorithm to extract triangle meshes. [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def KdTreeExtraction(f_theta, x_l, x_u, c, t, max_depth, current_depth):
    """
    Recursively finds active cells containing the isosurface.
    
    Args:
        f_theta: The INR function.
        x_l: Lower bound of the current node (vector).
        x_u: Upper bound of the current node (vector).
        c: Iso-value.
        t: Confidence interval hyperparameter.
        max_depth: Maximum allowed depth.
        current_depth: Current recursion depth.
        
    Returns:
        Set of active cells, where each cell is a tuple (cell_x_l, cell_x_u).
    """
    # 1. Calculate Node Center and PAF
    x_c = (x_l + x_u) / 2.0
    
    # V is a diagonal matrix with elements 1/sqrt(3) * (x_u - x_c)
    # In practice, this scales the random variables epsilon
    # We represent the region as a PAF. For implementation, we need
    # to propagate the uncertainty. The source defines X_hat = x_c + V * epsilon.
    # We assume a helper function propagate_uncertainty that takes the PAF parameters
    # (center and scale) and the INR, returning the output PAF parameters (mu, sigma).
    
    # Calculate scale factors for the PAF
    # The source uses V = Diag(1/sqrt(3) * (x_u - x_c))
    # The uniform distribution on [-x1, x1] has std dev x1/sqrt(3).
    # So the scale factor for each dimension i is (x_u[i] - x_c[i]) / sqrt(3)
    scale = (x_u - x_c) / (3.0 ** 0.5)
    
    # 2. Uncertainty Propagation
    # Propagate the PAF defined by center x_c and scale 'scale' through f_theta
    # This returns the mean mu and std dev sigma of the output distribution
    mu, sigma = propagate_uncertainty(f_theta, x_c, scale)
    
    # 3. Check Confidence Interval
    if c < (mu - t * sigma) or c > (mu + t * sigma):
        return set()
    
    # 4. Check Max Depth
    if current_depth >= max_depth:
        return {(x_l, x_u)}
    
    # 5. Subdivide
    # Split along the widest dimension
    dims = x_u - x_l
    widest_dim = argmax(dims)
    
    mid = (x_l[widest_dim] + x_u[widest_dim]) / 2.0
    
    # Create left child bounds
    x_l_left = x_l.copy()
    x_u_left = x_u.copy()
    x_u_left[widest_dim] = mid
    
    # Create right child bounds
    x_l_right = x_l.copy()
    x_u_right = x_u.copy()
    x_l_right[widest_dim] = mid
    
    # Recursive calls
    A_left = KdTreeExtraction(f_theta, x_l_left, x_u_left, c, t, max_depth, current_depth + 1)
    A_right = KdTreeExtraction(f_theta, x_l_right, x_u_right, c, t, max_depth, current_depth + 1)
    
    return A_left.union(A_right)

def extract_mesh(active_cells, f_theta):
    """
    Queries INR at corners of active cells and runs marching cubes.
    """
    vertices = []
    faces = []
    
    for (x_l, x_u) in active_cells:
        # Get 8 corners
        corners = get_corners(x_l, x_u)
        
        # Query INR at corners
        values = [f_theta(corner) for corner in corners]
        
        # Run marching cubes on this cell
        # Note: Standard marching cubes expects a 2x2x2 grid of values.
        # The source states: "perform the marching cubes algorithm to extract 
        # the triangle meshes inside the cells."
        cell_faces = marching_cubes(values, corners)
        
        # Add to global mesh
        # (Implementation detail: handle vertex indexing)
        pass
        
    return vertices, faces
```

**Pitfalls:**
*   **Cube Constraint:** The maximum depth must be a multiple of 3. If not, the active cells will not be cubes, which may cause issues with standard marching cubes implementations that assume uniform cubic cells. [S1]
*   **Uncertainty Propagation:** The accuracy of the pruning depends on the validity of the CLT approximation for the output PAF. If the INR has strong non-linearities, the Gaussian assumption may be poor. [S1]

## Operating characteristics
*   **Time Complexity:** Depends on the number of active cells and the cost of INR evaluation. By pruning non-active regions, it avoids evaluating the INR everywhere. However, uncertainty propagation is more expensive than a single INR query. [S1]
*   **Memory Complexity:** Proportional to the number of active cells and the depth of the k-d tree. [S1]
*   **Strengths:** Reduces computation by querying the INR only at active cell corners. Adaptive resolution based on the iso-value location. [S1]
*   **Limitations:** Requires uncertainty propagation, which is computationally heavier than simple sampling. The quality of the Gaussian approximation affects pruning efficiency. [S1]
*   **When to use:** When the INR evaluation is expensive and the isosurface is sparse or localized. [S1]
*   **When not to use:** When the isosurface is dense and fills the domain, as the overhead of uncertainty propagation may outweigh the benefits. [S1]

## Background
The method was proposed in "Improving Efficiency of Iso-Surface Extraction on Implicit Neural Representations Using Uncertainty Propagation" (2024) by authors from the IEEE Transactions on Visualization and Computer Graphics. It addresses the inefficiency of dense reconstruction from INRs by using a k-d tree to adaptively subdivide the domain. The technique builds on uncertainty propagation through INRs, treating input regions as probabilistic affine forms. [S1]

## Sources
[S1] Improving Efficiency of Iso-Surface Extraction on Implicit Neural Representations Using Uncertainty Propagation, https://doi.org/10.1109/tvcg.2024.3365089, fetched 2026-10-26
