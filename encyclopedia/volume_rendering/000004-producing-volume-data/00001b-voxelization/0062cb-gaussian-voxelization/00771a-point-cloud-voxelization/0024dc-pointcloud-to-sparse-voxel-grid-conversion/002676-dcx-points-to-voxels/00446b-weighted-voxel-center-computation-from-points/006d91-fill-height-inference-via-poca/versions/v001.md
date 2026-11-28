# Fill-Height Inference via PoCA

Also called: TomOpt fill-level estimation, Weighted PoCA Height Prediction, Weighted PoCA z-position inference, zpred calculation

## Overview
Fill-Height Inference via PoCA is a regression method that predicts the fill-level of a furnace by computing a weighted average of the z-positions of Points of Closest Approach (PoCAs) from muon trajectories [S1]. It differs from voxel-based density reconstruction methods by directly estimating the fill-height indicator without computing X0 (radiation length) predictions for voxels, making it computationally more efficient [S1]. The method sets itself apart by incorporating transverse (xy) weighting to mitigate bias from PoCAs located in furnace walls, leveraging the monotonic relationship between the mean PoCA z-position and the fill-height [S1].

## How it works
**Inputs:**
*   A set of muon trajectories, each associated with a PoCA point $p$.
*   For each PoCA $p$:
    *   $z_p$: The z-position of the PoCA.
    *   $\epsilon_p$: The muon trajectory efficiency.
    *   $\Delta z_p$: The uncertainty of the PoCA z-position.
    *   $\sigma_{x,p}, \sigma_{y,p}$: The xy uncertainties of the PoCA position.
    *   $(x_p, y_p)$: The xy position of the PoCA.
*   Voxel grid definition for the transverse area of the furnace (x, y dimensions).
*   Furnace geometry parameters defining the center and walls for the xy weighting function.

**Outputs:**
*   $z_{pred}$: The predicted z-position, which serves as the fill-height indicator.

**Data Structures:**
*   List of PoCA objects, each containing fields: `z`, `epsilon`, `dz` (uncertainty), `sigma_x`, `sigma_y`, `x`, `y`.
*   Voxel grid structure covering the furnace transverse area.

**Steps:**

1.  **Compute Base Weight:** For each PoCA $p$, calculate the base weight component based on trajectory efficiency and z-position uncertainty:
    $$ w_{base,p} = \frac{\epsilon_p}{(\Delta z_p)^2} $$

2.  **Define Transverse Distributions:** For each PoCA $p$, define a 2D Gaussian distribution in the xy-plane centered at $(x_p, y_p)$ with standard deviations $\sigma_{x,p}$ and $\sigma_{y,p}$:
    $$ G_p(x, y) = \frac{1}{2\pi \sigma_{x,p} \sigma_{y,p}} \exp\left( -\frac{(x-x_p)^2}{2\sigma_{x,p}^2} - \frac{(y-y_p)^2}{2\sigma_{y,p}^2} \right) $$

3.  **Integrate over Voxels:** For each PoCA $p$, integrate $G_p(x, y)$ over each voxel $v$ in the transverse area of the furnace. Let $V_v$ be the area of voxel $v$. The integral $I_{p,v}$ is:
    $$ I_{p,v} = \iint_{V_v} G_p(x, y) \, dx \, dy $$
    [G] In implementation, this integral can be computed using the error function (erf) for rectangular voxels.

4.  **Apply Transverse Weighting Function:** Define a double-sigmoid-based function $W_{xy}(x, y)$ that ascribes a weight to each voxel position. This function smoothly decreases to zero at the furnace walls and maximizes to one at the center. Let $C_{xy}$ be the center of the furnace in xy, and $R_{wall}$ be the effective radius/extent to the walls. A typical form is:
    $$ W_{xy}(x, y) = \text{sigmoid}\left( \frac{d_{center} - \text{dist}((x,y), C_{xy})}{k} \right) \times \text{sigmoid}\left( \frac{\text{dist}((x,y), C_{xy}) - d_{inner}}{k} \right) $$
    [G] The exact parameters $d_{center}, d_{inner}, k$ depend on the specific furnace geometry and are tuned to suppress contributions from wall regions. The source states this function is "double-sigmoid-based" and "smoothly decreases to zero at the furnace walls, and maximises to one at the centre" [S1].

