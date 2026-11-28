# Cubic Interpolation
Also called: 3D Cubic Interpolation, Tri-cubic Interpolation
## Overview
Cubic interpolation is a higher-order polynomial reconstruction method that samples 64 neighboring voxels to approximate a scalar value at any location within a volume, producing smoother results than linear interpolation [S1, S2, unofficial]. It distinguishes itself from its siblings by utilizing a full cubic kernel in three dimensions to minimize high-frequency leakage artifacts, at the cost of significantly increased computational complexity [S1, unofficial]. Builds on [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md).
## How it works
**Inputs and outputs:**
- Input: A 3D volume dataset $V$ with dimensions $R \times S \times T$, and a query point $\mathbf{p} = (x, y, z)$ in continuous space [G].
- Output: A scalar value $V(\mathbf{p})$ [G].

**Data structures:**
- The volume data is stored in a 3D array [G]. For efficient access, the integer coordinates of the 64 surrounding voxels are computed [G].

**Steps:**
1.  **Determine the bounding box:** Identify the integer coordinates $(i, j, k)$ such that $i \le x < i+1$, $j \le y < j+1$, and $k \le z < k+1$. This defines the primary voxel containing the query point [G].
2.  **Identify neighbor indices:** In 3D, cubic interpolation requires 4 points along each axis. The indices for the x-axis are $i-1, i, i+1, i+2$. Similarly for y ($j-1, j, j+1, j+2$) and z ($k-1, k, k+1, k+2$). This results in a $4 \times 4 \times 4$ grid of 64 voxels [S1, unofficial].
3.  **Compute fractional offsets:** Calculate the fractional parts of the query coordinates:
    $$ f_x = x - i, \quad f_y = y - j, \quad f_z = z - k $$
    where $f_x, f_y, f_z \in [0, 1)$ [G].
4.  **Apply 1D Cubic Kernel:** For each dimension, apply a cubic filter kernel $K(u)$ to the 4 sampled values. The general form of the cubic kernel depends on parameters $B$ and $C$ (see [Cubic Interpolation (Mitchell-Netravali)](2073-cubic-interpolation-mitchell-netravali.md)). For a specific dimension with values $v_{-1}, v_0, v_1, v_2$ and offset $f$, the interpolated value $v'$ is:
    $$ v' = K(f) \cdot v_0 + K(1+f) \cdot v_{-1} + K(1-f) \cdot v_1 + K(2-f) \cdot v_2 $$
    *(Note: The exact coefficients of $K$ depend on the chosen cubic variant, e.g., Catmull-Rom or Mitchell-Netravali.)* [G]
