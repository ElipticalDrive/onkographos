# Simplex-based Encoding
Also called: Simplex Structures, Simplex-based structure encoding

## Overview
Simplex-based encoding is a method for representing graphics primitives (such as images, Signed Distance Fields, and NeRFs) using simplex-based structures instead of traditional grid-based structures [S1, unofficial]. It solves the problem of exponential computational overhead ($2^n$ vertices) inherent in grid-based representations by reducing the number of vertices per cell to linear scaling ($n+1$ vertices) with dimensionality [S1, unofficial]. It sets itself apart from its siblings, such as [Simplex Noise](978-simplex-noise.md) which generates scalar fields, by using the simplex topology specifically for efficient feature storage and interpolation in high-dimensional encoding spaces, often combined with hash tables and neural networks [S1, unofficial].

## How it works
**Inputs:** Input coordinates $\mathbf{x} \in \mathbb{R}^n$ for a point in n-dimensional space; a pre-computed simplex grid with gradient vectors or features assigned to vertices.
**Outputs:** Interpolated feature vector or scalar value at $\mathbf{x}$.

**Data Structures:**
*   **Simplicial Grid:** A lattice of points derived from skewed coordinates. Unlike a regular grid, cells are simplices (triangles in 2D, tetrahedra in 3D).
*   **Hash Table:** Stores multiresolution features for all interest points in the simplicial grid. This allows for sparse storage and efficient lookup [S1, unofficial].
*   **Gradient/Feature Vectors:** Assigned to each vertex of the simplex grid. In the context of encoding, these are typically learned parameters passed into a neural network [S1, unofficial].

**Steps:**

1.  **Coordinate Skewing:** Transform the input Cartesian coordinate $\mathbf{x}$ into a simplex coordinate $\mathbf{x}'$ using an affine transformation. This maps the unit n-hypercube into an n-parallelpiped, effectively skewing the grid so that the cells become regular simplices [S1, unofficial].
    $$ \mathbf{x}' = \mathbf{x} + \mathbf{1}_n^T \cdot F_n \sum_i x_i $$
    Where:
    *   $\mathbf{1}_n$ is a vector of ones of length $n$.
    *   $F_n = \frac{\sqrt{n+1}-1}{n}$ is the skewing factor [S1, unofficial].
    *   $\sum_i x_i$ is the sum of the components of $\mathbf{x}$ [S1, unofficial].

2.  **Simplicial Subdivision (Cell Identification):** Determine the containing simplex for the skewed point $\mathbf{x}'$.
    *   Compute the floor $\lfloor \mathbf{x}' \rfloor$ and ceiling $\lceil \mathbf{x}' \rceil$ of the skewed coordinates. The point lies within a hypercube spanned by these integer coordinates.
    *   Sort the coordinates of $\mathbf{x}'$ in decreasing order to determine the "skew" direction. This sorting identifies the order in which the simplex boundaries are crossed [S1, unofficial].
    *   Start with the base vertex $\mathbf{v}_0 = \lfloor \mathbf{x}' \rfloor$.
    *   Successively add 1 to the coordinate index corresponding to the largest remaining component in the sorted order to find the subsequent vertices $\mathbf{v}_1, \dots, \mathbf{v}_n$.
    *   This yields $n+1$ vertices defining the containing simplex [S1, unofficial].

3.  **Interpolation (Noise Evaluation / Feature Sampling):**
    *   Retrieve the feature vectors or gradient values assigned to the $n+1$ vertices of the identified simplex from the hash table or grid [S1, unofficial].
    *   Perform barycentric interpolation of these values using the skewed coordinates $\mathbf{x}'$ relative to the simplex vertices. The specific weights depend on the distance from $\mathbf{x}'$ to each vertex within the simplex space [S1, unofficial].
    *   *Note:* In the context of pure Simplex Noise, this step involves dotting the gradient vectors with the distance vectors from the point to the vertices. In Simplex-based Encoding, this step yields the raw feature representation [S1, unofficial].

4.  **Unskewing (Kernel Summation):** If the output requires mapping back to Cartesian space for further processing or visualization, or if the interpolation weights require Cartesian distance metrics, unskew the coordinate. However, for pure feature encoding, the interpolated value in the simplex space is often used directly or passed to a neural network [S1, unofficial].
    $$ \mathbf{x} = \mathbf{x}' - \mathbf{1}_n^T \cdot G_n \sum_i x'_i $$
    Where:
    *   $G_n = \frac{1 - 1/\sqrt{n+1}}{n}$ is the unskewing factor [S1, unofficial].

**Pseudocode:**

