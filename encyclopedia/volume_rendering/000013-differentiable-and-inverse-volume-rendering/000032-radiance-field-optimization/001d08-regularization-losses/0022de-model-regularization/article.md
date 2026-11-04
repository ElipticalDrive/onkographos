# Model Regularization

Also called: Zero Regularization Loss

## Overview
Model regularization in radiance-field optimization refers to auxiliary loss terms applied directly to the neural network's internal predictions to stabilize training and prevent pathological behaviors, such as uncertainty values exploding or overfitting to noise. Unlike photometric losses that compare rendered images to ground truth, model regularization constrains the latent space or intermediate outputs (e.g., uncertainty, density) based on prior assumptions about the scene or network stability. This article focuses on the specific implementation of uncertainty regularization and the general framework for aggregating these losses, distinguishing it from geometric or perceptual sibling losses.

## How it works
The method operates by extracting specific internal variables from the radiance field model after forward propagation and computing penalty terms. These terms are weighted by hyperparameters and added to the total optimization objective.

### Inputs and Outputs
- **Inputs**: The trained radiance field model (e.g., NeRF), a batch of rays $\mathcal{R}$, and configuration parameters including regularization weights $\lambda$.
- **Outputs**: A scalar regularization loss value $\mathcal{L}_{model}$ added to the total loss.

### Data Structures
The implementation typically uses a dictionary or object to store intermediate model outputs.
- `out`: A dictionary or object containing model predictions. Key fields include:
  - `uncertainty`: A tensor of shape $(N,)$ or $(B, N)$ containing predicted uncertainty values $\delta_{c_i}$ for each sample point $i$ along rays in batch $B$.
- `losses_model_reg`: A dictionary mapping regularization loss names (strings) to their computed scalar values.

### Steps
1. **Forward Pass**: Compute model predictions for the input rays.
2. **Extract Regularization Terms**: Call the model's regularization method to compute specific loss components.
   - For uncertainty regularization, compute $\mathcal{L}_0$ using the predicted uncertainty $\delta_{c_i}$.
3. **Aggregate Losses**: Iterate through the computed regularization losses.
4. **Weight and Sum**: Multiply each loss by its corresponding lambda weight from the configuration and sum them to get the total model regularization loss.

### Formulas
The primary model regularization loss cited is the **Zero Regularization Loss** ($\mathcal{L}_0$), designed to prevent predicted uncertainty $\delta_i$ from blowing up to large values.

$$
\mathcal{L}_0 = \sum_{r \in \mathcal{R}} \sum_{i=1}^N \max(\delta_{c_i}, 0)
$$

Where:
- $\mathcal{R}$ is the set of rays.
- $N$ is the number of sample points per ray.
- $c_i$ is the query coordinate of the $i$-th sample point on ray $r$.
- $\delta_{c_i}$ is the predicted uncertainty for query coordinate $c_i$.
- $\max(\delta_{c_i}, 0)$ ensures only positive uncertainty values contribute to the penalty, encouraging $\delta_i$ to be close to zero.

The total uncertainty loss $\mathcal{L}_{unct}$ combines this with a standard uncertainty loss $\mathcal{L}_{SE}$ (not detailed here) as:

$$
\mathcal{L}_{unct} = \alpha_1 \mathcal{L}_{SE} + \alpha_2 \mathcal{L}_0
$$

Typical weights are $\alpha_1 = 1.0$ and $\alpha_2 = 0.01$ [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_model_regularization(model, rays, config):
    """
    Computes model regularization losses.
    
    Args:
        model: The radiance field model instance.
        rays: Input rays for the current batch.
        config: Configuration object containing loss weights.
        
    Returns:
        total_reg_loss: Scalar tensor of the total regularization loss.
    """
    # Step 1: Forward pass and extraction of regularization terms
    # The model returns a dictionary/object 'out' containing internal predictions
    out = model(rays)
    
    # Step 2: Obtain regularization losses from the model
    # This extracts specific terms like uncertainty regularization
    losses_dict = model.regularizations(out)
    
    total_reg_loss = 0.0
    
    # Step 3 & 4: Iterate, weight, and sum
    for name, loss_value in losses_dict.items():
        # Get the lambda weight for this specific regularization term
        lambda_key = f"lambda_{name}"
        lambda_val = config.system.loss[lambda_key]
        
        # Add weighted loss to total
        weighted_loss = lambda_val * loss_value
        total_reg_loss += weighted_loss
        
        # Logging (optional, for debugging/tracking)
        # log_loss(name, weighted_loss)
        
    return total_reg_loss
```

### Edge Cases and Pitfalls
- **Uncertainty Explosion**: If $\delta_{c_i}$ becomes very large, $\mathcal{L}_0$ provides a linear penalty. Without this, uncertainty predictions might diverge, especially in shallow networks where predicting exact error $E(r)$ is difficult [S2].
- **Positive Only**: The use of $\max(\delta_{c_i}, 0)$ means negative uncertainty predictions (if allowed by the network) do not penalize the loss. This assumes uncertainty should be non-negative.
- **Weight Sensitivity**: The weight $\alpha_2$ (or $\lambda_{\mathcal{L}_0}$) is typically small (e.g., 0.01) relative to the primary loss. Too high a weight may suppress useful uncertainty estimates.

## Subtopics

### Zero Regularization Loss
A specific regularization term $\mathcal{L}_0$ that penalizes positive predicted uncertainty values to prevent them from growing excessively large during training. It is used in conjunction with other uncertainty losses to stabilize training of neural radiance fields, particularly when the network architecture makes direct error prediction difficult. It is chosen when uncertainty prediction is part of the model's output and stability is a concern [S2].

## Operating characteristics
- **Time Complexity**: $O(N \cdot B)$, where $N$ is samples per ray and $B$ is batch size, as it iterates over all sample points to compute the max operation.
- **Memory Complexity**: $O(N \cdot B)$ to store intermediate uncertainty values before reduction.
- **Strengths**: Simple to implement, computationally cheap, effectively prevents uncertainty blow-up.
- **Limitations**: Only addresses uncertainty magnitude, not spatial smoothness or geometric consistency. Does not replace photometric or geometric losses.
- **When to use**: When the model predicts uncertainty or other internal variables that can diverge.
- **When not to use**: When the model does not output such variables, or when other regularization (e.g., Total Variation) is more appropriate for the artifact being addressed.

## Background
The concept of model regularization in this context originates from efforts to stabilize the training of neural radiance fields, particularly those predicting uncertainty. The specific $\mathcal{L}_0$ loss was introduced in the context of Recursive-NeRF to handle the difficulty of predicting error $E(r)$ in shallow networks [S2]. The general framework of aggregating model regularization losses via a dictionary and lambda weights is commonly found in modern NeRF implementations like Instant-NSR-PL [S1, unofficial].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
