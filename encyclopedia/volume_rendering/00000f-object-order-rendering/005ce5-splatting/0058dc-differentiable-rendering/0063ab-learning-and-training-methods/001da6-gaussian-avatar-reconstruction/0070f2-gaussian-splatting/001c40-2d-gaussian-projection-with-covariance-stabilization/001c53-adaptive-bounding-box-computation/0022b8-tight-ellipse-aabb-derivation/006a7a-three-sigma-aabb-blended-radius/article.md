# Three-Sigma AABB Blended Radius

Also called: splat extent calculation, visual splat weight

## Overview
Computes a single scalar radius that defines the Axis-Aligned Bounding Box (AABB) for a 2D Gaussian splat, blending volume-equivalent and RMS radii with an aspect penalty to ensure the bounding box tightly encloses the splat's visual influence [S1, unofficial]. It differs from sibling methods like [AccuTile](3037-accutile.md) by providing a per-splat geometric bound rather than a tile-level accumulation strategy, and builds on [Tight Ellipse AABB Derivation](2826-tight-ellipse-aabb-derivation.md) by converting the derived ellipse basis vectors into a conservative AABB extent suitable for spatial indexing [S1, unofficial].

## How it works
**Inputs:**
- `splatPosition`: 3D vector $(p_x, p_y, p_z)$.
- `quatsXYZW`: Quaternion $(x, y, z, w)$ representing splat orientation.
- `scaleLog`: Log-scale vector $(\log s_x, \log s_y, \log s_z)$.
- `opacity`: Scalar $\alpha$.
- `n`: Number of splats (for cloud bounds).

**Outputs:**
- `blendedRadius`: Scalar radius for a single splat.
- `globalBounds`: Min/Max 3D vectors for a cloud of splats.

**Data Structures:**
- `extentScratch`: Array of 3 floats $(e_x, e_y, e_z)$ storing the three-sigma extents along world axes.
- `globalMin`, `globalMax`: Arrays of 3 floats initialized to $-\infty$ and $+\infty$ respectively.

**Parameters:**
- `THREE_SIGMA_AABB_ASPECT_PENALTY_EXPONENT`: $0.5$ [S1, unofficial].
- `ELLIPSOID_AREA_P`: $1.6075$ [S1, unofficial].
- `MIN_SPLAT_EXTENT`: $10^{-12}$ [S1, unofficial].

**Steps:**

1.  **Extract and Normalize Quaternion:**
    Extract $x, y, z, w$ from `quatsXYZW`. Compute squared length $len2 = x^2 + y^2 + z^2 + w^2$. If $len2 < 10^{-20}$, reset quaternion to $(0, 0, 0, 1)$ [S1, unofficial]. Otherwise, normalize: $inv = 1/\sqrt{len2}$, then $x \leftarrow x \cdot inv$, $y \leftarrow y \cdot inv$, $z \leftarrow z \cdot inv$, $w \leftarrow w \cdot inv$ [S1, unofficial].

2.  **Compute Quaternion Products:**
    Calculate the following products:
    $$
    \begin{aligned}
    xx &= x^2, & yy &= y^2, & zz &= z^2 \\
    xy &= x \cdot y, & xz &= x \cdot z, & yz &= y \cdot z \\
    wx &= w \cdot x, & wy &= w \cdot y, & wz &= w \cdot z
    \end{aligned}
    $$
    [S1, unofficial].

3.  **Compute Scales:**
    Extract log-scales $l_x, l_y, l_z$ from `scaleLog`. Compute linear scales:
    $$
    s_x = \exp(l_x), \quad s_y = \exp(l_y), \quad s_z = \exp(l_z)
    $$
    [S1, unofficial]. Ensure scales are non-negative and finite: $s_i = \max(s_i, 0.0)$ if finite, else $0.0$ [S1, unofficial].

4.  **Compute Rotation Matrix Elements:**
    Construct the rotation matrix $R$ from the normalized quaternion components:
    $$
    \begin{aligned}
    r_{00} &= 1 - 2(yy + zz), & r_{10} &= 2(xy - wz), & r_{20} &= 2(xz + wy) \\
    r_{01} &= 2(xy + wz), & r_{11} &= 1 - 2(xx + zz), & r_{21} &= 2(yz - wx) \\
    r_{02} &= 2(xz - wy), & r_{12} &= 2(yz + wx), & r_{22} &= 1 - 2(xx + yy)
    \end{aligned}
    $$
    [S1, unofficial].

5.  **Compute Covariance Diagonal Elements:**
    Calculate the diagonal elements of the covariance matrix projected onto the world axes using the squared scales $s_{2x}=s_x^2, s_{2y}=s_y^2, s_{2z}=s_z^2$:
    $$
    \begin{aligned}
    c_{00} &= r_{00}^2 s_{2x} + r_{10}^2 s_{2y} + r_{20}^2 s_{2z} \\
    c_{11} &= r_{01}^2 s_{2x} + r_{11}^2 s_{2y} + r_{21}^2 s_{2z} \\
    c_{22} &= r_{02}^2 s_{2x} + r_{12}^2 s_{2y} + r_{22}^2 s_{2z}
    \end{aligned}
    $$
    [S1, unofficial].

