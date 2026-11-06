# Vector Normalization

Also called: MaxIndex, Normalize 3-tuple, PV.unitize, Sun Direction Normalization

## Overview
Vector Normalization converts a 3D vector into a unit vector (magnitude of 1) while preserving its direction, or returns a zero vector if the input is degenerate [S3, unofficial]. It is a fundamental utility for preparing direction data (such as normals or sun vectors) for lighting calculations in volume rendering [S4, unofficial]. Unlike [Octahedral Normal Encoding](3373-octahedral-normal-encoding.md) which compresses direction into 2D, this method operates directly on 3D Cartesian coordinates [S1][S2, unofficial].

## How it works
**Inputs:** A 3D vector $v = (v_x, v_y, v_z)$ of type $T$ (typically `float` or `double`) [S1][S3, unofficial].
**Outputs:** A normalized 3D vector $u$ of type $T$ [S2, S3, unofficial].

**Data Structures:**
The input is a simple 3-component vector. No complex structures are required for the basic operation. For grid-based operations, a dictionary or sparse grid structure maps coordinates $(c, v)$ to vectors [S3, unofficial].

**Parameters:**
- $\epsilon(T)$: A small epsilon value dependent on the precision of $T$ (e.g., $10^{-7}$ for float, $10^{-15}$ for double) used to detect near-zero magnitudes [S3, unofficial].

**Steps:**
1. Compute the Euclidean magnitude (norm) $n$ of the input vector $v$:
   $$ n = \sqrt{v_x^2 + v_y^2 + v_z^2} $$
   [S3, S4, unofficial]

2. Check for degeneracy:
   - If $n < \epsilon(T)$, the vector is effectively zero. Return the zero vector $(0, 0, 0)$ [S3, unofficial].
   - Otherwise, proceed to normalization [S3, unofficial].

3. Normalize the components:
   - Divide each component of $v$ by $n$:
     $$ u_x = \frac{v_x}{n}, \quad u_y = \frac{v_y}{n}, \quad u_z = \frac{v_z}{n} $$
     [S2, S3, unofficial]
   - The result $u$ is the unit vector [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vector_normalize(v: tuple, eps: float) -> tuple:
    """
    Normalizes a 3D vector v to unit length.
    Returns (0,0,0) if magnitude is below eps.
    """
    # Step 1: Compute magnitude
    # Using math.hypot for stability if available, otherwise explicit sqrt
    n = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    
    # Step 2: Check for zero magnitude
    if n < eps:
        return (0.0, 0.0, 0.0)
    
    # Step 3: Normalize components
    u_x = v[0] / n
    u_y = v[1] / n
    u_z = v[2] / n
    
    return (u_x, u_y, u_z)
```
[G] The epsilon value `eps` is not explicitly defined in the sources but is standard practice for floating-point comparisons. A typical value for `float` is $1e-7$ and for `double` is $1e-15$.

**Edge Cases:**
- **Zero Vector:** If the input is $(0,0,0)$, division by zero is avoided by returning $(0,0,0)$ [S3, unofficial].
- **Precision:** Using `Math.hypot` (as in [S4, unofficial]) can prevent intermediate overflow/underflow during magnitude calculation compared to naive squaring and square-rooting, though standard `sqrt(sum(squares))` is common [S4, unofficial].

## Subtopics

### [Octahedral Normal Decoding](3373-octahedral-normal-decoding.md)
Decodes a 2D octahedral coordinate into a 3D unit vector. It serves as the inverse operation for octahedral normal encoding, mapping a compact 2D representation back to a 3D direction suitable for volume rendering or shading. Unlike generic normalization which preserves arbitrary input magnitudes, this method specifically reconstructs the geometric distribution implied by octahedral projection.

## Operating characteristics
- **Time Complexity:** $O(1)$ for a single vector. $O(N)$ for grid normalization where $N$ is the number of active voxels [S3, unofficial].
- **Memory Complexity:** $O(1)$ for single vector. $O(N)$ for grid operations if a new grid/dictionary is allocated [S3, unofficial].
- **Strengths:** Simple, fast, numerically stable with epsilon checks [S3, unofficial].
- **Limitations:** Does not handle dimensionality reduction (see [Octahedral Normal Encoding](3373-octahedral-normal-encoding.md)).
- **When to use:** When direction vectors are needed for lighting or shading calculations and the input data is already in 3D Cartesian format [S4, unofficial].

## Background
Vector normalization is a standard linear algebra operation. In the context of volume rendering utilities, it is used to ensure direction vectors (like sun directions or surface normals) have unit length for correct dot product calculations in lighting models [S4, unofficial]. The implementation details vary slightly across libraries: OpenVDB provides `MaxIndex` which finds the index of the maximum component, often used in conjunction with normalization logic [S1], while other implementations like `PV.unitize` or custom shaders perform the direct division [S2, S4, unofficial].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S2] chloearluck/volume_rendering, https://github.com/chloearluck/volume_rendering, fetched 2026-10-24 (unofficial)
[S3] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S4] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