5.  **Tensor Product:** The final 3D value is the tensor product of the 1D interpolations along each axis. This can be implemented by first interpolating along x for each of the $4 \times 4$ planes in y-z, then along y for each of the 4 lines in z, and finally along z [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_interpolate_3d(volume, x, y, z, kernel_func):
    """
    volume: 3D array (R, S, T)
    x, y, z: float coordinates
    kernel_func: function K(u) returning the weight for offset u
    """
    R, S, T = len(volume), len(volume[0]), len(volume[0][0])
    
    # 1. Integer base coordinates
    ix = int(x) if x >= 0 else int(x) - 1
    iy = int(y) if y >= 0 else int(y) - 1
    iz = int(z) if z >= 0 else int(z) - 1
    
    # 2. Fractional parts
    fx = x - ix
    fy = y - iy
    fz = z - iz
    
    # 3. Helper for 1D cubic interpolation
    def interp_1d(v0, v1, v2, v3, f):
        # v0 is at distance f, v1 at 1+f (previous), v2 at 1-f (next), v3 at 2-f (next-next)
        w0 = kernel_func(f)
        w1 = kernel_func(1.0 + f)
        w2 = kernel_func(1.0 - f)
        w3 = kernel_func(2.0 - f)
        return w0 * v0 + w1 * v1 + w2 * v2 + w3 * v3

    # 4. Boundary clamping helper
    def get_voxel(x_idx, y_idx, z_idx):
        x_idx = max(0, min(x_idx, R - 1))
        y_idx = max(0, min(y_idx, S - 1))
        z_idx = max(0, min(z_idx, T - 1))
        return volume[x_idx][y_idx][z_idx]

    # 5. Pre-calculate 1D interpolations along X for all Y, Z combinations
    temp_yz = [[0.0 for _ in range(4)] for _ in range(4)]
    for dy in range(4):
        for dz in range(4):
            v0 = get_voxel(ix, iy - 1 + dy, iz - 1 + dz)
            v1 = get_voxel(ix - 1, iy - 1 + dy, iz - 1 + dz)
            v2 = get_voxel(ix + 1, iy - 1 + dy, iz - 1 + dz)
            v3 = get_voxel(ix + 2, iy - 1 + dy, iz - 1 + dz)
            temp_yz[dy][dz] = interp_1d(v0, v1, v2, v3, fx)

    # 6. Interpolate along Y
    temp_z = [0.0 for _ in range(4)]
    for dz in range(4):
        v0 = temp_yz[0][dz]
        v1 = temp_yz[1][dz]
        v2 = temp_yz[2][dz]
        v3 = temp_yz[3][dz]
        temp_z[dz] = interp_1d(v0, v1, v2, v3, fy)

    # 7. Interpolate along Z
    v0 = temp_z[0]
    v1 = temp_z[1]
    v2 = temp_z[2]
    v3 = temp_z[3]
    return interp_1d(v0, v1, v2, v3, fz)
```

**Parameters:**
- Kernel parameters $B, C$: Define the specific cubic shape and control the influence on the final result, allowing smoothing or sharpening [S3, unofficial].
- Boundary handling: Clamping, wrapping, or reflection. Clamping is common to avoid artifacts at edges [G].

**Edge cases and pitfalls:**
- **Boundary conditions:** Accessing indices $i-1$ or $i+2$ may go out of bounds. Proper clamping or mirroring is required [G].
- **Precision:** High computational cost can lead to accumulation errors in large volumes if not optimized [S3, unofficial].
- **Artifacts:** Cubic interpolation removes ringing artifacts caused by linear methods [S4].

## Subtopics

### [Catmull-Rom Interpolation](13562-catmull-rom-interpolation.md)
Catmull-Rom interpolation is a specific interpolating cubic-spline scheme used in volume rendering to reconstruct continuous scalar fields from discrete voxel data. It solves the problem of banding artifacts inherent in linear interpolation by providing a continuous first derivative, while uniquely satisfying the condition for accurate straight-line reconstruction among all interpolating cubic splines. It is distinguished from its sibling [General Cubic Spline Interpolation](15870-general-cubic-spline-interpolation.md) by its fixed parameter values ($B=0, C=1/2$ or $a=-0.5$), which minimize third-order error for discrete functions.

### [General Cubic Spline Interpolation](15870-general-cubic-spline-interpolation.md)
General cubic spline interpolation defines a family of piecewise third-order polynomial reconstruction filters parameterized by $B$ and $C$. It solves the problem of smooth volume data sampling by providing a continuous function with a continuous first derivative for all parameter values, ensuring smoother results than linear interpolation. It is distinct from its siblings, such as the four-parameter polynomial family, by utilizing third-order polynomials rather than fourth-order, which limits its ability to guarantee second-derivative continuity but simplifies the parameter space to two variables.

### Quasi-Cubic Sampling
Quasi-cubic sampling is a texture sampling technique that applies a quasi-cubic filter kernel to 3D volume textures to reduce aliasing and improve interpolation quality compared to standard linear/trilinear filtering [S5, unofficial].
**Steps:**
1. Scale the normalized texture coordinate `u` by the volume dimensions `R` and add 0.5 to center the texels: $U_{scaled} = u \times R + 0.5$ [S5, unofficial].
2. Extract the fractional part $F = fract(U_{scaled})$ and the integer part [S5, unofficial].
3. Apply the quasi-cubic kernel to the fractional part: $U_{filtered} = floor(U_{scaled}) + F \times F \times (3.0 - 2.0 \times F)$ [S5, unofficial].
4. Sample the volume texture using the filtered coordinate normalized back to [0,1]: $sample = texture(volume, (U_{filtered} - 0.5) / R)$ [S5, unofficial].

### On-the-fly Tri-cubic Filtering
On-the-fly tri-cubic filtering is an interpolation method using tri-cubic filtering to improve visualization quality over tri-linear filtering, calculated on-the-fly during ray casting [S6, unofficial].
**Steps:**
1. Calculate bi-cubic interpolated values using only four samplings by leveraging embedded bi-linear texture filtering [S6, unofficial].
2. Perform a single tri-cubic sampling by making 8 basic tri-linear samplings, effectively calculating the data value determined by 64 nearest voxels [S6, unofficial].
**Pitfall:** High computational cost: On-the-fly tri-cubic filtering is costly. Interactive performance (>20-30 fps) is only guaranteed for small datasets (<=256^3 voxels) or simple techniques (DVR, opaque iso-surface) [S6, unofficial].

### WENO Interpolation
WENO (Weighted Essentially Non-Oscillatory) interpolation is a high-order polynomial reconstruction method for solving hyperbolic and convection-diffusion equations, achieving high-order accuracy in smooth regions and better representation of sharp gradients [S7].
**Steps:**
1. For an edge $E_i$ where isovalue $k$ lies between $f_i$ and $f_{i+1}$, construct a high-order polynomial $p_i(x)$ to approximate the function in the interval [S7].
2. Solve $p_i(x) = k$ for $x$. This is a root-finding problem for a cubic polynomial, solved using the cubic formula. If multiple valid roots exist, select the median solution [S7].
**Formula:**
$p_i(x) = w_1 p^{(1)}(x) + w_2 p^{(2)}(x) + w_3 p^{(3)}(x)$, where $p^{(1)}, p^{(2)}, p^{(3)}$ are third-order polynomials and $w_1, w_2, w_3$ are nonlinear weights summing to 1 [S7].
Smoothness indicators $\beta_j$ are calculated as:
$\beta_1 = \frac{13}{12}(f_{i-2} - 2f_{i-1} + f_i)^2 + \frac{1}{4}(f_{i-2} - 4f_{i-1} + 3f_i)^2$
$\beta_2 = \frac{13}{12}(f_{i-1} - 2f_i + f_{i+1})^2 + \frac{1}{4}(f_{i-1} - f_{i+1})^2$
$\beta_3 = \frac{13}{12}(f_i - 2f_{i+1} + f_{i+2})^2 + \frac{1}{4}(3f_i - 4f_{i+1} + f_{i+2})^2$ [S7].
Nonlinear weights $w_j = \alpha_j / (\alpha_1 + \alpha_2 + \alpha_3)$, where $\alpha_j = \gamma_j / (\epsilon + \beta_j)^2$, with constants $\gamma_1=1/10, \gamma_2=3/5, \gamma_3=3/10$ [S7].
**Parameter:**
$\epsilon$: A small constant to avoid division by zero, typically set to $10^{-6}$ [S7].

## Operating characteristics
**Time and memory complexity:**
- Time: Considerably high computational costs. In 3D, it requires sampling 64 voxels ($2^4$) compared to 8 for trilinear [S1, unofficial]. It requires two more multiplications per dimension compared to linear interpolation, leading to a considerable amount of instructions [S3, unofficial].
- Memory: No additional memory storage required for the interpolation itself, but the access pattern is more complex [G].

**Measured performance:**
- Increases total surface extraction time by about 3% compared to linear methods in some implementations [S4].
- Interactive performance (>20-30 fps) is only guaranteed for small datasets (<=256^3 voxels) or simple techniques when using on-the-fly tri-cubic filtering [S6, unofficial].

**Scaling:**
- Scales poorly with dataset size for interactive applications due to the fixed 64-sample requirement per query [S6, unofficial].

**Strengths and limitations:**
- Strengths: Produces smoother results than linear interpolation, reducing banding and diamond artifacts [S1, S2, unofficial]. Allows smoothing or sharpening via parameters [S3, unofficial].
- Limitations: High computational cost [S3, unofficial]. Removes ringing artifacts present in linear interpolation [S4].

**When to use it and when not:**
- Use when high visual quality is required and computational resources are sufficient (e.g., offline rendering, small datasets) [S3, unofficial].
- Do not use for real-time rendering of large volumes where performance is critical [S6, unofficial].

## Background
Cubic interpolation in volume rendering originated as a higher-order alternative to linear methods to improve visual smoothness. It is discussed in foundational lessons on voxel grid rendering [S1, unofficial] and GPU-based compressive volume rendering [S2, unofficial]. It has been applied in various contexts, including real-time mono- and multi-volume rendering [S3, unofficial], accurate isosurface interpolation with Hermite data [S4], and quasi-cubic texture sampling [S5, unofficial]. On-the-fly tri-cubic filtering was extended for GPU ray casting to reduce wood-like artifacts [S6, unofficial]. WENO variants have been proposed for high-accuracy isosurface extraction [S7].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S3] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S4] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
[S5] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S6] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
[S7] Estimation and Visualization of Isosurface Uncertainty from Linear and High-Order Interpolation Methods, https://arxiv.org/abs/2409.00043v1, fetched 2026-10-26