```python
import math

def simplex_encode_coordinate_skew(x, n):
    """
    Skews input coordinate x from Cartesian to Simplex space.
    x: list or array of n floats
    n: dimensionality
    Returns: list of n floats in simplex space
    """
    # Calculate skew factor F_n
    F_n = (math.sqrt(n + 1) - 1) / n
    
    # Calculate sum of coordinates
    sum_x = sum(x)
    
    # Apply skewing: x' = x + 1_n^T * F_n * sum(x_i)
    # This adds F_n * sum_x to every component of x
    x_prime = [xi + F_n * sum_x for xi in x]
    
    return x_prime

def simplex_encode_coordinate_unskew(x_prime, n):
    """
    Unskews coordinate x_prime from Simplex to Cartesian space.
    x_prime: list or array of n floats
    n: dimensionality
    Returns: list of n floats in Cartesian space
    """
    # Calculate unskew factor G_n
    G_n = (1 - 1 / math.sqrt(n + 1)) / n
    
    # Calculate sum of skewed coordinates
    sum_x_prime = sum(x_prime)
    
    # Apply unskewing: x = x' - 1_n^T * G_n * sum(x'_i)
    x = [xp - G_n * sum_x_prime for xp in x_prime]
    
    return x

def simplex_encode_find_simplex_vertices(x_prime, n):
    """
    Determines the n+1 vertices of the simplex containing x_prime.
    x_prime: list of n floats in simplex space
    n: dimensionality
    Returns: list of n+1 vertex coordinates (each a list of n ints)
    """
    # Step 1: Floor and Ceiling
    floor_coords = [math.floor(xp) for xp in x_prime]
    # ceil_coords = [math.ceil(xp) for xp in x_prime] # Implicitly floor + 1
    
    # Step 2: Sort coordinates to determine skew order
    # Create a list of (value, index) pairs to sort by value descending
    sorted_indices = sorted(range(n), key=lambda i: x_prime[i], reverse=True)
    
    # Step 3: Construct vertices
    # Start with the floor vector
    vertices = []
    current_vertex = list(floor_coords)
    vertices.append(current_vertex)
    
    # Successively add 1 to the coordinate index in the sorted order
    for idx in sorted_indices:
        # Create a copy of the current vertex
        next_vertex = list(current_vertex)
        # Add 1 to the specific index
        next_vertex[idx] += 1
        vertices.append(next_vertex)
        current_vertex = next_vertex
        
    return vertices

def simplex_encode_interpolate(vertices, features, x_prime, n):
    """
    Interpolates features at the vertices using barycentric coordinates in simplex space.
    Note: The exact interpolation formula (barycentric weights) is not explicitly detailed 
    in the source beyond "interpolates the gradient vectors... to obtain a weighted sum".
    Standard simplex interpolation uses distances from x_prime to vertices.
    
    vertices: list of n+1 vertex coordinates (list of n ints)
    features: list of n+1 feature vectors (or scalars) corresponding to vertices
    x_prime: list of n floats in simplex space
    n: dimensionality
    Returns: interpolated feature vector or scalar
    """
    # Placeholder for specific barycentric interpolation logic.
    # In Simplex Noise, this involves dot products with gradients.
    # In Encoding, this is likely a weighted sum based on distance in simplex space.
    # Since the source does not provide the explicit weight formula, 
    # we assume a standard linear interpolation over the simplex.
    
    # For implementation completeness, a common approach for simplex interpolation
    # is to compute weights based on the distance from x_prime to each vertex,
    # normalized such that they sum to 1.
    
    # Calculate distances squared from x_prime to each vertex
    distances_sq = []
    for v in vertices:
        dist_sq = sum((x_prime[i] - v[i])**2 for i in range(n))
        distances_sq.append(dist_sq)
        
    # Avoid division by zero if x_prime is exactly on a vertex
    # Use inverse distance weighting or barycentric coordinates derived from geometry.
    # The source mentions "barycentric interpolation scheme".
    # Without the explicit formula in the source, we note this as a gap [G].
    # A robust implementation would compute barycentric coordinates lambda_i
    # such that x_prime = sum(lambda_i * v_i) and sum(lambda_i) = 1.
    
    # For the purpose of this article, we state that the weighted sum is computed.
    # result = sum(weight_i * feature_i for i in range(n+1))
    
    return None # Implementation detail for weights is not provided in source [G]

## Operating characteristics
**Time Complexity:** The method scales with $O(n^2)$ complexity for $n$ dimensions, primarily due to the sorting step required for simplicial subdivision [S1, unofficial]. This is a significant improvement over grid-based structures (like those in Instant-NGP) which scale with $O(2^n)$ vertices per cell [S1, unofficial].

**Memory Complexity:** Uses hash tables to store multiresolution features, allowing for sparse storage of the simplicial grid. This reduces memory overhead compared to dense grid representations [S1, unofficial].

**Performance:**
*   In 2D image fitting tasks, the method fits a giga-pixel image with 9.4% less time compared to the instant-ngp baseline, while maintaining the same quality and compression rate [S1, unofficial].
*   In volumetric rendering setups, a maximum 41.2% speedup is observed when samples are dense enough [S1, unofficial].

**Strengths:**
*   Linear scaling of vertices with dimensionality ($n+1$ vs $2^n$) [S1, unofficial].
*   Reduced computational overhead in higher dimensions [S1, unofficial].
*   Isotropic properties inherited from simplex noise, reducing directional artifacts [S1, unofficial].

**Limitations:**
*   Requires careful implementation of coordinate skewing and unskewing [S1, unofficial].
*   Regular simplices cannot tile space in dimensions > 2 without skewing, necessitating the complex transformation logic [S1, unofficial].
*   The exact barycentric interpolation weights for general feature vectors are not explicitly detailed in the source, requiring derivation or assumption [G].

## Background
Simplex-based encoding was proposed in the paper "Efficient Encoding of Graphics Primitives with Simplex-based Structures" [S1, unofficial]. It builds upon the foundations of Simplex Noise, which was originally introduced to address the directional artifacts and computational cost of Perlin Noise [S1, unofficial]. The key insight is that while regular simplices cannot tile space in dimensions greater than 2, the skewed coordinate system used in Simplex Noise effectively creates a tiling of regular simplices in the transformed space [S1, unofficial]. This allows for efficient sampling and interpolation of graphics primitives, which are then parameterized by a tiny fully connected neural network [S1, unofficial]. The method was implemented in C++ and CUDA [S1, unofficial].

## Sources
[S1] Efficient Encoding of Graphics Primitives with Simplex-based Structures, https://doi.org/10.48550/arxiv.2311.15439, fetched 2026-10-26 (unofficial)
