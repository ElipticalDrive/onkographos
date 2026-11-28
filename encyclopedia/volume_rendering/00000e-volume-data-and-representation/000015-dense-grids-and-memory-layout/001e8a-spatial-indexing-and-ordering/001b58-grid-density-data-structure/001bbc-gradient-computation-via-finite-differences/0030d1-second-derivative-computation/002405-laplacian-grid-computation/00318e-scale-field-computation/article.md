# Scale Field Computation
Also called: Discrete Scale Field, Size-based Transfer Function Scale Field

## Overview
Scale Field Computation estimates a 3D scalar field $S$ that encodes the local feature size (scale) for every point in a volume, enabling size-based transfer functions for volume exploration [S1]. It differs from pyramid-based approaches by using iterative non-linear diffusion to generate a continuous scale-space, allowing for sub-voxel scale detection and better localization of features without the granularity limitations of subsampled pyramids [S1].

## How it works
The method decomposes into three steps: Scale-space computation, Scale detection, and Backprojection [S1].

### Inputs and Outputs
- **Input**: Discrete 3D volume $V$ on a regular grid [S1].
- **Output**: Scalar field $S(x)$ mapping every voxel $x$ to a local scale $t$ [S1].

### Data Structures
- **Scale Space Grid $L$**: 3D array of floats, same dimensions as input volume. $L(x; t)$ stores the diffused intensity at position $x$ and scale $t$ [S1].
- **Scale Abstraction $\mathcal{N}$**: List of tuples $(x_i, t_i)$, where $x_i$ is the 3D index of a Laplacian of Gaussian (LoG) extremum and $t_i$ is the detected scale [S1].

### Parameters
- $n$: Number of iterations. Controls maximum detectable size. Often set to $n = \text{Dim} / \Delta t$ where Dim is the volume dimension [S1].
- $\Delta t$: Scale sampling distance. Controls minimum detectable size. $\Delta t = 1$ corresponds to voxel size; $\Delta t < 1$ allows sub-voxel detection [S1].
- $\lambda$: Sensitivity parameter for the conductivity function. Small values ($0.01 \le \lambda \le 0.1$) localize maxima for convoluted structures; large values ($\lambda > 1$) make diffusion linear and robust to noise [S1].
- $h$: Local support parameter for the basis function. Typically $h = k \langle t_i \rangle$ with $k \approx 1$. Larger $h$ increases smoothness but reduces scale distinction [S1].
- $\tau$: Visibility threshold parameter. Used to modulate weights during backprojection to prevent scale bleeding [S1].

### Step 1: Scale-space Computation
Compute the multi-scale representation iteratively using forward Euler integration of the diffusion equation [S1].

**Formula**:
$$ L(x; t + \Delta t) = L(x; t) + \Delta t \sum_{ijk \in \mathcal{N}} \alpha_{ijk} \Phi(L(x_{ijk}; t) - L(x; t)) $$
where $\mathcal{N}$ is the neighbor stencil (6-, 18-, or 26-point), $\alpha_{ijk}$ are normalizing coefficients, and $\Phi$ is the flow function [S1].

**Flow Function**:
$$ \Phi(u) = u \cdot C(u) $$
where $C$ is the conductivity function [S1].

**Conductivity Function**:
The method uses a custom non-linear conductivity based on signed forward differences to control diffusion direction towards the medial axis [S1]:
$$ C(L(x_{ijk}) - L(x)) = \begin{cases} \frac{\lambda^2}{\lambda^2 + (L(x_{ijk}) - L(x))^2} & \text{if } L(x_{ijk}) - L(x) > 0 \\ 1 & \text{otherwise} \end{cases} $$
This differs from the standard Perona-Malik gradient magnitude approach, which stops diffusion at edges and causes false maxima [S1].

**Algorithm**:
1. Initialize $L(x; 0) = V(x)$ for all $x$ [S1].
2. For $k = 1$ to $n$:
    a. Create temporary grid $L_{new}$ [G].
    b. For each voxel $x$:
        i. Compute sum over neighbors $x_{ijk} \in \mathcal{N}$:
           $$ \text{diff} = L(x_{ijk}; t) - L(x; t) $$
           $$ \text{cond} = \begin{cases} \frac{\lambda^2}{\lambda^2 + \text{diff}^2} & \text{if } \text{diff} > 0 \\ 1 & \text{else} \end{cases} $$
           $$ \text{flux} = \text{diff} \cdot \text{cond} $$
        ii. Update:
           $$ L_{new}(x) = L(x; t) + \Delta t \sum_{ijk \in \mathcal{N}} \alpha_{ijk} \cdot \text{flux} $$
    c. Set $L(x; t + \Delta t) = L_{new}(x)$ [S1].
    d. Increment $t$ by $\Delta t$ [S1].

