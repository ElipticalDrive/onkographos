# Entropy Model for Rate Estimation

Also called: Bitrate Prediction, Entropy Model, Entropy-Based Rate Estimation for Loss, Laplace PMF approximation, PMF Estimation, Rate Estimation Model, Rate loss estimation

## Overview
An entropy model used during training to estimate the entropy of feature grids, serving as a lower bound for the bitrate after compression, allowing the rate metric to be integrated into the loss function [S1, S2, S3]. It approximates the probability mass function (PMF) of quantized feature values using a cumulative distribution function (CDF) [S2, S3]. Unlike [Gaussian Probability Modeling](2122-gaussian-probability-modeling.md) which may assume fixed or generic distributions, this method constructs a distribution within the entropy model to closely match the actual data distribution of the 3D grids, often using a Laplace distribution with trainable parameters [S1, S3].

## How it works
**Inputs and Outputs:**
- **Input:** Quantized feature grid values $\hat{y}$ (or simulated noisy values $\tilde{y}$ during training) [S1, S3].
- **Output:** Estimated entropy (rate loss) $L_{rate}$ and PMF values $P(\hat{y})$ [S3].

**Data Structures:**
- **Laplace Distribution Parameters:** Two trainable scalar parameters per grid or channel:
  - $\mu$: Mean (typically initialized to 0) [S3].
  - $b$: Scale (typically initialized to 0.01) [S3].
- **Simulated Quantization:** A noise term $\delta$ drawn from a uniform distribution $U[-\frac{1}{2}, \frac{1}{2}]$ is added to continuous features $y$ to simulate quantization: $\tilde{y} = y + \delta$ [S3].

**Steps:**
1. **Simulate Quantization (Training Only):** To preserve gradient flow, replace the non-differentiable quantization operation $Q(y)$ with the addition of uniform noise $\delta \sim U[-\frac{1}{2}, \frac{1}{2}]$ to the continuous feature $y$, resulting in $\tilde{y}$ [S3].
2. **Define Distribution:** Assume the parameters of the 3D grid follow a Laplace distribution defined by mean $\mu$ and scale $b$ [S3]. The CDF of the Laplace distribution is used to model the probability [S3].
3. **Calculate PMF:** Approximate the PMF of the quantized value $\hat{y}$ by computing the difference of the CDF at $\hat{y} + \frac{1}{2}$ and $\hat{y} - \frac{1}{2}$ [S1, S2, S3].
   $$ P(\hat{y}) = P_{CDF}\left(\hat{y} + \frac{1}{2}\right) - P_{CDF}\left(\hat{y} - \frac{1}{2}\right) $$
   Where $P_{CDF}$ is the cumulative distribution function of the assumed distribution [S1, S3].
4. **Estimate Rate Loss:** Compute the rate loss $L_{rate}$ as the negative log-likelihood of the estimated PMF over all parameters $y_i$ in the feature set $F$ [S3]:
   $$ L_{rate} = \frac{1}{N} \sum_{y_i \in F} -\log_2 [P(\hat{y}_i)] $$
   Where $N$ is the number of parameters $y_i$ in the set $F$ [S3].
