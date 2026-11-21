# Analytic Integration for Trilinear Variation

Also called: Polynomial integration for trilinear basis, Trilinear Basis Function Integration

## Overview
Analytic Integration for Trilinear Variation computes the exact integral of a scalar field varying according to a trilinear basis function along a ray segment within a voxel [S1]. It solves the problem of accurately evaluating volume rendering integrals (such as opacity and color accumulation) when the underlying data is interpolated using trilinear filtering, avoiding the aliasing and approximation errors inherent in numerical quadrature [S1]. Unlike [Analytic Integration for Linear Variation](5146-analytic-integration-for-linear-variation.md), which assumes linear variation along the ray, this method accounts for the full cubic polynomial structure introduced by the interaction of the ray's direction with the three-dimensional trilinear interpolation weights [S1].

## How it works
**Inputs:**
- Ray segment defined by origin $\mathbf{p}$ and normalized direction vector $\mathbf{r} = (r_x, r_y, r_z)$ in voxel space.
- Intersection parameters $s_1$ and $s_2$ representing the entry and exit points of the ray with the current voxel, expressed in unit coordinates relative to the voxel's local space.
- Scalar values at the 8 corners of the voxel: $p_{000}, p_{001}, p_{010}, p_{011}, p_{100}, p_{101}, p_{110}, p_{111}$. These correspond to the unit cube corners in the order of $p_{zyx}$ (i.e., the least significant bit corresponds to the x-axis, middle to y, most significant to z) [S1].

**Outputs:**
- The definite integral of the trilinearly interpolated function $f(s)$ along the ray segment from $s_1$ to $s_2$.

**Data Structures:**
- The method relies on precomputed coefficients $a_1$ through $a_8$ derived from the corner values. These are scalar floats.

**Steps:**
1. Compute the constants $a_1$ through $a_8$ using the corner values $p_{ijk}$:
   $$
   \begin{aligned}
   a_1 &= -p_{000} - p_{001} + p_{010} + p_{011} - p_{100} - p_{101} + p_{110} + p_{111} \\
   a_2 &= -p_{000} + p_{001} + p_{010} - p_{011} \\
   a_3 &= -p_{000} + p_{001} - p_{100} + p_{101} \\
   a_4 &= -p_{000} + p_{010} - p_{100} + p_{110} \\
   a_5 &= -p_{000} - p_{001} \\
   a_6 &= p_{000} - p_{010} \\
   a_7 &= p_{000} + p_{100} \\
   a_8 &= -p_{000}
   \end{aligned}
   $$
   [S1]

2. Construct the cubic polynomial $P(s) = C_3 s^3 + C_2 s^2 + C_1 s + C_0$ representing the integrand. The coefficients are defined as follows:
   
   - **Coefficient for $s^3$ ($C_3$):**
     $$ C_3 = a_1 r_x r_y r_z $$
     [S1]

   - **Coefficient for $s^2$ ($C_2$):**
     $$ C_2 = a_1 (r_x r_y p_z + p_x r_y r_z + p_y r_x r_z) + a_2 r_x r_y + a_3 r_x r_z + a_4 r_y r_z $$
     *Note: The source text lists $p_x, p_y, p_z$ in the $s^2$ term formula contextually alongside $r$. In the context of trilinear interpolation $f(x,y,z) = \sum w_i p_i$, the terms $p_x, p_y, p_z$ in the source's polynomial expansion likely refer to the coordinate components of the ray origin $\mathbf{p}$ within the voxel, or are typos for specific geometric terms. However, strictly following the provided source formula [S1]:*
     $$ C_2 = a_1 (r_x r_y p_z + p_x r_y r_z + p_y r_x r_z) + a_2 r_x r_y + a_3 r_x r_z + a_4 r_y r_z $$
     [S1]

   - **Coefficient for $s^1$ ($C_1$):**
     $$ C_1 = a_1 (p_z p_x r_y + p_z p_y r_x + p_x p_y r_z) + a_2 (p_x r_z + p_y r_x) + a_3 (p_x r_z + p_z r_x) + a_4 (p_y r_z + p_z r_y) + a_5 r_x + a_6 r_y + a_7 r_z $$
     [S1]

   - **Constant term ($C_0$):**
     $$ C_0 = a_1 p_x p_y p_z + a_2 p_x p_y + a_3 p_x p_y + a_4 p_y p_z + a_5 p_x + a_6 p_y + a_7 p_z + a_8 $$
     [S1]

