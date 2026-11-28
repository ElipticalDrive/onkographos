# Plenoxels Transmittance Offset

Also called: High transmittance offset

## Overview
Plenoxels Transmittance Offset is an optimization technique applied to Plenoxels that adds a high transmittance offset to the density parameters to ensure high rendering quality across various scene scaling factors $k$ [S1]. It sets itself apart from the baseline Plenoxels configuration by addressing the convergence failure of standard density rectification methods when scene scales vary, a problem where baseline models suffer significant quality degradation at large $k$ [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Plenoxels voxel grid containing spherical harmonic (SH) coefficients for radiance and density; scene scaling factor $k$; baseline learning rates $\eta_{init} = 0.05$ and $\eta_{final} = 0.005$ [S1].
*   **Output:** Optimized voxel grid parameters yielding high PSNR across scales, specifically correcting the quality drop seen in baseline models at $k \ge 1$ [S1].

**Data Structures:**
The method operates on the existing Plenoxels voxel grid. Each voxel stores SH coefficients. The density $\sigma$ is queried via trilinear interpolation of these coefficients [S1].

**Method Steps:**
1.  **Initialize:** Set up the Plenoxels voxel grid with SH coefficients. Configure the optimizer with initial learning rate $\eta_{init} = 0.05$ and final learning rate $\eta_{final} = 0.005$ [S1].
2.  **Density Query:** For a given ray, sample the voxel grid using trilinear interpolation to obtain the raw density value $\sigma_{raw}$ [S1].
3.  **Activation and Offset:** Apply the exponential activation function to the raw density and add a high transmittance offset $\tau$ to ensure high transmittance (low opacity) initially or throughout optimization:
    $$ \sigma = \exp(\sigma_{raw}) + \tau $$
    The sources define this as a "high transmittance offset" but do not provide the exact numerical value of $\tau$ or the specific formula for its calculation, noting only that it is "high" and "needed even at $k=1$" [S1]. [G] In practice, this offset is typically a small positive constant added to the density to prevent the exponential from collapsing to zero or to maintain a baseline transparency level.
4.  **Rendering:** Compute transmittance $T$ and color $C$ using standard volume rendering integration with the modified $\sigma$ [S1].
5.  **Optimization:** Update voxel coefficients using gradient-based optimization with the specified learning rate schedule [S1].

**Parameters:**
*   $\eta_{init} = 0.05$ [S1]
*   $\eta_{final} = 0.005$ [S1]
*   Transmittance Offset $\tau$: Value not given in the sources. Described as "high" [S1].

**Edge Cases and Pitfalls:**
*   **Insufficiency of Exp Activation:** Using only the exponential activation ($\sigma = \exp(\sigma_{raw})$) is insufficient for convergence at various scene scales [S1].
*   **Baseline Failure:** The baseline Plenoxels model (using ReLU or exp without offset) suffers significant quality drops at large $k$ (e.g., PSNR 13.05 vs 32.25 at $k=10$ on Blender Chair) [S1].
*   **Learning Rate:** The reduced learning rate ($\eta_{init} = 0.05, \eta_{final} = 0.005$) is used in conjunction with the offset; the baseline with ReLU uses a larger rate [S1].

## Subtopics

### Plenoxels Exp Activation Adjustment
This variant highlights that rectifying $\sigma$ with $exp$ activation alone is insufficient for convergence at various scene scales; a high transmittance offset is needed even at $k=1$ [S1]. It serves as a prerequisite step, indicating that the offset is the critical differentiator from a simple exp-activated baseline [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
On the Blender dataset, the method significantly outperforms the baseline Plenoxels at large scene scaling factors $k$ [S1]:
*   **Chair:** At $k=1$, baseline PSNR 31.80 vs. method 31.89. At $k=10$, baseline 13.05 vs. method 32.25 [S1].
*   **Ship:** At $k=1$, baseline 28.16 vs. method 28.25. At $k=10$, baseline 11.27 vs. method 28.21 [S1].
*   **Other Objects:** Similar trends observed for other Blender objects (e.g., Ours vs Default at $k=10$: 34.93 vs 13.03 for one object, 31.69 vs 11.64 for another) [S1].

**Strengths and Limitations:**
*   **Strengths:** Maintains high rendering quality across various scene scaling factors $k$, preventing the severe quality degradation seen in baseline models at large $k$ [S1].
*   **Limitations:** The specific value of the offset is not detailed, making reproducibility dependent on heuristic tuning [S1].

**When to Use:**
Use when optimizing Plenoxels for scenes with varying or large scaling factors $k$ where baseline models fail to converge or produce low-quality renders [S1].

## Background
The method originates from the work on "Alpha Invariance: On Inverse Scaling Between Distance and Volume Density in Neural Radiance Fields" [S1]. It addresses the issue where forcing rays to be bounded within predetermined global near and far planes produces significantly worsened rendering quality, as shown in related tables [S1]. The technique is part of a broader effort to stabilize Plenoxels optimization under scene scaling variations [S1].

## Sources
[S1] Alpha Invariance: On Inverse Scaling Between Distance and Volume Density in Neural Radiance Fields, https://www.youtube.com/watch?v=YQhBEXC464w, fetched 2026-10-31
