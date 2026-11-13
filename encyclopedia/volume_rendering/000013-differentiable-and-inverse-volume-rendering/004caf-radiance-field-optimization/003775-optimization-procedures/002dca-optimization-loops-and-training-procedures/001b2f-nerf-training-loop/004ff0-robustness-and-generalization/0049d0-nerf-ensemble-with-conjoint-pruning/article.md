# NeRF Ensemble with Conjoint Pruning

Also called: CPE, Ens-FT, IPM

## Overview
NeRF Ensemble with Conjoint Pruning (CPE) is a method that improves novel view synthesis quality by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture [S1]. It achieves superior performance compared to single models while maintaining a similar parameter count and memory footprint [S1]. Unlike vanilla ensembling, which yields suboptimal results, CPE employs a three-stage process involving independent training, fine-tuning of the averaged output (Ens-FT), and compression via conjoint pruning (CPE) [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Training dataset $D_{train}$, validation dataset $D_{val}$, and a fixed number of training iterations for Stage 1.
*   **Outputs:** An ensemble of two compressed NeRF models.

**Data Structures**
The ensemble consists of two separate NeRF models (DVGO architectures). Each model $n \in \{1, 2\}$ maintains its own set of parameters $w_n$ and a voxel grid. The models are trained with different grid resolutions [S1].

**Parameters**
*   **Ensemble size:** 2 NeRF models [S1].
*   **Grid Resolution:** The two models use different resolutions. Specific resolution values are not given in the sources [S1].
*   **Training Iterations:** Fixed number for Stage 1. Specific value is not given in the sources [S1].

**Steps**
1.  **Stage 1: Independent Training.** Train the two NeRF models independently on $D_{train}$. For each model, minimize the standard NeRF loss function for a fixed number of iterations [S1].
2.  **Stage 2: Ensemble Fine-Tuning (Ens-FT).** Fine-tune the ensemble by optimizing the averaged output. This stage continues until performance drops [S1].
3.  **Stage 3: Conjoint Pruning (CPE).** Compress the ensemble using $D_{val}$ via conjoint pruning. This step prevents performance degradation associated with pruning [S1].

**Formulas**
The loss function minimized during Stage 1 for a model with parameters $w$ is [S1]:

$$
L(w) = \frac{1}{M} \sum_{m=1}^{M} \| C_m - C(w, x_m, \omega_m) \|^2
$$

Where:
*   $M$ is the total number of training samples.
*   $C_m$ is the ground truth color of the $m$-th pixel.
*   $C(w, x_m, \omega_m)$ is the predicted color by the NeRF.
*   $x_m$ is the 3D point where the pixel lies.
*   $\omega_m$ is the corresponding viewing direction.

The ensemble output $C_{avg_m}$ used in Stage 2 is the simple average of the outputs of the two NeRF models ($n=1, 2$) [S1]:

$$
C_{avg_m} = \frac{1}{2} \sum_{n=1}^{2} C^n(w_n, x_m, \omega_m)
$$

Where $C^n$ is the output of the $n$-th NeRF with parameters $w_n$.

The loss function minimized during Stage 2 (Ens-FT) is the difference between the ground truth and the averaged ensemble output [S1]:

$$
L_{Ens-FT} = \frac{1}{M} \sum_{m=1}^{M} \| C_m - C_{avg_m} \|^2
$$

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_ensemble(D_train, D_val, num_iterations_stage1):
    """
    Trains a NeRF ensemble using Independent Training, Fine-Tuning, and Conjoint Pruning.
    
    Args:
        D_train: Training dataset.
        D_val: Validation dataset.
        num_iterations_stage1: Fixed number of iterations for independent training.
        
    Returns:
        ensemble: List of two compressed NeRF models.
    """
    # Stage 1: Independent Training
    # Train two NeRFs with different grid resolutions independently
    nerf_1 = initialize_nerf(resolution="high")
    nerf_2 = initialize_nerf(resolution="low")
    
    # Minimize standard NeRF loss for fixed iterations
    nerf_1 = optimize(nerf_1, D_train, loss_fn=standard_nerf_loss, iterations=num_iterations_stage1)
    nerf_2 = optimize(nerf_2, D_train, loss_fn=standard_nerf_loss, iterations=num_iterations_stage1)
    
    ensemble = [nerf_1, nerf_2]
    
    # Stage 2: Ensemble Fine-Tuning (Ens-FT)
    # Fine-tune the ensemble by optimizing the averaged output
    # Continue until performance drops
    while performance_has_not_dropped(ensemble, D_val):
        for nerf in ensemble:
            # Update parameters to minimize the difference between ground truth
            # and the averaged ensemble output
            nerf = optimize(nerf, D_train, loss_fn=ensemble_fine_tuning_loss, iterations=1)
            
    # Stage 3: Conjoint Pruning (CPE)
    # Compress the ensemble using the validation set
    ensemble = conjoint_pruning(ensemble, D_val)
    
    return ensemble

def standard_nerf_loss(w, x_m, omega_m, C_m):
    """
    Standard NeRF loss function.
    L(w) = (1/M) * sum(||C_m - C(w, x_m, omega_m)||^2)
    """
    predicted_color = render(nerf=w, point=x_m, direction=omega_m)
    return mean_squared_error(C_m, predicted_color)

def ensemble_fine_tuning_loss(ensemble, x_m, omega_m, C_m):
    """
    Loss function for Ens-FT stage.
    Minimizes difference between ground truth and averaged ensemble output.
    """
    # Calculate output for each model in the ensemble
    outputs = [render(nerf=nerf, point=x_m, direction=omega_m) for nerf in ensemble]
    
    # Average the outputs
    avg_output = sum(outputs) / len(outputs)
    
    return mean_squared_error(C_m, avg_output)

def conjoint_pruning(ensemble, D_val):
    """
    Compresses the ensemble using the validation set.
    Implementation details of the specific pruning algorithm are not provided in the sources.
    [G] Placeholder for the specific conjoint pruning logic described in the source.
    """
    # Note: The source mentions a "formulation to prevent performance degradation"
    # but does not provide the explicit algorithmic steps for the pruning itself.
    # An implementer would need to refer to the specific pruning mechanism 
    # defined in the original paper for the exact weight removal/thresholding logic.
    return ensemble
```

**Edge Cases and Pitfalls**
*   **Vanilla Ensembling:** Simply averaging two models without the proposed conjoint pruning and fine-tuning strategy yields suboptimal results [S1].
*   **Performance Drop:** The fine-tuning stage must be monitored to stop when performance drops, preventing overfitting or degradation [S1].

## Subtopics

### DVGO (Direct Voxel Grid Optimization)
The baseline architecture used for the NeRF models in this method. It uses explicit volumetric information, making optimization difficult when fine-graining voxel grids, but offers simplicity and effectiveness [S1]. See [DVGO](link-to-dvgo.md).

### NeRF Compression
The goal of Stage 3 is to compress the ensemble. Conjoint pruning is a specific technique to reduce parameters while maintaining quality [S1]. See [NeRF Compression](link-to-compression.md).

## Operating characteristics
*   **Time and Memory Complexity:** The method requires training two models independently, which increases training time compared to a single model. However, it maintains a similar parameter count and memory footprint compared to a single large model [S1].
*   **Measured Performance:** Experimental results on Synthetic-NeRF show that the ensemble approach beats vanilla approaches under the same parameter cardinality constraint and achieves superior results compared to state-of-the-art architectures [S1].
*   **Scaling:** Ensembling multiple models at different scales requires fewer parameters than training and compressing one large model directly under the same generated image quality constraint [S1].
*   **Strengths:** Improves novel view synthesis performance; maintains similar memory footprint to single models; robust against the suboptimality of vanilla ensembling.
*   **Limitations:** Requires training two models; sensitive to the fine-tuning and pruning stages; specific resolution and iteration parameters are not standardized in the source [S1].
*   **When to use:** When high-quality 3D scene modeling is required with limited memory/parameter constraints, and computational resources allow for training multiple models.
*   **When not to use:** When training time is strictly limited to a single model's duration, or when memory constraints are extremely tight (though it claims similarity to single models, it still holds two sets of weights during training).

## Background
*   **Origin:** Proposed by Francesco Di Sario, Riccardo Renzulli, Enzo Tartaglione, and Marco Grangetto in 2023 for ICIAP 2023 [S1].
*   **History:** The work addresses the high computational requirements and sensitivity to training data of NeRFs [S1]. It builds on DVGO, a popular baseline for NeRFs due to its simplicity [S1]. It is the first work proposing a joint ensembling and compression scheme for NeRF models [S1].

## Sources
[S1] Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble, https://doi.org/10.1007/978-3-031-43153-1_27, fetched 2026-10-26
