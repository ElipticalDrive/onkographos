# Plenoxels Interpolation Methods

Also called: Plenoxel Grid with Trilinear Interpolation, Trilinear Plenoxel

## Overview
Plenoxels interpolation methods define a continuous radiance field over a sparse 3D voxel grid by storing spherical harmonic (SH) coefficients and opacity at voxel centers, enabling stable optimization and higher effective resolution than discrete voxel assumptions [S1]. It distinguishes itself from nearest-neighbor Plenoxels by providing a continuous function approximation that is critical for successful gradient-based optimization, and from tricubic methods by offering a favorable trade-off between smoothness and computational cost [S1].

## How it works
**Inputs and Outputs:**
- Input: A sparse 3D voxel grid containing optimized parameters; a 3D sample point $\mathbf{p}$; a viewing direction $\mathbf{v}$.
- Output: Radiance color $\mathbf{c}(\mathbf{p}, \mathbf{v})$ and opacity $\sigma(\mathbf{p})$.

**Data Structures:**
The grid stores data at voxel centers. For each active voxel, the layout is:
- Opacity: 1 scalar value ($\sigma$).
- Spherical Harmonics Coefficients: 27 values (9 coefficients per color channel for degree 2 SH).
- Note: The sources specify storing RGB channels (zero-order SH) for background spheres to conserve memory [S1].

**Parameters:**
- Spherical Harmonics Degree: 2 [S1]. This requires 9 coefficients per color channel, totaling 27 harmonic coefficients per voxel plus 1 opacity coefficient [S1].
- Resolution: Typical synthetic scenes use $128^3$ to $512^3$ [S1].

**Steps:**
1. **Locate Neighbors:** Identify the 8 nearest voxel centers surrounding the sample point $\mathbf{p}$. Let these centers be $\mathbf{c}_i$ for $i \in \{0, \dots, 7\}$.
2. **Compute Weights:** Calculate trilinear interpolation weights $w_i$ based on the relative position of $\mathbf{p}$ within the unit cube defined by the 8 neighbors.
3. **Interpolate Opacity:** Compute the opacity $\sigma(\mathbf{p})$ by trilinearly interpolating the opacity values $\sigma_i$ from the 8 neighbors:
   $$ \sigma(\mathbf{p}) = \sum_{i=0}^{7} w_i \sigma_i $$
4. **Interpolate SH Coefficients:** For each color channel $k \in \{R, G, B\}$, trilinearly interpolate the 9 SH coefficients $C_{i,j,k}$ (where $j$ indexes the 9 coefficients) from the 8 neighbors to obtain the interpolated coefficients $\hat{C}_{j,k}$:
   $$ \hat{C}_{j,k} = \sum_{i=0}^{7} w_i C_{i,j,k} $$
5. **Evaluate Radiance:** Evaluate the spherical harmonic basis functions $Y_j(\mathbf{v})$ for the viewing direction $\mathbf{v}$. Compute the radiance for channel $k$ as:
   $$ c_k(\mathbf{p}, \mathbf{v}) = \sum_{j=0}^{8} \hat{C}_{j,k} Y_j(\mathbf{v}) $$
   The final color is $\mathbf{c} = [c_R, c_G, c_B]$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_plenoxel_sample(grid, p, v):
    # 1. Find 8 nearest voxel centers and weights
    # grid is a sparse structure mapping voxel_index -> (opacity, sh_coeffs[27])
    # p is a 3D vector
    neighbors = get_8_neighbors(grid, p)
    weights = compute_trilinear_weights(p, neighbors.corners)
    
    # 2. Interpolate opacity
    sigma = 0.0
    for i, neighbor in enumerate(neighbors):
        sigma += weights[i] * neighbor.opacity
        
    # 3. Interpolate SH coefficients (Degree 2 -> 9 coeffs per channel)
    # sh_coeffs layout: [R0..R8, G0..G8, B0..B8]
    interp_sh = [0.0] * 27
    for i, neighbor in enumerate(neighbors):
        for k in range(27):
            interp_sh[k] += weights[i] * neighbor.sh_coeffs[k]
            
    # 4. Evaluate SH for viewing direction v
    # Y is a function returning 9 basis values for direction v
    y_vals = evaluate_sh_basis(v, degree=2)
    
    color = [0.0, 0.0, 0.0]
    for channel in range(3):
        channel_offset = channel * 9
        radiance = 0.0
        for j in range(9):
            radiance += interp_sh[channel_offset + j] * y_vals[j]
        color[channel] = radiance
        
    return color, sigma
```

**Edge Cases and Pitfalls:**
- **Nearest Neighbor Instability:** Using nearest-neighbor interpolation produces a discontinuous model that is difficult to optimize and less stable with respect to learning rate variations [S1].
- **Tricubic Cost:** Tricubic interpolation offers continuous and smooth approximation but incurs substantial computation with negligible quality gain compared to trilinear [S1].
- **Initialization:** Coefficients are optimized directly without special initialization or pretraining with a neural network [S1].

## Subtopics

### Unbounded Scene Handling (NDC and Background Model)
This variant extends Plenoxels to real, unbounded scenes, including forward-facing and 360° settings [S1]. For forward-facing scenes, it uses normalized device coordinates (NDC) for the sparse voxel grid structure [S1]. For 360° scenes, it augments the foreground with a multisphere image (MSI) background model using learned voxel colors and opacities with trilinear interpolation [S1]. It places 64 spheres linearly in inverse radius from 1 to $\infty$, pre-scaling the inner scene to be approximately contained in the unit sphere [S1]. To conserve memory, the background stores only RGB channels (zero-order SH) and uses opacity thresholding for sparse storage [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Memory: Scales with the number of active voxels. Each voxel stores 28 scalars (1 opacity + 27 SH coefficients) [S1]. Background spheres store only 3 scalars (RGB) per voxel [S1].
- Time: Trilinear interpolation is significantly faster than tricubic, which incurs substantial computation [S1].

**Measured Performance:**
- On 8 NeRF synthetic scenes, Trilinear Plenoxels at $128^3$ resolution (PSNR 28.46, SSIM 0.926, LPIPS 0.100) outperform Nearest Neighbor at $256^3$ resolution (PSNR 27.17, SSIM 0.914, LPIPS 0.119) [S1].
- Trilinear interpolation at $256^3$ achieves PSNR 30.57, SSIM 0.950, LPIPS 0.065 [S1].

**Strengths and Limitations:**
- **Strengths:** Increases effective resolution by representing sub-voxel variations; produces a continuous function approximation critical for successful optimization; more stable with respect to learning rate variations than nearest neighbor [S1].
- **Limitations:** Tricubic offers smoother approximation but with negligible quality gain and higher cost [S1].

**When to Use:**
- Use trilinear interpolation for stable optimization and higher quality than nearest neighbor at lower resolutions [S1].
- Avoid nearest neighbor if optimization stability is a concern [S1].
- Avoid tricubic unless smoothness is strictly required and computational budget allows, as quality gain is negligible [S1].

## Background
Plenoxels were introduced in the paper "Plenoxels: Radiance Fields without Neural Networks" [S1]. The method builds on PlenOctrees but replaces the constant-voxel assumption with trilinear interpolation to enable direct optimization of opacity and SH coefficients without neural networks [S1]. The use of degree 2 spherical harmonics was adopted from PlenOctrees, as higher orders confer minimal benefit [S1]. The method supports coarse-to-fine optimization strategies, resizing grids between arbitrary resolutions using trilinear interpolation [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
