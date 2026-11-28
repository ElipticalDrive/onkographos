# Rigid Registration via Mutual Information Maximization

Also called: LPS to RAS transform mapping, Local Rigid Registration with Mattes Mutual Information, MI-based rigid registration, Rigid registration

## Overview
This method performs 3D/3D monomodal intra-subject rigid registration of medical volumes (specifically renal MRI), assuming the organ remains rigid (displaced but not deformed) during the acquisition window [S1, unofficial]. It solves the problem of aligning a moving volume to a fixed reference by maximizing Mutual Information (MI) as the similarity measure, optimized via Downhill Simplex [S1, unofficial] or RegularStepGradientDescent [S2, unofficial]. It is distinguished from sibling methods like [Correlation](52-name.md) by its reliance on joint probability distributions and entropy-based formulations rather than linear intensity assumptions, making it robust for non-linear intensity relationships [S2, unofficial]. It builds on [Geometric Moment Initialization](20995-geometric-moment-initialization.md) to provide initial parameters that avoid local extrema [S1, unofficial].

## How it works
**Inputs:**
1.  **Fixed Volume ($V_f$):** Reference 3D volume.
2.  **Moving Volume ($V_m$):** Target 3D volume to be aligned to $V_f$.
3.  **Initial Transform ($T_0$):** 6-parameter rigid transform (3 translation, 3 rotation) initialized by [Geometric Moment Initialization](20995-geometric-moment-initialization.md) [S1, unofficial].

**Outputs:**
1.  **Optimized Transform ($T_{opt}$):** Rigid transformation matrix mapping $V_m$ to $V_f$ space.
2.  **Resampled Volume ($V_{resampled}$):** $V_m$ resampled onto the grid of $V_f$ using $T_{opt}$ [S2, unofficial].

**Data Structures:**
*   **Joint Histogram ($H$):** 2D histogram storing counts of voxel intensity pairs $(I_f, I_m)$.
    *   Layout: 2D array of size $N_{bins} \times N_{bins}$.
    *   $N_{bins}$: Number of bins per axis (typically 48 for Mattes MI [S2, unofficial] or 64 for standard MI [S1, unofficial]).
    *   Values: Integer counts of overlapping voxel pairs.
*   **Transform Parameters ($\theta$):** Vector of 6 floats $[t_x, t_y, t_z, r_x, r_y, r_z]$ [S1, unofficial].

**Steps:**

1.  **Input Validation:**
    *   Verify $V_f$ and $V_m$ are 3D volumes [S2, unofficial].
    *   Verify minimum dimension $\ge 12$ [S2, unofficial].
    *   Verify all voxel values are finite ($\infty, -\infty, NaN$ are invalid) [S2, unofficial].
    *   Verify standard deviation of voxel values $> 10^{-6}$ [S2, unofficial].
    *   If validation fails, abort registration [S2, unofficial].

2.  **Initialization:**
    *   Compute initial transform $T_0$ using geometric moments [S1, unofficial].
    *   Cast volumes to `float32` precision [S2, unofficial].

3.  **Similarity Metric Calculation (Mutual Information):**
    *   Apply current transform $T$ to $V_m$ to define the spatial overlap region with $V_f$ [S1, unofficial].
    *   Extract intensity pairs $(I_f, I_m)$ for all voxels in the overlap region.
    *   **Sampling:** To reduce computational cost, sample a fraction of voxels (e.g., 0.12% or 0.0012) using a fixed seed (e.g., 42) [S2, unofficial].
    *   **Histogram Construction:**
        *   Normalize intensity ranges to $[0, 1]$ or use fixed bin edges.
        *   Populate 2D joint histogram $H$ of size $B \times B$ (where $B$ is bins per axis) [S1, unofficial].
        *   Use partial volume assignment if intensity falls between bin centers [S1, unofficial].
    *   **Probability Estimation:**
        *   Compute joint probability $P(i, j) = H(i, j) / \sum_{k,l} H(k, l)$ [S1, unofficial].
        *   Compute marginal probabilities:
            $$ P_f(i) = \sum_j P(i, j) $$
            $$ P_m(j) = \sum_i P(i, j) $$
    *   **MI Calculation:**
        $$ MI = \sum_{i,j} P(i, j) \log \left( \frac{P(i, j)}{P_f(i) P_m(j)} \right) $$
        *   Handle $\log(0)$ by ignoring terms where $P(i, j) = 0$ [S1, unofficial].

