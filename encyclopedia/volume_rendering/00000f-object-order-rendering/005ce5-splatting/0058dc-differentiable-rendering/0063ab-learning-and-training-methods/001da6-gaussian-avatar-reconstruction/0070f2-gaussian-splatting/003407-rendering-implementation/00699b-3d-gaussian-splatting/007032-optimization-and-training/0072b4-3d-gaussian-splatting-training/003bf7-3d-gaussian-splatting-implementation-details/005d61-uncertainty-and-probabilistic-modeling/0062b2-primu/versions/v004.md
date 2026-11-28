# PRIMU

Also called: Primitive-based Representations of Uncertainty

## Overview
PRIMU is a post-hoc uncertainty estimation (UE) framework for 3D Gaussian Splatting (3DGS) that constructs primitive-level representations of error and visibility/coverage from training views to estimate uncertainties for novel views [S1, unofficial]. It differs from sibling methods like FisherRF and Variational Bayes Gaussian Splatting by operating entirely as post-processing without requiring GS retraining, using 13 optionally direction-dependent features instead of a single uncertainty measure [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained 3DGS model (Gaussian primitives with parameters $\mu, \Sigma, g, c$), a set of training views with known poses, and optionally a small set of holdout views with ground truth for regression training.
*   **Outputs:** Pixel-wise uncertainty maps for novel views (color and depth), or active view selection (AVS) candidates.

**Data Structures:**
*   **Gaussian Primitives:** Each primitive $k$ has a 3D mean $\mu_k$, a 3x3 covariance matrix $\Sigma_k$ (or quaternion/scale representation), a scalar opacity factor $g$, and view-dependent color coefficients $c_k$ (encoded via spherical harmonics).
*   **Primitive Features:** A vector of 13 features per primitive, optionally direction-dependent. Key features include:
    *   Field-of-View (FoV) count $F(k)$.
    *   Training-view coverage $V(k)$.
    *   Rendering error $E(k)$.
    *   Other coverage/error statistics aggregated from training views.

**Numbered Steps:**

1.  **Project Training Errors and Coverage:**
    For each training view $v \in \Omega$ (where $\Omega$ is the set of training views):
    a. Render the scene using the trained 3DGS model to obtain predicted color $C_v(\bar{x})$ and depth $D_v(\bar{x})$ for each pixel $\bar{x}$.
    b. Compute per-pixel reconstruction error $e_{\bar{x}}$ (e.g., $\ell_1$ error for color or depth) against ground truth.
    c. For each Gaussian primitive $k$ visible in view $v$ (i.e., $\bar{x} \in v^{(k)}$ where opacity $\alpha_k(\bar{x}) > \text{threshold}$):
       i. Calculate the contribution factor: $w_{k, \bar{x}} = \alpha_k(\bar{x}) T_k(\bar{x})$, where $T_k(\bar{x}) = \prod_{j=1}^{k-1} (1 - \alpha_j(\bar{x}))$ is the accumulated transparency.
       ii. Aggregate contributions for primitive $k$ in view $v$ using an aggregation function $\text{agg} \in \{\text{max}, \text{sum}, \text{mean}\}$:
           $$ \text{Agg}_{v}(k) = \text{agg}_{\bar{x} \in v^{(k)}} (w_{k, \bar{x}}) $$
       iii. Compute the rendering error representation for primitive $k$ in view $v$:
           $$ E_v(k) = \text{mean}_{\bar{x} \in v^{(k)}} (e_{\bar{x}} \cdot w_{k, \bar{x}}) $$
           [S1, unofficial]

2.  **Construct Primitive-Level Representations:**
    For each primitive $k$:
    a. **FoV Count:** $F(k) = |\Omega_k|$, where $\Omega_k$ is the set of training views where primitive $k$ is visible.
    b. **Coverage:** $V(k) = \max_{v \in \Omega} (\text{Agg}_{v}(k))$. This captures the maximum aggregated contribution across all training views.
    c. **Error:** $E(k) = \text{mean}_{v \in \Omega} (E_v(k))$. This captures the mean error-weighted contribution.
    d. Extract the remaining 10 features (details not fully specified in sources, but noted as optionally direction-dependent). [S1, unofficial]

3.  **Render Uncertainty Feature Maps:**
    For a novel viewpoint:
    a. Project all Gaussian primitives onto the image plane using standard 3DGS splatting.
    b. For each pixel $\bar{x}$, render the primitive-level features (e.g., $F(k), V(k), E(k)$) as separate channels. This produces "uncertainty feature maps" where each pixel contains the aggregated features of the contributing primitives.
    c. The rendering of a feature map $M$ at pixel $\bar{x}$ is computed similarly to color:
       $$ M(\bar{x}) = \sum_{k=1}^{K} \alpha_k(\bar{x}) T_k(\bar{x}) \cdot \text{feature}_k $$
       where $\text{feature}_k$ is the value of the specific feature for primitive $k$. [S1, unofficial]

4.  **Aggregate via Regression:**
    a. If holdout data is available, train a gradient boosting regression model on the rendered uncertainty feature maps (from holdout views) to predict the true pixel-wise error (color or depth).
    b. For novel views, render the uncertainty feature maps and pass them through the trained regression model to obtain the final uncertainty estimate.
    c. If no regression is used, the uncertainty feature maps can be used directly for Active View Selection (AVS) by prioritizing viewpoints with high uncertainty indicators. [S1, unofficial]

**Formulas:**
*   **Opacity:** $\alpha(\bar{x}) = g \Phi(\bar{x}; \bar{\mu}, \bar{\Sigma})$, where $\Phi$ is the 2D Gaussian density, $\bar{\mu}$ is the mean, and $\bar{\Sigma}$ is the covariance. [S1, unofficial]
*   **Color:** $C(\bar{x}) = \sum_{k=1}^{K} \alpha_k(\bar{x}) T_k(\bar{x}) c_k(\vec{d})$, where $c_k(\vec{d})$ is the view-dependent color. [S1, unofficial]
*   **FoV Count:** $F(k) = |\Omega_k|$. [S1, unofficial]
*   **Coverage:** $V(k) = \max_{v \in \Omega} \text{agg}_{\bar{x} \in v^{(k)}} (\alpha_k(\bar{x}) T_k(\bar{x}))$. [S1, unofficial]
*   **Error:** $E(k) = \text{mean}_{v \in \Omega} \text{agg}_{\bar{x} \in v^{(k)}} (e_{\bar{x}} \alpha_k(\bar{x}) T_k(\bar{x}))$. [S1, unofficial]

**Parameters:**
*   Number of Gaussian-primitive features: 13 [S1, unofficial]
*   Regression model: Gradient boosting [S1, unofficial]
*   Aggregation functions: max, sum, or mean [S1, unofficial]
*   Opacity threshold for visibility: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Geometry Misalignment:** Gaussian splats may misalign with scene geometry even when rendered appearance is correct, leading to depth error between true geometry and splat distribution. This affects the accuracy of depth uncertainty estimates. [S1, unofficial]
*   **Undefined Depth Error:** Regions with undefined depth error (e.g., background) should be handled, potentially represented as light gray or excluded from regression. [S1, unofficial]
*   **Generalization:** While the method generalizes to unseen scenes to some extent, performance may degrade if the training data distribution differs significantly from the novel scene. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Post-hoc processing. Rendering feature maps is $O(N \cdot P)$ where $N$ is the number of primitives and $P$ is the number of pixels. Regression is fast once trained.
*   **Memory Complexity:** Requires storing 13 feature values per primitive. No additional model parameters for the UE itself (except the lightweight regression model).
*   **Strengths:** No retraining required; interpretable features; strong correlation with true errors, especially for depth and foreground objects; generalizes to unseen scenes. [S1, unofficial]
*   **Limitations:** Relies on the quality of the initial 3DGS reconstruction; may not capture all sources of uncertainty (e.g., model misspecification); requires holdout data for optimal regression performance (though not strictly necessary for AVS). [S1, unofficial]
*   **When to Use:** Safety-critical domains (robotics, medical imaging); Active View Selection; when retraining is not feasible. [S1, unofficial]
*   **When Not to Use:** When real-time uncertainty estimation during training is required; when computational resources for regression are unavailable (though minimal). [S1, unofficial]

## Background
PRIMU was introduced by Stehr et al. in 2025 (preprint) as a post-hoc uncertainty estimation framework for Gaussian Splatting. It addresses the challenge of reliable uncertainty quantification in 3DGS, which is critical for deploying GS in robotics and medicine. Existing methods often estimate per-primitive uncertainty during training using Fisher information or learned distributions, whereas PRIMU operates entirely as post-processing. [S1, unofficial]

## Sources
[S1] PRIMU: Uncertainty Estimation for Novel Views in Gaussian Splatting from Primitive-Based Representations of Error and Coverage, https://arxiv.org/abs/2508.02443v2, fetched 2026-10-26 (unofficial)
