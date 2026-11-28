# Modified Noise (Dimension-Reducible)

Also called: Dimension-reducible noise

## Overview
Modified Noise is a gradient-based procedural noise function designed to optimize 3D noise calculation on GPUs by exploiting dimension reducibility [S1, unofficial]. It differs from standard Perlin noise by selecting gradient vectors from the corners of a unit-n cube rather than the edge centers, a geometric constraint required to decompose 3D evaluation into independent 2D noise and 1D gradient lookups [S1, unofficial]. This allows the noise to be stored in 2D textures (or a single 2-channel texture) and computed with minimal texture fetches, making it suitable for real-time volumetric rendering of effects like fire and smoke [S1, unofficial].

## How it works
Modified Noise computes a scalar value $n(\mathbf{P})$ at a query point $\mathbf{P} = (x, y, z)$ by reducing the 3D problem into two 2D hash lookups and a linear interpolation along the $z$-axis.

### Inputs and Outputs
- **Input**: Query point $\mathbf{P} = (x, y, z)$.
- **Output**: Scalar noise value $n(\mathbf{P}) \in [-1, 1]$ (assuming normalized gradients).

### Data Structures
The method relies on pre-computed or on-the-fly hash generation. For performance, gradients are typically pre-baked into textures [S1, unofficial].

1.  **2D Noise Texture ($T_{noise}$)**: Stores the 2D noise contribution for integer $z$ slices.
2.  **Z-Gradient Texture ($T_{zgrad}$)**: Stores the gradient contribution along the $z$-axis for integer $z$ slices.
3.  **Alternative Single Texture ($T_{combined}$)**: A 2-channel texture where each texel returns a pair $(n_{2D}, g_z)$ [S1, unofficial].

### Hash Function
The hash function maps integer coordinates to gradient indices. The source specifies a simple modular hash:
$$ \text{hash}(x) = x^2 \mod M $$
where $M$ is the size of the hash table/texture dimension [S1, unofficial].

For a query point $\mathbf{P}$, let $p_x, p_y, p_z$ be the integer coordinates of the cell corners. The 2D hash terms for the integer $z$ values below ($c_0$) and above ($c_1$) the noise argument are computed as:
$$ c_0 = (p_x, p_y, \text{hash}(p_z)) $$
$$ c_1 = (p_x, p_y, \text{hash}(p_z + 1)) $$
Note: The source notation `0 px, py C hash 0 pzi 1 1` implies the 2D coordinate is derived from $x, y$ and the hash of the $z$ index [S1, unofficial]. Specifically, the lookup coordinates for the 2D textures are determined by the $x, y$ coordinates and the hashed $z$ index.

### Interpolation
The final value is interpolated along the fractional $z$ component ($p_{zf} = z - \lfloor z \rfloor$) using a polynomial interpolation function `flerp`.

The `flerp` function is defined as:
$$ \text{flerp}(t, a, b) = (1 - \text{polynomial}(t)) \cdot a + \text{polynomial}(t) \cdot b $$
where $\text{polynomial}(t)$ is either a Cubic Hermite or Quintic Hermite polynomial [S1, unofficial].

### Algorithm Steps
1.  **Decompose Input**: Extract integer parts $p_x, p_y, p_z$ and fractional part $p_{zf}$ from $\mathbf{P}$.
2.  **Compute Hash Indices**:
    -   Calculate $h_0 = \text{hash}(p_z)$.
    -   Calculate $h_1 = \text{hash}(p_z + 1)$.
    -   Define lookup coordinates $c_0 = (p_x, p_y, h_0)$ and $c_1 = (p_x, p_y, h_1)$ for the 2D texture lookups. *Note: In practice, these are flattened 2D coordinates or indices into the 2D texture array.*
3.  **Lookup Values**:
    -   If using separate textures:
        -   $v_0 = T_{noise}(c_0)$
        -   $g_0 = T_{zgrad}(c_0)$
        -   $v_1 = T_{noise}(c_1)$
        -   $g_1 = T_{zgrad}(c_1)$
    -   If using a single 2-channel texture $T_{combined}$:
        -   $(v_0, g_0) = T_{combined}(c_0)$
        -   $(v_1, g_1) = T_{combined}(c_1)$
