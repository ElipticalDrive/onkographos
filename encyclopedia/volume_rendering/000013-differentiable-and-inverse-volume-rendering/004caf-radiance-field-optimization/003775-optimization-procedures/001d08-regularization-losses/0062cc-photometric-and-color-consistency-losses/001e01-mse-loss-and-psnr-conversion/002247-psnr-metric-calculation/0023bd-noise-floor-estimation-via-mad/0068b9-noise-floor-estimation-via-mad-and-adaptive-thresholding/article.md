# Noise Floor Estimation via MAD and Adaptive Thresholding

Also called: Adaptive Noise Floor Estimation, Static Canvas Decomposition

## Overview
This method generates a soft [0,1] mask from a residual field, gated by a robust per-frame noise floor calculated from the median and Median Absolute Deviation (MAD) of valid residuals [S1, unofficial]. It distinguishes itself from basic MAD estimation by introducing adaptive thresholding based on per-pixel noise statistics and spatial support filtering to isolate static background from dynamic foreground or noise [S1, unofficial]. It builds on [Noise Floor Estimation via MAD](3105-noise-floor-estimation-via-mad.md).

## How it works
**Inputs:**
- `stack`: Array of shape $(T, H, W, C)$ containing multi-channel data (e.g., RGB or RGB-D) across $T$ frames.
- `subject_masks`: Optional list of $T$ binary/soft masks to force inclusion in the static canvas.
- Parameters: `MASK_LO_FLOOR`, `MASK_SIGMA_LO`, `MASK_HI_FLOOR`, `MASK_SIGMA_HI` (constants for thresholding).

**Outputs:**
- `m`: Soft mask array of shape $(T, H, W)$ with values in $[0, 1]$.
- `static_srgb`: Static RGB canvas of shape $(H, W, 3)$.
- `static_depth`: Static depth canvas of shape $(H, W)$.

**Data Structures:**
- `resid`: Array $(T, H, W)$, residual magnitude.
- `sigma`: Array $(H, W)$, per-pixel noise floor estimate.
- `sigma_g`: Scalar, global sensor noise estimate.
- `lo`, `hi`: Arrays $(H, W)$, lower and upper thresholds.
- `strong`: Boolean array $(T, H, W)$, pixels where $m \ge 0.5$.
- `mstack`: Array $(H, W)$, mean mask value across frames.

**Steps:**

1. **Compute Residuals:**
   For each frame $t$, compute the residual as the maximum absolute difference across channels relative to the median channel value:
   $$ \text{resid}[t] = \max_c | \text{stack}[t, :, :, c] - \text{median}(\text{stack}[t, :, :, :], \text{axis}=-1) | $$
   [S1, unofficial]

2. **Estimate Noise Statistics:**
   - Compute per-pixel noise floor $\sigma$ as the median of the residual across frames for each pixel:
     $$ \sigma = \text{median}(\text{resid}, \text{axis}=0) $$
   - Compute global sensor noise $\sigma_g$ as the median of $\sigma$:
     $$ \sigma_g = \text{median}(\sigma) $$
   - Cap $\sigma$ by the global estimate to prevent overestimation in noisy regions:
     $$ \sigma \leftarrow \min(\sigma, 3.0 \cdot \sigma_g + 4.0) $$
   [S1, unofficial]

3. **Compute Adaptive Thresholds:**
   For each pixel, calculate lower ($lo$) and upper ($hi$) thresholds using the per-pixel $\sigma$ and constants:
   $$ lo = \max(\text{MASK\_LO\_FLOOR}, \text{MASK\_SIGMA\_LO} \cdot \sigma) $$
   $$ hi = \max(\text{MASK\_HI\_FLOOR}, \text{MASK\_SIGMA\_HI} \cdot \sigma) $$
   [S1, unofficial]

4. **Generate Initial Soft Mask:**
   For each frame $t$, compute the mask $m[t]$ by normalizing the residual against the adaptive thresholds and clipping to $[0, 1]$:
   $$ m[t] = \text{clip}\left( \frac{\text{resid}[t] - lo}{\max(hi - lo, 10^{-6})}, 0, 1 \right) $$
   [S1, unofficial]

