# QuadBox


## Overview
QuadBox is a Gaussian-tile intersection method that uses four axis-aligned bounding boxes (AABBs) to tightly encapsulate projected 2D Gaussians, reducing tile over-approximation and redundant computations in the rasterizer preprocess stage [S1]. It sets itself apart from its siblings by replacing the single conservative AABB (used in standard 3DGS and AdR-Gaussian) with four adaptive sub-boxes that conform to the ellipse's anisotropic shape, combined with a branch-free single-pass traversal algorithm (QPass) [S1]. This approach minimizes false-positive tile visits compared to [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md) and [Gaussian Opacity Fields (GOF)](5992-gaussian-opacity-fields-gof.md) while maintaining deterministic cache-friendly access patterns [S1].

## How it works
QuadBox operates in two main phases: **QuadBox Construction** (preprocessing) and **QPass** (tile traversal). The method assumes the image plane is partitioned into $16 \times 16$ tiles [S1].

### Inputs and Outputs
*   **Input**: A set of 3D Gaussians $\{\mathcal{G}_i\}$, each with center $\mu_i$, 3D covariance $\Sigma_i$, opacity $o_i$, and color $c_i$. For each Gaussian, the 2D projection parameters are required: center $\mu_i^{2D}$, 2D covariance $\Sigma_i^{2D}$, and splatted opacity $\alpha_i$.
*   **Output**: A list of tile-Gaussian pairs, where each pair consists of a tile ID and the Gaussian index, sorted by depth for front-to-back compositing.

### Data Structures
1.  **QuadBox**: A structure containing four AABBs. Each AABB is defined by its minimum and maximum tile coordinates $(t_{x_{min}}, t_{y_{min}})$ and $(t_{x_{max}}, t_{y_{max}})$.
2.  **Tile List**: A dynamic array or hash map storing the tile IDs intersected by the QuadBox.

### Parameters
*   $\alpha_{\min}$: Opacity threshold for filtering. Typical value: $1/255$ [S1].
*   Tile Size: $16 \times 16$ pixels [S1].

### Algorithm Steps

#### Phase 1: Initialization
1.  Filter out Gaussians where the splatted opacity $\alpha_i < \alpha_{\min}$ [S1]. This ensures numerical stability and defines the support region boundary [S1].

#### Phase 2: DualBox Extraction
1.  Compute the inverse 2D covariance matrix $\Lambda = (\Sigma_i^{2D})^{-1} = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$ [S1].
2.  Determine the major axis of the ellipse defined by $\Lambda$.
3.  Extract the **DualBox** by retaining only the two quadrants of the Adaptive Radius (AdR) AABB that are aligned with the ellipse's major axis [S1]. This suppresses empty tile coverage along the minor axis [S1].

#### Phase 3: Adaptive Stretching (QuadBox Construction)
The goal is to tightly enclose the remaining parts of the ellipse not covered by the DualBox using two additional sub-boxes, resulting in four total AABBs.

1.  **Define the Ellipse Equation**:
    The support region is defined by the confidence inequality [S1]:
    $$ 2 \ln\left(\frac{\theta_i}{\alpha_{\min}}\right) \leq (u - \mu_i^{2D})(\Sigma_i^{2D})^{-1}(u - \mu_i^{2D})^\top $$
    Let $\gamma = 2 \ln\left(\frac{\theta_i}{\alpha_{\min}}\right)$ [S1]. Center coordinates as $(x_d, y_d) = u - \mu_i^{2D}$. The ellipse equation is [S1]:
    $$ F(x_d, y_d) = ax_d^2 + 2bx_dy_d + cy_d^2 - \gamma \leq 0 $$

2.  **Compute Axis Intercepts**:
    Set $x_d = 0$ to find the y-intercept:
    $$ y_{\text{inter}} = \sqrt{\gamma/c} $$
    Set $y_d = 0$ to find the x-intercept:
    $$ x_{\text{inter}} = \sqrt{\gamma/a} $$

