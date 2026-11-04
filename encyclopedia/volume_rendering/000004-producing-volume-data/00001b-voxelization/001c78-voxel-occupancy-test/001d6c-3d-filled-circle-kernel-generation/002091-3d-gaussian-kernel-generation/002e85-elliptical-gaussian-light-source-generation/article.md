# Elliptical Gaussian Light Source Generation

Also called: Poisson Disk Light Sampling

## Overview
Generates spatially distributed light sources using either Poisson disk or uniform sampling, then constructs a luminance field by evaluating scaled multivariate Gaussian PDFs at query points [S1, unofficial]. It differs from its parent [3D Gaussian Kernel Generation](2297-3d-gaussian-kernel-generation.md) by focusing on the *placement* and *covariance orientation* of multiple distinct sources rather than a single smoothing kernel with a zeroed center.

## How it works
**Inputs:**
- `ls_type`: String, either `'poisson'` or `'uniform'`.
- `num_sources`: Integer, number of light sources to generate.
- `seed`: Integer, random seed.
- `query_points`: Array of shape `(N, 3)`, points at which to evaluate the luminance.

**Outputs:**
- `luminance`: Array of shape `(N,)`, the evaluated luminance values at `query_points`.

**Data Structures:**
- `source_locs`: Array of shape `(num_sources, 3)`, coordinates of light source centers.
- `covs`: Array of shape `(num_sources, 3, 3)`, covariance matrices for each source.
- `Us`: Array of shape `(num_sources, 3, 3)`, precomputed transformation matrices for efficient PDF evaluation.
- `logdets`: Array of shape `(num_sources,)`, log-determinants of the covariance matrices.

**Steps:**

1. **Sample Source Locations:**
   - If `ls_type == 'poisson'`:
     - Initialize random number generator with `seed`.
     - Create a Poisson disk sampler with dimension $d=3$, radius $r=0.13$, and the initialized RNG.
     - Sample `num_sources` points from the unit cube $[0,1]^3$.
   - If `ls_type == 'uniform'`:
     - Initialize JAX PRNGKey with `seed`.
     - Sample points uniformly in $[0,1]^3$ until `num_sources` are collected.
     - Stack the first `num_sources` points.

2. **Generate Covariances:**
   - Define base covariance matrix $C_{base}$:
     $$
     C_{base} = \begin{bmatrix} 2 & 0 & 0 \\ 0 & 0.5 & 0 \\ 0 & 0 & 0.5 \end{bmatrix} \times 10^{-3}
     $$
   - Rotate $C_{base}$ for each source to create `covs`. The rotation is determined by the `seed` and `num_sources` using a helper function `rotate_cov`. This results in `covs` of shape `(num_sources, 3, 3)`.

3. **Initialize Light Source Object:**
   - Pass `source_locs` and `covs` to the `Light_SourcesCov` class.
   - Inside `__init__`:
     - Compute eigenvalues $\lambda_i$ and eigenvectors $v_i$ of each covariance matrix in `covs` using symmetric eigendecomposition:
       $$
       \text{vals}, \text{vecs} = \text{eigh}(\text{covs})
       $$
       where `vals` has shape `(num_sources, 3)` and `vecs` has shape `(num_sources, 3, 3)`.
     - Compute inverse square roots of eigenvalues:
       $$
       \text{valsinvs}_i = \frac{1}{\lambda_i}
       $$
     - Compute transformation matrices $U_i$ for each source:
       $$
       U_i = V_i \cdot \text{diag}(\sqrt{\text{valsinvs}_i})
       $$
       In code, this is implemented as `self.Us = vecs * jnp.sqrt(valsinvs)[:, None]`.
     - Compute log-determinants:
       $$
       \log \det(C_i) = \sum_{j=1}^{3} \log(\lambda_{i,j})
       $$
       Stored in `self.logdets`.

4. **Evaluate Luminance:**
   - For each query point $x$, compute the luminance contribution from all sources. The multivariate Gaussian PDF for source $i$ is:
     $$
     \mathcal{N}(x; \mu_i, C_i) = \frac{1}{\sqrt{(2\pi)^3 \det(C_i)}} \exp\left( -\frac{1}{2} (x - \mu_i)^T C_i^{-1} (x - \mu_i) \right)
     $$
   - Using the precomputed $U_i$ where $C_i^{-1} = U_i^T U_i$, the exponent becomes:
     $$
     -\frac{1}{2} \| U_i (x - \mu_i) \|^2
     $$
   - Sum contributions from all sources (or select based on application logic, e.g., nearest neighbor or all).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
import jax.random as random
from jax import lax

def rotate_cov(cov, num_sources, seed):
    """
    Generates rotated covariance matrices for num_sources sources.
    Implementation detail: Not given in the sources. 
    Assumes a standard rotation generation method using the seed.
    """
    # Placeholder for actual rotation logic
    # Typically involves generating random rotation matrices and applying them
    # to the base covariance.
    raise NotImplementedError("rotate_cov implementation not provided in source")

