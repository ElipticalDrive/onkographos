# GPS-Gaussian

Also called: Gaussian Parameter Maps

## Overview
GPS-Gaussian is a training initialization and acceleration method for 3D Gaussian Splatting that eliminates the need for iterative optimization by regressing Gaussian parameters directly from sparse source views. It solves the problem of slow convergence and high computational cost in standard 3DGS training by introducing Gaussian parameter maps on sparse input images, jointly regressing the 3D Gaussian attributes alongside a depth estimation module [S1]. It is distinguished from standard [Base 3DGS Optimization](7549-base-3dgs-optimization.md) by its ability to produce a high-quality initial state without any fine-tuning or optimization steps, and from [FaCT-GS](10199-fact-gs.md) by its reliance on image-based regression rather than voxelization-based initialization [S1].

## How it works
GPS-Gaussian constructs an initial set of 3D Gaussians directly from 2D image data, bypassing the traditional point-cloud densification and optimization loop.

**Inputs and Outputs:**
- **Inputs:** Sparse set of source views (images and camera poses).
- **Outputs:** A set of 3D Gaussians (position, covariance, opacity, spherical harmonics coefficients) ready for rendering, with no subsequent optimization required.

**Data Structures:**
The method relies on two primary neural network outputs mapped to the image plane:
1.  **Depth Map:** A per-pixel depth estimate $Z(x,y)$.
2.  **Gaussian Parameter Maps:** A set of per-pixel tensors containing the parameters for the Gaussian corresponding to that pixel. For a standard 3DGS Gaussian, these typically include:
    -   Scale: 3 values (or quaternion + 3 log-scales).
    -   Rotation: 4 values (quaternion).
    -   Opacity: 1 value.
    -   Color: $C \times 3$ values (where $C$ is the number of SH bands).

**Steps:**
1.  **Depth Estimation:** A depth estimation network processes the sparse source views to generate a dense depth map for each view.
2.  **Gaussian Parameter Regression:** A separate regression network (or the same network with multiple heads) processes the source views to generate Gaussian parameter maps. This network predicts the geometric and appearance attributes for a Gaussian centered at the projected pixel location.
3.  **3D Reconstruction:** For each pixel $(u,v)$ in each source view:
    -   Retrieve the predicted depth $Z_{pred}$ and camera intrinsics/extrinsics.
    -   Back-project the pixel to 3D space to obtain the mean position $\mu$.
    -   Retrieve the predicted scale, rotation, opacity, and color from the parameter maps.
    -   Construct the 3D Gaussian using these parameters.
4.  **Rendering:** The resulting set of Gaussians is rendered using the standard 3DGS rasterization pipeline. No gradient-based optimization is performed on these parameters.

**Formulas:**
The core mechanism is the direct mapping from 2D image coordinates to 3D Gaussian parameters. Let $f_{depth}$ and $f_{params}$ be the neural networks for depth and parameter regression, respectively.
For a pixel $p$ in view $i$:
$$ \mu_i = K^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} Z_{pred}(p) $$
where $K$ is the camera intrinsic matrix.
The Gaussian attributes $\theta_i$ are obtained directly:
$$ \theta_i = f_{params}(Image_i, p) $$
where $\theta_i$ includes scale, rotation, opacity, and SH coefficients [S1].

**Parameters:**
-   **Sparse View Count:** The number of source views used for initialization. The method is designed to work with sparse inputs, though the exact minimum is "Not given in the sources."
-   **Network Architecture:** The specific architecture of the depth and parameter regression networks is "Not given in the sources."

**Edge Cases and Pitfalls:**
-   **Occlusion:** Since the method relies on sparse views, occluded regions in the scene may lack sufficient depth or parameter information. The method assumes the sparse views provide adequate coverage for the initial reconstruction [S1].
-   **Precision:** The quality of the initial reconstruction is heavily dependent on the accuracy of the depth estimation and parameter regression networks. Errors in these predictions propagate directly to the final render without the corrective power of optimization [S1].

## Subtopics

### Conditional Slicing
Conditional Slicing is a mechanism used in related high-dimensional Gaussian methods (e.g., 7DGS) that projects high-dimensional Gaussians into 3D counterparts conditioned on specific time and view direction, preserving computational efficiency [S2]. It is distinguished from GPS-Gaussian by its focus on dynamic, view-dependent modeling rather than static initialization from sparse views [S2].

**Definition:**
A mechanism that projects high-dimensional (7D) Gaussians into 3D counterparts conditioned on specific time and view direction, preserving computational efficiency of 3DGS [S2].

**Step:**
Condition the 6D Gaussian on the viewing direction $d$ (where $X_d = d$) to yield a conditional 3D Gaussian for the spatial component, using the conditional distribution formulas from 6DGS [S2].

**Formula:**
The conditional mean is calculated as:
$$ \mu_{cond} = \mu_p + \Sigma_{pd} \Sigma_d^{-1} (d - \mu_d) $$
where $\mu_p$ is the spatial mean, $\Sigma_{pd}$ is the cross-covariance between position and direction, $\Sigma_d$ is the directional covariance, $d$ is the viewing direction, and $\mu_d$ is the directional mean [S2].

The conditional covariance is calculated as:
$$ \Sigma_{cond} = \Sigma_p - \Sigma_{pd} \Sigma_d^{-1} \Sigma_{pd}^\top $$
where $\Sigma_p$ is the spatial covariance [S2].

The opacity of each Gaussian is modulated to reflect the alignment between the current view direction and the Gaussian’s preferred direction:
$$ f_{cond} = \exp \left( -\lambda (d - \mu_d)^\top \Sigma_d^{-1} (d - \mu_d) \right) $$
$$ \alpha_{cond} = \alpha \cdot f_{cond} $$
where $\lambda$ is a positive scaling parameter controlling the sensitivity of the opacity modulation to view direction alignment [S2].

**Parameter:**
-   $\lambda$: A positive scaling parameter controlling the sensitivity of the opacity modulation to view direction alignment [S2].

## Operating characteristics
-   **Time Complexity:** Training time is significantly reduced compared to standard 3DGS because the optimization loop is bypassed. The time is dominated by the forward pass of the depth and parameter regression networks [S1].
-   **Memory Complexity:** Memory usage is comparable to standard 3DGS for the Gaussian storage, but requires additional memory for the neural network weights used for regression [S1].
-   **Performance:** GPS-Gaussian enables high-fidelity rendering with high resolution on consumer-level devices by avoiding the iterative optimization process [S1].
-   **Strengths:** Fast initialization, no fine-tuning required, compact representation [S1].
-   **Limitations:** Quality is limited by the accuracy of the regression networks; cannot correct for errors in the initial prediction [S1].
-   **When to use:** When rapid initialization from sparse views is required and iterative optimization is computationally prohibitive [S1].
-   **When not to use:** When high accuracy is required and the initial regression predictions are likely to be insufficient [S1].

## Background
GPS-Gaussian was introduced to address the computational cost and slow convergence of 3D Gaussian Splatting training. It builds on the dual-graph mechanism for high-fidelity rendering [S1]. The method was proposed to achieve higher rendering speeds with high resolution on consumer-level devices by regressing Gaussian parameters jointly with a depth estimation module without any fine-tuning or optimization [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
