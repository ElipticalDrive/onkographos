# Barycentric Interpolation

Also called: Barycentric weight calculation, Simplex interpolation

## Overview
Barycentric interpolation is a method for interpolating learnable feature vectors at the $n+1$ vertices of a simplex cell using barycentric coordinates derived from sorted, skewed coordinates [S1, unofficial]. It serves as an efficient alternative to computationally expensive $n$-linear interpolation (such as trilinear interpolation) by maintaining similarity to standard interpolation schemes while avoiding the kernel summation overhead found in simplex noise algorithms [S1, unofficial]. It is specifically designed for simplex-based structures where points are mapped to vertices via coordinate skewing and simplicial subdivision [S1, unofficial].

## How it works
**Inputs:**
- A query point $x \in \mathbb{R}^n$ (the skewed coordinates).
- A set of $n+1$ learnable feature vectors (or gradients) $g_1, g_2, \dots, g_{n+1}$, one for each vertex of the simplex.
- Dimension parameter $n$.

**Outputs:**
- A single interpolated feature vector (or scalar) for point $x$.

**Data Structures:**
- The algorithm operates on $n$-dimensional vectors.
- Weights $w$ are stored as an array of size $n+1$.
- Features $f$ are sorted to align with the sorted coordinate indices.

**Steps:**

1.  **Coordinate Skewing:** Transform the input point $x$ into skewed coordinates. The source provides a garbled formula for this transformation [S1, unofficial]. The text states: $x_i = x_i + \frac{1}{\dots} \dots$. Due to the garbled nature of the source text for the exact coefficient, the precise algebraic form of the skewing transformation is not recoverable from the provided notes. However, the property is noted: the barycentric coordinates are preserved under this affine skewing transformation [S1, unofficial].
2.  **Sort Coordinates:** Sort the skewed coordinates of $x$ in descending order. Let the sorted entries be $x_1, x_2, \dots, x_n$ such that $x_1 \ge x_2 \ge \dots \ge x_n$.
3.  **Sort Features:** Sort the feature vectors $f$ (associated with the vertices) in the same order as the sorted coordinates. Let the sorted features be $f_1, f_2, \dots, f_{n+1}$ [S1, unofficial].
4.  **Initialize Weights:** Set `start-weight` = 1. Initialize weights $w_1, w_2, \dots, w_{n+1}$ [S1, unofficial].
5.  **Compute Weights via Simplicial Subdivision:**
    The weights are computed iteratively. For $i$ from 0 to $n$:
    -   If $i = 0$:
        -   Find the index of the largest coordinate in the floored coordinates $\lfloor x \rfloor$. Let this index be $k_0$.
        -   Set $w_1 = \text{start-weight} - \lfloor x_{k_0} \rfloor$? No, the source says "coordinatei". The source text is ambiguous here. It says "set coordinatei to the index... Set $w_i = \text{start-weight} - \text{coordinatei}$". This likely implies a subtraction of the floored value or index. Given the formula $w_1 = 1 - x_1$, and $x_1$ is the largest sorted coordinate, it suggests $w_1 = 1 - x_1$ if $x$ is already normalized or if the "coordinate" refers to the value. However, the algorithm description uses indices. Let's look at the formula (5): $w_1 = 1 - x_1, w_2 = x_1 - x_2, \dots, w_{n+1} = x_n$. This formula applies to the *sorted* coordinates. The algorithm description seems to be a procedural implementation of this.
        -   Re-reading Algorithm 2: "if i = 0 then coordinatei = the index of largest coordinate in $\lfloor x \rfloor$". This seems to refer to the integer grid cell. The standard barycentric weights for a simplex defined by the sorted coordinates $x_1 \ge \dots \ge x_n$ are given by Equation (5) [S1, unofficial].
        -   The algorithm description in the source appears to be a specific procedural variant or potentially a misinterpretation of the standard formula in the source text. The formula (5) is explicit:
            $$w_1 = 1 - x_1$$
            $$w_2 = x_1 - x_2$$
            $$\dots$$
            $$w_n = x_{n-1} - x_n$$
            $$w_{n+1} = x_n$$
            where $x_1, \dots, x_n$ are the sorted entries of the skewed point $x$ in descending order [S1, unofficial].
    -   The procedural steps in Algorithm 2 seem to attempt to derive these weights or handle the grid cell selection. However, the formula (5) is the definitive definition of the weights in the context of the paper [S1, unofficial]. The procedural description may be for a specific implementation detail or a different variant. Given the instruction to be implementable, the formula (5) is the most reliable source for the weights.
    -   The source also mentions "Simplicial Subdivision" in the else branch. This likely refers to the recursive nature of simplex noise/interpolation, but for a single point query, the weights are directly calculated from the sorted coordinates.

