# Entropy Model for Rate Estimation

Also called: Bitrate Prediction, Entropy Model, PMF Estimation, Rate Estimation Model

## Overview
The Entropy Model for Rate Estimation is a differentiable surrogate used during the training of compressed volume representations to estimate the compression bitrate [S1, S2]. It serves as a lower bound for the actual bitrate after entropy encoding, allowing the rate metric to be integrated directly into the loss function [S1, S2]. Unlike standard Gaussian Probability Modeling which assumes a fixed distribution, this method constructs a novel distribution within the model to closely match the actual data distribution of the 3D feature grids, rather than presuming a preconceived data distribution [S1, S2].

## How it works
The method estimates the probability mass function (PMF) of quantized feature values to compute the entropy, which approximates the bit consumption [S1, S2].

**Inputs and Outputs:**
- Input: Quantized feature values $\hat{y}$ from the feature grid [S1, S2].
- Input: Cumulative Distribution Function (CDF) parameters predicted by the network or derived from the data distribution [S1, S2].
- Output: Estimated entropy (rate) for the feature grid [S1, S2].

**Data Structures:**
- The CDF is typically parameterized by the network to predict local statistics (e.g., mean and scale) for each bin or feature channel, though specific parameterization details (e.g., number of bins, network architecture) are not given in the sources [S1, S2].
- Quantized values $\hat{y}$ are integers resulting from the quantization step $Q$ [S1, S2].

**Steps:**
1. **Quantization:** The continuous feature values $x$ are quantized to produce discrete values $\hat{y}$. The sources describe the quantization process generally as $Q(\cdot)$, often involving scaling and rounding [S1, S2]. Specifically, JointRF uses $Q(q \cdot x - q \cdot \min(x))$ [S1].
2. **CDF Evaluation:** Evaluate the Cumulative Distribution Function $P_{CDF}$ at the boundaries of the quantization bin for the quantized value $\hat{y}$ [S1, S2].
3. **PMF Calculation:** Compute the probability mass for $\hat{y}$ by taking the difference of the CDF values at the upper and lower boundaries of the bin [S1, S2].
4. **Entropy Calculation:** Compute the entropy $H$ as the negative expected log-probability: $H = -\sum_{\hat{y}} P(\hat{y}) \log_2 P(\hat{y})$ [G]. This entropy serves as the rate estimate in the loss function [S1, S2].

**Formulas:**
The core formula for the PMF of a quantized feature $\hat{y}$ is [S1, S2]:

$$
P(\hat{y}) = P_{CDF}\left(\hat{y} + \frac{1}{2}\right) - P_{CDF}\left(\hat{y} - \frac{1}{2}\right)
$$

Where:
- $P(\hat{y})$ is the probability mass function (PMF) of the quantized feature $\hat{y}$ [S1, S2].
- $P_{CDF}$ is the cumulative distribution function [S1, S2].
- $\hat{y}$ is the quantized value [S1, S2].

**Parameters:**
- The specific parameters of the CDF (e.g., mean $\mu$, scale $\sigma$) are not explicitly defined in the provided notes, but are implied to be learned or predicted to match the actual data distribution [S1, S2].
- Quantization step size $q$ is used in JointRF [S1].

**Edge Cases and Pitfalls:**
- **Gradient Flow:** Entropy encoding (e.g., range coding) is non-differentiable and disrupts gradient flow [S1, S2]. The entropy model is used *during training* to provide a differentiable approximation, while actual compression uses entropy encoding (e.g., range coder) during inference/encoding [S1, S2].
- **Distribution Mismatch:** Presuming a preconceived data distribution may lead to precision loss. The method constructs a novel distribution to closely match the actual data distribution [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_rate(features, cdf_params):
    """
    Estimate the bitrate (entropy) of quantized features using an entropy model.
    
    Args:
        features: Tensor of continuous feature values.
        cdf_params: Parameters for the CDF (e.g., mean, scale).
        
    Returns:
        estimated_rate: Scalar tensor representing the estimated bitrate.
    """
    # Step 1: Quantization
    # Note: Exact quantization formula varies by implementation (e.g., JointRF uses scaling and min-offset).
    # Here we assume a generic quantization Q.
    # [G] Assuming standard rounding quantization for demonstration.
    # In JointRF: q * x - q * min(x), then round.
    quantized_features = torch.round(features) 
    
    # Step 2 & 3: PMF Calculation
    # P(y_hat) = P_CDF(y_hat + 0.5) - P_CDF(y_hat - 0.5)
    # [G] Assuming cdf_params are used to evaluate a specific CDF (e.g., Gaussian).
    # The sources do not specify the exact CDF function or how cdf_params are used.
    # We assume a function evaluate_cdf(y, params) exists.
    
    upper_bound = quantized_features + 0.5
    lower_bound = quantized_features - 0.5
    
    cdf_upper = evaluate_cdf(upper_bound, cdf_params)
    cdf_lower = evaluate_cdf(lower_bound, cdf_params)
    
    pmf = cdf_upper - cdf_lower
    
    # Step 4: Entropy Calculation
    # H = - sum(P * log2(P))
    # [G] Add small epsilon to avoid log(0)
    epsilon = 1e-10
    pmf_clamped = torch.clamp(pmf, min=epsilon)
    
    entropy_per_element = -pmf_clamped * torch.log2(pmf_clamped)
    
    # Sum over all elements to get total estimated rate
    estimated_rate = torch.sum(entropy_per_element)
    
    return estimated_rate

def evaluate_cdf(y, cdf_params):
    """
    Evaluate the CDF at point y given parameters.
    [G] Placeholder for specific CDF implementation (e.g., Gaussian CDF).
    The sources do not specify the exact CDF form.
    """
    # Example: Gaussian CDF
    # mu, sigma = cdf_params
    # return 0.5 * (1 + torch.erf((y - mu) / (sigma * math.sqrt(2))))
    raise NotImplementedError("Specific CDF implementation not provided in sources.")
```

## Subtopics

### Gaussian Probability Modeling
Assumes a Gaussian distribution for feature values. Simpler but may not match actual data distribution as well as the novel distribution constructed in the Entropy Model for Rate Estimation. Choose when data is known to be Gaussian or for simplicity. [Link](2122-gaussian-probability-modeling.md)

## Operating characteristics
- **Time Complexity**: Depends on the complexity of the CDF evaluation and the number of feature elements. Generally linear with respect to the number of quantized feature values [S1, S2].
- **Memory Complexity**: Requires storing CDF parameters and intermediate CDF values. Not given in the sources [S1, S2].
- **Measured Performance**: Not given in the sources [S1, S2].
- **Scaling**: The method scales with the size of the feature grid [S1, S2].
- **Strengths**: Provides a differentiable estimate of bitrate, enabling end-to-end optimization of compression and quality. Constructs a distribution to match actual data, potentially improving accuracy over fixed distributions [S1, S2].
- **Limitations**: Requires careful construction of the distribution to match actual data. The actual compression still uses non-differentiable entropy encoding [S1, S2].
- **When to use**: When training compressed volume representations and needing to optimize for bitrate directly in the loss function [S1, S2].
- **When not to use**: When the data distribution is known to be well-modeled by a simpler distribution (e.g., Gaussian) and computational efficiency is critical during training [G].

## Background
The method originates from research on end-to-end joint optimization for dynamic neural radiance field representation and compression [S1] and hierarchical progressive coding for volumetric video [S2]. It addresses the challenge of integrating bitrate constraints into the training of neural radiance fields and volumetric video representations, where entropy encoding is non-differentiable [S1, S2].

## Sources
[S1] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S2] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