4.  **Optimization:**
    *   **Method A (Downhill Simplex):**
        *   Use the initial parameters $\theta_0$ from geometric moments [S1, unofficial].
        *   Minimize $-MI$ (or maximize MI) using the Nelder-Mead simplex algorithm [S1, unofficial].
        *   This method is robust to local minima but requires good initialization [S1, unofficial].
    *   **Method B (RegularStepGradientDescent):**
        *   Initialize optimizer with $\theta_0$ [S2, unofficial].
        *   Parameters:
            *   Learning Rate: 2.0 [S2, unofficial].
            *   Minimum Step: 0.015 [S2, unofficial].
            *   Max Iterations: 180 per level [S2, unofficial].
            *   Relaxation Factor: 0.5 [S2, unofficial].
            *   Gradient Tolerance: $10^{-6}$ [S2, unofficial].
        *   Scale updates from physical units [S2, unofficial].

5.  **Multi-Resolution Strategy (Optional but Recommended):**
    *   Create a Gaussian pyramid for $V_f$ and $V_m$ [S2, unofficial].
    *   Levels: 3 levels with shrink factors $[4, 2, 1]$ and smoothing sigmas $[2, 1, 0]$ (physical units) [S2, unofficial].
    *   Register at coarsest level (shrink 4) using $T_0$.
    *   Pass optimized transform to next level (shrink 2), then finest (shrink 1) [S2, unofficial].

6.  **Coordinate System Conversion (If applicable):**
    *   If using ITK (LPS coordinate system) and outputting to NIfTI (RAS coordinate system):
        *   Convert the fixed-to-moving transform matrix $M_{ITK}$ to $M_{NIfTI}$ [S2, unofficial].
        *   Apply flip to diagonal: $M_{NIfTI} = M_{ITK} \times \text{diag}([-1, -1, 1, 1])$ [S2, unofficial].
        *   Adjust origin and axis transformation accordingly [S2, unofficial].

7.  **Result Validation:**
    *   Verify $T_{opt}$ and final MI value are finite [S2, unofficial].
    *   Resample a binary mask of ones from $V_m$ to $V_f$ space using nearest-neighbor interpolation [S2, unofficial].
    *   Compute mean overlap between resampled mask and $V_f$'s valid region.
    *   If overlap $< 0.25$, raise error (registration failed due to insufficient overlap) [S2, unofficial].

