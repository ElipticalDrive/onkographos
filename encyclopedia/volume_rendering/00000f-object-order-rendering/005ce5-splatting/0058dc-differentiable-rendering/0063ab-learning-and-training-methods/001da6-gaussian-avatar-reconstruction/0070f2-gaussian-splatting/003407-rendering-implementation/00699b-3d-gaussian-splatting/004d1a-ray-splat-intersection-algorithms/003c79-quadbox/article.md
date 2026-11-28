# QuadBox
## Overview
QuadBox is a Gaussian-tile intersection method that uses four axis-aligned bounding boxes (AABBs) to tightly encapsulate projected 2D Gaussians, reducing tile over-approximation and redundant computations in the rasterizer preprocess stage [S1]. It sets itself apart from its siblings by replacing the single conservative AABB (used in standard 3DGS) and the single adaptive AABB (used in AdR-Gaussian) with four symmetrically positioned, adaptively stretched sub-boxes that fit the elliptical footprint more precisely [S1]. This tight fit suppresses empty tile coverage along the minor axis, which is a known weakness of single-box approaches for anisotropic Gaussians [S1].

## How it works
QuadBox consists of two main components: the construction of the QuadBox geometry and the QPass tile traversal algorithm [S1]. The method operates on the 2D projected Gaussian defined by center $\mu_i^{2D}$ and covariance $\Sigma_i^{2D}$ [S1].

### Inputs and Outputs
*   **Input**: 2D Gaussian parameters: center $\mu_i^{2D} = (\mu_x, \mu_y)$, 2D covariance $\Sigma_i^{2D}$, opacity $\theta_i$, and opacity threshold $\alpha_{\min}$ [S1].
*   **Output**: A set of four tile-aligned AABBs (the QuadBox) and a list of tiles intersected by these boxes [S1].

### Data Structures
*   **Inverse Covariance Matrix ($\Lambda$)**: A $2 \times 2$ symmetric matrix $\Lambda = (\Sigma_i^{2D})^{-1} = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$ [S1].
*   **QuadBox**: A collection of four AABBs. Each AABB is defined by its minimum and maximum coordinates in tile space: $(x_{min}, y_{min}, x_{max}, y_{max})$ [S1].
*   **Tile Grid**: The image plane is partitioned into $16 \times 16$ tiles [S1].

### Parameters
*   **Opacity Threshold ($\alpha_{\min}$)**: Gaussians with splatted opacity below this value are ignored. Typical value: $1/255$ [S1].
*   **Tile Size**: $16 \times 16$ pixels [S1].

### Step-by-Step Method

#### Phase 1: Initialization
1. Calculate the splatted opacity $\alpha$ for the Gaussian [S1].
2. If $\alpha < \alpha_{\min}$, discard the Gaussian [S1].
3. Define the confidence inequality for the support region:
   $$ 2 \ln\left(\frac{\theta_i}{\alpha_{\min}}\right) \leq (u - \mu_i^{2D})(\Sigma_i^{2D})^{-1}(u - \mu_i^{2D})^\top $$
   Let $\gamma = 2 \ln\left(\frac{\theta_i}{\alpha_{\min}}\right)$ [S1].
4. Define the ellipse equation centered at $\mu_i^{2D}$ with coordinates $(x_d, y_d)$:
   $$ F(x_d, y_d) = ax_d^2 + 2bx_dy_d + cy_d^2 - \gamma \leq 0 $$
   where $\Lambda = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$ [S1].

#### Phase 2: DualBox Extraction
1. Compute the initial AdR-AABB (Adaptive Radius AABB) bounds based on the ellipse [S1].
2. Identify the major axis of the ellipse [S1].
3. Retain only the two quadrants of the AdR-AABB that are aligned with the major axis. This forms the "DualBox" [S1].
   *   *Note*: This step suppresses empty tile coverage along the minor axis [S1].

#### Phase 3: Adaptive Stretching (QuadBox Construction)
1. Compute the stretching factor $f$. This factor depends only on the entries of $\Lambda$ and is independent of $\gamma$ and center translation [S1]:
   $$ f = \sqrt{1 - \frac{b^2}{ac}} $$
   *   *Constraint*: Since $\Lambda \succ 0$, $a > 0, c > 0, ac - b^2 > 0$, ensuring $0 < f \leq 1$ [S1].
   *   *Optimization*: Use the purely algebraic closed form to avoid excessive square roots and divisions where possible [S1].
2. Calculate the maximal extents $x_{\max}$ and $y_{\max}$ of the ellipse:
   $$ x_{\max} = \sqrt{\frac{\gamma c}{ac - b^2}}, \quad y_{\max} = \sqrt{\frac{\gamma a}{ac - b^2}} $$
   Alternatively, using the inverse matrix elements $(\Lambda^{-1})_{11}$ and $(\Lambda^{-1})_{22}$:
   $$ x_{\max} = \sqrt{\gamma (\Lambda^{-1})_{11}}, \quad y_{\max} = \sqrt{\gamma (\Lambda^{-1})_{22}} $$
   where $\Lambda^{-1} = \frac{1}{ac - b^2} \begin{bmatrix} c & -b \\ -b & a \end{bmatrix}$ [S1].