5.  **Calculate xy Weight for Each PoCA:** For each PoCA $p$, calculate its transverse weight $w_{xy,p}$ as the sum of the weighted voxel integrals:
    $$ w_{xy,p} = \sum_{v} I_{p,v} \cdot W_{xy}(x_v, y_v) $$
    where $(x_v, y_v)$ is the center of voxel $v$.

6.  **Compute Total Weight for Each PoCA:** Combine the base weight and the transverse weight:
    $$ w_p = w_{base,p} \cdot w_{xy,p} = \frac{\epsilon_p}{(\Delta z_p)^2} w_{xy,p} $$

7.  **Calculate Predicted z-position:** Compute the weighted average of all PoCA z-positions:
    $$ z_{pred} = \frac{\sum_p w_p z_p}{\sum_p w_p} $$

**Parameters:**
*   $\epsilon_p$: Trajectory efficiency. Typical values depend on detector acceptance and muon energy.
*   $\Delta z_p$: Z-position uncertainty. Determined by detector resolution and track fit quality.
*   $\sigma_{x,p}, \sigma_{y,p}$: XY position uncertainties. Determined by detector resolution.
*   Double-sigmoid parameters: Tuned to furnace geometry. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Wall Bias:** PoCAs located in furnace walls can bias the prediction and reduce precision. The xy weighting function mitigates this by reducing weights near walls [S1].
*   **Fill-Height vs. Center of Mass:** The mean of PoCA z-positions corresponds to the center of the fill material, not the fill-height itself. A linear transformation ($h_{pred} = a \cdot z_{pred} + b$) is required to convert $z_{pred}$ to fill-height [S1].
*   **Inherent Bias:** The PoCA method is inherently biased by ascribing entire muon scattering to a single point. This bias is detector-configuration-dependent [S1].
*   **Monotonicity:** The method relies on the monotonic relationship between mean PoCA z-position and fill-height. If this relationship is not monotonic (e.g., due to complex density distributions), the prediction may be inaccurate [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of PoCAs and the number of voxels in the transverse area. Specifically, $O(N_{PoCA} \cdot N_{voxels})$ for the integration step, where $N_{PoCA}$ is the number of muon trajectories and $N_{voxels}$ is the number of transverse voxels.
*   **Memory Complexity:** Linear with respect to the number of PoCAs and voxels to store intermediate integrals and weights.
*   **Measured Performance:** The source does not provide specific performance metrics (e.g., MSE, runtime) for this method on a specific hardware or resolution. It notes that it is "more computationally efficient" than computing X0 predictions for voxels [S1].
*   **Strengths:** Computationally efficient; does not require voxel-based X0 reconstruction; mitigates wall bias via xy weighting; leverages monotonic relationship for robust regression.
*   **Limitations:** Requires linear transformation to map to fill-height; inherently biased by PoCA approximation; bias is detector-dependent; requires accurate knowledge of furnace geometry for xy weighting.
*   **When to Use:** When computational efficiency is critical and the goal is to estimate fill-height rather than full 3D density reconstruction. Suitable for detector optimization pipelines where gradients are needed [S1].
*   **When Not to Use:** When full 3D density information is required; when the monotonic relationship between mean PoCA z-position and fill-height is not valid for the specific material distribution.

## Background
*   **Origin:** Developed in the context of muon tomography for monitoring industrial furnaces, specifically for the "TomOpt" project [S1].
*   **History:** The method was developed as part of an end-to-end optimization pipeline for task- and constraint-aware design of particle detectors [S1]. It was chosen because the mean z-position of PoCAs was observed to be monotonic with fill-height, allowing for efficient regression without full voxel reconstruction [S1].
*   **Usage:** Used for predicting furnace fill-heights in industrial settings, particularly in steel or similar metallurgical furnaces, using muon scattering data [S1].

## Sources
[S1] TomOpt: Differential optimisation for task- and constraint-aware design of particle detectors in the context of muon tomography, https://doi.org/10.1088/2632-2153/ad52e7, fetched 2026-10-26
