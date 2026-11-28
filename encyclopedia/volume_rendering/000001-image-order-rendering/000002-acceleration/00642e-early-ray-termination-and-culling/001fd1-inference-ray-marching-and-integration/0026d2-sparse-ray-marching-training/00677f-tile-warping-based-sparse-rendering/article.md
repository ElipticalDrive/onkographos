# Tile Warping-based Sparse Rendering

Also called: Sparse rendering via viewpoint transformation, Tile Warping

## Overview
Tile Warping-based Sparse Rendering (TWSR) is a sparse rendering strategy for 3D Gaussian Splatting (3DGS) that reuses pixels from a reference frame to a target frame by operating at the tile level rather than the pixel level [S1]. It addresses the conflict between standard pixel warping and the 3DGS tile-based architecture, where preprocessing and sorting overheads negate the benefits of pixel-level sparsity [S1]. Unlike pixel-based methods that suffer from floating artifacts and high overhead for tiles with even one missing pixel, TWSR interpolates tiles with few missing pixels and re-renders tiles with many missing pixels, bypassing rasterization, preprocessing, and sorting for interpolated tiles [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference frame $F_{ref}$, reference scene depth map $D_{ref}$, reference truncated depth map $D_{ref}^{max}$, reference viewpoint $V_{ref}$, target viewpoint $V_{tgt}$, and a re-rendering threshold $N_0$ [S1].
*   **Outputs:** Target frame $F_{tgt}$ and a group of tiles $T$ with a tile-level truncated depth map $D_{tgt}^{max}$ [S1].

**Data Structures:**
*   **Frame ($F$):** Image buffer containing color values for each pixel.
*   **Depth Map ($D$):** Per-pixel depth values.
*   **Truncated Depth Map ($D^{max}$):** Per-pixel maximum depth bounds used for culling [S1].
*   **Point Cloud ($P$):** 3D coordinates $(x, y, z)$ derived from back-projection.
*   **Tile ($t$):** A rectangular region of the image. The total number of pixels in a tile is $W \times H$, where $W$ and $H$ are the tile dimensions [S1].

**Parameters:**
*   **$N_0$:** The threshold for the number of missing pixels in a tile. If the count of missing pixels $N > N_0$, the tile is re-rendered; otherwise, it is interpolated [S1]. The text notes $N_0$ is empirically set to less than one-sixth of the total pixels in the tile for small missing pixel counts, but Algorithm 1 uses $N_0$ as the explicit threshold [S1].
*   **Warping Window Size $n$:** Number of frames between every two fully rendered frames. Default value $n=5$ [S1]. One frame is fully rendered every $n+1$ frames [S1].

**Steps:**
1.  **Initialize:** Initialize the target frame $F_{tgt}$ and truncated depth map $D_{tgt}^{max}$ [S1].
2.  **Back-project:** Project pixels from the reference frame $F_{ref}$ into 3D space using $D_{ref}$ and $D_{ref}^{max}$ to form a reference point cloud $P_{ref}$ and truncated point cloud $P_{ref}^{max}$ [S1].
3.  **Transform:** Transform the 3D points $P_{ref}$ and $P_{ref}^{max}$ according to the target frame’s camera pose $V_{tgt}$ to obtain target point clouds $P_{tgt}$ and $P_{tgt}^{max}$ [S1].
4.  **Reproject:** Reproject $P_{tgt}$ and $P_{tgt}^{max}$ onto the target image plane to generate an initial $F_{tgt}$ and $D_{tgt}^{max}$ [S1].
5.  **Iterate Tiles:** For each tile $t$ in the target frame:
    a. Count the number of valid (non-missing) pixels $N$ in $F_{tgt}$ for tile $t$ [S1].
    b. If $N > N_0$:
        i. Interpolate the missing pixels in $F_{tgt}$ for tile $t$ [S1].
        ii. Mark these interpolated pixels as "blank" (masked) to prevent error accumulation in subsequent frames [S1].
    c. Else ($N \le N_0$):
        i. Re-render the entire tile $t$ to ensure visual fidelity [S1].
        ii. Update $D_{tgt}^{max}$ for the tile using the maximum of the current $D_{tgt}^{max}$ and the newly rendered depth [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_warping_sparse_rendering(F_ref, D_ref, D_ref_max, V_ref, V_tgt, N0, n):
    """
    Performs Tile Warping-based Sparse Rendering.
    
    Args:
        F_ref: Reference frame image buffer.
        D_ref: Reference depth map.
        D_ref_max: Reference truncated depth map.
        V_ref: Reference camera viewpoint.
        V_tgt: Target camera viewpoint.
        N0: Threshold for missing pixels.
        n: Warping window size (frames between full renders).
        
    Returns:
        F_tgt: Target frame image buffer.
        D_tgt_max: Target truncated depth map.
    """
    # Step 1: Initialize output structures
    F_tgt = InitializeEmptyFrame()
    D_tgt_max = InitializeEmptyDepthMap()
    
    # Step 2: Back-project reference frame to 3D
    # ProjectTo3D uses camera intrinsic parameters (implied by V_ref)
    # and depth maps to create 3D points.
    P_ref, P_ref_max = ProjectTo3D(F_ref, D_ref, D_ref_max, V_ref)
    
    # Step 3: Transform 3D points to target view
    # ViewTransfer applies the transformation from V_ref to V_tgt
    P_tgt, P_tgt_max = ViewTransfer(P_ref, P_ref_max, V_ref, V_tgt)
    
    # Step 4: Reproject to target image plane
    # This creates a preliminary target frame based on warped geometry
    F_tgt, D_tgt = Reproject(P_tgt, P_tgt_max, V_tgt)
    
    # Step 5: Process each tile
    T = GetTiles(F_tgt) # Get list of all tiles in the image
    
    for t in T:
        # Count valid (non-missing) pixels in the current tile of F_tgt
        # Missing pixels are those where the reprojection failed or depth was invalid
        N = CountValidPixels(F_tgt, t)
        
        if N > N0:
            # Too many missing pixels: Re-render the tile
            # This ensures high visual fidelity for regions with significant occlusion/view change
            ReRenderTile(F_tgt, t, V_tgt)
            
            # Update truncated depth map for this tile
            # Use max to preserve depth bounds
            D_tgt_max[t] = Max(D_tgt_max[t], GetDepthForTile(t))
        else:
            # Few missing pixels: Interpolate
            # Interpolate missing pixels in F_tgt for tile t
            InterpolateTile(F_tgt, t)
            
            # Masking mechanism: Treat interpolated pixels as "blank"
            # This prevents them from contributing to future warping steps
            # to mitigate accumulation of interpolation errors
            MaskInterpolatedPixels(F_tgt, t)
            
    return F_tgt, D_tgt_max

def ProjectTo3D(F, D, D_max, V):
    """
    Back-projects pixels from a 2D frame into 3D space.
    Returns 3D points and truncated 3D points.
    """
    points_3d = []
    points_3d_max = []
    for each pixel (u, v) in F:
        depth = D[u, v]
        depth_max = D_max[u, v]
        if depth is valid:
            # Convert pixel coordinates and depth to 3D using camera pose V
            p = InverseProject(u, v, depth, V)
            points_3d.append(p)
            p_max = InverseProject(u, v, depth_max, V)
            points_3d_max.append(p_max)
    return points_3d, points_3d_max

def ViewTransfer(P_ref, P_ref_max, V_ref, V_tgt):
    """
    Transforms 3D points from reference view to target view.
    """
    P_tgt = []
    P_tgt_max = []
    for p in P_ref:
        # Transform point p from V_ref coordinate system to V_tgt
        p_tgt = TransformPoint(p, V_ref, V_tgt)
        P_tgt.append(p_tgt)
    for p_max in P_ref_max:
        p_tgt_max = TransformPoint(p_max, V_ref, V_tgt)
        P_tgt_max.append(p_tgt_max)
    return P_tgt, P_tgt_max

def Reproject(P, P_max, V):
    """
    Projects 3D points back to 2D image plane.
    """
    F_out = InitializeEmptyFrame()
    D_out = InitializeEmptyDepthMap()
    for p in P:
        u, v = ProjectPoint(p, V)
        if u, v are within image bounds:
            F_out[u, v] = GetColorFromPoint(p) # Color from reference frame
            D_out[u, v] = GetDepthFromPoint(p)
    # Note: P_max is used to determine valid depth ranges, 
    # but the primary output is the color frame F_tgt
    return F_out, D_out

def CountValidPixels(F, t):
    """
    Counts pixels in tile t that are not missing/invalid.
    """
    count = 0
    for pixel in t:
        if F[pixel] is not missing:
            count += 1
    return count

def ReRenderTile(F, t, V):
    """
    Fully renders tile t using 3DGS rasterization.
    """
    # Standard 3DGS rasterization for the tile
    RenderTile3DGS(t, V)

def InterpolateTile(F, t):
    """
    Interpolates missing pixels in tile t.
    """
    for pixel in t:
        if F[pixel] is missing:
            F[pixel] = InterpolateNeighbors(pixel, F)

def MaskInterpolatedPixels(F, t):
    """
    Marks interpolated pixels as blank for future warping.
    """
    for pixel in t:
        if F[pixel] was interpolated:
            F[pixel] = BLANK

def GetDepthForTile(t):
    """
    Returns the depth values for the newly rendered tile.
    """
    return GetCurrentDepth(t)
```

**Edge Cases and Pitfalls:**
*   **Accumulation of Interpolation Errors:** Repeated viewpoint transformations without re-rendering degrade image quality over time [S1]. This is mitigated by the masking mechanism, where interpolated pixels are treated as "blank" in subsequent projections, forcing re-rendering in areas with high error accumulation [S1].
*   **Depth Estimation Errors:** Unlike NeRF, 3DGS estimates depth in real-time by computing an opacity-weighted sum over the depths of contributing Gaussians. This introduces minor errors that can be amplified in regions with limited pixel reuse, causing projected pixel positions to deviate from actual results [S1].
*   **Tile Preprocessing Overhead:** In 3DGS, preprocessing and sorting cannot be skipped unless no pixel within a tile requires rendering. TWSR avoids this overhead for tiles with few missing pixels by interpolating them, whereas pixel-based methods still incur this cost if any pixel in the tile is missing [S1].

## Operating characteristics
*   **Time Complexity:** Reduces computational load by skipping rasterization, preprocessing, and sorting for tiles with few missing pixels [S1]. The speedup increases with the warping window size $n$, but quality degrades [S1].
*   **Memory Complexity:** Requires storage for reference frames, depth maps, and point clouds. The masking mechanism adds minimal overhead [S1].
*   **Measured Performance:**
    *   On Synthetic-NeRF dataset: TWSR average SSIM loss 0.005, PSNR loss 1.4 dB [S1]. Potamoi SSIM loss 0.063, PSNR loss 6.8 dB [S1].
    *   On Jetson AGX Orin: LS-Gaussian (using TWSR) achieves 1.85x speedup over AdR-Gaussian and 1.75x over SeeLe [S1].
    *   On Deep Blending dataset: LS-Gaussian achieves 5.41x speedup over GPU baseline, satisfying 90 FPS real-time rendering [S1].
*   **Scaling:** Indoor scenes with flattened structures and uniform color patterns achieve better rendering quality and higher speedups compared to outdoor scenes with more edges and corners [S1].
*   **Strengths:** Avoids floating pixels and artifacts caused by pixel-based inpainting; bypasses preprocessing and sorting for interpolated tiles; balances efficiency and image quality via masking [S1].
*   **Limitations:** Quality degrades with larger warping window sizes $n$; sensitive to depth estimation errors; indoor scenes perform better than outdoor scenes [S1].
*   **When to Use:** Real-time rendering on edge devices (e.g., Jetson AGX Orin) where computational resources are limited and scene geometry is relatively stable (e.g., indoor environments) [S1].
*   **When Not to Use:** Scenes with high dynamic range, large viewpoint changes, or complex geometry where interpolation errors accumulate rapidly [S1].

## Background
Tile Warping-based Sparse Rendering (TWSR) was proposed in the paper "No Redundancy, No Stall: Lightweight Streaming 3D Gaussian Splatting for Real-time Rendering" [S1]. It addresses the limitations of existing pixel warping-based sparse rendering (PWSR) methods like Potamoi, which conflict with the 3DGS tile-based architecture [S1]. The method was developed to enable real-time rendering on edge devices such as the Jetson AGX Orin [S1].

## Sources
[S1] No Redundancy, No Stall: Lightweight Streaming 3D Gaussian Splatting for Real-time Rendering, https://doi.org/10.1109/iccad66269.2025.11240950, fetched 2026-10-30