5. **Spatial Support Filtering:**
   - Identify strong pixels: `strong` = $(m \ge 0.5)$.
   - Compute spatial support using a $3 \times 3$ box filter (sum of neighbors including self) on `strong`. Let $S$ be this support count.
   - If a strong pixel has support $S < \frac{2.5}{9.0}$ (i.e., isolated speckles), multiply its mask value by $0.25$:
     $$ m[t] \leftarrow m[t] \cdot \begin{cases} 0.25 & \text{if } \text{strong}[t] \text{ and } S < 2.5/9.0 \\ 1 & \text{otherwise} \end{cases} $$
   [S1, unofficial]

6. **Soft Halo Application:**
   Apply a soft halo to mask edges using a $3 \times 3$ box filter on $m$:
   $$ m \leftarrow \max\left(m, \text{box3}(m) \cdot 0.8\right) $$
   [S1, unofficial]

7. **Subject Mask Integration:**
   If `subject_masks` are provided, OR them into the mask:
   $$ m[t] \leftarrow \max(m[t], \text{clip}(\text{subject\_masks}[t], 0, 1)) $$
   [S1, unofficial]

8. **Static Canvas Extraction:**
   - Compute `mstack` as the mean of $m$ across all frames for each pixel.
   - Identify quiet pixels where `mstack < 0.4`.
   - Compute `static_srgb` and `static_depth` as the NaN-median of the input channels over frames where the pixel is quiet.
   [S1, unofficial]

