```python
    import math

    def compute_depth_variance_loss(samples, voxel_size):
        """
        Computes the depth variance regularization loss.
        
        Args:
            samples: List of lists/arrays. Each inner list contains dicts/arrays for samples along a ray.
                     Each sample i has:
                     - T_i: Transmittance before sample i
                     - sigma_i: Density at sample i
                     - delta_i: Distance of sample i
                     - t_i: Depth position of sample i
            voxel_size: float, the size of one voxel (delta_0).
            
        Returns:
            float: The depth variance loss value.
        """
        tau = 1.5 * voxel_size
        lambda_dv = 0.01
        total_loss = 0.0
        ray_count = 0
        
        for ray in samples:
            expected_depth = 0.0
            variance = 0.0
            
            # Compute weights and expected depth
            for sample in ray:
                weight = sample['T_i'] * (1.0 - math.exp(-sample['sigma_i'] * sample['delta_i']))
                expected_depth += weight * sample['t_i']
                
            # Compute variance
            for sample in ray:
                weight = sample['T_i'] * (1.0 - math.exp(-sample['sigma_i'] * sample['delta_i']))
                diff = expected_depth - sample['t_i']
                variance += weight * (diff ** 2)
                
            # Apply regularization based on threshold tau
            # Note: The exact penalty function applied when variance exceeds tau is not specified in the sources.
            if variance > tau:
                total_loss += lambda_dv * (variance - tau)
                ray_count += 1
                
        if ray_count > 0:
            return total_loss / ray_count
        return 0.0
    ```

    **Edge Cases and Pitfalls:**
    - **Precision:** Computing $1 - \exp(-\sigma_i \delta_i)$ can suffer from precision issues if $\sigma_i \delta_i$ is very small. Use `1 - math.expm1(-sigma_i * delta_i)` for better numerical stability [G].
    - **Boundaries:** If a ray has no valid samples (e.g., fully transparent), variance is undefined. Handle empty rays by skipping them or assigning zero loss [G].
    - **Degenerate Input:** If $\delta_0$ is not uniform, $\tau$ should be adapted per voxel or use an average voxel size [G].

    ## Subtopics

    ### Geometric Loss with Depth Variance Normalization
    A loss function that measures the absolute difference between predicted and true depths, normalized by the predicted depth variance to discourage weights with high uncertainty [S2]. It is defined as $L_{Geo} = \sum_{r \in R} \frac{|\hat{D}(r) - D(r)|}{\sqrt{\hat{D}_{var}(r)}}$, where $\hat{D}_{var}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) (\hat{D}(r) - t_i)^2$ is the depth variance of the image [S2]. This variant is chosen when ground truth depth is available and one wants to regularize the geometric consistency relative to the model's own uncertainty [S2]. It differs from standard Depth Variance Regularization by incorporating ground truth depth supervision [S2].

    ## Operating characteristics
    - **Time Complexity:** Not given in the sources [S1, S2].
    - **Memory Complexity:** Not given in the sources [S1, S2].
    - **Strengths:** Effectively prevents semi-opaque voxels and floating artifacts, leading to sharper surfaces [S1].
    - **Limitations:** Requires careful tuning of the threshold $\tau$ and weight $\lambda_{DV}$ [S1]. It does not provide geometric supervision if ground truth depth is unavailable (unlike the variant in Subtopics) [S1].
    - **When to use:** Use when training volume rendering models (e.g., VoxGRAF) and observing floating artifacts or semi-transparent noise [S1].
    - **When not to use:** Avoid if ground truth depth is available and a geometric loss with normalization is preferred for better alignment [S2].

    ## Background
    Not given in the sources.

    ## Sources
    [S1] VoxGRAF: Fast 3D-Aware Image Synthesis with Sparse Voxel Grids, https://doi.org/10.52202/068431-2464, fetched 2026-10-26
    [S2] 360FusionNeRF: Panoramic Neural Radiance Fields with Joint Guidance, https://doi.org/10.1109/iros55552.2023.10341346, fetched 2026-10-26
