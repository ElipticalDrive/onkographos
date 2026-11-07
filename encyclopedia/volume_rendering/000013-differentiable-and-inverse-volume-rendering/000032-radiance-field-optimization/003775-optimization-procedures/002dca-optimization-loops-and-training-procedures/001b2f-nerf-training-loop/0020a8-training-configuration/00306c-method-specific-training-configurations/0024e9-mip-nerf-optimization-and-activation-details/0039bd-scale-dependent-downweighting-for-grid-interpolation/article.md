# Scale-Dependent Downweighting for Grid Interpolation
Also called: Gaussian CDF Downweighting, Interpolation Weighting

## Overview
Scale-Dependent Downweighting is a technique to anti-alias trilinear interpolation in neural radiance fields by downweighting features interpolated from grid cells that are small relative to the sampling Gaussian's spread, preventing aliasing artifacts from high-frequency content [S1]. Unlike naive trilinear interpolation which causes aliasing at high frequencies, this method uses the Gaussian CDF to weight samples, similar to Mip-NeRF's IPE features but applied to grid-based representations [S1].

## How it works
**Inputs:**
- A set of multisampled isotropic Gaussians, each defined by a mean coordinate $\mathbf{x}_j$ and standard deviation $\sigma_j$ [S1].
- A set of feature grids $\{V_\ell\}$ at different resolution levels $\ell$, where each grid has a linear size (resolution) $n_\ell$ [S1].

**Outputs:**
- A concatenated feature vector provided as input to the subsequent MLP, consisting of downweighted interpolated features $\{\mathbf{f}_\ell\}$ and featurized downweighting factors $\{\omega_{j,\ell}\}$ [S1].

**Data Structures:**
- $V_\ell$: A 3D tensor (or hash grid) storing feature vectors. The specific dimensionality of the feature vector is $c_\ell$ [S1].
- $\mathbf{x}_j$: A 3D coordinate vector representing the mean of the $j$-th Gaussian [S1].
- $\sigma_j$: A scalar representing the standard deviation of the $j$-th Gaussian [S1].
- $n_\ell$: A scalar representing the linear size (resolution) of grid level $\ell$ [S1].

**Parameters:**
- $n_\ell$: The linear size (resolution) of the grid at level $\ell$. Used to scale the coordinate $\mathbf{x}$ before interpolation [S1].
- $\sigma_j$: The standard deviation of the isotropic Gaussian for multisample $j$. Represents the spatial extent of the sample [S1].

**Steps:**
1. For each multisample $j$ with mean $\mathbf{x}_j$ and standard deviation $\sigma_j$, and for each grid level $\ell$ with linear size $n_\ell$ and feature grid $V_\ell$:
    a. Compute the downweighting factor $\omega_{j,\ell}$ based on the fraction of the Gaussian's PDF inside the grid cell. The formula is:
       $$ \omega_{j,\ell} = \sqrt{\operatorname{erf}\left(\frac{1}{8\sigma_j^2 n_\ell^2}\right)} $$
       where $\omega_{j,\ell} \in [0, 1]$ [S1].
    b. Scale the coordinate $\mathbf{x}_j$ by the grid's linear size $n_\ell$ to get $\mathbf{x}'_j = n_\ell \cdot \mathbf{x}_j$ [S1].
    c. Perform trilinear interpolation on grid $V_\ell$ at coordinate $\mathbf{x}'_j$ to obtain the interpolated feature vector $\mathbf{f}_{j,\ell}$ [S1].
    d. Compute the downweighted feature for this sample and level as $\omega_{j,\ell} \cdot \mathbf{f}_{j,\ell}$ [S1].

