# Model Regularization

Also called: Residual Rotation Regularization, Rotation Regularization

## Overview
Model Regularization applies penalties to the raw parameters of a neural scene representation (such as residual rotations or spherical harmonics coefficients) to prevent instability and overfitting during optimization [S2]. It distinguishes itself from output-based regularizations (like [Distortion Loss](2755-distortion-loss.md)) by constraining the internal latent variables or decoder outputs directly, ensuring that the predicted geometric or appearance attributes remain within physically plausible or numerically stable bounds [S2].

## How it works
**Inputs and Outputs:**
- **Inputs:** The raw outputs from the neural network's decoder layers, including residual rotation vectors $\mathbf{r}_n$, spherical harmonics coefficients $c$, and scale parameters $s_n$ [S2].
- **Outputs:** A scalar regularization loss value $L_{reg}$ that is weighted by a hyperparameter $\lambda$ and added to the total training objective [S1, unofficial].

**Data Structures:**
- **Residual Rotations ($\mathbf{r}_n$):** A vector of size 6 representing the 6D rotation residual predicted by the decoder [S2].
- **SH Coefficients ($c$):** A vector of spherical harmonics coefficients for each Gaussian or voxel [S2].
- **Scales ($s_n$):** Log-space scale parameters predicted by the decoder [S2].

**Steps:**
1. **Retrieve Decoder Outputs:** Extract the raw parameters from the model's forward pass. In implementation frameworks, this is often done via a dedicated regularization hook that returns a dictionary of raw parameters [S1, unofficial].
2. **Compute Individual Loss Terms:** Calculate specific regularization losses based on the parameter type using the formulas below [S2].
3. **Aggregate and Weight:** Multiply each loss term by its corresponding weight (lambda) from the configuration dictionary and sum them into the total loss [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_model_regularization_loss(raw_params, config):
    # raw_params: dict with keys 'rotations' (N, 6), 'sh_coeffs' (N, C), 'scales' (N,)
    # config: dict with config.system.loss = {name: lambda_val}
    N = raw_params['rotations'].shape[0]
    losses = {}

    # 1. Rotation regularization: quadratic penalty on raw residual 6D rotations [S2]
    losses['rotation'] = mean(norm(raw_params['rotations'], axis=1)**2)

    # 2. SH soft-cap regularization: soft penalty for coefficients exceeding cap [S2]
    c = raw_params['sh_coeffs']
    c_max = config.get('c_max', 1.0) # Not given in sources [G]
    tau_SH = config.get('tau_SH', 1.0) # Not given in sources [G]
    p = config.get('p', 2)
    sh_term = softplus((abs(c) - c_max) / tau_SH) * tau_SH
    losses['sh_softcap'] = mean(sh_term**p)

    # 3. Scale regularization: quadratic penalty for log-scales exceeding max [S2]
    s = raw_params['scales']
    s_max = config.get('s_max', 1.0) # Not given in sources [G]
    losses['scale'] = mean(max(log(s) - log(s_max), 0)**2)

    # 4. Aggregate and weight per loss name from config [S1, unofficial]
    total_loss = 0.0
    for name, loss_val in losses.items():
        lambda_val = config.system.loss.get(f"lambda_{name}", 0.0)
        weighted = loss_val * lambda_val
        total_loss += weighted
        print(f"Logged {name}: {weighted}")

    return total_loss
```

**Formulas:**
- **Rotation Loss ($L_{rot}$):** Applies a quadratic penalty to raw residual 6D rotations $\mathbf{r}_n$ predicted by the decoder [S2].
  $$
  L_{rot} = \frac{1}{N} \sum_{n=1}^{N} \|\mathbf{r}_n\|_2^2
  $$
  where $N$ is the number of samples or Gaussians [S2].

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
- **$\lambda_{name}$:** Weight for each regularization term. Configurable per loss name via a dictionary lookup [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Scales are predicted in log-space [S2].

## Subtopics

### [L1 Norm Regularization for Transfer Functions](7273-l1-norm-regularization-for-transfer-functions.md)
Applies an L1 penalty to the opacity and color parameters of a transfer function to avoid unnecessary complexity. It addresses convergence to complex transfer functions with redundant color mappings. Choose this when interpretability of the transfer function is critical.

### [Neural Latent Regularization](10334-neural-latent-regularization.md)
Uses a small auxiliary neural network to reconstruct grid features from canonical coordinates, preventing overfitting of the radiance field without affecting inference speed. It regularizes the grid feature representation itself, ensuring spatial-temporal encoding remains smooth. Choose this for dynamic radiance fields where grid consistency is key.

### [L1 Regularization on Residual Grids](11655-l1-regularization-on-residual-grids.md)
Applies L1 regularization to residual grids ($R_t$) during dynamic radiance field optimization to enforce temporal continuity. It is a core component of JointRF, targeting temporal residual structure for compression and stability. Choose this for dynamic scene reconstruction.

### [MLP Output Regularization](16242-mlp-output-regularization.md)
Applies loss terms to the outputs of a neural network (e.g., deformation fields) to enforce physical plausibility, constraining translation/rotation residuals. It differs from weight decay by targeting semantic output vectors. Choose this for deformation fields or scene manipulators.

### Zero Regularization Loss
Encourages predicted uncertainty $\delta_i$ to be close to zero to prevent blowing up [S3].
$$
\sigma_0 = \sum_{r \in \sigma} \sum_{i=1}^N \max(\delta_{c_i}, 0)
$$
Used in uncertainty-aware NeRFs [S3].

### Scaffold-GS Volume Regularization
Volume regularization loss $L_{vol} = \sum_{i=1}^N \text{Prod}(s_i)$, weighted at 0.01, used in Scaffold-GS to control Gaussian volume [S4].

### Compactness Regularizer
Adds a compactness regularizer based on volume and opacity to the loss function, adopted from 3D-GS-MCMC [S5]. Improves PSNR by 0.05 dB and reduces Gaussian count by 2.5% [S5].

### SH Soft-Cap Regularization
Softly penalizes spherical harmonics coefficients exceeding a magnitude cap to avoid unstable appearance [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Prevents numerical instability (exploding scales/coefficients) [S2]. Improves PSNR by 0.05 dB and reduces Gaussian count by 2.5% [S5].
- **Limitations:** Requires tuning of caps ($c_{max}$) and weights ($\lambda$) [S2].
- **When to Use:** Not given in the sources.

## Background
Originated from techniques in 3D Gaussian Splatting and NeRF optimization [S2, S5]. Rotation regularization and SH soft-cap regularization are detailed in GlobalSplat [S2]. Uncertainty regularization ($L_0$) was introduced in Recursive-NeRF [S3]. Compactness regularization was adopted from 3D-GS-MCMC [S5]. Scaffold-GS uses volume regularization [S4]. The general framework of calling `model.regularizations()` is seen in instant-nsr-pl implementations [S1, unofficial].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] GlobalSplat: Efficient Feed-Forward 3D Gaussian Splatting via Global Scene Tokens, https://doi.org/10.1007/978-3-032-37595-7_10, fetched 2026-10-29
[S3] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S4] Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians, https://doi.org/10.1109/tpami.2025.3568201, fetched 2026-10-27
[S5] DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting, https://europepmc.org/article/PMC/PMC13705904, fetched 2026-10-27
