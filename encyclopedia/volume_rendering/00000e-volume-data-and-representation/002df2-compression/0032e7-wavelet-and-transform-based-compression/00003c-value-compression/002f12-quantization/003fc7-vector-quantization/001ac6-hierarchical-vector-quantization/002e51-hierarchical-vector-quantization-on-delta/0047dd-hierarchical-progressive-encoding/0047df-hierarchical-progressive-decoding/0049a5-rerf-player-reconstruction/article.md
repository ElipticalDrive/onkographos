# ReRF Player Reconstruction

Also called: Network Streaming ReRF Player Reconstruction, ReRF Player Decoding

## Overview
ReRF Player Reconstruction is the decoding phase of the Neural Residual Radiance Fields (ReRF) codec, designed to recover feature grids from a compressed bitstream for the real-time rendering of dynamic free-viewpoint videos (FVV) [S1]. It differs from static volume rendering by utilizing temporal redundancy through motion-compensated prediction and residual encoding, and supports downloading coded bitstreams from streaming media servers [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A coded bitstream containing syntax elements for I-frames (keyframes) and P-frames (predictive frames); previously reconstructed feature grids $\tilde{f}_{t-1}$; decoded motion fields $M_t$ [S1].
*   **Outputs:** Reconstructed feature grids $\tilde{f}_t$ for each frame $t$, which are passed to a renderer (global MLP $\Phi$) to generate pixel colors via volume rendering [S1].

**Data Structures:**
*   **Feature Grid ($f$):** A 3D explicit grid representing the union of density and color features [S1]. It has $C=13$ channels [S1]. The spatial dimensions are typically $N_x \times N_y \times N_z$ (e.g., $250 \times 250 \times 250$) [S1].
*   **Motion Field ($M_t$):** A grid representing position offsets from the current frame $t$ to the previous frame $t-1$ [S1]. It has a shape of $3 \times N_x \times N_y \times N_z$ [S1].
*   **Residual Grid ($r$):** The difference between the actual and predicted features, stored in transform domain during compression [S1].
*   **PCA Matrix ($V$):** The basis matrix used for back-projection of residuals [S1].

**Algorithm Steps:**

1.  **Download Bitstream:** Receive the coded bitstream from the streaming media server [S1].
2.  **Reconstruct I-Frame (First Frame of Group of Frames - GOF):**
    *   Decode the syntax elements for the I-feature grid $f_{e1}$ [S1].
    *   Perform inverse quantization and inverse transform on the quantized transform coefficients to reconstruct the initial feature grid [S1].
    *   Set $\tilde{f}_0 = f_{e1}$ (or $\tilde{f}_1$ depending on indexing convention; the source denotes the first reconstructed grid as $f_{e1}$) [S1].
3.  **Reconstruct P-Frame (Subsequent Frames $t > 1$):**
    *   **Step 3.1: Decode Residual:** Generate the initial reconstructed residual grid $\hat{r}'_t$ by performing inverse quantization and inverse transform on the quantized transform coefficients of the P-frame [S1].
    *   **Step 3.2: Back-Project Residual:** Transform $\hat{r}'_t$ back to the origin space using the transpose of the PCA matrix $V$ to obtain $\hat{r}_t$:
        $$ \hat{\mathbf{r}}_t = \hat{\mathbf{r}}'_t \mathbf{V}^T $$
        where $\hat{\mathbf{r}}_t$ is the residual grid in the original feature space [S1].
    *   **Step 3.3: Predict Feature Grid:** Deform the previously reconstructed feature grid $\tilde{f}_{t-1}$ using the decoded motion field $M_t$ [S1]. For each grid index $p$, the predicted feature grid $\bar{f}_t(p)$ is:
        $$ \bar{f}_t(\mathbf{p}) = \tilde{f}_{t-1}(\mathbf{p} + \mathbf{M}_t(\mathbf{p})) $$
        where $\mathbf{p}$ denotes the index of the explicit grids [S1].
    *   **Step 3.4: Combine:** Add the predicted feature grid and the reconstructed residual grid to produce the final reconstructed feature grid $\tilde{f}_t$:
        $$ \tilde{\mathbf{f}}_t = \bar{\mathbf{f}}_t + \hat{\mathbf{r}}_t $$
        [S1].
4.  **Render:** Output $\tilde{f}_t$ to the renderer [S1]. The renderer queries $\tilde{f}_t$ at point $p_t$ along the pixel ray using trilinear interpolation:
    $$ \text{Tri-Interp}(p_t, \tilde{f}_t) \rightarrow \mathbb{R}^C $$
    The global MLP $\Phi$ decodes this feature vector into radiance fields to calculate pixel color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_frame(bitstream, prev_feature_grid, motion_field, pca_matrix_V):
    """
    Reconstructs the feature grid for the current frame t.
    
    Args:
        bitstream: Contains syntax elements for residual coefficients.
        prev_feature_grid: The reconstructed feature grid from frame t-1 (tilde_f_{t-1}). Shape: (C, Nx, Ny, Nz).
        motion_field: The decoded motion field M_t. Shape: (3, Nx, Ny, Nz).
        pca_matrix_V: The PCA basis matrix V. Shape: (C, C) or (C, K).
        
    Returns:
        current_feature_grid: The reconstructed feature grid for frame t (tilde_f_t).
    """
    # Step 1: Decode residual coefficients from bitstream
    # [G] Standard entropy decoding, inverse quantization, and inverse 3D DCT steps.
    residual_transform_coeffs = entropy_decode(bitstream)
    residual_transform_coeffs = inverse_quantize(residual_transform_coeffs)
    residual_transform_coeffs = inverse_3d_dct(residual_transform_coeffs)
    
    # Step 2: Back-project to origin space using PCA matrix transpose
    # r_hat_t = r_hat'_t * V^T
    # Flatten spatial dimensions for matrix multiplication: (Nx*Ny*Nz, C) @ (C, K).T -> (Nx*Ny*Nz, K)
    residual_flat = residual_transform_coeffs.reshape(-1, residual_transform_coeffs.shape[0])
    residual_origin_space_flat = residual_flat @ pca_matrix_V.T
    residual_origin_space = residual_origin_space_flat.reshape(residual_transform_coeffs.shape)
    
    # Step 3: Predict feature grid by deforming previous frame
    # f_bar_t(p) = f_tilde_{t-1}(p + M_t(p))
    predicted_feature_grid = deform_grid(prev_feature_grid, motion_field)
    
    # Step 4: Add predicted grid and residual
    # f_tilde_t = f_bar_t + r_hat_t
    current_feature_grid = predicted_feature_grid + residual_origin_space
    
    return current_feature_grid

def deform_grid(feature_grid, motion_field):
    """
    Deforms the feature grid using the motion field via trilinear interpolation.
    
    Args:
        feature_grid: The previous feature grid f_tilde_{t-1} (C x Nx x Ny x Nz).
        motion_field: The motion field M_t (3 x Nx x Ny x Nz).
        
    Returns:
        deformed_grid: The predicted feature grid f_bar_t.
    """
    C, Nx, Ny, Nz = feature_grid.shape
    deformed_grid = [[[[0.0 for _ in range(Nz)] for _ in range(Ny)] for _ in range(Nx)] for _ in range(C)]
    
    for x in range(Nx):
        for y in range(Ny):
            for z in range(Nz):
                # Get displacement vector M_t(p)
                displacement = [motion_field[0, x, y, z], motion_field[1, x, y, z], motion_field[2, x, y, z]]
                
                # Calculate source coordinate p + M_t(p)
                source_x = x + displacement[0]
                source_y = y + displacement[1]
                source_z = z + displacement[2]
                
                # Sample feature_grid at (source_x, source_y, source_z) using trilinear interpolation
                sampled_features = trilinear_interpolate(feature_grid, source_x, source_y, source_z)
                
                # Store in deformed grid
                for c in range(C):
                    deformed_grid[c][x][y][z] = sampled_features[c]
                    
    return deformed_grid

def trilinear_interpolate(grid, x, y, z):
    """
    Performs trilinear interpolation on a 3D grid.
    Handles boundary conditions by clamping coordinates to [0, N-1].
    """
    C, Nx, Ny, Nz = len(grid), len(grid[0]), len(grid[0][0]), len(grid[0][0][0])
    
    # Clamp coordinates to valid range
    x = max(0.0, min(float(x), float(Nx - 1)))
    y = max(0.0, min(float(y), float(Ny - 1)))
    z = max(0.0, min(float(z), float(Nz - 1)))
    
    # Extract integer and fractional parts
    x0, x1 = int(x), min(int(x) + 1, Nx - 1)
    y0, y1 = int(y), min(int(y) + 1, Ny - 1)
    z0, z1 = int(z), min(int(z) + 1, Nz - 1)
    
    dx, dy, dz = x - x0, y - y0, z - z0
    
    # Interpolate along each axis
    # [G] Standard trilinear interpolation formula
    val = (1 - dx) * (1 - dy) * (1 - dz) * grid[0][x0][y0][z0] + \
          dx * (1 - dy) * (1 - dz) * grid[0][x1][y0][z0] + \
          (1 - dx) * dy * (1 - dz) * grid[0][x0][y1][z0] + \
          dx * dy * (1 - dz) * grid[0][x1][y1][z0] + \
          (1 - dx) * (1 - dy) * dz * grid[0][x0][y0][z1] + \
          dx * (1 - dy) * dz * grid[0][x1][y0][z1] + \
          (1 - dx) * dy * dz * grid[0][x0][y1][z1] + \
          dx * dy * dz * grid[0][x1][y1][z1]
          
    return [val]
```

**Parameters:**
*   **Grid Resolution:** Progressive training uses resolutions $(125 \times 125 \times 125) \rightarrow (150 \times 150 \times 150) \rightarrow (200 \times 200 \times 200) \rightarrow (250 \times 250 \times 250)$ [S1].
*   **Channels:** $C=13$ (union of density and color features) [S1].
*   **Motion Field Resolution:** Compact low-resolution motion grid derived from densely estimated motion field $D_t$ [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Trilinear interpolation during deformation may query coordinates outside the grid bounds; coordinates must be clamped to valid ranges [S1].
*   **Temporal Drift:** Errors in $\tilde{f}_{t-1}$ propagate to $\tilde{f}_t$ through deformation; residual encoding mitigates accumulated error [G].

## Subtopics

### ReRF Codec Pipeline
The encoder compresses the input signal using PCA, 3D DCT, quantization, and entropy coding to produce the bitstream [S1]. The decoder reverses this process [S1]. The feature grid representation unions density and color features ($C=13$) [S1]. The motion field $D_t$ has shape $3 \times N_x \times N_y \times N_z$ and contains warping information from frame $t$ to $t-1$ [S1].

## Operating characteristics
*   **Time Complexity:** Reconstruction involves inverse transforms, matrix multiplication (PCA back-projection), and grid deformation (trilinear interpolation over all voxels) [S1].
*   **Memory Complexity:** Requires storing the current and previous feature grids ($C \times N_x \times N_y \times N_z$) and the motion field ($3 \times N_x \times N_y \times N_z$) [S1].
*   **Performance:** The coded bitstream consists of successive GOFs; the first frame in a GOF is an I-feature grid (keyframe) which contains an independently coded feature, allowing fast seeking to a new position during playback [S1].
*   **Strengths:** Efficient compression of dynamic scenes via motion compensation and residual encoding; real-time rendering capability [S1].

## Background
ReRF Player Reconstruction is part of the Neural Residual Radiance Fields (ReRF) method proposed by [S1]. It enables streamable free-viewpoint videos by combining neural radiance fields with video coding techniques (I/P frames, motion compensation) [S1].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