8.  **Resampling:**
    *   Resample $V_m$ to $V_f$ grid using linear interpolation and $T_{opt}$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rigid_registration_mi(fixed_vol, moving_vol, initial_transform, 
                          num_bins=48, sampling_percentage=0.0012, 
                          seed=42, optimizer='gradient_descent'):
    """
    Rigid registration using Mutual Information maximization.
    
    Args:
        fixed_vol: 3D numpy array, reference volume.
        moving_vol: 3D numpy array, volume to align.
        initial_transform: 4x4 numpy array, initial rigid transform (moving -> fixed).
        num_bins: int, number of bins for joint histogram.
        sampling_percentage: float, fraction of voxels to sample for MI calculation.
        seed: int, random seed for sampling.
        optimizer: str, 'simplex' or 'gradient_descent'.
        
    Returns:
        optimized_transform: 4x4 numpy array, final rigid transform.
        resampled_vol: 3D numpy array, moving_vol resampled to fixed_vol grid.
    """
    
    # 1. Input Validation
    if fixed_vol.ndim != 3 or moving_vol.ndim != 3:
        raise ValueError("Volumes must be 3D")
    if min(fixed_vol.shape) < 12 or min(moving_vol.shape) < 12:
        raise ValueError("Minimum dimension must be >= 12")
    if not np.isfinite(fixed_vol).all() or not np.isfinite(moving_vol).all():
        raise ValueError("All voxel values must be finite")
    if np.std(fixed_vol) < 1e-6 or np.std(moving_vol) < 1e-6:
        raise ValueError("Standard deviation must be > 1e-6")

    # 2. Cast to float32
    fixed_vol = fixed_vol.astype(np.float32)
    moving_vol = moving_vol.astype(np.float32)

    # 3. Define MI Metric Function
    def compute_mi(transform_matrix, fixed, moving, bins, sample_frac, rng):
        # Apply transform to moving volume coordinates to find overlap
        # Note: In practice, this involves resampling or coordinate mapping
        # For simplicity, we assume a helper 'resample_volume' exists
        transformed_moving = resample_volume(moving, transform_matrix, fixed.shape, mode='linear')
        
        # Get overlapping voxels
        # Mask of valid voxels in fixed volume (non-zero or non-NaN)
        valid_mask = np.isfinite(fixed) & np.isfinite(transformed_moving)
        
        if not np.any(valid_mask):
            return -np.inf # No overlap
        
        # Sample voxels
        num_samples = int(np.sum(valid_mask) * sample_frac)
        if num_samples == 0:
            return -np.inf
            
        indices = np.where(valid_mask)
        flat_indices = np.ravel_multi_index(indices, fixed.shape)
        rng.shuffle(flat_indices)
        selected_indices = flat_indices[:num_samples]
        
        # Extract intensities
        f_vals = fixed.flat[selected_indices]
        m_vals = transformed_moving.flat[selected_indices]
        
        # Build Joint Histogram
        # Normalize intensities to [0, 1] for binning
        f_min, f_max = np.min(f_vals), np.max(f_vals)
        m_min, m_max = np.min(m_vals), np.max(m_vals)
        
        # Avoid division by zero in normalization
        if f_max == f_min: f_max = f_min + 1e-6
        if m_max == m_min: m_max = m_min + 1e-6
        
        f_norm = (f_vals - f_min) / (f_max - f_min)
        m_norm = (m_vals - m_min) / (m_max - m_min)
        
        # Bin indices
        f_bins = np.clip((f_norm * (bins - 1)).astype(int), 0, bins - 1)
        m_bins = np.clip((m_norm * (bins - 1)).astype(int), 0, bins - 1)
        
        # Count occurrences
        joint_hist = np.zeros((bins, bins), dtype=np.int32)
        for i, j in zip(f_bins, m_bins):
            joint_hist[i, j] += 1
            
        # Normalize to probability
        total = np.sum(joint_hist)
        if total == 0:
            return -np.inf
        p_joint = joint_hist / total
        
        # Marginals
        p_f = np.sum(p_joint, axis=1)
        p_m = np.sum(p_joint, axis=0)
        
        # Compute MI
        mi = 0.0
        for i in range(bins):
            for j in range(bins):
                if p_joint[i, j] > 0 and p_f[i] > 0 and p_m[j] > 0:
                    mi += p_joint[i, j] * np.log(p_joint[i, j] / (p_f[i] * p_m[j]))
                    
        return mi

    # 4. Optimization
    rng = np.random.default_rng(seed)
    
    if optimizer == 'simplex':
        # Downhill Simplex (Nelder-Mead)
        # Flatten transform to 6 parameters (3 trans, 3 rot)
        # Note: Implementing full Nelder-Mead from scratch is complex; 
        # using scipy.optimize.minimize with method='Nelder-Mead' is standard.
        # Here we outline the structure.
        
        def objective(params):
            # Reconstruct transform from params
            # params = [tx, ty, tz, rx, ry, rz]
            T = euler_to_matrix(params[:3], params[3:]) # Helper function
            # Combine with initial transform
            T_combined = T @ initial_transform
            # Maximize MI -> Minimize -MI
            return -compute_mi(T_combined, fixed_vol, moving_vol, num_bins, sampling_percentage, rng)
            
        # Initial guess from geometric moments
        initial_params = geometric_moment_initialization(fixed_vol, moving_vol)
        
        result = minimize(objective, initial_params, method='Nelder-Mead')
        optimized_params = result.x
        
    elif optimizer == 'gradient_descent':
        # RegularStepGradientDescent
        # This requires computing gradient of MI w.r.t transform parameters.
        # Approximate gradient via finite differences or analytical derivative.
        # For brevity, we assume a gradient descent loop.
        
        current_params = geometric_moment_initialization(fixed_vol, moving_vol)
        learning_rate = 2.0
        min_step = 0.015
        max_iter = 180
        relaxation = 0.5
        tol = 1e-6
        
        best_mi = -np.inf
        best_params = current_params
        
        for level_shrink in [4, 2, 1]:
            # Downsample volumes for multi-resolution
            f_res = downsample(fixed_vol, level_shrink)
            m_res = downsample(moving_vol, level_shrink)
            
            # Adjust initial transform for shrinkage if necessary
            # ...
            
            for i in range(max_iter):
                # Compute gradient
                grad = compute_mi_gradient(current_params, f_res, m_res, num_bins, sampling_percentage, rng)
                
                # Update parameters
                step = learning_rate * grad
                if np.linalg.norm(step) < min_step:
                    break
                    
                current_params = current_params - step * relaxation
                
                # Evaluate MI
                T = euler_to_matrix(current_params[:3], current_params[3:])
                T_combined = T @ initial_transform
                current_mi = compute_mi(T_combined, f_res, m_res, num_bins, sampling_percentage, rng)
                
                if current_mi > best_mi:
                    best_mi = current_mi
                    best_params = current_params
                    
                if np.linalg.norm(grad) < tol:
                    break
                    
        optimized_params = best_params

    # 5. Final Transform Construction
    optimized_transform = euler_to_matrix(optimized_params[:3], optimized_params[3:]) @ initial_transform
    
    # 6. Validation
    # Resample mask to check overlap
    mask_moving = np.ones_like(moving_vol)
    resampled_mask = resample_volume(mask_moving, optimized_transform, fixed_vol.shape, mode='nearest')
    overlap = np.mean(resampled_mask * np.isfinite(fixed_vol))
    
    if overlap < 0.25:
        raise RuntimeError("Insufficient overlap (< 0.25). Registration failed.")
        
    # 7. Resample Moving Volume
    resampled_vol = resample_volume(moving_vol, optimized_transform, fixed_vol.shape, mode='linear')
    
    return optimized_transform, resampled_vol

