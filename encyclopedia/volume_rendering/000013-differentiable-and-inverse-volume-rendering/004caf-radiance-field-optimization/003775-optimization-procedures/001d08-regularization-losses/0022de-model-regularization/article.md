# Model Regularization

Also called: Residual Rotation Regularization, Rotation Regularization

## Overview
Model Regularization applies penalties to the raw parameters of a neural scene representation (such as residual rotations or spherical harmonics coefficients) to prevent instability and overfitting during optimization [S2]. It distinguishes itself from output-based regularizations (like [Distortion Loss](2755-distortion-loss.md)) by constraining the internal latent variables or decoder outputs directly, ensuring that the predicted geometric or appearance attributes remain within physically plausible or numerically stable bounds [S2].

## How it works
**Inputs and Outputs:**
- **Inputs:** The raw outputs from the neural network's decoder layers (e.g., residual rotation vectors $\mathbf{r}_n$, spherical harmonics coefficients $c$, or scale parameters $s_n$).
- **Outputs:** A scalar regularization loss value $L_{reg}$ added to the total training objective.

**Data Structures:**
- **Residual Rotations ($\mathbf{r}_n$):** A vector of size 6 (for 6D rotation representation) or similar, predicted by the decoder.
- **SH Coefficients ($c$):** A vector of spherical harmonics coefficients for each Gaussian or voxel.
- **Scales ($s_n$):** Log-space scale parameters.

**Steps:**
1. **Retrieve Decoder Outputs:** Extract the raw parameters from the model's forward pass. In implementation frameworks, this is often done via a dedicated regularization hook:
   ```python
   # Pseudocode for retrieving regularization terms
   # Assumes 'model' has a method that returns a dict of raw parameters
   raw_params = self.model.regularizations(out) 
   ```
2. **Compute Individual Loss Terms:** Calculate specific regularization losses based on the parameter type.
   - **Rotation Regularization:** Compute the L2 norm squared of the residual rotation vectors [S2].
   - **SH Soft-Cap Regularization:** Compute the soft penalty for SH coefficients exceeding a magnitude cap [S2].
   - **Scale Regularization:** Compute the penalty for scales exceeding a maximum log-value [S2].
3. **Aggregate and Weight:** Multiply each loss term by its corresponding weight (lambda) from the configuration and sum them into the total loss [S1, unofficial].
   ```python
   total_loss = 0.0
   for name, loss_value in losses_model_reg.items():
       # Get lambda from config, default to 0 if not specified
       lambda_val = self.config.system.loss.get(f"lambda_{name}", 0.0)
       weighted_loss = loss_value * lambda_val
       total_loss += weighted_loss
       # Logging step
       log_loss(name, weighted_loss)
   ```

**Formulas:**
- **Rotation Loss ($L_{rot}$):** Applies a quadratic penalty to raw residual 6D rotations $\mathbf{r}_n$ predicted by the decoder [S2].
  $$
  L_{rot} = \frac{1}{N} \sum_{n=1}^{N} \|\mathbf{r}_n\|_2^2
  $$
  where $N$ is the number of samples/Gaussians [S2].

- **SH Soft-Cap Loss ($L_{SH}$):** Softly penalizes spherical harmonics coefficients $c$ whose magnitude exceeds a prescribed cap $c_{max}$ [S2].
  $$
  L_{SH} = \mathbb{E}\left[ \left( \operatorname{softplus}\left( \frac{|c| - c_{max}}{\tau_{SH}} \right) \tau_{SH} \right)^p \right]
  $$
  where $\tau_{SH}$ is the temperature parameter for the softplus function, and $p$ is the power (typically 2) [S2].

- **Scale Regularization ($L_{scale}$):** Penalizes scales $s_n$ that exceed a predefined maximum $s_{max}$ in log-space [S2].
  $$
  L_{scale} = \frac{1}{N} \sum_{n=1}^{N} \max(\log s_n - \log s_{max}, 0)^2
  $$