3. Construct the four sub-boxes. The DualBox covers the major axis quadrants. The remaining parts of the ellipse (along the minor axis) are covered by two additional sub-boxes. These sub-boxes are adaptively stretched by factor $f$ to tightly fit the ellipse curvature [S1].
   *   The axis intercepts are calculated as $x_{\text{inter}} = \sqrt{\gamma/a}$ and $y_{\text{inter}} = \sqrt{\gamma/c}$ [S1].
   *   The final QuadBox consists of four AABBs that collectively enclose the ellipse $F(x_d, y_d) \leq 0$ [S1].

#### Phase 4: QPass Tile Traversal
1. **Global Range**: Compute the global tile range covered by the QuadBox using simple integer divisions based on the QuadBox bounds [S1].
2. **Scanline Aggregation**: Scan along the shorter axis of the bounding volume. For each scanline, aggregate the active tile interval using fast min/max tests over the intersecting sub-boxes [S1].
3. **Tile Intersection**: Perform tile intersection checks using simple interval tests (integer comparisons) for each tile in the aggregated range. This ensures each tile is visited exactly once [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_inverse_covariance(sigma_2d):
    """
    Computes Lambda = (Sigma_2D)^-1.
    Sigma_2d is a 2x2 matrix [[s11, s12], [s21, s22]].
    Returns Lambda = [[a, b], [b, c]].
    """
    det = sigma_2d[0][0] * sigma_2d[1][1] - sigma_2d[0][1] * sigma_2d[1][0]
    if det == 0:
        return None # Degenerate case
    a = sigma_2d[1][1] / det
    b = -sigma_2d[0][1] / det
    c = sigma_2d[0][0] / det
    return [[a, b], [b, c]]

def compute_stretching_factor(lambda_mat):
    """
    Computes f = sqrt(1 - b^2 / (a*c)).
    lambda_mat is [[a, b], [b, c]].
    """
    a, b, c = lambda_mat[0][0], lambda_mat[0][1], lambda_mat[1][1]
    val = 1.0 - (b * b) / (a * c)
    if val < 0: val = 0.0 # Clamp due to precision
    if val > 1: val = 1.0
    return math.sqrt(val)

def compute_max_extents(lambda_mat, gamma):
    """
    Computes x_max and y_max of the ellipse.
    Lambda = [[a, b], [b, c]].
    """
    a, b, c = lambda_mat[0][0], lambda_mat[0][1], lambda_mat[1][1]
    det = a * c - b * b
    if det == 0:
        return 0.0, 0.0
    x_max = math.sqrt(gamma * c / det)
    y_max = math.sqrt(gamma * a / det)
    return x_max, y_max

def quadbox_construction(gaussian, alpha_min):
    """
    Constructs the QuadBox for a single Gaussian.
    gaussian: object with .mu_2d (center), .sigma_2d (covariance), .opacity
    alpha_min: threshold (e.g., 1/255)
    Returns: List of 4 AABBs (each as [x_min, y_min, x_max, y_max] in pixel coords)
    """
    theta = gaussian.opacity
    if theta < alpha_min:
        return []

    gamma = 2.0 * math.log(theta / alpha_min)
    mu = gaussian.mu_2d
    sigma_2d = gaussian.sigma_2d
    
    lambda_mat = compute_inverse_covariance(sigma_2d)
    if lambda_mat is None:
        return []
    
    a, b, c = lambda_mat[0][0], lambda_mat[0][1], lambda_mat[1][1]
    
    f = compute_stretching_factor(lambda_mat)
    x_max, y_max = compute_max_extents(lambda_mat, gamma)
    
    x_inter = math.sqrt(gamma / a)
    y_inter = math.sqrt(gamma / c)
    
    # The exact geometric mapping of the 4 AABB corners from the ellipse 
    # equation and stretching factor is not explicitly given in the sources.
    # Implementers must derive the 4 AABB corners from the ellipse equation 
    # F(xd, yd) <= 0 using the computed x_max, y_max and f.
    # For example, determining major axis orientation from lambda_mat and 
    # applying the stretching factor to the minor axis extents.
    boxes = [] 
    return boxes

def qpass_tile_traversal(quadbox_boxes, image_width, image_height, tile_size=16):
    """
    QPass algorithm to find intersected tiles.
    quadbox_boxes: List of 4 AABBs in pixel coordinates.
    image_width, image_height: Image dimensions.
    tile_size: 16.
    Returns: Set of tile coordinates (tx, ty).
    """
    tiles = set()
    if not quadbox_boxes:
        return tiles
    
    # Step 1: Compute global tile range
    min_x = min(box[0] for box in quadbox_boxes)
    min_y = min(box[1] for box in quadbox_boxes)
    max_x = max(box[2] for box in quadbox_boxes)
    max_y = max(box[3] for box in quadbox_boxes)
    
    tx_min = max(0, min_x // tile_size)
    ty_min = max(0, min_y // tile_size)
    tx_max = min((image_width - 1) // tile_size, max_x // tile_size)
    ty_max = min((image_height - 1) // tile_size, max_y // tile_size)
    
    # Step 2 & 3: Scan and Aggregate
    # Determine shorter axis for scanning
    scan_y = (max_y - min_y) < (max_x - min_x)
    
    if scan_y:
        for ty in range(ty_min, ty_max + 1):
            y_start = ty * tile_size
            y_end = (ty + 1) * tile_size
            active_x_intervals = []
            for box in quadbox_boxes:
                bx_min, by_min, bx_max, by_max = box
                if by_max > y_start and by_min < y_end:
                    active_x_intervals.append((bx_min, bx_max))
            if not active_x_intervals:
                continue
            active_x_intervals.sort(key=lambda x: x[0])
            merged = []
            curr_s, curr_e = active_x_intervals[0]
            for s, e in active_x_intervals[1:]:
                if s < curr_e:
                    curr_e = max(curr_e, e)
                else:
                    merged.append((curr_s, curr_e))
                    curr_s, curr_e = s, e
            merged.append((curr_s, curr_e))
            for s, e in merged:
                tx_start = max(tx_min, s // tile_size)
                tx_end = min(tx_max, e // tile_size)
                for tx in range(tx_start, tx_end + 1):
                    tiles.add((tx, ty))
    else:
        for tx in range(tx_min, tx_max + 1):
            x_start = tx * tile_size
            x_end = (tx + 1) * tile_size
            active_y_intervals = []
            for box in quadbox_boxes:
                bx_min, by_min, bx_max, by_max = box
                if bx_max > x_start and bx_min < x_end:
                    active_y_intervals.append((by_min, by_max))
            if not active_y_intervals:
                continue
            active_y_intervals.sort(key=lambda x: x[0])
            merged = []
            curr_s, curr_e = active_y_intervals[0]
            for s, e in active_y_intervals[1:]:
                if s < curr_e:
                    curr_e = max(curr_e, e)
                else:
                    merged.append((curr_s, curr_e))
                    curr_s, curr_e = s, e
            merged.append((curr_s, curr_e))
            for s, e in merged:
                ty_start = max(ty_min, s // tile_size)
                ty_end = min(ty_max, e // tile_size)
                for ty in range(ty_start, ty_end + 1):
                    tiles.add((tx, ty))
    return tiles
```

### Edge Cases and Pitfalls
*   **Numerical Stability**: Gaussians with very low opacity can cause instability. This is handled by filtering Gaussians with opacity $\alpha < \alpha_{\min}$ [S1].
*   **Computational Overhead**: Excessive use of square roots ($\sqrt{}$) and division operations inflates computational overhead. The method derives a purely algebraic closed form for $f$ to avoid these operations where possible [S1].
*   **Degenerate Input**: If the determinant of the covariance matrix is zero, the inverse is undefined. The pseudocode includes a guard for this [G].
*   **Warp Divergence**: A naive iteration over sub-boxes causes redundant tile checks and warp divergence. QPass avoids this by aggregating intervals [S1].

## Operating characteristics
*   **Time Complexity**: The preprocessing step (QuadBox construction) is $O(1)$ per Gaussian, as it involves a fixed number of arithmetic operations. The tile traversal (QPass) is $O(T)$ where $T$ is the number of tiles covered by the QuadBox, which is significantly reduced compared to standard 3DGS due to tighter bounds [S1].
*   **Memory Complexity**: $O(1)$ per Gaussian for storing the QuadBox parameters [S1].
*   **Measured Performance**: QuadBox achieves an average over 1.8× speedup compared to standard 3DGS on Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets [S1]. It outperforms AdR-Gaussian's AABB algorithm by 1.85× [S1].
*   **Strengths**: Tighter bounding reduces false positives in tile assignment, leading to fewer redundant computations in the render kernel. The QPass algorithm is branch-free and cache-friendly [S1].
*   **Limitations**: The method assumes the Gaussian projection is well-approximated by the ellipse. Extreme aspect ratios or numerical issues in covariance inversion may require careful handling [S1].

## Background
QuadBox was introduced in the paper "QuadBox: Accelerating 3D Gaussian Splatting with Geometry-Aware Boxes" [S1]. It addresses the inefficiency of conservative bounding boxes in 3D Gaussian Splatting rasterization. The method builds on the concept of adaptive bounding boxes (AdR-Gaussian) but improves upon them by using four boxes to better fit anisotropic shapes [S1].

## Sources
[S1] QuadBox: Accelerating 3D Gaussian Splatting with Geometry-Aware Boxes, https://doi.org/10.1109/icip61757.2026.11629922, fetched 2026-10-29