def euler_to_matrix(tx, ty, tz, rx, ry, rz):
    """Convert Euler angles (rad) and translation to 4x4 matrix."""
    # Rx
    Rx = np.array([
        [1, 0, 0, 0],
        [0, np.cos(rx), -np.sin(rx), 0],
        [0, np.sin(rx), np.cos(rx), 0],
        [0, 0, 0, 1]
    ])
    # Ry
    Ry = np.array([
        [np.cos(ry), 0, np.sin(ry), 0],
        [0, 1, 0, 0],
        [-np.sin(ry), 0, np.cos(ry), 0],
        [0, 0, 0, 1]
    ])
    # Rz
    Rz = np.array([
        [np.cos(rz), -np.sin(rz), 0, 0],
        [np.sin(rz), np.cos(rz), 0, 0],
        [0, 0, 1, 0],
        [0, 0, 0, 1]
    ])
    # Translation
    T = np.array([
        [1, 0, 0, tx],
        [0, 1, 0, ty],
        [0, 0, 1, tz],
        [0, 0, 0, 1]
    ])
    return T @ Rz @ Ry @ Rx

def resample_volume(volume, transform_matrix, output_shape, mode='linear'):
    """
    Resample volume using transform_matrix (4x4) to output_shape.
    Assumes transform maps output coordinates to input coordinates.
    """
    # Create grid for output space
    x = np.arange(output_shape[0])
    y = np.arange(output_shape[1])
    z = np.arange(output_shape[2])
    X, Y, Z = np.meshgrid(x, y, z, indexing='ij')
    
    # Stack coordinates
    coords = np.stack([X.ravel(), Y.ravel(), Z.ravel(), np.ones_like(X.ravel())], axis=1)
    
    # Apply inverse transform to map output -> input
    # Note: transform_matrix is usually moving->fixed. 
    # We need fixed->moving to sample.
    inv_transform = np.linalg.inv(transform_matrix)
    input_coords = (inv_transform @ coords.T).T
    
    # Extract x, y, z
    ix = input_coords[:, 0]
    iy = input_coords[:, 1]
    iz = input_coords[:, 2]
    
    # Clip to bounds
    ix = np.clip(ix, 0, volume.shape[0] - 1)
    iy = np.clip(iy, 0, volume.shape[1] - 1)
    iz = np.clip(iz, 0, volume.shape[2] - 1)
    
    if mode == 'nearest':
        # Round to nearest integer
        ix = np.round(ix).astype(int)
        iy = np.round(iy).astype(int)
        iz = np.round(iz).astype(int)
        resampled = volume[ix, iy, iz]
    elif mode == 'linear':
        # Trilinear interpolation
        # Implementation omitted for brevity, but standard in graphics libraries
        resampled = trilinear_interpolate(volume, ix, iy, iz)
    else:
        raise ValueError("Mode must be 'nearest' or 'linear'")
        
    return resampled.reshape(output_shape)