3.  **Compute Maximal Extents**:
    The maximal support in direction $v$ is $\max v^\top u = \sqrt{\gamma v^\top \Lambda^{-1} v}$ [S1].
    For the coordinate axes $e_x = (1, 0)^\top$ and $e_y = (0, 1)^\top$:
    $$ x_{\max} = \sqrt{\gamma (\Lambda^{-1})_{11}}, \quad y_{\max} = \sqrt{\gamma (\Lambda^{-1})_{22}} $$
    Using the inverse of the $2 \times 2$ symmetric matrix $\Lambda$:
    $$ \Lambda^{-1} = \frac{1}{ac - b^2} \begin{bmatrix} c & -b \\ -b & a \end{bmatrix} $$
    Substituting the diagonal elements:
    $$ x_{\max} = \sqrt{\frac{\gamma c}{ac - b^2}}, \quad y_{\max} = \sqrt{\frac{\gamma a}{ac - b^2}} $$

4.  **Compute Stretching Factor**:
    The stretching factor $f$ is the ratio of the intercept to the maximal extent. It is derived as a purely algebraic closed form to avoid excessive square roots and divisions [S1]:
    $$ f = \sqrt{1 - \frac{b^2}{ac}} $$
    This factor satisfies $0 < f \leq 1$ and depends only on $\Lambda$ entries, independent of $\gamma$ and center translation [S1].

5.  **Construct Four AABBs**:
    The DualBox covers the major axis quadrants. The remaining two quadrants (aligned with the minor axis) are covered by two additional AABBs. These sub-boxes are adaptively stretched by factor $f$ to tightly fit the ellipse boundary [S1]. The final QuadBox consists of these four AABBs.

#### Phase 4: QPass (Tile Traversal)
A naive iteration over sub-boxes causes redundant tile checks and warp divergence. QPass aggregates intervals to visit each tile exactly once [S1].

1.  **Global Tile Range**:
    Compute the global tile range $[T_{x_{min}}, T_{x_{max}}] \times [T_{y_{min}}, T_{y_{max}}]$ by taking the min/max of the four AABBs using simple integer divisions based on pixel coordinates [S1].

2.  **Scanline Aggregation**:
    Iterate along the shorter axis of the global range. For each scanline (row or column), aggregate the active tile interval by performing fast min/max tests over the intersecting sub-boxes [S1].