2. Approximate the expected grid feature for level $\ell$ by taking the mean over all multisamples $j$:
   $$ \mathbf{f}_\ell = \operatorname{mean}_j\left( \omega_{j,\ell} \cdot \operatorname{trilerp}(n_\ell \cdot \mathbf{x}_j; V_\ell) \right) $$
   This approximation relies on the assumption that grid values are zero-mean and normally distributed due to weight decay, allowing the expected feature to be approximated as $\omega_{j,\ell} \cdot \mathbf{f}_{j,\ell} + (1 - \omega_{j,\ell}) \cdot 0 = \omega_{j,\ell} \cdot \mathbf{f}_{j,\ell}$ [S1].

3. Concatenate the downweighted features $\{\mathbf{f}_\ell\}$ from all levels and the featurized versions of $\{\omega_{j,\ell}\}$ to form the input for the subsequent MLP [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def scale_dependent_downweighting(x_means, sigmas, grids, grid_sizes):
    """
    Compute scale-dependent downweighted features for grid interpolation.
    
    Args:
        x_means: Array of shape (N, 3) containing mean coordinates for N multisamples.
        sigmas: Array of shape (N,) containing standard deviations for N multisamples.
        grids: List of 3D arrays (or hash grids) V_l for each level l.
        grid_sizes: List of scalars n_l for each level l.
        
    Returns:
        concatenated_features: Concatenated feature vector for the MLP input.
    """
    N = len(x_means)
    num_levels = len(grids)
    all_level_features = []
    all_weight_featurizations = []
    
    for l in range(num_levels):
        V_l = grids[l]
        n_l = grid_sizes[l]
        downweighted_features_sum = np.zeros_like(V_l[0,0,0]) # Assuming feature dim matches grid cell output
        count = 0
        
        for j in range(N):
            x_j = x_means[j]
            sigma_j = sigmas[j]
            
            # Step 1a: Compute downweighting factor
            # omega = sqrt(erf(1 / (8 * sigma^2 * n^2)))
            # Note: erf is the error function. 
            # The formula in source is sqrt(erf(1 / (8 * sigma_j^2 * n_l^2)))
            arg = 1.0 / (8.0 * (sigma_j ** 2) * (n_l ** 2))
            # Ensure arg is within valid range for erf if necessary, though erf handles large inputs
            omega = math.sqrt(math.erf(arg))
            
            # Step 1b & 1c: Scale coordinate and interpolate
            # trilerp takes scaled coordinate and grid
            # Assuming trilerp handles boundary conditions or coordinates are normalized [-0.5, 0.5] before scaling
            # or grid is indexed appropriately. Source implies standard trilinear interpolation.
            scaled_x = n_l * x_j
            f_j_l = trilinear_interpolation(scaled_x, V_l)
            
            # Step 1d: Downweight
            # f_j_l is a vector. omega is scalar.
            downweighted_f = omega * f_j_l
            
            # Accumulate for mean
            downweighted_features_sum += downweighted_f
            count += 1
            
        # Step 2: Mean over samples
        if count > 0:
            f_l = downweighted_features_sum / count
        else:
            f_l = np.zeros_like(V_l[0,0,0])
            
        all_level_features.append(f_l)
        
        # Step 3: Featurize weights (Source says "featurized versions", details in supplement)
        # Since supplement details are not provided, we concatenate the scalar omega directly or a simple encoding.
        # A common practice is to concatenate the scalar or a small set of powers.
        # Given the constraint to not guess, we note the concatenation.
        # We will concatenate the scalar omega for each sample as a placeholder for "featurized version"
        # However, the output must be self-contained. The source says "concatenate featurized versions of {omega_j,l}".
        # Without the supplement, we cannot define the exact featurization.
        # We will assume a simple concatenation of the scalar omega for now, or note it.
        # To be safe and implementable, we concatenate the scalar omega for each sample j.
        # This creates a vector of size N for this level.
        weight_vec = np.array([omega for j in range(N)]) # Re-compute omega for consistency
        all_weight_featurizations.append(weight_vec)

    # Concatenate all level features
    # Assuming all f_l have same dimension or are concatenated along feature axis
    # The source says "This set of features {f_l} is concatenated".
    # If dimensions differ, they are likely concatenated along the channel/feature dimension.
    concatenated_features = np.concatenate(all_level_features)
    
    # Concatenate weight featurizations
    # Source says "concatenated and provided as input".
    # We concatenate the weight vectors for all levels.
    concatenated_weights = np.concatenate(all_weight_featurizations)
    
    final_input = np.concatenate([concatenated_features, concatenated_weights])
    
    return final_input

def trilinear_interpolation(coord, grid):
    """
    Standard trilinear interpolation.
    coord: (3,) array, scaled coordinates.
    grid: 3D array.
    """
    # Implementation of trilinear interpolation
    # This is a standard operation.
    # We assume grid is defined on integer indices.
    # coord is in continuous space.
    
    # Get base indices
    x, y, z = coord
    ix, iy, iz = int(math.floor(x)), int(math.floor(y)), int(math.floor(z))
    
    # Get fractional parts
    fx, fy, fz = x - ix, y - iy, z - iz
    
    # Get grid dimensions
    Dx, Dy, Dz = grid.shape
    
    # Handle boundaries (clamp or wrap? Source doesn't specify, assume clamping or valid range)
    # For robustness, we clamp indices to [0, D-1]
    ix = np.clip(ix, 0, Dx - 2)
    iy = np.clip(iy, 0, Dy - 2)
    iz = np.clip(iz, 0, Dz - 2)
    
    # Get 8 neighbors
    c000 = grid[ix, iy, iz]
    c100 = grid[ix+1, iy, iz]
    c010 = grid[ix, iy+1, iz]
    c110 = grid[ix+1, iy+1, iz]
    c001 = grid[ix, iy, iz+1]
    c101 = grid[ix+1, iy, iz+1]
    c011 = grid[ix, iy+1, iz+1]
    c111 = grid[ix+1, iy+1, iz+1]
    
    # Interpolate x
    c00 = c000 * (1-fx) + c100 * fx
    c01 = c001 * (1-fx) + c101 * fx
    c10 = c010 * (1-fx) + c110 * fx
    c11 = c011 * (1-fx) + c111 * fx
    
    # Interpolate y
    c0 = c00 * (1-fy) + c10 * fy
    c1 = c01 * (1-fy) + c11 * fy
    
    # Interpolate z
    result = c0 * (1-fz) + c1 * fz
    
    return result
```

**Edge Cases and Pitfalls:**
- If the Gaussian is much larger than the grid cell, the interpolated feature is unreliable. The downweighting factor $\omega$ approaches 0, effectively discarding this unreliable sample [S1].
- The approximation relies on the assumption that grid values are zero-mean and normally distributed due to weight decay. If this assumption does not hold, the approximation $\omega_{j,\ell} \cdot \mathbf{f}_{j,\ell}$ may be inaccurate [S1].

## Operating characteristics
- **Time Complexity:** The method requires computing the error function and trilinear interpolation for each multisample at each grid level. This adds a constant factor overhead per sample per level compared to naive interpolation.
- **Memory Complexity:** No additional memory structures are required beyond the grids and multisample parameters. The output is a concatenated feature vector.
- **Strengths:** Effectively reduces spatial aliasing artifacts caused by high-frequency content when using multisampling with trilinear interpolation [S1].
- **Limitations:** Relies on the assumption of zero-mean, normally distributed grid values, which is encouraged by weight decay but may not always be perfectly met [S1].
- **When to use:** When training grid-based neural radiance fields (like iNGP or Zip-NeRF) with multisampling to prevent aliasing artifacts [S1].
- **When not to use:** When naive interpolation is sufficient (e.g., low resolution, no multisampling) or when the zero-mean assumption is violated and cannot be corrected by weight decay.

## Background
This technique is described in the context of Zip-NeRF, an anti-aliased grid-based neural radiance field method [S1]. It is part of the optimization and activation details for training mip-NeRF and JaxNeRF, designed to improve training stability and robustness [S1]. The method is inspired by Mip-NeRF's IPE features but adapted for grid-based representations [S1].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
