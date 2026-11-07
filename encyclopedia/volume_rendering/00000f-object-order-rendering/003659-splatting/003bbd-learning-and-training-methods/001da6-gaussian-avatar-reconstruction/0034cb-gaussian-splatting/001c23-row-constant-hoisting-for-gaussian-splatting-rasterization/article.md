# Row-constant hoisting for Gaussian Splatting Rasterization


## Overview
Row-constant hoisting is an optimization for the inner pixel loop of a Gaussian splatting rasterizer that hoists row-constant terms outside the inner (column) loop to minimize arithmetic operations per pixel [S1, unofficial]. It reduces the computational cost of evaluating the 2D Gaussian exponent by precomputing terms dependent only on the row coordinate [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A projected Gaussian primitive $p$ with screen-space center $(sx, sy)$, bounding box $[x_0, y_0, x_1, y_1]$, inverse covariance matrix elements $a, b, d$, opacity $\alpha_p$, and color $C_p$ [S1, unofficial].
- **Input**: A framebuffer `fb` of width $w$, where each cell stores accumulated color and opacity [S1, unofficial].
- **Input**: Parameters `saturation` (maximum accumulated opacity) and `alpha_threshold` (minimum alpha for contribution) [S1, unofficial].
- **Output**: Updated framebuffer cells for pixels covered by the splat [S1, unofficial].

### Data Structures
- **Framebuffer Cell**: A tuple `(Vec3, f32)` storing the accumulated color (3 floats) and accumulated opacity (1 float) [S1, unofficial].
- **Projected Gaussian**: Contains screen coordinates, bounding box, inverse covariance matrix components, opacity, and color [S1, unofficial].

### Numbered Steps
1. Extract inverse covariance elements $a, b, d$ from the projected Gaussian's inverse covariance matrix [S1, unofficial].
2. Compute coefficients for the quadratic form:
   - $dx_{coeff} = -0.5 \cdot a$
   - $dy_{coeff} = -0.5 \cdot d$
   - $cross_{coeff} = -b$ (derived from $-0.5 \cdot 2 \cdot b$) [S1, unofficial]
3. Initialize `saturation` and `alpha_threshold` from parameters [S1, unofficial].
4. Iterate over each row $py$ from $y_0$ to $y_1$ inclusive:
   a. Compute $dy = py - sy$ [S1, unofficial].
   b. Precompute row-constant terms:
      - $row_{base} = dy_{coeff} \cdot dy^2$
      - $row_{slope} = cross_{coeff} \cdot dy$
      - $row_{offset} = py \cdot w$ (for framebuffer indexing) [S1, unofficial]
   c. Iterate over each column $px$ from $x_0$ to $x_1$ inclusive:
      i. Calculate framebuffer index $idx = row_{offset} + px$ [S1, unofficial].
      ii. Check saturation: if `fb[idx].opacity >= saturation`, skip to next pixel [S1, unofficial].
      iii. Compute $dx = px - sx$ [S1, unofficial].
      iv. Calculate the power term: $power = dx_{coeff} \cdot dx^2 + row_{slope} \cdot dx + row_{base}$ [S1, unofficial].
      v. If $power > 0.0$, skip the pixel (exponential would exceed 1 or be invalid) [S1, unofficial].
      vi. Compute alpha: $\alpha = (\alpha_p \cdot e^{power}).\min(0.999)$ [S1, unofficial].
      vii. If $\alpha < alpha_{threshold}$, skip the pixel [S1, unofficial].
      viii. Compute contribution: $t = 1.0 - fb[idx].opacity$, $contrib = t \cdot \alpha$ [S1, unofficial].
      ix. Update framebuffer: `fb[idx].color += contrib * C_p`, `fb[idx].opacity += contrib` [S1, unofficial].

### Formulas
The Gaussian exponent power is calculated as:
$$power = dx_{coeff} \cdot dx^2 + row_{slope} \cdot dx + row_{base}$$
where:
- $dx_{coeff} = -0.5 \cdot a$
- $row_{slope} = -b \cdot dy$
- $row_{base} = -0.5 \cdot d \cdot dy^2$
- $a, b, d$ are elements of the inverse covariance matrix
- $dx, dy$ are screen-space offsets from the Gaussian center [S1, unofficial]

### Parameters
- **saturation**: The maximum accumulated opacity value in the framebuffer cell. If reached, further splats are skipped [S1, unofficial].
- **alpha_threshold**: The minimum alpha value for a splat to contribute. Splats with alpha below this are skipped [S1, unofficial].

### Edge Cases and Pitfalls
- If the computed power is positive, the exponential term would exceed 1 (or be invalid for the Gaussian model), so the pixel is skipped [S1, unofficial].
- Alpha is clamped to 0.999 to prevent numerical issues when blending [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def composite_splat(p, fb, w, params):
    x0, y0, x1, y1 = p.bbox
    a = p.cov2d_inv[0, 0]  # (0,0)
    b = p.cov2d_inv[0, 1]  # (0,1) = (1,0) since symmetric
    d = p.cov2d_inv[1, 1]  # (1,1)
    
    dx_coeff = -0.5 * a
    dy_coeff = -0.5 * d
    cross_coeff = -b  # -0.5 * 2 * b
    
    saturation = params.saturation
    alpha_threshold = params.alpha_threshold
    opacity = p.opacity
    color = p.color
    sx = p.screen.x
    sy = p.screen.y
    
    for py in range(y0, y1 + 1):
        dy = py - sy
        row_base = dy_coeff * dy * dy
        row_slope = cross_coeff * dy
        row_offset = py * w
        
        for px in range(x0, x1 + 1):
            idx = row_offset + px
            color_acc, opacity_acc = fb[idx]
            
            if opacity_acc >= saturation:
                continue
                
            dx = px - sx
            power = dx_coeff * dx * dx + row_slope * dx + row_base
            
            if power > 0.0:
                continue
                
            alpha = min(opacity * math.exp(power), 0.999)
            
            if alpha < alpha_threshold:
                continue
                
            t = 1.0 - opacity_acc
            contrib = t * alpha
            fb[idx] = (color_acc + contrib * color, opacity_acc + contrib)
```

## Subtopics

### [Gaussian Splatting](946-gaussian-splatting.md)
The parent method that uses 3D Gaussians for scene representation. Row-constant hoisting optimizes its rasterization step.

### [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md)
An alternative rasterization approach that marches rays through Gaussians, differing in computational strategy.

### [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md)
A technique for stabilizing 2D Gaussian projections, which may be used in conjunction with row-constant hoisting.

### [Transformer-based Reconstruction](1794-transformer-based-reconstruction.md)
A different approach to scene reconstruction using transformers, not directly related to rasterization optimization.

## Operating characteristics
Not given in the sources.

## Background
This optimization is described in the tsplat project, which implements Gaussian splatting rasterization in a terminal environment [S1, unofficial].

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
