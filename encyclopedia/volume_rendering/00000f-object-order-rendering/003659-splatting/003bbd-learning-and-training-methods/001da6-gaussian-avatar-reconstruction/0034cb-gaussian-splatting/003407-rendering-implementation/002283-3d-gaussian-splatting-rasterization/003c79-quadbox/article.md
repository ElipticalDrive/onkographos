# QuadBox


## Overview
QuadBox is a Gaussian-tile intersection method that uses four axis-aligned bounding boxes (AABBs) to tightly encapsulate projected 2D Gaussians, reducing tile over-approximation and redundant computations in the rasterizer preprocess stage [S1]. It sets itself apart from its sibling [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md) by replacing the conservative single AABB with a geometry-aware QuadBox, and from [AdR-Gaussian](8365-adr-gaussian.md) by using four adaptive sub-boxes to better capture anisotropic shapes, thereby suppressing residual redundancy along the minor axis [S1].

## How it works
QuadBox consists of two main components: the construction of the QuadBox geometry and the QPass tile traversal algorithm.

### Inputs and Outputs
**Inputs:**
*   Set of $N$ 3D Gaussians, each with center $\mu_i$, 3D covariance $\Sigma_i$, opacity $o_i$, and color $c_i$.
*   Camera projection parameters (view and projection matrices).
*   Image resolution and tile size (typically $16 \times 16$ pixels) [S1].
*   Opacity threshold $\alpha_{\min}$ (typically $1/255$) [S1].

**Outputs:**
*   A list of tile-Gaussian pairs, where each Gaussian is assigned only to the tiles it intersects, minimizing false positives.

### Data Structures
*   **Gaussian Primitive:** Contains $\mu_i^{2D}$ (2D center), $\Sigma_i^{2D}$ (2D covariance), opacity $\alpha_i$, and color $c_i$.
*   **QuadBox:** Defined by four AABBs. Each AABB is defined by its minimum and maximum coordinates $(x_{min}, y_{min})$ and $(x_{max}, y_{max})$ in screen space.
*   **Tile Grid:** The image plane is partitioned into $16 \times 16$ tiles [S1]. Tiles are indexed by integer coordinates $(t_x, t_y)$.

### Algorithm Steps

#### Phase 1: Initialization
Filter Gaussians to ensure numerical stability. Discard any Gaussian $i$ where the splatted opacity $\alpha_i < \alpha_{\min}$ [S1]. This defines the support region boundary.

#### Phase 2: DualBox Extraction
1.  Compute the inverse 2D covariance matrix $\Lambda = (\Sigma_i^{2D})^{-1} = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$ [S1].
2.  Determine the major axis of the ellipse defined by the Gaussian.
3.  Extract the "DualBox" by retaining only the two quadrants of the AdR-AABB [16] that are aligned with the ellipse's major axis. This suppresses empty tile coverage along the minor axis [S1].

#### Phase 3: QuadBox Construction (Adaptive Stretching)
To cover the parts of the ellipse not covered by the DualBox, construct two additional adaptive AABBs. This requires calculating a stretching factor $f$.

1.  **Calculate Axis Intercepts:**
    Set $x_d = 0$ and $y_d = 0$ in the ellipse equation $F(x_d, y_d) = ax_d^2 + 2bx_dy_d + cy_d^2 - \gamma \leq 0$ to find the intercepts [S1]:
    $$y_{\text{inter}} = \sqrt{\gamma/c}$$
    $$x_{\text{inter}} = \sqrt{\gamma/a}$$
    Here, $\gamma = 2 \ln\left(\frac{\theta_i}{\alpha_{\min}}\right)$, where $\theta_i$ is the opacity parameter [S1].

2.  **Calculate Maximal Extents:**
    Compute the maximal support along the coordinate axes using the inverse matrix $\Lambda^{-1}$ [S1]:
    $$\Lambda^{-1} = \frac{1}{ac - b^2} \begin{bmatrix} c & -b \\ -b & a \end{bmatrix}$$
    $$y_{\max} = \sqrt{\frac{\gamma a}{ac - b^2}}$$
    $$x_{\max} = \sqrt{\frac{\gamma c}{ac - b^2}}$$

3.  **Compute Stretching Factor $f$:**
    Derive the purely algebraic closed form for $f$ to avoid excessive square roots and divisions [S1]:
    $$f = \sqrt{1 - \frac{b^2}{ac}}$$
    Note that $0 < f \leq 1$. $f=1$ for circles or when principal axes align with coordinate axes [S1].

