# Volumetric Supervision

Also called: CT-Guided Training, Hybrid Supervision

## Overview
Volumetric Supervision is a training strategy that leverages high-quality volumetric reconstructions from conventional tomography to supervise neural network training at intermediate time steps where only sparse projection data is available [S1, unofficial]. It solves the problem of temporal interpolation in dynamic tomography by using dense ground-truth volumes at the start and end of an experiment to regularize the learning of the canonical model and deformation field [S1, unofficial]. Unlike sibling methods that primarily enforce geometric consistency via depth or gradient losses, this method directly minimizes the discrepancy between the predicted 3D volume and a physically reconstructed 3D volume [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Sparse projection data for intermediate time steps $t \in (0, 1)$ [S1, unofficial].
    *   Dense projection data at boundary time steps $t=0$ and $t=1$ [S1, unofficial].
    *   Conventional tomography reconstructions (ground truth volumes) $\mathbf{y}$ at $t=0$ and $t=1$ [S1, unofficial].
    *   Neural network parameters $\theta$ defining the canonical volume and deformation field [S1, unofficial].
*   **Outputs:**
    *   Updated network parameters $\theta$ [S1, unofficial].
    *   Predicted volume $\hat{\mathbf{y}}$ at intermediate times [S1, unofficial].

**Data Structures:**
*   **Target Volume $\mathbf{y}$:** A 3D voxel grid representing the ground-truth density/attenuation field reconstructed via conventional tomography at $t=0$ or $t=1$ [S1, unofficial].
*   **Predicted Volume $\hat{\mathbf{y}}$:** A 3D voxel grid sampled from the neural network's canonical representation, warped by the deformation field to match the intermediate time step [G].
*   **Note:** The exact resolution and bit-depth of these volumes are not specified in the sources [S1, unofficial].

**Steps:**
1.  **Boundary Reconstruction:** Obtain rich projection data at the start ($t=0$) and end ($t=1$) of the experiment [S1, unofficial]. Reconstruct the 3D volumetric data $\mathbf{y}_0$ and $\mathbf{y}_1$ using conventional tomography algorithms [S1, unofficial].
2.  **Intermediate Prediction:** For an intermediate time step $t \in (0, 1)$ with sparse projections, query the neural network to predict the volume $\hat{\mathbf{y}}_t$ [S1, unofficial]. This involves evaluating the canonical volume and applying the learned deformation field to account for temporal changes [G].
3.  **Correlation Calculation:** Compute the normalized correlation coefficient $C_{cor}$ between the target volume $\mathbf{y}$ (interpolated or selected from $t=0$ or $t=1$ based on temporal proximity or specific protocol) and the predicted volume $\hat{\mathbf{y}}_t$ [S1, unofficial].
4.  **Loss Computation:** Calculate the volumetric loss $\mathcal{L}_{vol}$ as the negative normalized correlation coefficient [S1, unofficial]:
    $$ \mathcal{L}_{vol} = -C_{cor}(\mathbf{y}, \hat{\mathbf{y}}_t) $$
5.  **Optimization:** Minimize $\mathcal{L}_{vol}$ along with other task-specific losses (e.g., projection consistency) using gradient descent to update $\theta$ [S1, unofficial].

**Formula:**
The volumetric loss is defined as the negative normalized correlation coefficient between the target CT volume $\mathbf{y}$ and the predicted volume $\hat{\mathbf{y}}$ [S1, unofficial]:
$$ \mathcal{L}_{vol} = -C_{cor}(\mathbf{y}, \hat{\mathbf{y}}) $$
The normalized correlation coefficient is used because density level scales may differ between the canonical volume and the CT-reconstructed volume [S1, unofficial].

**Parameters:**
*   **Time Steps:** $t=0$ and $t=1$ denote the start and end of the experiment [S1, unofficial]. Intermediate $t$ values are not given specific typical values [S1, unofficial].
*   **Correlation Metric:** Normalized correlation coefficient is used to handle scale differences in density levels between the canonical volume and the CT-reconstructed volume [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Scale Mismatch:** Density levels may differ between the canonical volume and the CT-reconstructed volume, which the normalized correlation coefficient mitigates [S1, unofficial].
*   **Sparse Data:** The method is specifically designed for the "sparse few-projection regime" at intermediate times [S1, unofficial].
*   **Temporal Alignment:** The method assumes that the deformation field can accurately warp the canonical volume to match the physical state at time $t$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volumetric_loss(target_volume, predicted_volume):
    """Computes the volumetric loss as the negative normalized correlation coefficient."""
    n = len(target_volume)
    mean_target = sum(target_volume) / n
    mean_pred = sum(predicted_volume) / n
    dev_target = [x - mean_target for x in target_volume]
    dev_pred = [x - mean_pred for x in predicted_volume]
    numerator = sum(dt * dp for dt, dp in zip(dev_target, dev_pred))
    denom_target = sum(dt ** 2 for dt in dev_target) ** 0.5
    denom_pred = sum(dp ** 2 for dp in dev_pred) ** 0.5
    if denom_target == 0 or denom_pred == 0:
        correlation = 0.0
    else:
        correlation = numerator / (denom_target * denom_pred)
    return -correlation

def train_step(sparse_projections, target_volume, network, optimizer, t):
    """Single training step for Volumetric Supervision."""
    optimizer.zero_grad()
    # Evaluate canonical volume and apply deformation field to match time t [G]
    predicted_volume = network.predict_volume_at_time(t)
    loss_vol = compute_volumetric_loss(target_volume, predicted_volume)
    total_loss = loss_vol
    total_loss.backward()
    optimizer.step()
    return total_loss.item()
```

## Subtopics

### Monte Carlo Estimation for Volumetric Supervision
An optimization strategy that approximates the dense voxel objective using a subset of voxels while preserving consistency with the full-volume optimization [S2]. It uses importance-weighted Monte Carlo estimation to approximate dense reconstruction objectives from sparse voxel subsets [S2]. Instead of computing the reconstruction loss over the full voxel grid, it uses the Monte Carlo estimator to approximate the objective [S2]. Importance-weighted sampling improves coverage of informative regions [S2]. This updates Gaussian parameters using more representative voxel samples and stabilizes optimization [S2]. However, without slice supervision, the model lacks structured planar constraints and shows lower performance in rendered views than methods combining sparse voxel sampling with slice-based sampling [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial].
*   **Scaling:** Not given in the sources [S1, unofficial].
*   **Strengths:** Directly supervises the 3D structure, which can improve the accuracy of the canonical model and deformation field in dynamic tomography [S1, unofficial]. Handles scale differences between canonical and CT volumes via normalized correlation [S1, unofficial].
*   **Limitations:** Requires high-quality volumetric reconstructions at the start and end of the experiment, which may not always be available [S1, unofficial]. Relies on the quality of the conventional tomography reconstruction [S1, unofficial].
*   **When to Use:** When dense volumetric ground truth is available at boundary time steps and sparse projections are available at intermediate times [S1, unofficial].

## Background
Volumetric Supervision was introduced in the context of dynamic tomography, where the goal is to reconstruct 4D (3D + time) volumes from sparse projections [S1, unofficial]. The method builds on the idea that while intermediate time steps may have sparse data, the start and end states often have rich data that can be reconstructed using conventional tomography [S1, unofficial]. This approach was proposed to improve the training of neural networks for dynamic tomography by providing strong volumetric constraints [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
[S2] Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, https://doi.org/10.1007/978-3-032-37461-5_35, fetched 2026-10-26
