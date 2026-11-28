# BARF (Bundle-Adjusting Neural Radiance Fields)
Also called: Bundle-Adjusting NeRF
## Overview
BARF reconstructs a Neural Radiance Field (NeRF) from multi-view images when the input camera poses contain errors [S1, unofficial]. It addresses the sensitivity of NeRF to inaccurate camera poses, which typically leads to poor reconstruction quality [S1, unofficial]. Unlike standard NeRF which assumes known poses, BARF jointly optimizes the scene representation and camera parameters using a coarse-to-fine strategy that gradually introduces high-frequency positional encoding details [S1, unofficial].
## How it works
**Inputs and outputs:**
- Inputs: A set of $N$ images $\{I_i\}_{i=1}^N$ and their corresponding initial camera poses $\{P_i\}_{i=1}^N$, where $P_i$ may contain errors.
- Outputs: A refined NeRF model (MLP parameters for density $\sigma$ and color $c$) and refined camera poses $\{\hat{P}_i\}_{i=1}^N$.

**Data structures:**
- The NeRF is represented by two multi-layer perceptrons (MLPs): one for density $\sigma$ and one for view-dependent color $c$.
- Camera poses are represented as $4 \times 4$ transformation matrices.
- Positional encoding uses a hierarchy of frequencies.

**Steps:**
1. **Initialize:** Start with the initial noisy camera poses $P_i$ and standard NeRF MLPs initialized with small weights.
2. **Coarse-to-Fine Schedule:** Define a schedule for the positional encoding. Let $L$ be the total number of frequency bands. Initially, only the lowest frequency band (or a subset) is active.
3. **Joint Optimization:** For each training step:
   a. Sample rays through the current camera poses $P_i$.
   b. Encode the 3D points on the rays using the *currently active* subset of positional encoding frequencies.
   c. Query the density and color MLPs to compute volume rendering.
   d. Compute the photometric loss between rendered images and ground truth images $I_i$.
   e. Backpropagate to update both the MLP weights and the camera pose parameters $P_i$.
4. **Frequency Annealing:** Gradually increase the number of active high-frequency positional encoding bands over the course of training. This allows the optimization to first converge on the low-frequency structure (coarse alignment) before refining high-frequency details (fine alignment) [S1, unofficial].
5. **Final Output:** Return the final MLP weights and refined poses $\hat{P}_i$.

**Formulas:**
The volume rendering equation remains standard:
$$
\hat{C}(r) = \sum_{i=1}^n T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) c_i
$$
where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance.

The positional encoding $\gamma(p)$ for a point $p$ is truncated to the first $k$ frequencies during the coarse phase, where $k$ increases over time [S1, unofficial].

**Parameters:**
- Initial frequency bands: Typically only the lowest frequency (or no positional encoding initially) is used.
- Annealing schedule: Linear or step-wise increase of active frequencies over training iterations.
- Learning rate: Standard Adam optimizer settings are used.

**Edge cases and pitfalls:**
- If the initial pose error is too large, the coarse-to-fine strategy may fail to converge if the low-frequency signal is insufficient to guide the optimization [S1, unofficial].
- Precision issues may arise when optimizing rotation parameters directly; using exponential maps or quaternions is recommended [G].
- Sparse views exacerbate the problem, requiring careful regularization [S1, unofficial].
## Subtopics
## Operating characteristics
- **Time complexity:** Higher than standard NeRF due to the joint optimization of camera poses. The coarse-to-fine strategy helps convergence speed by avoiding local minima associated with high-frequency noise early on.
- **Memory complexity:** Similar to standard NeRF, plus storage for camera pose gradients.
- **Strengths:** Robust to initial pose errors; does not require precise camera calibration.
- **Limitations:** Requires iterative optimization; sensitive to the annealing schedule; may struggle with very sparse views without additional regularization [S1, unofficial].
- **When to use:** When camera poses are estimated from SfM or other approximate methods and contain significant errors.
- **When not to use:** When precise ground-truth poses are available, as standard NeRF is faster and simpler.
## Background
BARF was introduced in "Bundle-Adjusting Neural Radiance Fields" (ICCV 2021) [S1, unofficial]. It addresses the limitation that standard NeRF is not robust to inaccurate camera poses, a common issue in real-world scenarios where poses are estimated from Structure-from-Motion (SfM) pipelines [S1, unofficial]. The method leverages the observation that high-frequency positional encoding can hinder optimization when poses are inaccurate, leading to a coarse-to-fine refinement strategy.
## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