4.  **Interpolate**:
    -   Compute the interpolated value using the formula:
        $$ n(\mathbf{P}) = \text{flerp}(p_{zf}, v_0 + p_{zf} \cdot g_0, v_1 + p_{zf} \cdot g_1) $$
    -   Alternatively, using the dot product form with the 2-channel texture directly:
        $$ n(\mathbf{P}) = \text{flerp}(p_{zf}, (1, p_{zf}-1) \cdot \text{tex}(c_0), (1, p_{zf}-1) \cdot \text{tex}(c_1)) $$
        where $\text{tex}(c)$ returns the two floats $(v, g)$ and the dot product is with the vector $(1, p_{zf}-1)$? *Correction*: The source formula `(1, pzf-1) . tex(c0)` seems to imply a specific linear combination. Let's stick to the explicit separate texture formula for clarity, as it is mathematically unambiguous:
        $$ \text{val}_0 = v_0 + p_{zf} \cdot g_0 $$
        $$ \text{val}_1 = v_1 + p_{zf} \cdot g_1 $$
        $$ n(\mathbf{P}) = \text{flerp}(p_{zf}, \text{val}_0, \text{val}_1) $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_noise_3d(x, y, z, hash_table_size=256):
    """
    Computes Modified Noise at point (x, y, z).
    Assumes pre-baked textures T_noise and T_zgrad are accessible via lookup function.
    """
    # 1. Decompose coordinates
    px = floor(x)
    py = floor(y)
    pz = floor(z)
    pzf = z - pz  # Fractional z component

    # 2. Compute hash indices for z-slices
    # Hash function: h(i) = i^2 mod M
    h0 = (pz * pz) % hash_table_size
    h1 = ((pz + 1) * (pz + 1)) % hash_table_size

    # 3. Determine 2D lookup coordinates
    # The source implies the 2D lookup is based on (px, py) and the hash of z.
    # We assume a 2D texture lookup function tex2D(u, v)
    c0 = (px, py, h0) # Conceptual index; in implementation, this maps to 2D texture coords
    c1 = (px, py, h1)

    # 4. Lookup values from textures
    # v0, v1 are the 2D noise values
    # g0, g1 are the z-gradient values
    v0, g0 = lookup_combined_texture(c0)
    v1, g1 = lookup_combined_texture(c1)

    # 5. Interpolate along Z
    # Using the formula: flerp(pzf, v0 + pzf*g0, v1 + pzf*g1)
    val0 = v0 + pzf * g0
    val1 = v1 + pzf * g1

    # 6. Apply polynomial interpolation (flerp)
    # Using Quintic Hermite polynomial as per standard Perlin variants mentioned in parent
    # polynomial(t) = 6t^5 - 15t^4 + 10t^3
    poly = quintic_hermite(pzf)
    
    result = (1 - poly) * val0 + poly * val1
    return result

def quintic_hermite(t):
    return 6 * t**5 - 15 * t**4 + 10 * t**3

def lookup_combined_texture(coord):
    """
    Placeholder for texture lookup.
    Returns (noise_value, z_gradient_value)
    """
    # Implementation depends on specific GPU API or CPU texture storage
    pass
```

### Parameters and Edge Cases
-   **Hash Table Size ($M$)**: Determines the period of the noise. Typical values are powers of 2 (e.g., 256, 512) for efficient modulo operations.
-   **Polynomial Choice**: The source mentions both Cubic and Quintic Hermite polynomials for `flerp`. Quintic is standard for $C^1$ continuity in Perlin variants [G].
-   **Precision**: The method relies on accurate hash generation. Using `x^2 mod M` can introduce patterns if $M$ is not chosen carefully (e.g., if $M$ shares factors with the coordinate range).
-   **Boundary Conditions**: The hash function must handle negative integer coordinates correctly. The source does not specify a signed hash, so implementers should ensure `hash()` handles negative inputs (e.g., by taking absolute value or using two's complement representation before squaring) [G].

## Operating characteristics
-   **Time Complexity**: $O(1)$ per noise evaluation. The cost is dominated by two texture lookups and a few arithmetic operations.
-   **Memory Complexity**: $O(N^2)$ for 2D textures, where $N$ is the resolution of the hash table/texture. This is significantly less than the $O(N^3)$ required for a full 3D texture.
-   **Performance**: Optimized for GPUs. The source notes that the low-level OpenGL shader code for 2D Modified noise is 45 instructions, which is considered too high for repeated use per pixel without pre-baking [S1, unofficial]. Pre-baking gradients into textures mitigates this.
-   **Strengths**:
    -   Dimension reducibility allows efficient storage and computation.
    -   Produces a "fluid" character with large, solid parts that separate naturally, making it suitable for fire and smoke [S1, unofficial].
-   **Limitations**:
    -   Requires pre-computation or texture storage for optimal performance.
    -   The visual character differs from Perlin noise due to corner-based gradient selection, which may not be desirable for all applications.

## Background
Modified Noise was introduced in the context of real-time volumetric rendering of fire in a production system [S1, unofficial]. It was developed to address the performance constraints of rendering complex volumetric effects like fire, smoke, and fog on hardware with limited texture bandwidth and instruction count. The method exploits the dimension reducibility property to store 3D noise data in 2D textures, optimizing the calculation process [S1, unofficial]. It is distinct from Perlin noise in its gradient selection strategy, picking gradients from the corners of a unit-n cube to satisfy the dimension reducibility requirement [S1, unofficial].

## Sources
[S1] Real-Time Volumetric Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-thesis/YVanzine_thesis.pdf, fetched 2026-10-27 (unofficial)