### Step 2: Scale Detection
Detect blobs by finding local maxima of the normalized Laplacian in both space and scale [S1].

**Formula**:
$$ t \nabla^2 L = t (L_{xx} + L_{yy} + L_{zz}) = t \cdot \text{Tr}(H(x)) $$
where $H(x)$ is the Hessian matrix [S1].

**Algorithm**:
1. For each scale $t$ in the computed scale-space:
    a. Compute the normalized Laplacian $N(x) = t \cdot \nabla^2 L(x; t)$ for all voxels [S1].
    b. Identify voxels $x$ that are local maxima in the 3D spatial neighborhood (e.g., 26-connected) at scale $t$ [G].
    c. Check if the scale $t$ is a local maximum along the scale axis (compare with $t-\Delta t$ and $t+\Delta t$) [G].
    d. If both conditions are met, add $(x, t)$ to the Scale Abstraction $\mathcal{N}$ [S1].

### Step 3: Backprojection
Fit a continuous scale field $S(x)$ to the discrete scale abstraction $\mathcal{N}$ using scattered data interpolation [S1].

**Basis Function**:
Fourth-degree Wendland polynomial with compact support [S1]:
$$ \Theta(d) = \left[ 1 - \frac{d}{h} \right]_{0}^{1} \left( 4 \frac{d}{h} + 1 \right) $$
where $[ \cdot ]_0^1$ clamps the value between 0 and 1, and $d = \| x - x_i \|$ [S1].

**Interpolation Methods**:
1. **Shepard’s Interpolation (Sum)**:
   $$ S(x) = \sum_{x_i \in \mathcal{N}} \Theta(\| x - x_i \|) t_i $$
2. **Maximum Blending**:
   $$ S(x) = \max_{x_i \in \mathcal{N}} \{ \Theta(\| x - x_i \|) t_i \} $$
   The maximum operator is preferred to give priority to larger features and avoid smoothing away scale variation [S1].