3.  **Tile Intersection Check**:
    For each tile in the aggregated range, perform a simple interval test (integer comparison) to confirm intersection. Add the tile-Gaussian pair to the output list [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_inverse_2x2_symmetric(a, b, c):
    """
    Computes the inverse of Lambda = [[a, b], [b, c]].
    Returns (inv_a, inv_b, inv_c) corresponding to [[inv_a, inv_b], [inv_b, inv_c]].
    """
    det = a * c - b * b
    if det <= 0:
        # Degenerate case: not positive definite. 
        # Fallback to standard 3DGS bounding box or skip. [G]
        return None
    inv_a = c / det
    inv_b = -b / det
    inv_c = a / det
    return inv_a, inv_b, inv_c

def quadbox_construction(gaussian):
    """
    Constructs the QuadBox for a single Gaussian.
    Returns a list of 4 AABBs, each defined as (x_min, y_min, x_max, y_max) in tile coordinates.
    """
    # 1. Initialization
    alpha = gaussian.opacity * gaussian.gaussian_value(gaussian.center) # Simplified opacity calc
    alpha_min = 1.0 / 255.0
    if alpha < alpha_min:
        return []

    # 2. Extract 2D Covariance Inverse Lambda
    # Assume gaussian provides 2D covariance Sigma_2D
    Sigma_2D = gaussian.get_2d_covariance()
    a = Sigma_2D[0][0]
    b = Sigma_2D[0][1]
    c = Sigma_2D[1][1]
    
    # Compute Lambda = Sigma_2D^-1
    # Note: The source defines Lambda = (Sigma_2D)^-1. 
    # If Sigma_2D is given, we invert it. 
    # However, the formulas use a,b,c as entries of Lambda.
    # Let's assume we have Lambda directly or compute it.
    # Standard inversion of 2x2 symmetric matrix Sigma:
    det_sigma = Sigma_2D[0][0]*Sigma_2D[1][1] - Sigma_2D[0][1]**2
    if det_sigma <= 0:
        return [] # Degenerate
    
    # Lambda = 1/det * [[S11, -S12], [-S12, S00]]
    lam_a = Sigma_2D[1][1] / det_sigma
    lam_b = -Sigma_2D[0][1] / det_sigma
    lam_c = Sigma_2D[0][0] / det_sigma
    
    # 3. DualBox Extraction
    # Determine major axis. Eigenvectors of Lambda give principal axes.
    # For simplicity, we assume the logic to select the two quadrants aligned with the major axis.
    # This requires computing eigenvectors or comparing a and c if axes are aligned.
    # General case: compute eigenvalues/vectors.
    trace = lam_a + lam_c
    det_lam = lam_a * lam_c - lam_b**2
    sqrt_disc = math.sqrt(trace**2 - 4*det_lam)
    eig1 = (trace + sqrt_disc) / 2
    eig2 = (trace - sqrt_disc) / 2
    
    # Major axis corresponds to smaller eigenvalue of Lambda (larger variance in Sigma)
    # If eig1 < eig2, major axis is associated with eig1.
    # We need to determine which quadrants are "aligned".
    # For implementation, we compute the 4 sub-boxes based on the ellipse geometry.
    
    # 4. Adaptive Stretching
    # Compute stretching factor f
    # f = sqrt(1 - b^2 / (a*c)) where a,b,c are entries of Lambda
    # Using lam_a, lam_b, lam_c
    if lam_a * lam_c == 0:
        f = 1.0 # Avoid division by zero
    else:
        val = 1.0 - (lam_b**2) / (lam_a * lam_c)
        if val < 0: val = 0 # Numerical stability
        f = math.sqrt(val)
        
    # Compute gamma
    # gamma = 2 * ln(theta / alpha_min)
    # theta is opacity. In source, theta_i is opacity.
    theta = gaussian.opacity
    gamma = 2.0 * math.log(theta / alpha_min)
    
    # Compute intercepts
    y_inter = math.sqrt(gamma / lam_c) if lam_c > 0 else 0
    x_inter = math.sqrt(gamma / lam_a) if lam_a > 0 else 0
    
    # Compute max extents
    # x_max = sqrt(gamma * c / (ac - b^2)) -> using Lambda entries: sqrt(gamma * lam_a / det_lam) ?
    # Source Eq 11: x_max = sqrt(gamma * c / (ac - b^2)) where a,b,c are Lambda entries.
    # So x_max = sqrt(gamma * lam_a / det_lam)
    # y_max = sqrt(gamma * lam_c / det_lam)
    
    det_lam_val = lam_a * lam_c - lam_b**2
    if det_lam_val <= 0:
        return []
        
    x_max = math.sqrt(gamma * lam_a / det_lam_val)
    y_max = math.sqrt(gamma * lam_c / det_lam_val)
    
    # Construct 4 AABBs
    # The QuadBox consists of 4 sub-boxes.
    # The DualBox covers the major axis quadrants.
    # The other two cover the minor axis quadrants, stretched by f.
    
    # Center in pixel coordinates
    cx, cy = gaussian.center_2d
    
    # Define the 4 quadrants relative to center
    # Quadrant 1: x>0, y>0
    # Quadrant 2: x<0, y>0
    # Quadrant 3: x<0, y<0
    # Quadrant 4: x>0, y<0
    
    # We need to know which axis is major to apply DualBox logic correctly.
    # Let's assume we split based on the major axis direction.
    # For a general implementation, we calculate the bounding box of the ellipse 
    # and then split it.
    
    # Simplified approach: Create 4 boxes that cover the ellipse.
    # Box 1 (Q1): x in [0, x_max], y in [0, y_max] -> Stretched?
    # The source says "adaptively stretched to reflect eccentricity".
    # The factor f is applied to the minor axis extent.
    
    # Let's define the 4 boxes explicitly.
    # If major axis is X:
    #   DualBox covers Q1 and Q4 (or Q2 and Q3) along X.
    #   The other two are stretched in Y.
    
    # To implement strictly from source:
    # The source derives f to stretch the sub-boxes.
    # We generate 4 AABBs.
    
    boxes = []
    
    # Helper to convert pixel bounds to tile bounds
    def pixel_to_tile(p_min, p_max, tile_size=16):
        t_min = int(p_min // tile_size)
        t_max = int(p_max // tile_size)
        if t_min > t_max:
            t_min, t_max = t_max, t_min
        return t_min, t_max

    # Determine orientation
    # If lam_a < lam_c, X is major axis (smaller inverse variance = larger variance)
    is_x_major = lam_a < lam_c
    
    if is_x_major:
        # Major axis is X.
        # DualBox: Quadrants 1 and 4 (positive and negative X, all Y? No, aligned with major axis)
        # Actually, DualBox retains two quadrants aligned with the major axis.
        # If X is major, the ellipse is wide. The "empty" parts are top/bottom.
        # The source says "suppress empty tile coverage along the minor axis".
        # So DualBox covers the left and right sides.
        
        # Box 1 (Right): x in [0, x_max], y in [-y_max, y_max] -> But we split into 4.
        # Let's create 4 boxes.
        # Q1: x>0, y>0. Extent: x_max, y_max * f?
        # Q2: x<0, y>0. Extent: x_max, y_max * f?
        # Q3: x<0, y<0. Extent: x_max, y_max * f?
        # Q4: x>0, y<0. Extent: x_max, y_max * f?
        
        # The source implies 4 boxes.
        # Let's assume the boxes are:
        # 1. [0, x_max] x [0, y_max * f]
        # 2. [-x_max, 0] x [0, y_max * f]
        # 3. [-x_max, 0] x [-y_max * f, 0]
        # 4. [0, x_max] x [-y_max * f, 0]
        
        # However, DualBox only keeps 2. Then QuadBox adds 2 more?
        # "QuadBox adds adaptively stretched sub-boxes to cover the minor axis"
        # This implies the final set is 4 boxes.
        
        # Let's define the 4 boxes with stretching f on the minor axis (Y)
        y_ext = y_max * f
        
        # Q1
        boxes.append((cx, cy, cx + x_max, cy + y_ext))
        # Q2
        boxes.append((cx - x_max, cy, cx, cy + y_ext))
        # Q3
        boxes.append((cx - x_max, cy - y_ext, cx, cy))
        # Q4
        boxes.append((cx, cy - y_ext, cx + x_max, cy))
        
    else:
        # Major axis is Y.
        # Stretch X by f.
        x_ext = x_max * f
        
        # Q1
        boxes.append((cx, cy, cx + x_ext, cy + y_max))
        # Q2
        boxes.append((cx - x_ext, cy, cx, cy + y_max))
        # Q3
        boxes.append((cx - x_ext, cy - y_max, cx, cy))
        # Q4
        boxes.append((cx, cy - y_max, cx + x_ext, cy))

    # Convert to tile coordinates
    tile_boxes = []
    for (x1, y1, x2, y2) in boxes:
        tx1, tx2 = pixel_to_tile(x1, x2)
        ty1, ty2 = pixel_to_tile(y1, y2)
        tile_boxes.append((tx1, ty1, tx2, ty2))
        
    return tile_boxes

def qpass(tile_boxes, gaussian_id):
    """
    Performs the QPass traversal to generate tile-Gaussian pairs.
    """
    if not tile_boxes:
        return []
        
    # 1. Global Tile Range
    all_tx1 = [b[0] for b in tile_boxes]
    all_tx2 = [b[2] for b in tile_boxes]
    all_ty1 = [b[1] for b in tile_boxes]
    all_ty2 = [b[3] for b in tile_boxes]
    
    global_tx_min = min(all_tx1)
    global_tx_max = max(all_tx2)
    global_ty_min = min(all_ty1)
    global_ty_max = max(all_ty2)
    
    # Determine shorter axis for scanline
    height = global_ty_max - global_ty_min
    width = global_tx_max - global_tx_min
    
    pairs = []
    
    if height <= width:
        # Scan along Y (rows)
        for ty in range(global_ty_min, global_ty_max + 1):
            # Aggregate active tile interval in X
            x_min = global_tx_min
            x_max = global_tx_max
            
            for (tx1, ty1, tx2, ty2) in tile_boxes:
                if ty1 <= ty <= ty2:
                    if tx1 < x_min:
                        x_min = tx1
                    if tx2 > x_max:
                        x_max = tx2
            
            # Add tiles in range [x_min, x_max]
            for tx in range(x_min, x_max + 1):
                pairs.append((tx, ty, gaussian_id))
    else:
        # Scan along X (columns)
        for tx in range(global_tx_min, global_tx_max + 1):
            # Aggregate active tile interval in Y
            y_min = global_ty_min
            y_max = global_ty_max
            
            for (tx1, ty1, tx2, ty2) in tile_boxes:
                if tx1 <= tx <= tx2:
                    if ty1 < y_min:
                        y_min = ty1
                    if ty2 > y_max:
                        y_max = ty2
            
            # Add tiles in range [y_min, y_max]
            for ty in range(y_min, y_max + 1):
                pairs.append((tx, ty, gaussian_id))
                
    return pairs
```

### Edge Cases and Pitfalls
*   **Numerical Stability**: Gaussians with very low opacity can cause instability. Handled by filtering Gaussians with opacity $\alpha < \alpha_{\min}$ [S1].
*   **Degenerate Covariance**: If the determinant of $\Sigma_i^{2D}$ is zero or negative, the inverse is undefined. The pseudocode includes a guard for this [G].
*   **Computational Overhead**: Excessive use of square roots and division operations inflates overhead. The stretching factor $f$ is computed using a purely algebraic closed form to mitigate this [S1].
*   **Warp Divergence**: Naive iteration over sub-boxes causes redundant tile checks and warp divergence. QPass avoids this by aggregating intervals [S1].

## Operating characteristics
*   **Time Complexity**: The preprocessing step (QuadBox construction) is $O(1)$ per Gaussian, involving constant number of arithmetic operations. The traversal (QPass) is proportional to the number of tiles covered by the QuadBox, which is significantly reduced compared to standard AABBs.
*   **Memory Complexity**: $O(N)$ for storing the tile-Gaussian pairs, where $N$ is the number of Gaussians. The QuadBox structure itself is small (4 AABBs per Gaussian).
*   **Measured Performance**: QuadBox achieves an average over $1.8\times$ speedup compared to standard 3DGS on Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets [S1]. It outperforms AdR-Gaussian's AABB algorithm by $1.85\times$ [S1].
*   **Strengths**: Tighter bounding reduces false-positive tile visits, leading to faster rendering. The branch-free QPass algorithm is cache-friendly and scalable [S1].
*   **Limitations**: The method relies on the assumption that the Gaussian projection is well-approximated by the ellipse. Degenerate cases (e.g., extremely thin or large Gaussians) may require careful handling of the bounding box limits [S1].

## Background
QuadBox was introduced in the paper "QuadBox: Accelerating 3D Gaussian Splatting with Geometry-Aware Boxes" [S1]. It addresses the inefficiency of conservative bounding boxes in 3D Gaussian Splatting rasterization. The method builds on the concept of adaptive radius bounding boxes (AdR-Gaussian) but improves upon them by using four sub-boxes to better fit the anisotropic shape of projected Gaussians [S1].

## Sources
[S1] QuadBox: Accelerating 3D Gaussian Splatting with Geometry-Aware Boxes, https://doi.org/10.1109/icip61757.2026.11629922, fetched 2026-10-29
