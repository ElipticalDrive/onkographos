# Conjoint Pruning of the Ensemble

Also called: Stage 3, CPE

## Overview
Conjoint Pruning of the Ensemble (CPE) is the final optimization phase in the JointRF training procedure, designed to jointly prune multiple pre-trained neural field models to achieve superior reconstruction quality compared to ensembling pre-trained models with a comparable or lower memory footprint [S1]. It distinguishes itself from sibling methods like [Rate-Distortion Joint Finetuning](12624-rate-distortion-joint-finetuning.md) by explicitly modifying the ensemble's aggregation logic from averaging to summation to prevent signal degradation during the pruning process [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** An ensemble of $n$ pre-trained models with weights $w_n$ and parameters $\omega_n$; a dataset of $M$ training views with input coordinates $x_m$ and target colors $C_m$; a target compression rate (Low or High); and a maximum number of optimization epochs [S1].
*   **Outputs:** Pruned ensemble models with optimized weights $w_n$ [S1].

**Data Structures:**
Not given in the sources.

**Steps:**
1.  **Define Ensemble Output:** Instead of averaging the outputs of the $n$ models, compute the summed ensemble output $C^2_m$ for input $x_m$ [S1]:
    $$ C^2_m = \sum_{n=1}^{n} C_n(w_n, x_m, \omega_n) $$
    where $C_n(w_n, x_m, \omega_n)$ is the output (e.g., color) of the $n$-th model [S1].

2.  **Define Loss Function:** Minimize the squared difference between the target color $C_m$ and the summed ensemble output using the loss function $L_{sum}$ [S1]:
    $$ L_{sum}(w) = \frac{1}{M} \sum_{m=1}^{M} \left( C_m - \sum_{n=1}^{n} C_n(w_n, x_m, \omega_n) \right)^2 $$
    where $M$ is the number of training views [S1].

3.  **Joint Optimization:** Perform gradient descent to update the weights $w$ of all models in the ensemble simultaneously to minimize $L_{sum}$ [S1].

**Parameters:**
*   **Compression Rates:** The source defines two specific rates based on the percentage of total parameters retained:
    *   **Low:** 87.50% of total parameters retained [S1].
    *   **High:** 96.87% of total parameters retained [S1].
*   **Number of Models:** The source experiments use an ensemble of $n=2$ models [S1].

**Edge Cases and Pitfalls:**
*   **Signal Reduction with Averaging:** Using the averaging loss function $L_{avg}$ during conjoint pruning causes significant signal reduction [S1]. Pruning creates null or near-zero values in model outputs; averaging these sparse outputs reduces the overall signal magnitude, leading to suboptimal results [S1].
*   **Pruning Progression:** The benefit of using $L_{sum}$ over $L_{avg}$ increases as the pruning process progresses and sparsity increases [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def conjoint_pruning_ensemble(models, training_data, compression_rate, max_epochs, learning_rate):
    """
    Jointly prunes an ensemble of models using summed output optimization.
    
    Args:
        models: List of neural field models [model_1, model_2, ...]
        training_data: List of tuples (x_m, C_m) for M views
        compression_rate: Target compression rate (e.g., 'Low' or 'High')
        max_epochs: Number of optimization iterations
        learning_rate: Step size for gradient descent
        
    Returns:
        Pruned models
    """
    for epoch in range(max_epochs):
        total_loss = 0.0
        for x_m, C_m in training_data:
            # Step 1: Compute summed ensemble output
            summed_output = 0.0
            for model in models:
                summed_output += model(x_m)
            
            # Step 2: Compute loss using summed output (L_sum)
            error = C_m - summed_output
            loss = error ** 2
            total_loss += loss
        
        # Average loss over M views
        avg_loss = total_loss / len(training_data)
        
        # Step 3: Backpropagate and update weights
        # Compute gradients for all models w.r.t. avg_loss
        for model in models:
            # [G] Standard automatic differentiation computes gradients
            grads = compute_gradients(model, avg_loss)
            # Standard SGD update: w <- w - lr * grad
            for param in model.parameters():
                param.data -= learning_rate * grads[param]
        
        # [G] Pruning masks are applied based on compression_rate after each epoch
        # The exact mask generation is handled by the underlying pruning mechanism.
    return models
```

## Subtopics

### [Magnitude-based Edge Pruning](14607-magnitude-based-edge-pruning.md)
Magnitude-based Edge Pruning is an unstructured pruning technique that eliminates individual weights (edges) in a neural network based on their absolute magnitude, typically discarding those below a specific threshold. It addresses the problem of model sparsity by reducing the parameter count while attempting to maintain inference quality. Unlike structured pruning methods that remove entire neurons or layers, edge pruning creates irregular memory patterns, which can hinder efficient hardware implementation but allows for fine-grained compression. It is distinguished from its sibling [Conjoint Pruning of the Ensemble](12978-conjoint-pruning-of-the-ensemble.md) by targeting individual connections rather than jointly pruning ensemble models for memory footprint reduction.

## Operating characteristics
*   **Time and Memory Complexity:** The method requires optimizing $n$ models jointly, increasing computational cost per iteration compared to pruning a single model [S1]. Memory footprint is comparable to or lower than ensembling pre-trained models, depending on the compression rate [S1].
*   **Measured Performance:** On the SyntheticNeRF dataset (800x800 resolution, 100 training views), CPE outperforms individually pruned models (IPM) and IPM followed by fine-tuning in ensembling (IPM+ENS-FT) [S1]. Performance degradation is minor as compression increases [S1].
*   **Strengths:** Superior performance compared to ensembling pre-trained models with comparable or lower memory footprint; avoids signal reduction issues inherent in averaging sparse outputs [S1].
*   **Limitations:** Requires joint optimization of multiple models, which is computationally more expensive than independent pruning; sensitive to the choice of aggregation function (summation vs. averaging) [S1].
*   **When to use:** When modeling specific objects from datasets like SyntheticNeRF with strict memory constraints, requiring joint fine-tuning of an ensemble [S1].
*   **When not to use:** When computational resources are limited to independent model training/pruning, or when the ensemble size is large enough that summation leads to overflow (not discussed in sources) [S1].

## Background
Conjoint Pruning of the Ensemble (CPE) was introduced by F. Di Sario et al. in the paper "Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble" [S1]. It is presented as Stage 3 of a three-stage training procedure for NeRF ensembles, following pre-training and individual pruning [S1]. The method was evaluated on the SyntheticNeRF dataset, which contains eight realistic objects synthesized from Blender [S1].

## Sources
[S1] Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble, https://doi.org/10.1007/978-3-031-43153-1_27, fetched 2026-10-26