**Visibility Test**:
To prevent "bleeding" of scale from large features into small embedded features, modulate the weight $\Theta$ based on visibility [S1].
- For a sample point $x$ and a LoG extremum $x_i$, cast a ray from $x_i$ to $x$ [G].
- If any voxel along the ray has a scalar value or gradient magnitude below threshold $\tau$, the point $x$ is not "visible" from $x_i$ [S1].
- If not visible, set the weight $\Theta$ for $x_i$ to 0 [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scale_field(volume, n_iterations, delta_t, lambda_param, h, tau, use_max_blend=True):
    """
    Computes the scale field for a 3D volume.
    """
    import numpy as np
    
    # Step 1: Scale-space Computation
    L = np.copy(volume).astype(np.float64)
    scale_space = []
    current_t = 0.0
    
    # 6-connected stencil shifts and normalized coefficients alpha_ijk = 1/6
    shifts = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
    
    for _ in range(n_iterations):
        L_new = np.copy(L)
        diff_sum = np.zeros_like(L)
        
        for dx, dy, dz in shifts:
            neighbor = np.roll(np.roll(np.roll(L, dx, axis=0), dy, axis=1), dz, axis=2)
            diff = neighbor - L
            cond = np.where(diff > 0, (lambda_param**2) / (lambda_param**2 + diff**2), 1.0)
            flux = diff * cond
            diff_sum += flux
            
        L_new = L + delta_t * (diff_sum / 6.0)
        L = L_new
        current_t += delta_t
        scale_space.append((current_t, np.copy(L)))
        
    # Step 2: Scale Detection
    scale_abstraction = []
    
    for t, L_t in scale_space:
        # Compute normalized Laplacian: t * Laplacian(L) [S1]
        # Standard 3D Laplacian stencil (6-connected)
        laplacian_L = np.zeros_like(L_t)
        for dx, dy, dz in shifts:
            neighbor = np.roll(np.roll(np.roll(L_t, dx, axis=0), dy, axis=1), dz, axis=2)
            laplacian_L += neighbor - L_t
        normalized_laplacian = t * laplacian_L
        
        # Find local maxima in space [G]
        maxima_mask = np.zeros_like(normalized_laplacian, dtype=bool)
        for dx, dy, dz in shifts:
            neighbor = np.roll(np.roll(np.roll(normalized_laplacian, dx, axis=0), dy, axis=1), dz, axis=2)
            maxima_mask |= (neighbor < normalized_laplacian)
        maxima_mask &= (normalized_laplacian > 0) # Blobs correspond to positive Laplacian maxima [G]
        
        # Check scale axis maximum [G]
        for idx in np.argwhere(maxima_mask):
            idx = tuple(idx)
            is_scale_max = True
            if t > 0:
                prev_t, prev_L = scale_space[-2] if len(scale_space) > 1 else (0, volume)
                if prev_L[idx] > normalized_laplacian[idx]:
                    is_scale_max = False
            if t < scale_space[-1][0]:
                next_t, next_L = scale_space[scale_space.index((t, L_t)) + 1]
                if next_L[idx] > normalized_laplacian[idx]:
                    is_scale_max = False
                    
            if is_scale_max:
                scale_abstraction.append((idx, t))
                
    # Step 3: Backprojection
    scale_field = np.zeros_like(volume, dtype=float)
    
    for x_idx in np.ndindex(volume.shape):
        x = np.array(x_idx)
        max_val = -np.inf
        sum_val = 0.0
        
        for (xi, ti) in scale_abstraction:
            xi = np.array(xi)
            dist = np.linalg.norm(x - xi)
            
            # Basis function
            if dist <= h:
                theta = (1.0 - dist / h) * (4.0 * dist / h + 1.0)
            else:
                theta = 0.0
            
            # Visibility test
            if not is_visible(xi, x, volume, tau):
                theta = 0.0
            
            contribution = theta * ti
            
            if use_max_blend:
                if contribution > max_val:
                    max_val = contribution
            else:
                sum_val += contribution
                
        if use_max_blend:
            scale_field[x_idx] = max_val
        else:
            scale_field[x_idx] = sum_val
            
    return scale_field, scale_abstraction

def is_visible(start, end, volume, tau):
    """Check if end is visible from start along ray."""
    # Cast ray, sample voxels, check intensity/gradient [G]
    steps = int(np.linalg.norm(end - start) * 10) + 1
    for i in range(steps + 1):
        t = i / steps
        point = start + t * (end - start)
        idx = np.round(point).astype(int)
        if 0 <= idx[0] < volume.shape[0] and 0 <= idx[1] < volume.shape[1] and 0 <= idx[2] < volume.shape[2]:
            if volume[idx] < tau:
                return False
    return True
```

## Subtopics

### Progressive Scale-Space Construction
A property where the scale-space is constructed progressively, allowing intermediate results to be valid approximations [S1]. The scale field at a given scale is the same whether obtained from the original volume or an already smoothed version of the data set [S1]. This implies that large data sets ($> 512^3$) do not need to be analyzed at their full resolution; the scale field can be of a lower resolution, yielding the same scales except for granularity limits [S1].

## Operating characteristics
- **Time Complexity**: Scale-space computation is $O(n \cdot V \cdot |\mathcal{N}|)$ where $V$ is number of voxels and $|\mathcal{N}|$ is stencil size. Backprojection is $O(V \cdot |\mathcal{N}_{abstraction}|)$, which can be expensive if many extrema are detected [G].
- **Memory Complexity**: Requires storing the scale space $L$ for all iterations if backprojection is done offline, or just the current layer if streaming. The abstraction list size depends on feature complexity [G].
- **Strengths**: Continuous scale representation; sub-voxel detection; non-linear diffusion preserves edges and prevents merging of nearby objects better than linear diffusion [S1].
- **Limitations**: Sensitive to parameter $\lambda$; visibility test adds computational overhead; bleeding can occur if visibility test is not used [S1].
- **When to use**: When size-based classification is needed for unsegmented volumes, especially for structures with varying sizes like vessels or aneurysms [S1].

## Background
The method was introduced in "Size-based Transfer Functions: A New Volume Exploration Technique" (Vis 2008) by Carlos Correa and Tiago A. Oliveira [S1]. It addresses the limitations of pyramid-based scale maps, which suffer from poor granularity and localization due to subsampling [S1]. The approach is based on continuous scale-space theory but adapted for discrete volumes using iterative diffusion instead of convolution [S1].

## Sources
[S1] Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