6.  **Compute Three-Sigma Extents:**
    Calculate the extent along each world axis, ensuring positivity and a minimum floor:
    $$
    \begin{aligned}
    e_x &= 3 \cdot \sqrt{\max(c_{00}, 10^{-20})} \\
    e_y &= 3 \cdot \sqrt{\max(c_{11}, 10^{-20})} \\
    e_z &= 3 \cdot \sqrt{\max(c_{22}, 10^{-20})}
    \end{aligned}
    $$
    [S1, unofficial]. Store these in `extentScratch`.

7.  **Compute Blended Radius:**
    Calculate the volume-equivalent radius $r_{vol}$ and RMS radius $r_{rms}$:
    $$
    r_{vol} = \sqrt[3]{s_x s_y s_z}
    $$
    $$
    r_{rms} = \sqrt{\frac{s_x^2 + s_y^2 + s_z^2}{3.0}}
    $$
    [S1, unofficial]. Compute the aspect penalty:
    $$
    \text{aspectPenalty} = \frac{r_{rms}}{r_{vol}}
    $$
    [S1, unofficial]. The blended radius is:
    $$
    \text{blendedRadius} = r_{vol} \cdot \text{aspectPenalty}^{0.5}
    $$
    [S1, unofficial].

8.  **Compute Cloud Bounds (Optional):**
    Initialize `globalMin` to $[\infty, \infty, \infty]$ and `globalMax` to $[-\infty, -\infty, -\infty]$ [S1, unofficial]. For each splat $i$:
    - Compute $e_x, e_y, e_z$ as above.
    - Get position $p$.
    - Update bounds:
      $$
      \begin{aligned}
      \text{globalMin}_x &= \min(\text{globalMin}_x, p_x - e_x) \\
      \text{globalMax}_x &= \max(\text{globalMax}_x, p_x + e_x)
      \end{aligned}
      $$
      (Repeat for $y$ and $z$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_three_sigma_aabb_blended_radius(splat_position, quats_xyzw, scale_log, opacity):
    """
    Computes the blended radius for a single splat.
    
    Args:
        splat_position: tuple (px, py, pz)
        quats_xyzw: tuple (x, y, z, w)
        scale_log: tuple (log_sx, log_sy, log_sz)
        opacity: float
        
    Returns:
        blended_radius: float
    """
    # 1. Extract and Normalize Quaternion
    x, y, z, w = quats_xyzw
    len2 = x*x + y*y + z*z + w*w
    
    if len2 < 1e-20:
        x, y, z, w = 0.0, 0.0, 0.0, 1.0
    else:
        inv = 1.0 / math.sqrt(len2)
        x *= inv
        y *= inv
        z *= inv
        w *= inv

    # 2. Compute Quaternion Products
    xx = x * x
    yy = y * y
    zz = z * z
    xy = x * y
    xz = x * z
    yz = y * z
    wx = w * x
    wy = w * y
    wz = w * z

    # 3. Compute Scales
    sx = math.exp(scale_log[0])
    sy = math.exp(scale_log[1])
    sz = math.exp(scale_log[2])
    
    # Ensure non-negative and finite
    if not math.isfinite(sx): sx = 0.0
    if not math.isfinite(sy): sy = 0.0
    if not math.isfinite(sz): sz = 0.0
    sx = max(sx, 0.0)
    sy = max(sy, 0.0)
    sz = max(sz, 0.0)

    s2x = sx * sx
    s2y = sy * sy
    s2z = sz * sz

    # 4. Compute Rotation Matrix Elements
    r00 = 1.0 - 2.0 * (yy + zz)
    r10 = 2.0 * (xy - wz)
    r20 = 2.0 * (xz + wy)
    r01 = 2.0 * (xy + wz)
    r11 = 1.0 - 2.0 * (xx + zz)
    r21 = 2.0 * (yz - wx)
    r02 = 2.0 * (xz - wy)
    r12 = 2.0 * (yz + wx)
    r22 = 1.0 - 2.0 * (xx + yy)

    # 5. Compute Covariance Diagonal Elements
    c00 = r00*r00 * s2x + r10*r10 * s2y + r20*r20 * s2z
    c11 = r01*r01 * s2x + r11*r11 * s2y + r21*r21 * s2z
    c22 = r02*r02 * s2x + r12*r12 * s2y + r22*r22 * s2z

    # 6. Compute Three-Sigma Extents
    MIN_EXTENT = 1e-12
    ex = 3.0 * math.sqrt(max(c00, 1e-20))
    ey = 3.0 * math.sqrt(max(c11, 1e-20))
    ez = 3.0 * math.sqrt(max(c22, 1e-20))

    # 7. Compute Blended Radius
    # Volume equivalent radius
    if sx * sy * sz <= 0:
        r_vol = 0.0
    else:
        r_vol = math.pow(sx * sy * sz, 1.0/3.0)
        
    # RMS radius
    r_rms = math.sqrt((s2x + s2y + s2z) / 3.0)
    
    # Aspect penalty
    if r_vol == 0:
        aspect_penalty = 1.0 # Avoid division by zero, though blended radius will be 0
    else:
        aspect_penalty = r_rms / r_vol
        
    blended_radius = r_vol * math.pow(aspect_penalty, 0.5)
    
    return blended_radius

def compute_cloud_bounds(splats):
    """
    Computes global AABB for a cloud of splats.
    
    Args:
        splats: list of dicts with keys 'position', 'quats_xyzw', 'scale_log'
        
    Returns:
        global_min: tuple (min_x, min_y, min_z)
        global_max: tuple (max_x, max_y, max_z)
    """
    INF = float('inf')
    global_min = [INF, INF, INF]
    global_max = [-INF, -INF, -INF]
    
    for splat in splats:
        pos = splat['position']
        quats = splat['quats_xyzw']
        s_log = splat['scale_log']
        
        # Reuse logic from above to get extents
        x, y, z, w = quats
        len2 = x*x + y*y + z*z + w*w
        if len2 < 1e-20:
            x, y, z, w = 0.0, 0.0, 0.0, 1.0
        else:
            inv = 1.0 / math.sqrt(len2)
            x *= inv; y *= inv; z *= inv; w *= inv
            
        xx = x*x; yy = y*y; zz = z*z
        xy = x*y; xz = x*z; yz = y*z
        wx = w*x; wy = w*y; wz = w*z
        
        sx = math.exp(s_log[0]); sy = math.exp(s_log[1]); sz = math.exp(s_log[2])
        if not math.isfinite(sx): sx = 0.0
        if not math.isfinite(sy): sy = 0.0
        if not math.isfinite(sz): sz = 0.0
        sx = max(sx, 0.0); sy = max(sy, 0.0); sz = max(sz, 0.0)
        
        s2x = sx*sx; s2y = sy*sy; s2z = sz*sz
        
        r00 = 1.0 - 2.0*(yy+zz); r10 = 2.0*(xy-wz); r20 = 2.0*(xz+wy)
        r01 = 2.0*(xy+wz); r11 = 1.0 - 2.0*(xx+zz); r21 = 2.0*(yz-wx)
        r02 = 2.0*(xz-wy); r12 = 2.0*(yz+wx); r22 = 1.0 - 2.0*(xx+yy)
        
        c00 = r00*r00*s2x + r10*r10*s2y + r20*r20*s2z
        c11 = r01*r01*s2x + r11*r11*s2y + r21*r21*s2z
        c22 = r02*r02*s2x + r12*r12*s2y + r22*r22*s2z
        
        ex = 3.0 * math.sqrt(max(c00, 1e-20))
        ey = 3.0 * math.sqrt(max(c11, 1e-20))
        ez = 3.0 * math.sqrt(max(c22, 1e-20))
        
        px, py, pz = pos
        
        global_min[0] = min(global_min[0], px - ex)
        global_max[0] = max(global_max[0], px + ex)
        global_min[1] = min(global_min[1], py - ey)
        global_max[1] = max(global_max[1], py + ey)
        global_min[2] = min(global_min[2], pz - ez)
        global_max[2] = max(global_max[2], pz + ez)
        
    return tuple(global_min), tuple(global_max)
```

## Subtopics

### AccuTile
An alternative method for managing splat bounds at the tile level, focusing on accumulation and culling rather than per-splat geometric radius calculation [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per splat for radius computation; $O(N)$ for cloud bounds where $N$ is the number of splats.
- **Memory Complexity**: $O(1)$ auxiliary space per splat (scratch vectors).
- **Strengths**: Provides a tight, aspect-aware bounding box that accounts for the splat's orientation and scale distribution. The three-sigma extent ensures high probability coverage of the Gaussian's mass.
- **Limitations**: The blended radius is a scalar approximation; for highly anisotropic splats, the AABB may still be loose compared to the actual ellipse shape. Precision issues can arise if quaternion normalization fails or scales are extremely small/large.
- **When to use**: When spatial indexing (e.g., BVH, grid) is required for efficient splat rendering or culling.
- **When not to use**: When exact ellipse intersection tests are feasible and performance allows, or when tile-level culling (AccuTile) is more appropriate for the rendering pipeline.

## Background
The method originates from implementations of 3D Gaussian Splatting for 3D Tiles conversion, specifically in the `lzy-0618/3DGS-PLY-3DTiles-Converter` repository [S1, unofficial]. It adapts standard 3DGS covariance representations into a form suitable for spatial partitioning in web-based 3D viewers. The aspect penalty exponent and ellipsoid area approximation parameters are tuned for visual fidelity in this context.

## Sources
[S1] lzy-0618/3DGS-PLY-3DTiles-Converter, https://github.com/lzy-0618/3DGS-PLY-3DTiles-Converter, fetched 2026-10-27 (unofficial)
