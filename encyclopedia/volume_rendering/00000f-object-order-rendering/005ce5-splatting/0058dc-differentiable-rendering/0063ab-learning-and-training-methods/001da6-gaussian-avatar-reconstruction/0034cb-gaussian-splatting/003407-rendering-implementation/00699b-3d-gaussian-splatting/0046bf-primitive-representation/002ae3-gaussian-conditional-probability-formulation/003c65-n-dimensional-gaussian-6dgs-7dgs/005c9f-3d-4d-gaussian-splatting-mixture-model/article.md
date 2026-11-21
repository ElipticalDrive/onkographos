# 3D/4D Gaussian Splatting Mixture Model

Also called: 3D/4DGS Mixture Model

## Overview
The 3D/4D Gaussian Splatting Mixture Model treats a radiance field as a probabilistic mixture of 3D Gaussian primitives, where each component is defined by location, shape, opacity, and color [S1]. It differs from deterministic splatting by framing the rendering process as the evaluation of expected color values conditioned on spatial location, allowing for the modeling of data uncertainty and distributional parameters rather than single-point estimates [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $M$ Gaussian components, each defined by parameters $\theta_k$. For standard 3DGS, these are location $\mu_k$, shape $\Sigma_k$, opacity $o_k$, and color $s_k$ (via spherical harmonics) [S1]. For Variational Bayesian GS (VBGS), parameters include distributions over these attributes (e.g., Normal-Inverse-Wishart for color/position, Dirichlet for assignments) [S2, unofficial].
*   **Outputs:** The expected color $C(d)$ for each pixel $d$ in the rendered image [S1][S2, unofficial].

**Data Structures:**
*   **Standard 3DGS Component:**
    *   $\mu_k \in \mathbb{R}^3$: Mean location.
    *   $\Sigma_k \in \mathbb{R}^{3 \times 3}$: Covariance matrix (shape).
    *   $o_k \in [0, 1]$: Opacity.
    *   $s_k$: Color coefficients (Spherical Harmonics).
*   **VBGS Component (Probabilistic):**
    *   Represents a distribution over parameters.
    *   Requires 29 parameters per component (vs. 14 in gradient baselines) to model the Normal-Inverse-Wishart likelihood for color/position and Dirichlet prior [S2, unofficial].
    *   Stores posterior probabilities $p(z_n = k|s)$, representing the probability that data point $n$ belongs to component $k$ given spatial location $s$ [S2, unofficial].

**Rendering Formula:**
The final color $C(d)$ of the $d$-th pixel is computed by summing contributions from $N$ intersecting Gaussians [S1]:

$$
C(d) = \sum_{i=1}^{N} s_i o_i G_{i,2D}(d) \prod_{j=1}^{i-1} (1 - o_j G_{j,2D}(d))
$$

Where:
*   $s_i$: Color of the $i$-th Gaussian.
*   $o_i$: Opacity of the $i$-th Gaussian.
*   $G_{i,2D}(d)$: The value of the 2D Gaussian projected onto the image plane at pixel $d$.
*   The term $\prod_{j=1}^{i-1} (1 - o_j G_{j,2D}(d))$ represents the alpha blending transmittance from previous Gaussians.

**VBGS Specifics:**
In VBGS, rendering evaluates the expected color $E_{p(c|s)}[c]$ [S2, unofficial]. This is approximated as:

$$
E_{p(c|s)}[c] \approx \sum_{k} p(z_n = k|s) c_k
$$

Where $c_k$ is the mean color of component $k$ and $p(z_n = k|s)$ is the posterior assignment probability [S2, unofficial]. Note that VBGS sets alpha values to 1 (opaque) for all components, bypassing standard alpha blending as it does not optimize for image reconstruction in the same manner [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize Gaussian parameters. For VBGS, initialize variational distributions (Normal-Inverse-Wishart, Dirichlet) [S2, unofficial].
2.  **Projection:** Transform 3D Gaussians to 2D Gaussians on the image plane [S1].
3.  **Assignment (VBGS only):** Compute posterior probabilities $p(z_n = k|s)$ for each data point $n$ and component $k$ [S2, unofficial].
4.  **Rendering:**
    *   For standard GS: Apply alpha blending formula [S1].
    *   For VBGS: Compute weighted sum of expected colors using posterior probabilities [S2, unofficial].
5.  **Optimization (VBGS only):** Perform Coordinate Ascent Variational Inference (CAVI) [S2, unofficial]:
    *   Step A: Compute model assignments (Complexity $O(nm)$) [S2, unofficial].
    *   Step B: Compute parameter updates (Complexity $O(n)$) [S2, unofficial].

**Parameters:**
*   **VBGS Memory:** Increases by a factor of 2 compared to gradient baselines for the same number of Gaussians due to storing 29 parameters per component [S2, unofficial].
*   **Data Normalization:** Data is normalized to zero mean and unit standard deviation before training [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **VBGS Alpha Blending:** Standard alpha blending is not used in VBGS because it does not optimize for image reconstruction; all components are treated as opaque ($\alpha=1$) [S2, unofficial].
*   **Uncertainty:** Standard 4DGS does not model asynchronous cameras or multiple types of uncertainty; this requires extensions like GraphiXS [S1].

## Subtopics

### VBGS (Variational Bayesian Gaussian Splatting)
A probabilistic variant that represents a distribution over Gaussian Mixture Model parameters using Normal-Inverse-Wishart likelihoods and Dirichlet priors [S2, unofficial]. It differs from standard GS by optimizing via CAVI rather than gradient descent, allowing for uncertainty modeling [S2, unofficial]. Choose it when modeling data uncertainty or when a probabilistic representation is required, accepting higher memory usage (2x) and computational cost [S2, unofficial]. Link: [VBGS](link-to-vbgs-if-exists).

## Operating characteristics
*   **Time Complexity:** Standard GS rendering is dominated by sorting and blending $N$ Gaussians. VBGS training involves two steps: assignment $O(nm)$ and parameter updates $O(n)$, where $n$ is data points and $m$ is components [S2, unofficial].
*   **Memory Complexity:** VBGS requires 29 parameters per component compared to 14 in standard GS, doubling memory requirements for the same number of Gaussians [S2, unofficial].
*   **Performance:** VBGS training can be parallelized on GPU for both components and data [S2, unofficial].
*   **Strengths:** Handles data uncertainty; provides probabilistic outputs [S1][S2, unofficial].
*   **Limitations:** Higher memory and computational cost for VBGS; standard GS lacks uncertainty modeling [S1][S2, unofficial].

## Background
3D Gaussian Splatting was introduced as a new paradigm for neural rendering, representing radiance fields with 3D Gaussians [S1]. Extensions like GraphiXS address data uncertainty in 4DGS [S1]. VBGS applies variational Bayesian methods to Gaussian Splatting, treating it as a Gaussian Mixture Model [S2, unofficial].

## Sources
[S1] Graphical X Splatting (GraphiXS): A Graphical Model for 4D Gaussian Splatting under Uncertainty, https://doi.org/10.1145/3799902.3811085, fetched 2026-11-05
[S2] Variational Bayes Gaussian Splatting, https://arxiv.org/abs/2410.03592v2, fetched 2026-10-26 (unofficial)
