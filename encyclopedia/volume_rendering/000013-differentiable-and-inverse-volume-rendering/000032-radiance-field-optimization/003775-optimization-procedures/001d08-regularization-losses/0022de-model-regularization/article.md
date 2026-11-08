# Model Regularization

Also called: Model Regularization Loss

## Overview
Model regularization applies explicit constraints to the parameters of the radiance field representation (such as density, color, or uncertainty) to prevent overfitting, ensure physical plausibility, or stabilize optimization. Unlike sibling methods that regularize the output image or geometric consistency, this method operates directly on the scene parameters during the backward pass. It complements the standard emission-absorption rendering equation by adding penalty terms to the total loss function [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- **Input:** The current state of the model parameters (e.g., density $\sigma$, step size $\delta$, uncertainty $\delta_c$) and the current batch of rays.
- **Output:** A scalar regularization loss value $\mathcal{L}_{reg}$ added to the total training loss.

**Data Structures:**
- The regularization loss is typically computed as a dictionary of named losses, where each key corresponds to a specific regularization term (e.g., "l1", "zero") and the value is the computed scalar loss [S1, unofficial].

**Steps:**
1. **Compute Regularization Terms:** Evaluate the specific regularization function(s) defined for the model parameters. For example, if regularizing uncertainty $\delta$, compute the penalty for non-zero values [S2].
2. **Aggregate Losses:** Sum the individual regularization terms, optionally weighted by hyperparameters $\lambda$ [S1, unofficial].
3. **Backpropagate:** The gradient of the regularization loss is computed with respect to the model parameters and added to the gradient accumulator.

**Formulas:**
The general form of the total loss is:
$$
\mathcal{L}_{total} = \mathcal{L}_{rendering} + \sum_{k} \lambda_k \mathcal{L}_{reg, k}
$$
where $\mathcal{L}_{rendering}$ is the photometric error and $\lambda_k$ are weights for each regularization term [S1, unofficial].

A specific example is the **Zero Regularization Loss** for uncertainty $\delta_{c_i}$, which encourages uncertainty to be close to zero to prevent it from "blowing up" during optimization [S2]:
$$
\mathcal{L}_0 = \sum_{r \in \mathcal{R}} \sum_{i=1}^N \max(\delta_{c_i}, 0)
$$
where $\mathcal{R}$ is the set of rays, $N$ is the number of samples per ray, and $\delta_{c_i}$ is the predicted uncertainty for the query coordinate $c_i$ [S2].

**Parameters:**
- $\alpha_1, \alpha_2$: Weights for the uncertainty loss components. In [S2], $\alpha_1 = 1.0$ and $\alpha_2 = 0.01$.
- $\lambda_{name}$: Weights for each regularization term, configured in the system (e.g., `self.config.system.loss[f"lambda_{name}"]`) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The $\max(\delta_{c_i}, 0)$ operation ensures that only positive uncertainties contribute to the loss, preventing negative values from reducing the loss [S2].
- **Stability:** Using regularization instead of direct L1 loss for uncertainty prediction can be easier for shallow networks to optimize, as predicting exact uncertainty is difficult [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_model_regularization(model, rays):
    """
    Computes model regularization losses.
    Args:
        model: The radiance field model instance.
        rays: The batch of rays being processed.
    Returns:
        dict: A dictionary of regularization losses.
    """
    losses = {}
    
    # Example: Zero regularization for uncertainty
    # Assuming model has a method or attribute for uncertainty delta
    if hasattr(model, 'uncertainty'):
        delta = model.uncertainty
        # L0 regularization: encourage delta to be close to zero
        # max(delta, 0) ensures only positive values penalize
        l0_loss = torch.sum(torch.maximum(delta, torch.tensor(0.0)))
        losses['zero_reg'] = l0_loss
        
    # Additional regularization terms can be added here
    # e.g., L1 on density, etc.
    
    return losses

def apply_regularization_losses(losses, config):
    """
    Applies weights to regularization losses and adds to total loss.
    Args:
        losses: Dictionary of regularization losses.
        config: Configuration object containing lambda weights.
    Returns:
        float: The total weighted regularization loss.
    """
    total_reg_loss = torch.tensor(0.0)
    for name, value in losses.items():
        # Get lambda weight from config, default to 0 if not present
        lambda_key = f"lambda_{name}"
        lambda_val = config.system.loss.get(lambda_key, 0.0)
        total_reg_loss += lambda_val * value
    return total_reg_loss
```

## Subtopics

### Zero Regularization Loss
A specific regularization term applied to uncertainty parameters ($\delta$) to prevent them from growing excessively large during training. It uses the L0 norm (sum of absolute values, approximated by max with 0) to penalize non-zero uncertainty [S2]. This is particularly useful when the network struggles to predict accurate uncertainty directly, as it provides a simpler optimization target [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples, as it involves a simple element-wise operation and summation.
- **Memory Complexity:** $O(1)$ additional memory, as it does not require storing intermediate states beyond the current parameter values.
- **Strengths:** Stabilizes optimization of parameters like uncertainty; prevents overfitting to noise; computationally inexpensive.
- **Limitations:** Requires careful tuning of weights ($\lambda$); may oversimplify the representation if too strong.
- **When to use:** When optimizing parameters that are prone to instability (e.g., uncertainty) or when a simpler representation is desired.

## Background
Model regularization is a general technique in machine learning applied to neural radiance fields (NeRFs) to improve training stability and generalization. The specific formulation of zero regularization for uncertainty is introduced in Recursive-NeRF [S2]. The integration of multiple regularization losses into a unified training loop is exemplified in instant-nsr-pl [S1, unofficial].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
