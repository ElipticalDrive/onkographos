# Progressive Multi-Pass Fitting

Also called: Progressive fitting, Residual fitting, Single-Pass and Multi-Pass Residual Fitting

## Overview
Progressive multi-pass fitting distributes a nominal splat budget across sequential passes, with each pass optimizing the residual left by frozen earlier passes [S1, unofficial]. It serves as an alternative to single-pass fitting within the Regularised Gaussian Fitting Loss framework [3104-regularised-gaussian-fitting-loss.md], trading marginal PSNR gains for improved SSIM on specific complex structures [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A 3D volume dataset, a nominal initial splat count ($N_{\text{seed}}$), a total iteration budget ($I_{\text{total}}$), and the number of passes ($P$) [S1, unofficial].
- **Output**: A set of optimized Gaussian splats [S1, unofficial].

**Data Structures**
- **Splat State**: Each splat stores position, covariance, opacity, and color attributes [G]. During multi-pass fitting, parameters of splats from previous passes are frozen (gradient updates disabled) in subsequent passes [S1, unofficial].
- **Residual Volume**: Implicitly computed as the difference between the target volume intensity and the current rendered image from frozen splats [S1, unofficial].

**Parameters**
- $N_{\text{seed}} = 32,000$: Nominal initial splat count [S1, unofficial].
- $I_{\text{total}} = 10,000$: Total optimization steps [S1, unofficial].
- $P \in \{1, 2, \dots, 8\}$: Number of passes. Single-pass corresponds to $P=1$ [S1, unofficial].
- **Iteration Split**: Iterations are distributed equally across passes [S1, unofficial]. For $P$ passes, iterations per pass is $I_{\text{pass}} = \lfloor I_{\text{total}} / P \rfloor$ [S1, unofficial].

**Algorithm**
1. **Initialization**: Generate $N_{\text{seed}}$ initial splats [S1, unofficial]. The default seeder draws 60% of seeds from edges (using Sobel gradients with normalized magnitude $> 0.1$, sampled via gradient-weighted Poisson-disk sampling with a 2-voxel exclusion radius) and 40% from a uniform grid [S1, unofficial].
2. **Loop Passes** ($p = 1$ to $P$):
   a. **Freeze Previous**: If $p > 1$, freeze the parameters (position, covariance, opacity, color) of all splats optimized in passes $1$ to $p-1$ [S1, unofficial].
   b. **Optimize**: Run optimization for $I_{\text{pass}}$ iterations [S1, unofficial]. The loss function is the Regularised Gaussian Fitting Loss [3104-regularised-gaussian-fitting-loss.md], which combines L1 reconstruction loss with regularization terms for splat amplitude and extent [S1, unofficial]. The optimization targets the residual error relative to the current frozen state [S1, unofficial].
   c. **Cull**: Apply post-fit culling to remove low-contribution splats [S1, unofficial].
3. **Finalize**: Return the union of all splats from all passes [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_multi_pass_fitting(volume, seed_count=32000, total_iterations=10000, num_passes=2):
    # Step 1: Initialization
    # Default seeder: 60% edge seeds (Sobel gradient > 0.1, gradient-weighted Poisson-disk, 2-voxel exclusion), 40% uniform grid [S1, unofficial]
    splats = generate_seeds(volume, seed_count)  # [G] Standard seeder implementation
    
    iterations_per_pass = total_iterations // num_passes
    
    for p in range(1, num_passes + 1):
        # Step 2a: Freeze previous splats
        if p > 1:
            # Freeze parameters (position, covariance, opacity, color) of splats from passes 1 to p-1 [S1, unofficial]
            for i in range(len(splats)):
                if splats[i].pass_index < p:
                    splats[i].requires_grad = False  # [G] Disable gradient updates
        
        # Step 2b: Optimize current pass
        # Run optimization for I_pass iterations targeting residual error [S1, unofficial]
        # Loss function: Regularised Gaussian Fitting Loss (L1 reconstruction + amplitude/extent regularization) [3104-regularised-gaussian-fitting-loss.md]
        for _ in range(iterations_per_pass):
            # [G] Render current splats to compute residual
            residual = volume - render_splats(splats, volume)
            # [G] Compute loss and gradients, then update parameters
            loss = compute_loss(residual, splats)
            gradients = compute_gradients(loss, splats)
            update_parameters(splats, gradients)
            
        # Step 2c: Cull
        # Apply post-fit culling to remove low-contribution splats [S1, unofficial]
        # [G] Exact culling threshold and criteria are not specified in the sources
        splats = [s for s in splats if s.contribution > threshold]
        
    return splats
```

**Edge Cases and Pitfalls**
- **Precision**: The smallest PSNR margin (Fly brain neurons, 0.07 dB) is below the largest run-to-run spread (0.093 dB), indicating that differences between single-pass and 2-pass may be statistically insignificant for some volumes [S1, unofficial].
- **Boundaries**: Four or more passes trail single-pass on every volume, suggesting diminishing returns or overfitting with excessive passes [S1, unofficial].

## Subtopics

### Single-Pass Fitting
Optimizes all splats in one pass. Yields higher training PSNR on 15 of 17 volumes by 0.07–6.02 dB over 2-pass. Choose when PSNR is the primary metric and wall time is comparable. [Single-Pass Fitting](3104-regularised-gaussian-fitting-loss.md)

### Multi-Pass Fitting
General term for $P > 1$. Used when SSIM is prioritized or for specific complex structures (e.g., Mouse heart nuclei +0.40 dB, Zebrafish embryo nuclei +0.85 dB). [Multi-Pass Fitting](3104-regularised-gaussian-fitting-loss.md)

## Operating characteristics
- **Time Complexity**: Wall time for 1-pass vs 2-pass is comparable, with a ratio of 0.70–1.75× (median 1.07×) on an otherwise idle GPU [S1, unofficial]. Multi-pass overhead increases with $P$ [S1, unofficial].
- **Memory Complexity**: Not given in the sources [S1, unofficial].
- **Measured Performance**:
  - **PSNR**: Single-pass wins on 15 of 17 volumes by 0.07–6.02 dB over 2-pass. 2-pass wins on Mouse heart nuclei (+0.40 dB) and Zebrafish embryo nuclei (+0.85 dB) [S1, unofficial].
  - **SSIM**: 2-pass yields higher SSIM on 12 of 17 volumes [S1, unofficial].
  - **Error**: Single-pass has lower maximum absolute error on 12 of 17 volumes [S1, unofficial].
- **Scaling**: Four or more passes trail single-pass on every volume in PSNR [S1, unofficial].
- **Strengths**: Can improve SSIM and handle specific complex structures better than single-pass [S1, unofficial].
- **Limitations**: Generally lower PSNR than single-pass. Higher wall time for marginal gains [S1, unofficial].
- **When to Use**: When SSIM is more important than PSNR, or for specific volumes like Mouse heart nuclei [S1, unofficial].
- **When Not to Use**: When PSNR is the primary metric or when computational efficiency is critical [S1, unofficial].

## Background
Progressive multi-pass fitting was evaluated in the context of Luxar, a system for Gaussian splatting for microscopy and scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. The method was compared against single-pass fitting across 17 volumes [S1, unofficial]. The study found that while single-pass is generally superior in PSNR, multi-pass (specifically 2-pass) offers advantages in SSIM and specific structural fidelity [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
