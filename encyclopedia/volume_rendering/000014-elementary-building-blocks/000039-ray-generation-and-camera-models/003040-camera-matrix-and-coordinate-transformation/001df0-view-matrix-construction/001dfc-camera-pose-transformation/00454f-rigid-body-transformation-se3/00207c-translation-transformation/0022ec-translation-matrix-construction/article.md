# Translation Matrix Construction

Also called: Mat.translation, Translation Matrix

## Overview
Constructs a 4x4 homogeneous transformation matrix that represents a pure translation by a given 4D vector. It solves the problem of embedding a 3D displacement vector into a 4x4 linear algebra structure compatible with standard graphics pipelines, allowing translation to be composed with other affine transformations via matrix multiplication. It differs from [Coordinate Translation](2281-translation-transformation.md) by operating on matrix algebra rather than directly manipulating coordinate data frames.

## How it works
**Inputs:** A 4D vector $v = [v_x, v_y, v_z, v_w]^T$. In practice, $v_w$ is ignored for the translation components, and the vector is treated as a direction or point offset.
**Outputs:** A 4x4 matrix $M$ in row-major or column-major layout (implementation dependent, but standard OpenGL uses column-major storage for the resulting matrix object).

**Data Structure:**
The matrix $M$ is a 4x4 array of floats.
$$
M = \begin{bmatrix}
1 & 0 & 0 & v_x \\
0 & 1 & 0 & v_y \\
0 & 0 & 1 & v_z \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

**Steps:**
1. Initialize a 4x4 matrix $M$ with zeros.
2. Set the diagonal elements $M_{0,0}, M_{1,1}, M_{2,2}$ to $1.0$.
3. Set the translation components:
   - $M_{0,3} = v_x$
   - $M_{1,3} = v_y$
   - $M_{2,3} = v_z$
4. Set the bottom row $M_{3,0}, M_{3,1}, M_{3,2}$ to $0.0$ and $M_{3,3}$ to $1.0$.
5. Return $M$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def translation_matrix(v: list[float]) -> list[list[float]]:
    # Initialize 4x4 matrix with zeros
    mat = [[0.0 for _ in range(4)] for _ in range(4)]
    
    # Set identity part
    mat[0][0] = 1.0
    mat[1][1] = 1.0
    mat[2][2] = 1.0
    
    # Set translation components from input vector v
    # v is expected to have at least 3 elements [x, y, z]
    mat[0][3] = v[0]
    mat[1][3] = v[1]
    mat[2][3] = v[2]
    
    # Set bottom row for homogeneous coordinates
    mat[3][3] = 1.0
    
    return mat
```

**Parameters:**
- `v`: A list or array of 3 or 4 floats representing the translation vector.
- Typical values: Any real numbers.

**Edge Cases:**
- If $v$ has fewer than 3 components, the missing components are treated as 0 [G].
- Precision errors in accumulation of multiple translations can lead to drift; use double precision for high-accuracy requirements [G].

## Subtopics

### Coordinate Translation
A method that directly modifies coordinate data frames by adding offset values to x, y, and z columns. It is distinct from matrix construction as it operates on tabular data structures rather than linear algebra objects. It is chosen when working with R-style data frames or when avoiding matrix overhead is preferred. It returns a data frame of transformed coordinates [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$, as it involves a fixed number of assignments to a 4x4 matrix.
- **Memory Complexity:** $O(1)$, storing a fixed 4x4 matrix.
- **Strengths:** Simple, fast, and standard for GPU shader uniforms.
- **Limitations:** Does not handle scaling or rotation; must be combined with other matrices for complex transformations.
- **When to use:** When applying a simple positional offset in a graphics pipeline.
- **When not to use:** When direct coordinate manipulation is required without matrix overhead, or when non-linear transformations are needed.

## Background
The concept of using a 4x4 matrix for translation in homogeneous coordinates was introduced by William Kingdon Clifford and later popularized in computer graphics by Edwin Catmull and others in the 1970s. The specific implementation as a utility function `Mat.translation` is found in modern WebGL utility libraries [S1, unofficial]. The related `coord_translate` function in R packages like `isocubes` provides a data-frame oriented alternative [S2, unofficial].

## Sources
[S1] chloearluck/volume_rendering, https://github.com/chloearluck/volume_rendering, fetched 2026-10-24 (unofficial)
[S2] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