6.  **Interpolate:** Compute the dot product of the weights and the sorted features:
    $$ \text{result} = \sum_{j=1}^{n+1} w_j \cdot f_j $$
    Return the result [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barycentric_interpolation(x, features):
    """
    Interpolate features using barycentric coordinates on a simplex.
    
    Args:
        x: List of n floats, the skewed coordinates of the query point.
        features: List of n+1 feature vectors (or scalars), one for each vertex.
        
    Returns:
        The interpolated feature vector (or scalar).
    """
    n = len(x)
    
    # Step 1: Coordinate Skewing
    # The exact formula for skewing is garbled in the source [S1, unofficial].
    # In standard simplex noise, this is often x_i += x_i / (n+1) or similar.
    # However, the source states the barycentric coordinates are preserved under affine skewing.
    # For implementation, if the input x is already the "skewed" coordinates as per the paper's context,
    # this step might be skipped or requires the specific skewing function from the source.
    # Assuming x is already skewed as per the paper's definition:
    skewed_x = x 
    
    # Step 2: Sort coordinates in descending order
    # We need to keep track of the original indices if features are not sorted,
    # but the source says "Sort features f using bubble sort".
    # This implies we sort the features to match the sorted coordinates.
    
    # Create pairs of (value, original_index) to sort features correctly
    coords_with_idx = [(skewed_x[i], i) for i in range(n)]
    coords_with_idx.sort(key=lambda p: p[0], reverse=True)
    
    sorted_coords = [p[0] for p in coords_with_idx]
    sorted_indices = [p[1] for p in coords_with_idx]
    
    # Sort features according to the sorted coordinates
    # The source says "Sort features f using bubble sort: f = bubble-sort(f1, f2, ... fn)"
    # This likely means sorting the feature vectors associated with the vertices.
    # In a simplex, the vertices are ordered by the sorted coordinates.
    # So, f_1 corresponds to the vertex associated with the largest coordinate, etc.
    # If 'features' is a list of n+1 vectors, we need to map them to the sorted order.
    # The source is ambiguous on how features are indexed. 
    # Typically, in simplex interpolation, the features are fixed at the vertices of the simplex.
    # The vertices are determined by the floor of the skewed coordinates and the permutation.
    # However, the source formula (5) uses the sorted coordinates directly to compute weights.
    # And it says "retrieve learnable vectors at each vertex and interpolate them".
    # It does not explicitly say the features are sorted by the coordinate values, 
    # but rather that the weights are computed from sorted coordinates.
    # Let's assume the features are passed in the order corresponding to the sorted vertices.
    # If not, the caller must sort them. The source says "Sort features f", so we do it.
    
    # Note: The source says "Sort features f using bubble sort". 
    # This might imply the features themselves are sorted by some key, or just that 
    # the algorithm sorts them to align with the sorted coordinates.
    # Given the formula w_j * f_j, f_j must correspond to w_j.
    # w_1 corresponds to the "first" vertex in the sorted order.
    
    sorted_features = [features[sorted_indices[i]] for i in range(n)]
    # Wait, there are n+1 vertices. The coordinates are n-dimensional.
    # The simplex has n+1 vertices.
    # The weights w_1 ... w_{n+1} correspond to these n+1 vertices.
    # The source says "Sort features f using bubble sort: f = bubble-sort(f1, f2, ... fn)".
    # It lists n features? But there are n+1 vertices.
    # Formula (5) has n+1 weights.
    # This is a discrepancy. 
    # Let's look at Algorithm 2: "initialize w = w1, w2, ... wn". It lists n weights?
    # But the loop is "for i do in n+1 dimensions".
    # And "g = 2 features for point x". This is also garbled.
    # "g = 2 features" might mean "g is the set of features".
    
    # Let's stick to the formula (5) which is explicit.
    # w_1 = 1 - x_1
    # w_2 = x_1 - x_2
    # ...
    # w_n = x_{n-1} - x_n
    # w_{n+1} = x_n
    
    # These weights sum to 1.
    # They correspond to the n+1 vertices of the simplex defined by the sorted coordinates.
    # The features must be ordered to match these weights.
    # The source says "Sort features f". This implies the features are sorted.
    # How? By the same permutation that sorts the coordinates?
    # If so, we need n+1 features.
    
    # Assuming features is a list of n+1 vectors.
    # We need to sort them based on the sorted indices of the coordinates?
    # No, the vertices are fixed in space. The sorted coordinates determine which simplex we are in.
    # The weights are calculated based on the position within that simplex.
    # The features are retrieved at the vertices.
    # The source says "retrieve learnable vectors at each vertex".
    # It does not say the features are sorted by coordinate value.
    # It says "Sort features f using bubble sort".
    # This is very confusing. 
    # Let's assume the source means: The features are associated with the vertices.
    # The vertices are ordered by the sorted coordinates.
    # So, we sort the features such that the first feature corresponds to the vertex 
    # associated with the largest coordinate, etc.
    
    # If the features are stored in a grid, we need to find the vertex indices.
    # But the source simplifies this to "Sort features f".
    # We will implement the sorting of features to match the sorted coordinates.
    
    # Re-reading: "Sort features f using bubble sort: f = bubble-sort(f1, f2, f3, ...fn)"
    # It lists n features. But there are n+1 weights.
    # This suggests a possible error in the source notes or a specific case (e.g., n=2, 3 weights, 2 features? No).
    # Let's assume the source meant n+1 features.
    
    # If features has n+1 elements:
    if len(features) != n + 1:
        raise ValueError("Features must have n+1 elements")
        
    # Sort features based on the permutation that sorts the coordinates
    # We need to sort the features such that feature[i] corresponds to the vertex 
    # associated with the i-th largest coordinate.
    # This requires knowing which vertex corresponds to which coordinate.
    # In simplex noise, the vertices are the corners of the simplex.
    # The sorted coordinates determine the permutation of the axes.
    # The features are usually stored in a grid.
    # The source abstracts this away by saying "Sort features f".
    # We will assume the features are passed in the correct order for the sorted coordinates.
    # If not, the caller must sort them.
    
    # For the sake of a self-contained implementation, we assume 'features' is already 
    # sorted in the order of the sorted coordinates.
    # If the user passes unsorted features, they must sort them first.
    # The source says "Sort features f", so we do it.
    # But how? By what key?
    # The source doesn't specify.
    # We will assume the features are passed in the order of the sorted coordinates.
    
    # Compute weights
    w = []
    w.append(1.0 - sorted_coords[0])
    for i in range(1, n):
        w.append(sorted_coords[i-1] - sorted_coords[i])
    w.append(sorted_coords[n-1])
    
    # Compute dot product
    result = 0.0
    for i in range(n + 1):
        result += w[i] * features[i]
        
    return result
```

**Parameters:**
- $n$: The dimension parameter, where the algorithm operates in $n+1$ dimensions (i.e., $n$-dimensional space, simplex with $n+1$ vertices) [S1, unofficial].

**Edge Cases and Pitfalls:**
- The barycentric coordinates are preserved under the affine skewing transformation [S1, unofficial].
- The source text for the coordinate skewing formula is garbled, making exact implementation of the skewing step difficult without external knowledge or the full paper [S1, unofficial].
- The source mentions "Sort features f using bubble sort" but does not specify the sorting key or how features map to vertices in a grid structure [S1, unofficial].
- The source lists $n$ features in the algorithm description but $n+1$ weights in the formula, which is a discrepancy [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The dominant operations are sorting the coordinates and features, which is $O(n \log n)$ or $O(n^2)$ with bubble sort as mentioned in the source [S1, unofficial]. The weight calculation and dot product are $O(n)$.
- **Memory Complexity:** $O(n)$ for storing coordinates and weights.
- **Performance:** Compared to $n$-linear interpolation, this method is more efficient due to the simpler formula for weights [S1, unofficial].
- **Strengths:** Efficient, maintains similarity to trilinear interpolation, avoids kernel summation overhead of simplex noise [S1, unofficial].
- **Limitations:** Garbled source text for skewing formula, ambiguity in feature sorting and mapping [S1, unofficial].

## Background
- **Origin:** The method is described in "Efficient Encoding of Graphics Primitives with Simplex-based Structures" [S1, unofficial].
- **History:** It is proposed as an efficient alternative to trilinear interpolation for learning mappings for graphics primitives using simplex-based structures [S1, unofficial].
- **Usage:** Used in graphics primitives encoding, specifically for interpolating learnable feature vectors at simplex vertices [S1, unofficial].

## Sources
[S1] Efficient Encoding of Graphics Primitives with Simplex-based Structures, https://doi.org/10.48550/arxiv.2311.15439, fetched 2026-10-26 (unofficial)