def trilinear_interpolate(volume, x, y, z):
    # Placeholder for actual trilinear interpolation logic
    # [G] Standard trilinear interpolation: interpolate along x, then y, then z
    # using floor and ceil coordinates and weights.
    pass

def geometric_moment_initialization(fixed, moving):
    """
    Placeholder for Geometric Moment Initialization.
    Returns [tx, ty, tz, rx, ry, rz].
    """
    # [G] Compute centroids and second-order moments of both volumes.
    # Solve for rigid transform that aligns moments.
    pass

def compute_mi_gradient(params, fixed, moving, bins, sample_frac, rng):
    """
    Compute gradient of MI w.r.t. transform parameters.
    [G] Typically computed via finite differences or analytical derivative of joint histogram.
    """
    pass

def downsample(volume, factor):
    """Downsample volume by integer factor."""
    # [G] Average pooling or simple strided slicing
    pass
```

## Subtopics

### [Similarity Measures for Volume Registration](23318-similarity-measures-for-volume-registration.md)
Defines metrics like Correlation, Normalized MI, and Tsallis Entropy. Choose MI when intensity relationships are non-linear or monomodal but intensity shifts occur. Choose Correlation for linear intensity relationships.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot B^2)$ per iteration, where $N$ is the number of sampled voxels and $B$ is the number of bins. Multi-resolution reduces $N$ significantly at coarse levels [S1, unofficial].
*   **Memory Complexity:** $O(B^2)$ for the joint histogram. $O(V)$ for storing volumes [S1, unofficial].
*   **Performance:** On synthetic renal data (0.65 mm resolution), max translation error $< 0.08$ mm and max rotation error $< 0.1^\circ$ [S1, unofficial].
*   **Strengths:** Robust to intensity non-linearities; effective for monomodal 3D/3D registration [S1, unofficial].
*   **Limitations:** Sensitive to initialization (requires geometric moments or similar) to avoid local minima [S1, unofficial]. Computationally expensive for large volumes without sampling or multi-resolution [S1, unofficial].
*   **When to use:** Intra-subject rigid registration of organs (e.g., kidneys) where shape is preserved but position changes [S1, unofficial].
*   **When not to use:** Deformable registration, inter-subject registration with large intensity differences, or volumes with constant intensity [S2, unofficial].

## Background
Originated in medical image analysis for renal volume registration in preoperative planning systems [S1, unofficial]. The method assumes renal rigidity during short acquisition windows (20-30 minutes) [S1, unofficial]. It was evaluated against other similarity measures (Correlation, Normalized MI, Correlation Ratio, Tsallis Entropy) and found MI to be superior for monomodal renal data [S1, unofficial]. Implementation details often use ITK/SimpleITK for optimization and resampling [S2, unofficial].

## Sources
[S1] Research of New Medical Volume Visualization Methods: Application in the Kidney Preoperative Planning System, https://theses.hal.science/tel-00355629v1, fetched 2026-10-26 (unofficial)
[S2] lev1nson/OpenMRI, https://github.com/lev1nson/OpenMRI, fetched 2026-10-26 (unofficial)