class Light_SourcesCov:
    def __init__(self, source_locs, covs):
        self.source_locs = source_locs
        self.covs = covs
        
        # Eigen decomposition
        vals, vecs = jnp.linalg.eigh(self.covs)
        
        # Inverse square roots of eigenvalues
        valsinvs = 1.0 / vals
        
        # Compute Us: vecs * sqrt(valsinvs)[:, None]
        # vals shape: (num_sources, 3)
        # vecs shape: (num_sources, 3, 3)
        # valsinvs shape: (num_sources, 3)
        # sqrt(valsinvs)[:, None] shape: (num_sources, 3, 1)
        self.Us = vecs * jnp.sqrt(valsinvs)[:, None]
        
        # Log determinants
        self.logdets = jnp.sum(jnp.log(vals), axis=1)

    def evaluate(self, query_points):
        """
        Evaluate luminance at query_points.
        query_points shape: (N, 3)
        Returns: (N,)
        """
        # source_locs shape: (num_sources, 3)
        # Us shape: (num_sources, 3, 3)
        # logdets shape: (num_sources,)
        
        # Calculate differences: (N, 1, 3) - (1, num_sources, 3) -> (N, num_sources, 3)
        diffs = query_points[:, None, :] - self.source_locs[None, :, :]
        
        # Transform differences: (N, num_sources, 3) @ (num_sources, 3, 3)^T
        # Note: Us[i] is such that C_i^{-1} = Us[i]^T @ Us[i]
        # So we want (x - mu)^T C^{-1} (x - mu) = || Us @ (x - mu) ||^2
        # Transformed diffs: (N, num_sources, 3)
        transformed = jnp.einsum('ijk,ijk->ij', diffs, jnp.einsum('ijk,ikl->ijl', self.Us, diffs))
        
        # Exponent: -0.5 * sum(transformed^2) ? 
        # Actually, transformed above is already the dot product if we did it right.
        # Let's re-verify: 
        # y = Us @ diff -> (N, num_sources, 3)
        # norm_sq = sum(y^2, axis=-1) -> (N, num_sources)
        
        y = jnp.einsum('ijk,ilk->ijl', self.Us, diffs) # (N, num_sources, 3)
        norm_sq = jnp.sum(y**2, axis=-1) # (N, num_sources)
        
        # PDF normalization constant: 1 / sqrt((2pi)^3 * det(C))
        # log_norm = -1.5 * log(2pi) - 0.5 * log(det(C))
        log_norm = -1.5 * jnp.log(2 * jnp.pi) - 0.5 * self.logdets # (num_sources,)
        
        # Log PDF: -0.5 * norm_sq + log_norm
        log_pdf = -0.5 * norm_sq + log_norm # (N, num_sources)
        
        # Sum over sources (assuming additive luminance)
        # Use log-sum-exp for numerical stability if needed, but simple sum is fine if values are small
        pdf = jnp.exp(log_pdf) # (N, num_sources)
        luminance = jnp.sum(pdf, axis=1) # (N,)
        
        return luminance

def generate_light_sources(ls_type, num_sources, seed, query_points):
    if ls_type == 'poisson':
        # Initialize RNG
        rng = random.PRNGKey(seed)
        # Note: qmc.PoissonDisk is not standard JAX. 
        # Assuming a wrapper or external library. 
        # For pure JAX implementation, this step is not directly available.
        # [G] Placeholder for Poisson Disk sampling logic.
        source_locs = None # Placeholder
    elif ls_type == 'uniform':
        rng = random.PRNGKey(seed)
        # Sample until num_sources are collected
        # [G] Standard rejection sampling or direct sampling from uniform distribution
        # Since we just need num_sources points in [0,1]^3:
        source_locs = random.uniform(rng, shape=(num_sources, 3), minval=0.0, maxval=1.0)
    else:
        raise ValueError("ls_type must be 'poisson' or 'uniform'")

    # Base covariance
    cov_base = jnp.array([[[2, 0, 0], [0, 0.5, 0], [0, 0, 0.5]]]) * 1e-3
    
    # Rotate covariances
    covs = rotate_cov(cov_base, num_sources, seed)
    
    # Initialize Light Source Object
    light_sources = Light_SourcesCov(source_locs, covs)
    
    # Evaluate
    luminance = light_sources.evaluate(query_points)
    
    return luminance
```

**Edge Cases and Pitfalls:**
- **Precision:** Eigenvalue decomposition of near-singular covariance matrices can be unstable. The base covariance is well-conditioned, but rotated versions should be checked.
- **Boundaries:** Poisson disk sampling in a bounded cube may result in fewer points if the radius is large relative to the domain. The source code samples 1000 points for Poisson, implying a fixed size or a rejection-based approach that might not yield exactly `num_sources` if not handled carefully. The uniform case explicitly stacks the first `num_sources`.
- **Degenerate Input:** If `num_sources` is 0, the method should return an empty array or zero luminance.

## Operating characteristics
- **Time Complexity:** $O(N \cdot S \cdot d^2)$ for evaluation, where $N$ is the number of query points, $S$ is the number of sources, and $d=3$ is the dimension. Eigen decomposition is $O(S \cdot d^3)$.
- **Memory Complexity:** $O(S \cdot d^2)$ to store covariances and precomputed matrices.
- **Strengths:** Poisson disk sampling provides more uniform spatial distribution than uniform sampling, reducing clustering artifacts. Precomputing $U$ and log-determinants speeds up repeated evaluations.
- **Limitations:** Evaluation cost scales linearly with the number of sources. For large $S$, this can be expensive.
- **When to use:** When a smooth, spatially distributed light field is needed, and uniformity of source placement is important.
- **When not to use:** When real-time performance is critical and $S$ is large, or when sources are highly clustered (use a different sampling strategy).

## Background
Originated in the context of neural radiance fields (NeRF) and volume rendering optimization, specifically in the `rtnf` repository [S1, unofficial]. It is used to generate initial light source distributions for training neural networks to represent light fields.

## Sources
[S1] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