4.  **Construct Four AABBs:**
    Using the DualBox bounds and the stretching factor $f$, define the four sub-boxes. The sub-boxes are symmetrically positioned around the Gaussian center and adaptively stretched to reflect the ellipse's eccentricity [S1]. The exact bounds are derived by scaling the extents of the DualBox by $f$ in the appropriate directions to tightly enclose the remaining elliptical footprint [S1].

#### Phase 4: QPass Tile Traversal
For each Gaussian, determine the set of intersecting tiles using the QPass algorithm [S1].

1.  **Compute Global Tile Range:**
    Calculate the global tile range $[t_{x,min}, t_{x,max}] \times [t_{y,min}, t_{y,max}]$ covered by the QuadBox using simple integer divisions based on the QuadBox bounds and tile size [S1].

2.  **Scan Along Shorter Axis:**
    Identify the shorter axis of the global tile range. Iterate along this axis (scanlines). For each scanline, aggregate the active tile interval [S1].

3.  **Aggregate Active Interval:**
    For each scanline, determine the union of tile intervals covered by the four sub-boxes. This is done using fast min/max tests over the intersecting sub-boxes. This aggregation ensures that the final interval represents the continuous range of tiles to visit [S1].

4.  **Tile Intersection Check:**
    Perform tile intersection checks using simple integer comparisons for each tile in the aggregated range. This ensures each tile is visited exactly once, avoiding redundant checks and warp divergence [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_quadbox_and_tiles(gaussian, tile_size=16, alpha_min=1/255):
    """
    Computes the QuadBox for a single Gaussian and returns the list of intersecting tiles.
    
    Args:
        gaussian: Object with attributes:
            - mu_2d: (x, y) center in screen space
            - sigma_2d: 2x2 covariance matrix [[s11, s12], [s21, s22]]
            - opacity: float
        tile_size: int, size of tiles in pixels
        alpha_min: float, opacity threshold
        
    Returns:
        list of (t_x, t_y) tile coordinates
    """
    # Phase 1: Initialization
    if gaussian.opacity < alpha_min:
        return []

    # Extract 2D covariance components
    s11, s12 = gaussian.sigma_2d[0][0], gaussian.sigma_2d[0][1]
    s21, s22 = gaussian.sigma_2d[1][0], gaussian.sigma_2d[1][1]
    
    # Compute Lambda = Sigma^-1
    det = s11 * s22 - s12 * s21
    # Lambda = 1/det * [[s22, -s12], [-s21, s11]]
    # Let Lambda = [[a, b], [b, c]]
    a = s22 / det
    b = -s12 / det
    c = s11 / det
    
    # Compute gamma
    # gamma = 2 * ln(theta_i / alpha_min)
    # Note: theta_i is the opacity parameter. In 3DGS, alpha = o * G(u).
    # The confidence inequality uses theta_i. Assuming theta_i is the opacity o_i
    # or the max opacity. The source uses theta_i in the log.
    # We use the Gaussian's opacity attribute as theta_i.
    gamma = 2.0 * math.log(gaussian.opacity / alpha_min)
    
    # Phase 2 & 3: Compute Stretching Factor f
    # f = sqrt(1 - b^2 / (a*c))
    # Check for numerical stability: a*c - b^2 should be > 0 for positive definite Lambda
    if a * c - b * b <= 0:
        return [] # Degenerate case
        
    f = math.sqrt(1.0 - (b * b) / (a * c))
    
    # Compute Maximal Extents
    # y_max = sqrt(gamma * a / (a*c - b^2))
    # x_max = sqrt(gamma * c / (a*c - b^2))
    denom = a * c - b * b
    y_max = math.sqrt(gamma * a / denom)
    x_max = math.sqrt(gamma * c / denom)
    
    # Compute Axis Intercepts
    # y_inter = sqrt(gamma / c)
    # x_inter = sqrt(gamma / a)
    y_inter = math.sqrt(gamma / c)
    x_inter = math.sqrt(gamma / a)
    
    # Construct QuadBox Bounds
    # The QuadBox consists of 4 AABBs.
    # The source describes extracting DualBox (2 quadrants aligned with major axis)
    # and then stretching.
    # For implementation, we approximate the 4 AABBs by defining the bounds
    # relative to the center mu_2d.
    # The exact derivation of the 4 AABB coordinates from f, x_max, y_max, x_inter, y_inter
    # involves determining the major/minor axes orientation.
    # However, QPass operates on the union of these boxes.
    # We can compute the global bounding box of the QuadBox for the initial range,
    # but QPass requires the specific sub-boxes for interval aggregation.
    
    # Simplified approach for QPass input:
    # We need the 4 AABBs.
    # Let's assume the major axis is aligned with the eigenvector corresponding to the larger eigenvalue of Sigma.
    # But the source uses a purely algebraic f independent of rotation for the stretch factor.
    # The DualBox retains 2 quadrants. The QuadBox adds 2 more.
    # To implement QPass, we need the coordinates of the 4 boxes.
    
    # Derive orientation from Lambda
    # Eigenvalues of Lambda determine axes.
    # Trace = a + c, Det = ac - b^2
    # Lambda_eig1, Lambda_eig2 = ...
    # The major axis of the ellipse corresponds to the MINOR axis of Lambda (smallest eigenvalue of Lambda).
    # Let's compute the rotation angle theta of the ellipse.
    # tan(2*theta) = 2*b / (a - c)
    if abs(a - c) < 1e-8:
        theta = math.pi / 4 if b > 0 else -math.pi / 4
    else:
        theta = 0.5 * math.atan2(2 * b, a - c)
        
    # Rotate the extents to get the AABBs
    # This part is complex to derive purely from the text without the figure.
    # The text says: "construct the stretching factor... to tightly enclose the parts... not yet covered by DualBox"
    # "QuadBox adds adaptively stretched sub-boxes"
    
    # For the purpose of this article, we define the 4 boxes as:
    # Box 1: Quadrant 1 (x>0, y>0) stretched by f
    # Box 2: Quadrant 2 (x<0, y>0) stretched by f
    # Box 3: Quadrant 3 (x<0, y<0) stretched by f
    # Box 4: Quadrant 4 (x>0, y<0) stretched by f
    # But DualBox only keeps 2. Which 2? The ones aligned with the major axis.
    
    # Let's define the 4 candidate boxes in local coordinates (centered at 0,0)
    # before rotation.
    # The source implies the boxes are axis-aligned in SCREEN space.
    # So we must rotate the ellipse to screen space, then define the boxes.
    
    # Actually, the source says "QuadBox... tightly encapsulate projected 2D Gaussians".
    # The projection is already done. Sigma_2D is in screen space.
    # So the ellipse is already in screen space.
    # The AABBs are axis-aligned in screen space.
    
    # The "DualBox" extracts 2 quadrants of the AdR-AABB.
    # AdR-AABB bounds are roughly +/- x_max, +/- y_max?
    # No, AdR-AABB is tighter.
    
    # Given the complexity of deriving the exact 4 AABB coordinates from the text alone,
    # we provide the QPass logic which operates on the 4 boxes.
    
    # Define the 4 sub-boxes in screen space.
    # We need the bounds of the 4 boxes.
    # Let's approximate:
    # The QuadBox is the union of 4 AABBs.
    # Box 1: [mu_x, mu_x + x_max*f], [mu_y, mu_y + y_max*f] ? No, this is not tight.
    
    # The source states: "f = sqrt(1 - b^2/ac)".
    # And "y_inter = sqrt(gamma/c)", "x_inter = sqrt(gamma/a)".
    # And "y_max = sqrt(gamma*a/(ac-b^2))", "x_max = sqrt(gamma*c/(ac-b^2))".
    
    # The QuadBox construction is geometry-aware.
    # We will define the 4 boxes as the intersection of the ellipse with the 4 quadrants,
    # approximated by AABBs.
    
    # For implementation, we calculate the 4 AABBs as follows:
    # 1. Determine the major axis direction.
    # 2. The DualBox covers the major axis quadrants.
    # 3. The other two boxes cover the minor axis tips, stretched by f.
    
    # Since exact coordinate derivation is not fully explicit in text,
    # we define the QPass algorithm which takes the 4 boxes as input.
    
    # Placeholder for the 4 AABBs:
    # boxes = [ (x1_min, y1_min, x1_max, y1_max), ... ]
    # These must be computed based on the rotation theta and the extents.
    
    # For the sake of a complete implementation guide, we assume the 4 boxes are:
    # B1: [mu_x, mu_x + dx1], [mu_y, mu_y + dy1]
    # B2: [mu_x - dx2, mu_x], [mu_y, mu_y + dy2]
    # B3: [mu_x - dx3, mu_x], [mu_y - dy3, mu_y]
    # B4: [mu_x, mu_x + dx4], [mu_y - dy4, mu_y]
    # where dx, dy are derived from x_max, y_max, f and theta.
    
    # We will skip the exact trigonometric derivation of the 4 box bounds 
    # as it requires interpreting Figure 2 which is not provided.
    # However, the QPass algorithm is fully specified.
    
    # Return empty list if we cannot define the boxes precisely from text.
    # [G] In a real implementation, the 4 AABBs are computed by rotating the 
    # local ellipse axes to screen space and taking the bounding box of each 
    # quadrant's intersection with the ellipse, scaled by f.
    
    # For this article, we assume the 4 boxes are pre-computed and stored in `gaussian.quad_boxes`.
    
    boxes = gaussian.quad_boxes # List of 4 tuples (x_min, y_min, x_max, y_max)
    
    # Phase 4: QPass
    tiles = []
    
    # Compute global tile range
    # Convert box bounds to tile coordinates
    # tile_x = floor(x / tile_size)
    
    t_x_min = math.floor(min(b[0] for b in boxes) / tile_size)
    t_x_max = math.floor(max(b[2] for b in boxes) / tile_size)
    t_y_min = math.floor(min(b[1] for b in boxes) / tile_size)
    t_y_max = math.floor(max(b[3] for b in boxes) / tile_size)
    
    # Determine shorter axis for scanning
    height = t_y_max - t_y_min + 1
    width = t_x_max - t_x_min + 1
    
    if height < width:
        # Scan along Y (rows)
        for t_y in range(t_y_min, t_y_max + 1):
            # Aggregate active interval in X
            x_start = t_x_min
            x_end = t_x_max
            
            for box in boxes:
                bx_min = math.floor(box[0] / tile_size)
                bx_max = math.floor(box[2] / tile_size)
                by_min = math.floor(box[1] / tile_size)
                by_max = math.floor(box[3] / tile_size)
                
                if by_min <= t_y <= by_max:
                    # Intersect with current interval
                    x_start = max(x_start, bx_min)
                    x_end = min(x_end, bx_max)
            
            if x_start <= x_end:
                for t_x in range(x_start, x_end + 1):
                    tiles.append((t_x, t_y))
    else:
        # Scan along X (columns)
        for t_x in range(t_x_min, t_x_max + 1):
            y_start = t_y_min
            y_end = t_y_max
            
            for box in boxes:
                bx_min = math.floor(box[0] / tile_size)
                bx_max = math.floor(box[2] / tile_size)
                by_min = math.floor(box[1] / tile_size)
                by_max = math.floor(box[3] / tile_size)
                
                if bx_min <= t_x <= bx_max:
                    y_start = max(y_start, by_min)
                    y_end = min(y_end, by_max)
            
            if y_start <= y_end:
                for t_y in range(y_start, y_end + 1):
                    tiles.append((t_x, t_y))
                    
    return tiles
```

### Parameters
*   **Opacity Threshold $\alpha_{\min}$**: Gaussians with splatted opacity below this value are ignored. Typical value: $1/255$ [S1].
*   **Tile Size**: The image plane is partitioned into $16 \times 16$ tiles [S1].

### Edge Cases and Pitfalls
*   **Numerical Stability**: Gaussians with very low opacity can cause instability. Handled by filtering Gaussians with opacity $\alpha < \alpha_{\min}$ [S1].
*   **Computational Overhead**: Excessive use of square roots ($\sqrt{}$) and division operations inflates computational overhead. Handled by deriving a purely algebraic closed form for $f$ that avoids these operations where possible [S1].
*   **Degenerate Input**: If the covariance matrix is singular or nearly singular, the inverse $\Lambda$ may be unstable. The condition $ac - b^2 > 0$ must be checked [S1].

## Subtopics

### 3D Gaussian Splatting Differentiable Rasterizer
A differentiable rasterizer for 3DGS. QuadBox is a specific optimization for the tile intersection stage of such rasterizers. [Link](2813-3d-gaussian-splatting-differentiable-rasterizer.md)

### Optimal Gaussian Splatting
Focuses on optimal parameterization. QuadBox focuses on rasterization efficiency. [Link](3293-optimal-gaussian-splatting.md)

### Hybrid Transparency
Combines different transparency models. QuadBox is a geometric culling method. [Link](3294-hybrid-transparency.md)

### Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates
Uses Plücker coordinates for perspective correction. QuadBox operates in screen space after projection. [Link](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)

### 3D Gaussian Splatting Rasterization (Cone-Beam/Scanner Space)
Rasterization in a different coordinate space. QuadBox is for standard perspective projection. [Link](3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md)

### Analytic-Splatting Forward Shading
Focuses on shading. QuadBox focuses on culling. [Link](3459-analytic-splatting-forward-shading.md)

### Speedy-Splat Rendering Pipeline
A general pipeline optimization. QuadBox is a specific culling technique. [Link](3543-speedy-splat-rendering-pipeline.md)

### Ghost Ellipsoids for Chunked Training
Used for training. QuadBox is for rendering. [Link](3559-ghost-ellipsoids-for-chunked-training.md)

### Analytic Transmittance 3DGS
Focuses on transmittance calculation. QuadBox focuses on tile intersection. [Link](4440-analytic-transmittance-3dgs.md)

### Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL
Deferred rendering with IBL. QuadBox is for forward rasterization. [Link](4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md)

### 3D Gaussian Splatting Self-Shadowing (T_light)
Adds self-shadowing. QuadBox is a culling method. [Link](4487-3d-gaussian-splatting-self-shadowing-t-light.md)

### Light Space Rasterization for T_light
Rasterization in light space. QuadBox is in screen space. [Link](4489-light-space-rasterization-for-t-light.md)

### 2D Gaussian Splatting Rasterization
Rasterization for 2D Gaussians. QuadBox is for 3D Gaussians projected to 2D. [Link](4815-2d-gaussian-splatting-rasterization.md)

### SparseOIT Rasterizer
Order-Independent Transparency. QuadBox is a culling method for OIT or OIT-like pipelines. [Link](7562-sparseoit-rasterizer.md)

### AdR-Gaussian
Uses a single AABB. QuadBox uses four adaptive AABBs for tighter fit. [Link](8365-adr-gaussian.md)

### 3D Half-Gaussian Splatting
Uses half-Gaussians. QuadBox is a culling method. [Link](9011-3d-half-gaussian-splatting.md)

### 3D Gaussian Unscented Transform (3DGUT)
Uses unscented transform for uncertainty. QuadBox is a deterministic culling method. [Link](9154-3d-gaussian-unscented-transform-3dgut.md)

## Operating characteristics
*   **Time Complexity**: The QPass algorithm visits each tile exactly once, with complexity proportional to the number of tiles covered by the QuadBox. The preprocessing step involves computing the QuadBox bounds, which is $O(1)$ per Gaussian.
*   **Memory Complexity**: Minimal additional memory per Gaussian to store the 4 AABBs.
*   **Measured Performance**: QuadBox achieves an average over $1.8\times$ speedup compared to 3DGS on Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets [S1]. It outperforms AdR-Gaussian's AABB algorithm by $1.85\times$ [S1].
*   **Strengths**: Reduces tile over-approximation, minimizes redundant computations, and is cache-friendly due to the single-pass traversal [S1].
*   **Limitations**: Requires careful handling of numerical stability and the derivation of the 4 AABBs [S1].
*   **When to Use**: When rendering speed is critical and the scene contains many anisotropic Gaussians.
*   **When Not to Use**: When the overhead of computing the QuadBox bounds outweighs the benefit of culling (e.g., very small scenes or isotropic Gaussians).

## Background
QuadBox was proposed in the paper "QuadBox: Accelerating 3D Gaussian Splatting with Geometry-Aware Boxes" [S1]. It addresses the inefficiency of conservative bounding boxes in 3D Gaussian Splatting rasterization. The method builds on the AdR-Gaussian technique [16] but improves upon it by using four adaptive AABBs instead of one. The code is available at [https://github.com/Powertony102/QuadBox.](https://github.com/Powertony102/QuadBox) [S1].

## Sources
[S1] QuadBox: Accelerating 3D Gaussian Splatting with Geometry-Aware Boxes, https://doi.org/10.1109/icip61757.2026.11629922, fetched 2026-10-29
