# Depth Prediction for Early Stopping (DPES)
Also called: Early Stopping Depth Prediction

## Overview
Depth Prediction for Early Stopping (DPES) is a technique used in streaming 3D Gaussian Splatting to predict the early stopping position for each tile in a target frame by leveraging truncated depth information from a reference frame [S1]. It solves the problem of redundant computation and load imbalance during viewpoint changes by allowing the renderer to exclude Gaussians that are guaranteed to be occluded or beyond the visible depth range in the new view [S1]. Unlike standard early termination which relies on per-pixel opacity thresholds during rendering, DPES pre-computes a tile-level depth bound to prune the Gaussian sorting stage entirely [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference frame color buffer $F_{ref}$, reference scene depth map $D_{ref}$, reference truncated depth max map $D_{ref}^{max}$, reference viewpoint $V_{ref}$, target viewpoint $V_{tgt}$, and a rerendering threshold $N_0$ [S1].
*   **Outputs:** Target frame color buffer $F_{tgt}$, a group of tiles $T$, and a tile-level truncated depth map $D_T^{max}$ [S1].

**Data Structures:**
*   $F_{ref}, F_{tgt}$: 2D arrays of pixel colors.
*   $D_{ref}, D_{ref}^{max}$: 2D arrays of depth values. $D_{ref}$ stores the depth of the last traversed Gaussian or the early stopping depth [S1]. $D_{ref}^{max}$ stores the truncated depth bound [S1].
*   $D_T^{max}$: 2D array where each element corresponds to a tile and stores the maximum valid reprojected depth within that tile [S1].

**Steps:**
1.  **Record Reference Depths:** During the initial rendering of the reference frame, record the depth of each pixel. This depth corresponds to the depth of the last Gaussian traversed or the depth at which early stopping occurred [S1].
2.  **Project to 3D:** Convert the 2D pixel coordinates and depth values from the reference frame into 3D world coordinates using the reference viewpoint $V_{ref}$ [S1]. This yields 3D points $P_{ref}$ and truncated depth bounds $P_{ref}^{max}$ [S1].
3.  **View Transfer:** Transform the 3D points $P_{ref}$ and $P_{ref}^{max}$ from the reference viewpoint $V_{ref}$ to the target viewpoint $V_{tgt}$ [S1]. This yields transformed points $P_{tgt}$ and $P_{tgt}^{max}$ [S1].
4.  **Reproject to Target Frame:** Project the transformed 3D points $P_{tgt}$ and $P_{tgt}^{max}$ back onto the 2D image plane of the target viewpoint $V_{tgt}$ [S1]. This produces the target frame color buffer $F_{tgt}$ (interpolated or reprojected colors) and the target depth map $D_{tgt}$ [S1].
5.  **Tile-Level Processing:** Iterate through each tile $t$ in the target frame [S1]:
    *   Count the number of valid pixels $N$ within the tile $t$ in $F_{tgt}$ [S1].
    *   If $N > N_0$: Interpolate the color values for the tile [S1].
    *   If $N \le N_0$: Calculate the early stopping depth for the tile as the maximum depth of all valid reprojected pixels within that tile: $D_T^{max}(t) = \max(D_{tgt}, t)$ [S1]. Use this depth to re-render the tile, excluding any Gaussians with depth beyond $D_T^{max}(t)$ from the sorting stage [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_prediction_for_early_stopping(F_ref, D_ref, D_ref_max, V_ref, V_tgt, N0):
    """
    Predicts early stopping positions for tiles in the target frame.
    
    Args:
        F_ref: Reference frame color buffer.
        D_ref: Reference scene depth map.
        D_ref_max: Reference truncated depth max map.
        V_ref: Reference viewpoint (camera matrix/intrinsics).
        V_tgt: Target viewpoint (camera matrix/intrinsics).
        N0: Rerendering threshold.
        
    Returns:
        F_tgt: Target frame color buffer.
        T: List of tiles.
        D_T_max: Tile-level truncated depth map.
    """
    # Initialize outputs
    F_tgt = Initialize()
    D_T_max = Initialize()
    T = GetTiles()
    
    # Step 2: Project to 3D
    # Convert 2D pixels and depths to 3D world coordinates
    P_ref, P_ref_max = ProjectTo3D(F_ref, D_ref, D_ref_max, V_ref)
    
    # Step 3: View Transfer
    # Transform 3D points to target camera space
    P_tgt, P_tgt_max = ViewTransfer(P_ref, P_ref_max, V_ref, V_tgt)
    
    # Step 4: Reproject to Target Frame
    # Project 3D points back to 2D target image plane
    F_tgt, D_tgt = Reproject(P_tgt, P_tgt_max, V_tgt)
    
    # Step 5: Tile-Level Processing
    for t in T:
        # Count valid pixels in the tile
        N = ValidPixelSum(F_tgt, t)
        
        if N > N0:
            # Interpolate if enough valid pixels are available
            Interpolate(F_tgt, t)
        else:
            # Calculate max depth for early stopping
            D_T_max[t] = Max(D_tgt, t)
            # Re-render with depth pruning
            Re_render(F_tgt, t, D_T_max[t])
            
    return F_tgt, T, D_T_max

def ProjectTo3D(F, D, D_max, V):
    """
    Projects 2D pixels to 3D world coordinates.
    """
    # Implementation depends on camera intrinsics V
    # For each pixel (u, v) with depth d:
    # x = (u - cx) * d / fx
    # y = (v - cy) * d / fy
    # z = d
    # Return 3D points and corresponding max depths
    pass

def ViewTransfer(P, P_max, V_ref, V_tgt):
    """
    Transforms 3D points from reference to target viewpoint.
    """
    # Apply inverse of V_ref to get world coordinates
    # Apply V_tgt to get target camera coordinates
    # Return transformed points
    pass

def Reproject(P, P_max, V):
    """
    Projects 3D points to 2D target image plane.
    """
    # Apply V to get clip space
    # Perspective divide
    # Return 2D coordinates and depths
    pass

def ValidPixelSum(F, t):
    """
    Counts valid (non-empty/valid depth) pixels in tile t.
    """
    count = 0
    for pixel in t:
        if IsValid(F[pixel]):
            count += 1
    return count

def Max(D, t):
    """
    Returns the maximum depth value in tile t.
    """
    max_val = -infinity
    for pixel in t:
        if IsValid(D[pixel]):
            max_val = max(max_val, D[pixel])
    return max_val
```

**Parameters:**
*   $N_0$: Rerendering threshold. Heuristically defined as $3\lambda_1$ and $3\lambda_2$ in the source, though exact values for $\lambda_1$ and $\lambda_2$ are not provided in the excerpt [S1].

**Edge Cases and Pitfalls:**
*   **Sparse Valid Pixels:** If the number of valid reprojected pixels $N$ is low ($N \le N_0$), the method triggers re-rendering with depth pruning. If $N$ is too low, the depth estimate $D_T^{max}$ may be noisy or incomplete, potentially leading to incorrect Gaussian culling if the max depth is underestimated [S1].
*   **Occlusion:** The method assumes that the depth from the reference frame is a valid upper bound for the target frame. If significant occlusion changes occur, the predicted depth might be inaccurate [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** The method adds an overhead of projecting and reprojecting depths for the reference frame. However, it reduces the sorting complexity for tiles that are re-rendered by excluding Gaussians beyond the predicted depth [S1].
*   **Memory Complexity:** Requires storing the reference depth maps $D_{ref}$ and $D_{ref}^{max}$ and the target depth map $D_{tgt}$ [S1].
*   **Strengths:** Enables balanced workload distribution across rasterization units by predicting expected load per tile [S1]. Reduces redundant computation by pruning Gaussians early [S1].
*   **Limitations:** Relies on the accuracy of the reference depth map. Performance degrades if the viewpoint change is large or if the scene has significant dynamic occlusion [S1].
*   **When to use:** Real-time streaming 3D Gaussian Splatting where viewpoint changes are frequent and computational resources are limited [S1].

## Background
*   **Origin:** Proposed in "No Redundancy, No Stall: Lightweight Streaming 3D Gaussian Splatting for Real-time Rendering" [S1].
*   **History:** Developed to address the issue of redundant computation and load imbalance in streaming 3D Gaussian Splatting [S1].
*   **Usage:** Used in real-time rendering of 3D Gaussian Splatting scenes [S1].

## Sources
[S1] No Redundancy, No Stall: Lightweight Streaming 3D Gaussian Splatting for Real-time Rendering, https://doi.org/10.1109/iccad66269.2025.11240950, fetched 2026-10-30