**Parameters:**
- **$c_{max}$:** Prescribed cap for SH coefficient magnitude. Not given in the sources.
- **$\tau_{SH}$:** Temperature parameter for the softplus function. Not given in the sources.
- **$s_{max}$:** Maximum allowed scale. Not given in the sources.
- **$\lambda_{name}$:** Weight for each regularization term. Configurable per loss name [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Regularization:** If no regularization is applied, the model may suffer from unstable appearance coefficients or exploding scales [S2].
- **Precision:** Scales are predicted in log-space to handle large dynamic ranges; ensure numerical stability when computing $\log s_n$ [S2].

## Subtopics

### [L1 Norm Regularization for Transfer Functions](7273-l1-norm-regularization-for-transfer-functions.md)
Applies an L1 penalty to the opacity and color parameters of a transfer function to avoid unnecessary complexity. It addresses convergence to complex transfer functions with redundant color mappings. Choose this when interpretability of the transfer function is critical. [L1 Norm Regularization for Transfer Functions](7273-l1-norm-regularization-for-transfer-functions.md)

### [Neural Latent Regularization](10334-neural-latent-regularization.md)
Uses a small auxiliary neural network to reconstruct grid features from canonical coordinates, preventing overfitting of the radiance field without affecting inference speed. It regularizes the grid feature representation itself, ensuring spatial-temporal encoding remains smooth. Choose this for dynamic radiance fields where grid consistency is key. [Neural Latent Regularization](10334-neural-latent-regularization.md)

### [L1 Regularization on Residual Grids](11655-l1-regularization-on-residual-grids.md)
Applies L1 regularization to residual grids ($R_t$) during dynamic radiance field optimization to enforce temporal continuity. It is a core component of JointRF, targeting temporal residual structure for compression and stability. Choose this for dynamic scene reconstruction. [L1 Regularization on Residual Grids](11655-l1-regularization-on-residual-grids.md)

### [MLP Output Regularization](16242-mlp-output-regularization.md)
Applies loss terms to the outputs of a neural network (e.g., deformation fields) to enforce physical plausibility, constraining translation/rotation residuals. It differs from weight decay by targeting semantic output vectors. Choose this for deformation fields or scene manipulators. [MLP Output Regularization](16242-mlp-output-regularization.md)

### Zero Regularization Loss
Encourages predicted uncertainty $\delta_i$ to be close to zero to prevent blowing up [S3].
$$
\sigma_0 = \sum_{r \in \sigma} \sum_{i=1}^N \max(\delta_{c_i}, 0)
$$
Used in uncertainty-aware NeRFs [S3]. [Zero Regularization Loss](#)

### Scaffold-GS Volume Regularization
Volume regularization loss $L_{vol} = \sum_{i=1}^N \text{Prod}(s_i)$, weighted at 0.01, used in Scaffold-GS to control Gaussian volume [S4]. [Scaffold-GS Volume Regularization](#)

### Compactness Regularizer
Adds a compactness regularizer based on volume and opacity to the loss function, adopted from 3D-GS-MCMC [S5]. Improves PSNR by 0.05 dB and reduces Gaussian count by 2.5% [S5]. [Compactness Regularizer](#)

### SH Soft-Cap Regularization
Softly penalizes spherical harmonics coefficients exceeding a magnitude cap to avoid unstable appearance [S2]. [SH Soft-Cap Regularization](#)

## Operating characteristics
- **Time Complexity:** $O(N)$ per regularization term, where $N$ is the number of samples or Gaussians. The overhead is minimal as it operates on decoder outputs.
- **Memory Complexity:** $O(1)$ additional memory, as it does not require storing intermediate gradients for the regularization terms beyond standard AD.
- **Strengths:** Prevents numerical instability (exploding scales/coefficients), encourages simpler models (L1), and improves generalization by constraining latent space [S2, S5].
- **Limitations:** Requires careful tuning of weights ($\lambda$) and caps ($c_{max}$); overly strong regularization may bias the model towards simpler solutions that miss fine details [S2].
- **When to Use:** When training neural radiance fields or Gaussian splatting models with unstable convergence, large dynamic ranges in parameters, or when interpretability of latent parameters is desired.

## Background
Originated from techniques in 3D Gaussian Splatting and NeRF optimization to stabilize training. Rotation regularization and SH soft-cap regularization are explicitly detailed in GlobalSplat [S2]. Uncertainty regularization ($L_0$) was introduced in Recursive-NeRF [S3]. Compactness regularization was adopted from 3D-GS-MCMC [S5]. Scaffold-GS uses volume regularization [S4]. The general framework of calling `model.regularizations()` is seen in instant-nsr-pl implementations [S1, unofficial].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] GlobalSplat: Efficient Feed-Forward 3D Gaussian Splatting via Global Scene Tokens, https://doi.org/10.1007/978-3-032-37595-7_10, fetched 2026-10-29
[S3] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S4] Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians, https://doi.org/10.1109/tpami.2025.3568201, fetched 2026-10-27
[S5] DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting, https://europepmc.org/article/PMC/PMC13705904, fetched 2026-10-27
