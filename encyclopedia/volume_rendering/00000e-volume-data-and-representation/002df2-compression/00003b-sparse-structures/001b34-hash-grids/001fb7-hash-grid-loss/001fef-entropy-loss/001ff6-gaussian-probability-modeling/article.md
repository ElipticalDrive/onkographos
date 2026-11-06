# Gaussian Probability Modeling

Also called: Context model Gaussian modeling

## Overview
Gaussian Probability Modeling is a technique used within entropy loss calculations for sparse volume representations, where quantized anchor attributes are modeled as Gaussian distributions to estimate bit consumption [S1, unofficial]. It differs from uniform or discrete entropy models by leveraging the Gaussian-like distribution of neural network features, allowing the context model to output adaptive mean and variance parameters for each attribute component [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Quantized anchor attributes $\hat{\mathbf{f}}_i$ for $i=1 \dots N^{\text{at}}$, where $N^{\text{at}}$ is the number of anchors [S1, unofficial].
*   **Output:** Probability density estimates $p(\hat{\mathbf{f}}_i)$ derived from Gaussian parameters [S1, unofficial].

**Data Structures:**
The context model outputs two scalar values per attribute component $i$:
1.  Mean $\mu_i$
2.  Variance $\sigma_i$

**Steps:**
1.  **Quantization:** Original anchor attributes are quantized using a predefined step $Q$. The quantization step $Q$ varies by component due to different numerical scales. Typical values cited are $Q=1$ for attributes $\mathbf{f}_a$, $Q=0.001$ for $\mathbf{l}_s$ and $\mathbf{l}_r$, and $Q=0.2$ for $\Delta \mathbf{x}_g$ [S1, unofficial]. During training, the quantized attribute $\hat{\mathbf{f}}_i$ substitutes the original to achieve quantization-aware training [S1, unofficial].
2.  **Parameter Generation:** The context model processes the data to output the mean $\mu_i$ and variance $\sigma_i$ for each quantized attribute $\hat{\mathbf{f}}_i$ [S1, unofficial].
3.  **Probability Estimation:** The probability $p(\hat{\mathbf{f}}_i)$ is formulated using the Gaussian distribution $\mathcal{N}(\mu_i, \sigma_i)$ [S1, unofficial].
4.  **Entropy Loss Calculation:** The total entropy loss is computed as the sum of negative log-probabilities:
    $$
    L_{\text{entropy}} = \sum_{\mathbf{f} \in \{\mathbf{A}\}} \sum_{i=1}^{N^{\text{at}}} \left( -\log_2 p(\hat{\mathbf{f}}_i) \right)
    $$
    where $\mathbf{A}$ represents the set of attributes and $N^{\text{at}}$ is the number of anchors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_probability_modeling(attributes, context_model, quantization_steps):
    """
    Computes entropy loss by modeling quantized attributes as Gaussian distributions.
    
    Args:
        attributes: List of original anchor attribute tensors.
        context_model: Function returning (mean, variance) for given inputs.
        quantization_steps: Dict mapping attribute index to quantization step Q.
        
    Returns:
        entropy_loss: Scalar tensor representing the bit consumption estimate.
    """
    total_loss = 0.0
    num_anchors = len(attributes)
    
    for i in range(num_anchors):
        # Step 1: Quantize the attribute
        attr = attributes[i]
        Q = quantization_steps.get(i, 1.0) # Default Q=1 if not specified
        # Quantization-aware substitution: round to nearest multiple of Q
        # Note: Exact quantization function (e.g., round(x/Q)*Q) is implied by [S1, unofficial]
        quantized_attr = torch.round(attr / Q) * Q
        
        # Step 2: Get Gaussian parameters from context model
        # The context model outputs mean mu_i and variance sigma_i
        mu_i, sigma_i = context_model(quantized_attr)
        
        # Step 3: Compute probability density p(f_hat_i)
        # Gaussian PDF: p(x) = (1 / sqrt(2 * pi * sigma^2)) * exp(-(x - mu)^2 / (2 * sigma^2))
        # In practice, for entropy coding, we often use the log-probability directly
        # or approximate the discrete probability mass.
        # Standard Gaussian log-probability:
        log_prob = -0.5 * torch.log(2 * torch.pi * sigma_i**2) - \
                   (quantized_attr - mu_i)**2 / (2 * sigma_i**2)
        
        # Step 4: Accumulate entropy loss (-log2(p))
        # log2(p) = log(p) / log(2)
        loss_component = -log_prob / torch.log(torch.tensor(2.0))
        total_loss += loss_component
        
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Variance Zero:** If $\sigma_i$ approaches zero, numerical instability occurs in the log-probability calculation. A small epsilon should be added to $\sigma_i$ [G].
*   **Quantization Step Selection:** Incorrect $Q$ values can lead to poor probability modeling if the quantization bin size does not align with the distribution's scale [S1, unofficial].

## Subtopics

### Quantized Gaussian Sampling
A 3D normal distribution quantized to integer coordinates to provide a prior about where the output is expected and where the loss is defined, centered at the middle of the voxel grid [S2]. It is used to guide training by sampling loss according to these quantized coordinates, effectively masking out non-empty voxels with low density that are not visible in rendering [S2].

## Operating characteristics
*   **Complexity:** Linear with respect to the number of anchors $N^{\text{at}}$ and attributes [S1, unofficial].
*   **Performance:** Enables efficient compression by adapting the probability model to the specific statistical properties of each attribute component [S1, unofficial].
*   **Limitations:** Requires a context model capable of accurately predicting mean and variance; performance depends heavily on correct quantization step selection [S1, unofficial].

## Background
Originated in the context of compressive neural Gaussian splats for volume visualization, specifically in the ECoNGS method [S1, unofficial]. It addresses the need for efficient entropy coding of sparse 3D representations by leveraging the Gaussian nature of quantized features [S1, unofficial].

## Sources
[S1] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
[S2] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