5. **Update Parameters:** Continuously update $\mu$ and $b$ during training via gradient descent to model the distribution of the grid parameters [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class LaplaceEntropyModel(nn.Module):
    def __init__(self, num_features):
        super().__init__()
        # Trainable parameters for Laplace distribution
        # Initialized as per [S3]: mu=0, b=0.01
        self.mu = nn.Parameter(torch.zeros(num_features))
        self.b = nn.Parameter(torch.full((num_features,), 0.01))

    def laplace_cdf(self, x, mu, b):
        """
        Cumulative Distribution Function for Laplace distribution.
        """
        diff = x - mu
        abs_diff = torch.abs(diff)
        sign_diff = torch.sign(diff)
        # CDF(x) = 0.5 * (1 + sign(x - mu) * (1 - exp(-|x - mu| / b)))
        cdf = 0.5 * (1 + sign_diff * (1 - torch.exp(-abs_diff / b)))
        return cdf

    def forward(self, y_continuous):
        """
        Estimates rate loss for continuous features y_continuous.
        Simulates quantization by adding uniform noise.
        """
        # Step 1: Simulate quantization [S3]
        # delta ~ U[-0.5, 0.5]
        delta = torch.rand_like(y_continuous) - 0.5
        y_simulated = y_continuous + delta

        # Step 2 & 3: Calculate PMF using CDF difference [S1, S3]
        # P(y_hat) = CDF(y_hat + 0.5) - CDF(y_hat - 0.5)
        cdf_plus = self.laplace_cdf(y_simulated + 0.5, self.mu, self.b)
        cdf_minus = self.laplace_cdf(y_simulated - 0.5, self.mu, self.b)
        
        pmf = cdf_plus - cdf_minus
        
        # Clamp PMF to avoid log(0) [G]
        pmf = torch.clamp(pmf, min=1e-10)

        # Step 4: Estimate Rate Loss [S3]
        # L_rate = -1/N * sum(log2(P(y_hat)))
        log_pmf = torch.log2(pmf)
        rate_loss = -torch.mean(log_pmf)
        
        return rate_loss
```

**Parameters:**
- $\mu$: Mean of the Laplace distribution. Initialized to 0 [S3].
- $b$: Scale of the Laplace distribution. Initialized to 0.01 [S3].
- $\lambda_1$: Weight for the rate loss in the total loss function. Typical values: 0.0001, 0.0005, 0.002, 0.005 [S3].
- $\lambda_2$: Weight for regularization loss. Typical value: 0.0001 [S3].

**Edge Cases and Pitfalls:**
- **Gradient Flow:** Standard entropy encoding (e.g., range coding) is non-differentiable, so the entropy model provides a differentiable surrogate to enable gradient-based optimization [S1, S2, S3].
- **Precision:** Refraining from presuming a preconceived data distribution and constructing a novel distribution to match actual data helps maintain precision [S1].
- **Quantization Simulation:** Using uniform noise instead of hard quantization allows gradients to flow through the "quantization" step [S3].

## Subtopics

### [Context Model Entropy Estimation](16628-context-model-entropy-estimation.md)
Context Model Entropy Estimation calculates the differentiable bit consumption for binary feature values based on their predicted probabilities, serving as a differentiable proxy for the bitrate in loss functions. It distinguishes itself from sibling methods by explicitly modeling the spatial context of 3D features to refine probability estimates, rather than relying on global occurrence frequencies.

### [Training Loss Function](16640-training-loss-function.md)
The total loss function integrates the rate estimation into the optimization process. For Laplace-based models, the total loss is $L_{total} = \sum_{r \in R} \|C(r) - \hat{C}(r)\|^2 + \lambda_1 L_{rate} + \lambda_2 L_{reg}$, where the first term is the L2 distortion loss, $L_{rate}$ is the estimated entropy loss, and $L_{reg}$ is regularization. For Instant-NGP-based NeRF compression, the approach optimizes compressed 3D embeddings by integrating an entropy-based rate distortion term to control bitrate during training.

### Rate-Distortion Optimization for 3DGS
A scene compression framework that balances distortion and model size through a rate-distortion objective, using a learned parametric entropy model for entropy coding [S4]. It minimizes $D_{\boldsymbol{\theta}}(\mathbf{x}, \hat{\mathbf{x}}) + \lambda R_{\boldsymbol{\theta}}(\mathbf{G})$, where $R_{\boldsymbol{\theta}}(\mathbf{G}) = E[-\log_2 p_{\hat{y}}(\hat{y})]$ is the Shannon cross-entropy between the quantized latent encoding $\hat{y}$ and the learned parametric entropy model $p_{\hat{y}}$ [S4]. The trade-off parameter $\lambda$ controls the balance between reconstruction quality and storage cost [S4]. A key pitfall is that storage size is not determined solely by the number of Gaussians but also by parameter redundancy and the probabilistic modeling used for entropy coding [S4].

## Operating characteristics
- **Time Complexity:** The entropy estimation adds a small computational overhead during training, primarily involving the evaluation of the CDF and log operations on the feature grid [S1, S3]. It avoids the expensive actual entropy encoding/decoding during training [S1, S3].
- **Memory Complexity:** Requires storing the trainable parameters $\mu$ and $b$ for the distribution model [G].
- **Performance:** The method effectively imposes a bitrate constraint during network updates by encouraging a lower-entropy distribution of features [S1, S2]. It allows for variable bitrates by tuning $\lambda_1$ [S3].
- **Strengths:** Differentiable rate estimation enables end-to-end optimization of compression-aware neural representations [S1, S3]. It avoids the non-differentiable nature of actual entropy coding [S1, S2].
- **Limitations:** The estimated entropy is a lower bound and may not exactly match the actual compressed bitrate [S1, S2]. The accuracy depends on how well the modeled distribution matches the actual data distribution [S1, S3].

## Background
The entropy model for rate estimation was introduced to address the non-differentiability of entropy encoding in the training of neural radiance fields and volumetric video representations [S1, S2, S3]. It builds on the theory that the theoretical compression upper bound is the symbol's entropy, allowing the estimated entropy to be used as a loss function to minimize bitrate [S3]. The approach was developed in the context of JointRF [S1], HPC [S2], and Efficient Dynamic-NeRF [S3], and later adapted for 3D Gaussian Splatting [S4].

## Sources
[S1] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S2] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
[S3] Efficient Dynamic-NeRF Based Volumetric Video Coding with Rate Distortion Optimization, https://doi.org/10.1109/icme57554.2024.10687892, fetched 2026-10-28
[S4] Drop-In Perceptual Optimization for 3D Gaussian Splatting, https://doi.org/10.1007/978-3-032-37350-2_25, fetched 2026-10-30
