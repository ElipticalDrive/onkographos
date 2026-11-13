# NeRF Ensemble with Conjoint Pruning

Also called: CPE, Ens-FT, IPM

## Overview
NeRF Ensemble with Conjoint Pruning (CPE) is a method that improves novel view synthesis quality by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture [S1]. It addresses the limitations of standard NeRF optimization, such as high computational requirements and sensitivity to training data, by achieving superior performance compared to single models while maintaining a similar parameter count and memory footprint [S1]. Unlike [Aug-NeRF](10294-aug-nerf.md), which uses adversarial regularization, or [RoGUENeRF](13731-roguenerf.md), which relies on post-processing reprojection, CPE employs a three-stage process of independent training, ensemble fine-tuning, and conjoint pruning to prevent the performance degradation often seen in vanilla ensembling [S1].

## How it works
The method operates on an ensemble of two NeRF models, denoted as $n=1, 2$, where each model $n$ maintains its own set of parameters $w_n$ and a voxel grid. The two models are trained with different grid resolutions [S1].

**Inputs:**
*   Training set $D_{train}$ containing samples $(x_m, \omega_m, C_m)$, where $x_m$ is a 3D point, $\omega_m$ is the viewing direction, and $C_m$ is the ground truth color.
*   Validation set $D_{val}$ used for pruning decisions.
*   Ensemble size: 2 models [S1].

**Outputs:**
*   A compressed ensemble of NeRF models optimized for novel view synthesis.

**Data Structures:**
*   **NeRF Model ($M_n$):** Contains parameters $w_n$ and a voxel grid. The specific layout of $w_n$ depends on the underlying DVGO architecture, which stores density and color features in a 3D grid [S1].
*   **Ensemble:** A collection of two independent models $\{M_1, M_2\}$ [S1].

**Algorithm Steps:**

1.  **Stage 1: Independent Training**
    Train two NeRF models, $M_1$ and $M_2$, independently on the training set $D_{train}$. Each model uses a different grid resolution. The training minimizes the standard NeRF loss function for each model $n$:
    $$
    L(w_n) = \frac{1}{M} \sum_{m=1}^{M} \| C_m - C(w_n, x_m, \omega_m) \|^2
    $$
    where $C(w_n, x_m, \omega_m)$ is the predicted color for the $m$-th sample by model $n$, and $M$ is the total number of training samples [S1]. Training proceeds for a fixed number of iterations or until convergence [S1].

2.  **Stage 2: Ensemble Fine-Tuning (Ens-FT)**
    Construct an ensemble by averaging the outputs of the two trained models. The predicted color for a sample $m$ is:
    $$
    C_{avg, m} = \frac{1}{2} \sum_{n=1}^{2} C(w_n, x_m, \omega_m)
    $$
    Fine-tune the ensemble by optimizing the averaged output. The loss function minimized during this stage is:
    $$
    L_{ens}(w_1, w_2) = \frac{1}{M} \sum_{m=1}^{M} \| C_m - C_{avg, m} \|^2
    $$
    This stage continues until performance on the validation set begins to drop [S1].