3. Integrate the polynomial $P(s)$ from $s_1$ to $s_2$:
   $$ \text{Integral} = \left[ \frac{C_3}{4}s^4 + \frac{C_2}{3}s^3 + \frac{C_1}{2}s^2 + C_0 s \right]_{s_1}^{s_2} $$
   [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytic_integration_trilinear(p_origin, r_direction, s1, s2, p_corners):
    """
    Computes the analytic integral of a trilinearly interpolated field along a ray segment.
    
    Args:
        p_origin: Tuple (px, py, pz) representing the ray origin in voxel space.
        r_direction: Tuple (rx, ry, rz) representing the normalized ray direction.
        s1: Float, entry parameter of the ray into the voxel.
        s2: Float, exit parameter of the ray into the voxel.
        p_corners: List of 8 floats [p000, p001, p010, p011, p100, p101, p110, p111].
                   Order is p_zyx (index = z*4 + y*2 + x).
                   
    Returns:
        Float, the value of the integral from s1 to s2.
    """
    px, py, pz = p_origin
    rx, ry, rz = r_direction
    
    p000, p001, p010, p011, p100, p101, p110, p111 = p_corners
    
    # Step 1: Compute constants a1...a8
    a1 = -p000 - p001 + p010 + p011 - p100 - p101 + p110 + p111
    a2 = -p000 + p001 + p010 - p011
    a3 = -p000 + p001 - p100 + p101
    a4 = -p000 + p010 - p100 + p110
    a5 = -p000 - p001
    a6 = p000 - p010
    a7 = p000 + p100
    a8 = -p000
    
    # Step 2: Compute polynomial coefficients C3, C2, C1, C0
    # C3 coefficient for s^3
    C3 = a1 * rx * ry * rz
    
    # C2 coefficient for s^2
    # Note: Source formula uses px, py, pz in the a1 term. 
    # Assuming px, py, pz refer to the origin coordinates as per standard derivation context 
    # or strictly following the provided text structure.
    term_a1_s2 = rx * ry * pz + px * ry * rz + py * rx * rz
    C2 = a1 * term_a1_s2 + a2 * rx * ry + a3 * rx * rz + a4 * ry * rz
    
    # C1 coefficient for s^1
    term_a1_s1 = pz * px * ry + pz * py * rx + px * py * rz
    term_a2_s1 = px * rz + py * rx
    term_a3_s1 = px * rz + pz * rx
    term_a4_s1 = py * rz + pz * ry
    
    C1 = (a1 * term_a1_s1 + 
          a2 * term_a2_s1 + 
          a3 * term_a3_s1 + 
          a4 * term_a4_s1 + 
          a5 * rx + 
          a6 * ry + 
          a7 * rz)
          
    # C0 constant term
    term_a1_c0 = px * py * pz
    term_a2_c0 = px * py
    term_a3_c0 = px * py # Note: Source lists a3 * px * py here, distinct from a2
    term_a4_c0 = py * pz
    
    C0 = (a1 * term_a1_c0 + 
          a2 * term_a2_c0 + 
          a3 * term_a3_c0 + 
          a4 * term_a4_c0 + 
          a5 * px + 
          a6 * py + 
          a7 * pz + 
          a8)
          
    # Step 3: Integrate from s1 to s2
    # Antiderivative F(s) = (C3/4)s^4 + (C2/3)s^3 + (C1/2)s^2 + C0*s
    def antiderivative(s):
        return (C3 / 4.0) * (s ** 4) + \
               (C2 / 3.0) * (s ** 3) + \
               (C1 / 2.0) * (s ** 2) + \
               C0 * s
               
    return antiderivative(s2) - antiderivative(s1)
```

**Edge Cases and Pitfalls:**
- The proof for the transformation of curve integrals used to derive these coefficients is only valid for a linear curve $c_n$ and linear transformation $T$ [S1]. If the ray is not linear or the transformation is non-linear, this method is invalid.
- Precision issues may arise if $s_1$ and $s_2$ are very close, leading to catastrophic cancellation in the final subtraction.
- The order of corner values $p_{000} \dots p_{111}$ must strictly follow the $p_{zyx}$ indexing convention specified in the source [S1].

## Operating characteristics
**Time Complexity:** $O(1)$ per voxel. The method involves a fixed number of arithmetic operations (multiplications and additions) to compute the coefficients and the final integral.
**Memory Complexity:** $O(1)$. No additional data structures are required beyond the input values.
**Strengths:** Provides exact integration for trilinearly interpolated data, eliminating sampling artifacts associated with numerical methods like the Trapezoid Rule. It is computationally efficient compared to higher-order numerical quadrature.
**Limitations:** Only applicable to trilinear interpolation. Does not handle higher-order basis functions (e.g., bicubic or tetrahedral) without modification. Requires the ray to be linear within the voxel.
**When to use:** When high accuracy is required for volume rendering with trilinearly interpolated textures and performance constraints prevent expensive numerical integration.
**When not to use:** When the data uses higher-order interpolation or when the ray path is non-linear (e.g., in curved space or with specific distortion fields).

## Background
The method originates from work on image-based tomographic reconstruction and realistic fire simulation, specifically detailed in the paper "Image-based tomographic reconstruction of flames" by Beaudoin, Paquet, and Poulin (Graphics Interface 2001) [S1]. The mathematical foundation relies on the analytical integration of cubic polynomials representing trilinear basis functions along line segments. The source cites Åke Björck's "Numerical Methods for Least Squares Problems" (1996) for related numerical techniques [S1].

## Sources
[S1] Image-based tomographic reconstruction of flames, https://doi.org/10.1145/1028523.1028572, fetched 2026-10-26
