# Coarse-to-fine feature grid
Also called: Progressive Upscaling of Feature Grids, Coarse-to-fine grid optimization

## Overview
A training optimization scheme that reduces initial training cost and regularizes feature representations by initializing a 3D feature grid at a coarse resolution and progressively upscaling it to higher resolutions [S1]. It improves model convergence and makes camera refinement more effective compared to training at full resolution from the start [S2, unofficial]. Unlike static grid methods, this approach dynamically adapts the spatial detail of the feature representation during the optimization process [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** Initial coarse feature grid parameters (matrices and vectors), current iteration count, target final resolution.
- **Output:** Updated feature grid parameters at the current resolution stage.

**Data Structures**
The method operates on the 3D feature grid components defined in the parent [TensoRF Decomposition Models](13296-tensorf-decomposition-models.md). These typically consist of:
- **Feature Matrices:** 2D tensors representing features along specific axes (e.g., $X, Y, Z$).
- **Feature Vectors:** 1D tensors representing scalar or vector features at grid nodes.

**Numbered Steps**
1. **Initialization:** Initialize the 3D feature grid with a coarse resolution. [S1]
2. **Start Optimization:** Begin model optimization with an initial grid resolution of $d = 32$. [S2, unofficial]
3. **Iterative Upscaling:**
   - Check if the current iteration count is a multiple of 5,000. [S2, unofficial]
   - If yes, upsample the grid to a resolution that is two times larger in each dimension. [S2, unofficial]
   - Apply **bi-linear upscaling** to the feature matrices. [S1]
   - Apply **linear upscaling** to the feature vectors. [S1]
   - [G] Initialize the new grid entries with interpolated values from the coarse grid.
4. **Repeat:** Continue the optimization and upscaling process every 5,000 iterations. [S2, unofficial]
5. **Termination:** Stop upscaling when the final dimension reaches $d = 256$ or the desired fine resolution determined by scene complexity. [S1][S2, unofficial]

**Formulas**
- **Upsampling Interval:** Upsampling occurs at iterations $k$ where $k \mod 5000 = 0$. [S2, unofficial]
- **Resolution Progression:** $d_{new} = 2 \times d_{old}$. [S2, unofficial]
- **Matrix Upscaling:** Bi-linear interpolation is used for 2D feature matrices. [S1]
- **Vector Upscaling:** Linear interpolation is used for 1D feature vectors. [S1]

**Parameters**
- **Initial Resolution ($d_{init}$):** 32 [S2, unofficial].
- **Final Resolution ($d_{final}$):** 256 [S2, unofficial] or determined by scene complexity (input DVR or IR images) [S1].
- **Upsampling Frequency:** Every 5,000 iterations [S2, unofficial].
- **Upsampling Factor:** 2x per step [S2, unofficial].

**Edge Cases and Pitfalls**
- **Scene Complexity:** If the scene is simple, a lower final resolution may suffice, reducing memory usage [S1].
- **Camera Refinement:** Coarse-to-fine optimization is critical for effective camera refinement; skipping it may lead to poor pose estimation [S2, unofficial].
- **Precision:** Bi-linear and linear upscaling preserve continuity but may introduce smoothing artifacts if the scene has high-frequency details not captured by the coarse grid [S1].

## Subtopics

### [NeuManifold](9771-neumanifold.md)
Uses manifold-based representations for unbounded environments, often combined with coarse-to-fine strategies for better convergence.

### [Separable Component-Wise Convolution for Joint Pose Optimization](10576-separable-component-wise-convolution-for-joint-pose-optimiza.md)
Focuses on pose optimization, where coarse-to-fine grids help stabilize camera refinement.

### [StyleTRF](11815-styletrf.md)
Applies style-based transformations, potentially benefiting from progressive grid upscaling for multi-scale detail.

### [TensoRF Decomposition Models](13296-tensorf-decomposition-models.md)
The foundational representation that uses the feature grids optimized by this method.

## Operating characteristics
- **Time Complexity:** Reduces initial training cost by starting with a smaller grid [S1]. Total training time may increase due to upscaling steps, but convergence is faster [S2, unofficial].
- **Memory Complexity:** Scales with the cube of the resolution ($d^3$). Starting at $d=32$ significantly reduces peak memory usage compared to starting at $d=256$ [S2, unofficial].
- **Measured Performance:** Effective for static and dynamic scenes, improving novel view synthesis quality [S1].
- **Strengths:** Regularizes the optimization process, prevents overfitting in early stages, and improves camera refinement [S1][S2, unofficial].
- **Limitations:** Requires careful tuning of upscaling frequency and final resolution [S1][S2, unofficial].
- **When to Use:** When training cost is a concern, or when camera refinement is critical [S1][S2, unofficial].
- **When Not to Use:** For very simple scenes where full resolution can be trained quickly, or when real-time inference is prioritized over training time.

## Background
Originated in the context of TensoRF and HexPlane representations to address the high computational cost of training large 3D feature grids [S1]. It was adopted in ViSNeRF for efficient multidimensional neural radiance field representation [S1] and in Neural Assets for volumetric object capture [S2, unofficial]. The method builds on the observation that coarse-to-fine optimization stabilizes gradient descent and improves generalization [S1][S2, unofficial].

## Sources
[S1] ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes, https://doi.org/10.1109/pacificvis64226.2025.00029, fetched 2026-10-26
[S2] Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments, https://core.ac.uk/download/552466290.pdf, fetched 2026-10-30 (unofficial)