3.  **Stage 3: Conjoint Pruning (CPE)**
    Compress the ensemble using the validation set $D_{val}$. This stage involves a joint pruning strategy to remove redundant parameters from the ensemble while maintaining the performance gains achieved in Stage 2. The specific pruning mechanism is designed to prevent the performance degradation associated with vanilla ensembling [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_ensemble(D_train, D_val, grid_resolutions):
    """
    Trains a NeRF ensemble using Independent Training, Fine-Tuning, and Conjoint Pruning.
    
    Args:
        D_train: Training dataset of (x, omega, C) tuples.
        D_val: Validation dataset of (x, omega, C) tuples.
        grid_resolutions: List of two grid resolutions [res1, res2] for the two models.
        
    Returns:
        pruned_ensemble: List of two pruned NeRF models.
    """
    
    # Stage 1: Independent Training
    # Train two NeRFs independently with different grid resolutions
    model_1 = initialize_nerf(grid_resolutions[0])
    model_2 = initialize_nerf(grid_resolutions[1])
    
    # Minimize L(w) = (1/M) * sum(||C_m - C(w, x_m, omega_m)||^2)
    model_1 = optimize(model_1, D_train, loss_fn='mse')
    model_2 = optimize(model_2, D_train, loss_fn='mse')
    
    # Stage 2: Ensemble Fine-Tuning (Ens-FT)
    # Optimize the averaged output until performance drops
    # C_avg_m = 0.5 * (C(w1, x_m, omega_m) + C(w2, x_m, omega_m))
    # Loss: L_ens = (1/M) * sum(||C_m - C_avg_m||^2)
    
    ensemble = [model_1, model_2]
    best_ensemble = ensemble
    best_val_loss = infinity
    
    while True:
        # Compute ensemble predictions
        # Note: In practice, this involves forward passes through both models
        # and averaging the results.
        
        # Evaluate on validation set
        val_loss = compute_loss(ensemble, D_val)
        
        if val_loss > best_val_loss:
            # Performance dropped, stop fine-tuning
            break
        
        best_val_loss = val_loss
        best_ensemble = ensemble
        
        # Update parameters of both models to minimize ensemble loss
        # This requires computing gradients of the averaged output w.r.t. w1 and w2
        ensemble = fine_tune_ensemble(ensemble, D_train)
        
    # Stage 3: Conjoint Pruning (CPE)
    # Compress the ensemble using D_val
    # This step removes redundant parameters jointly from both models
    pruned_ensemble = conjoint_pruning(best_ensemble, D_val)
    
    return pruned_ensemble

def compute_loss(ensemble, dataset):
    """
    Computes the MSE loss between ground truth and ensemble average.
    """
    total_loss = 0.0
    count = 0
    for x, omega, C_gt in dataset:
        # Get predictions from both models
        pred_1 = ensemble[0].forward(x, omega)
        pred_2 = ensemble[1].forward(x, omega)
        
        # Average prediction
        pred_avg = 0.5 * (pred_1 + pred_2)
        
        # MSE loss
        loss = sum((C_gt - pred_avg) ** 2)
        total_loss += loss
        count += 1
        
    return total_loss / count

def conjoint_pruning(ensemble, D_val):
    """
    Compresses the ensemble by pruning parameters based on validation performance.
    The specific pruning algorithm is not detailed in the source, but it is
    a joint operation to prevent performance degradation.
    [G] Implementation detail: Typically involves evaluating parameter importance
    or redundancy across the ensemble and removing those with minimal impact
    on validation loss.
    """
    # Placeholder for the specific conjoint pruning logic
    # The source states this step uses D_val to compress the ensemble
    # without detailing the exact pruning criterion or threshold.
    return ensemble
```

**Parameters:**
*   **Ensemble Size:** 2 models [S1].
*   **Grid Resolutions:** The two models use different grid resolutions. The specific values are not given in the sources, but they are chosen to capture details at different scales [S1].
*   **Loss Function:** Mean Squared Error (MSE) between predicted and ground truth colors [S1].

**Edge Cases and Pitfalls:**
*   **Vanilla Ensembling:** Simply averaging two models without fine-tuning and conjoint pruning yields suboptimal results [S1].
*   **Performance Degradation:** Without conjoint pruning, compressing the ensemble may lead to significant performance loss [S1].
*   **Precision:** The method relies on the accuracy of the ensemble average; numerical stability should be considered when averaging outputs from models with different resolutions [S1].

## Operating characteristics
*   **Time Complexity:** The method requires training two models independently, which doubles the initial training time compared to a single model. The fine-tuning and pruning stages add further computational overhead [S1].
*   **Memory Footprint:** The ensemble maintains a similar parameter count and memory footprint to a single model of comparable quality, as the two models are trained at different scales and then compressed [S1].
*   **Performance:** Experimental results on Synthetic-NeRF show that this approach beats vanilla approaches under the same parameter cardinality constraint and achieves superior results compared to state-of-the-art architectures [S1].
*   **Strengths:** Improves novel view synthesis quality; requires fewer parameters than training and compressing one large model directly for the same image quality [S1].
*   **Limitations:** Higher computational cost during training due to the ensemble approach [S1].

## Background
The method was proposed by Francesco Di Sario, Riccardo Renzulli, Enzo Tartaglione, and Marco Grangetto in 2023 for the ICIAP 2023 conference [S1]. It builds on the Direct Voxel Grid Optimization (DVGO) architecture, which is a popular baseline for NeRFs due to its simplicity and effectiveness [S1]. The work addresses the high computational requirements and sensitivity to training data inherent in standard NeRF methods [S1].

## Sources
[S1] Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble, https://doi.org/10.1007/978-3-031-43153-1_27, fetched 2026-10-26