9. **Inpainting:**
   Identify holes in the static canvas where RGB or depth values are not finite. Inpaint these holes using a standard inpainting algorithm (e.g., nearest-neighbor or diffusion-based) to fill missing regions.
   [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def noise_floor_estimation_and_adaptive_thresholding(
    stack: np.ndarray,
    subject_masks: list[np.ndarray] | None = None,
    mask_lo_floor: float = 0.0,
    mask_sigma_lo: float = 1.0,
    mask_hi_floor: float = 1.0,
    mask_sigma_hi: float = 2.0
) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    """
    stack: (T, H, W, C)
    Returns: m (T, H, W), static_srgb (H, W, 3), static_depth (H, W)
    """
    T, H, W, C = stack.shape
    
    # Step 1: Compute Residuals
    # Median across channels for each frame
    med_channels = np.median(stack, axis=-1, keepdims=True) # (T, H, W, 1)
    # Max absolute diff across channels
    resid = np.abs(stack - med_channels).max(axis=-1) # (T, H, W)
    
    # Step 2: Estimate Noise Statistics
    # Per-pixel sigma: median across frames
    sigma = np.median(resid, axis=0) # (H, W)
    # Global sensor noise
    sigma_g = np.median(sigma)
    # Cap sigma
    sigma = np.minimum(sigma, 3.0 * sigma_g + 4.0)
    
    # Step 3: Compute Adaptive Thresholds
    lo = np.maximum(mask_lo_floor, mask_sigma_lo * sigma)
    hi = np.maximum(mask_hi_floor, mask_sigma_hi * sigma)
    
    # Step 4: Generate Initial Soft Mask
    m = np.zeros((T, H, W))
    for t in range(T):
        denom = np.maximum(hi - lo, 1e-6)
        m[t] = np.clip((resid[t] - lo) / denom, 0.0, 1.0)
        
    # Step 5: Spatial Support Filtering
    strong = (m >= 0.5)
    # Box3 filter: sum of 3x3 neighborhood
    # Using convolution with 1s kernel
    kernel = np.ones((3, 3))
    support = np.zeros_like(strong, dtype=float)
    for t in range(T):
        # Pad for convolution
        padded = np.pad(strong[t].astype(float), 1, mode='edge')
        # Simple box filter implementation
        support[t] = (padded[:-2, :-2] + padded[:-2, 1:-1] + padded[:-2, 2:] +
                      padded[1:-1, :-2] + padded[1:-1, 1:-1] + padded[1:-1, 2:] +
                      padded[2:, :-2] + padded[2:, 1:-1] + padded[2:, 2:])
        
    # Kill isolated speckles
    mask_factor = np.ones_like(m)
    isolated = (strong & (support < 2.5 / 9.0))
    mask_factor[isolated] = 0.25
    m = m * mask_factor
    
    # Step 6: Soft Halo
    # Apply box3 to m
    m_padded = np.pad(m, ((0,0), (1,1), (1,1)), mode='edge')
    m_halo = np.zeros_like(m)
    for t in range(T):
        m_halo[t] = (m_padded[t, :-2, :-2] + m_padded[t, :-2, 1:-1] + m_padded[t, :-2, 2:] +
                     m_padded[t, 1:-1, :-2] + m_padded[t, 1:-1, 1:-1] + m_padded[t, 1:-1, 2:] +
                     m_padded[t, 2:, :-2] + m_padded[t, 2:, 1:-1] + m_padded[t, 2:, 2:])
    m = np.maximum(m, m_halo * 0.8)
    
    # Step 7: Subject Mask Integration
    if subject_masks is not None:
        for t in range(T):
            m[t] = np.maximum(m[t], np.clip(subject_masks[t], 0.0, 1.0))
            
    # Step 8: Static Canvas Extraction
    mstack = np.mean(m, axis=0) # (H, W)
    quiet = (mstack < 0.4)
    
    # Compute static RGB and Depth
    # Assuming stack[:,:,0:3] is RGB and stack[:,:,3] is Depth if C=4, else handle appropriately
    # For generality, we assume first 3 channels are RGB, last is depth if available
    static_srgb = np.full((H, W, 3), np.nan)
    static_depth = np.full((H, W), np.nan)
    
    # Reshape for median calculation
    # We need to compute median per pixel over frames where quiet is True
    for y in range(H):
        for x in range(W):
            if quiet[y, x]:
                # Get frames where this pixel is quiet
                # Note: quiet is per-pixel, so it's constant for all t
                # Collect values
                rgb_vals = []
                depth_vals = []
                for t in range(T):
                    if C >= 3:
                        rgb_vals.append(stack[t, y, x, :3])
                    if C >= 4:
                        depth_vals.append(stack[t, y, x, 3])
                
                if rgb_vals:
                    static_srgb[y, x] = np.nanmedian(rgb_vals, axis=0)
                if depth_vals:
                    static_depth[y, x] = np.nanmedian(depth_vals, axis=0)
                    
    # Step 9: Inpainting
    # Placeholder for inpainting logic
    static_srgb = inpaint_holes(static_srgb)
    static_depth = inpaint_holes(static_depth)
    
    return m, static_srgb, static_depth

def inpaint_holes(data: np.ndarray) -> np.ndarray:
    """Placeholder for inpainting function."""
    # Not given in the sources.
    return data
```

## Operating characteristics
- **Time Complexity:** Dominated by median calculations and box filters. Median over $T$ frames is $O(T \cdot H \cdot W)$. Box filters are $O(H \cdot W)$ per frame. Overall $O(T \cdot H \cdot W)$.
- **Memory Complexity:** $O(T \cdot H \cdot W)$ for residuals and masks. $O(H \cdot W)$ for static canvases.
- **Strengths:** Robust to outliers due to MAD-based floor. Adaptive thresholds handle varying noise levels across the image. Spatial filtering removes noise speckles.
- **Limitations:** Assumes static background exists. Inpainting is required for holes, which may introduce artifacts. Performance depends on $T$.
- **When to use:** When separating static background from dynamic foreground or noise in a video sequence.
- **When not to use:** When no static background exists or when noise is non-stationary in a way that violates the MAD assumption.

## Background
This method is part of the reconstruction pipeline for volumetric video and 4D Gaussian Splatting, specifically for generating a static background canvas to improve rendering quality and training stability [S1, unofficial]. It is used in the `devarshh/volumetric-video-4dgs` repository [S1, unofficial].

## Sources
[S1] devarshh/volumetric-video-4dgs, https://github.com/devarshh/volumetric-video-4dgs, fetched 2026-10-26 (unofficial)
