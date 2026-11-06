# LinPrim Population Control
Also called: Adaptive Population Control, LinPrim-MMC, LinPrim-MCMC
## Overview
LinPrim Population Control manages the density of linear primitives (octahedra and tetrahedra) during differentiable volume rendering optimization by pruning, cloning, and splitting primitives to adapt to scene complexity [S1]. It differs from standard 3D Gaussian Splatting (3DGS) by redefining primitive size metrics to accommodate non-spherical kernels and adjusting sampling distributions to respect the anisotropic geometry of linear primitives [S1]. It builds on [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md) and adapts techniques from [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md).
## How it works
### Inputs and Outputs
**Inputs:**
- Current set of primitives $\mathcal{P}$, each with position $\mathbf{p}$, color $\mathbf{c}$, distances to corners $\mathbf{d}$, rotation quaternion $\mathbf{q}$, and view-space positional gradient $\nabla_{\mathbf{p}}$.
- Camera parameters for known views.
- Scene extent bounds.

**Outputs:**
- Updated set of primitives $\mathcal{P}'$ with modified count, positions, and sizes.

### Data Structures
Each primitive stores:
- `position`: 3D vector $\mathbf{p} \in \mathbb{R}^3$.
- `color`: 3D vector $\mathbf{c} \in \mathbb{R}^3$ (or SH coefficients).
- `distances`: Array of distances to corner vertices. For octahedra, 3 values (along axes); for tetrahedra, 4 values (to vertices).
- `rotation`: Quaternion $\mathbf{q} \in \mathbb{R}^4$.
- `gradient_momentum`: 3D vector for Adam optimizer state.
- `type`: Enum (OCTAHEDRON, TETRAHEDRON).

### Steps

#### 1. Pruning
Remove primitives that fail fidelity or efficiency criteria [S1].

**Size Definition:**
- For **Octahedra**: Size $S_{oct} = 2 \cdot \max(\mathbf{d})$, where $\mathbf{d}$ are the distances along the principal axes [S1].
- For **Tetrahedra**: Size $S_{tet} \approx \sqrt{2} \cdot d_{max}$, where $d_{max}$ is the distance to the furthest corner vertex [S1].

**Criteria:**
A primitive is pruned if:
1. Opacity $\alpha < \tau_{opacity}$ (threshold, e.g., 0.005) [S1].
2. $S > \tau_{scene\_scale} \cdot \text{SceneExtent}$ [S1].
3. Screen-space area $A_{screen} > \tau_{screen\_space}$ [S1].

#### 2. Initialization (One-time or Reset)
Initialize primitives from Structure-from-Motion (SfM) points [S1]:
1. For each SfM point $\mathbf{p}_i$ with color $\mathbf{c}_i$:
   - Find closest SfM neighbor distance $d_{min}$.
   - Set initial distances $\mathbf{d} = d_{min}$ for all corners.
   - Set rotation $\mathbf{q}$ to a uniformly random quaternion [S1].
   - Set position $\mathbf{p} = \mathbf{p}_i$.

#### 3. Cloning and Splitting
Select primitives with large view-space positional gradients $\|\nabla_{\mathbf{p}}\| > \tau_{grad}$ [S1].

**Cloning (for small primitives):**
1. Create new primitive $\mathbf{p}_{new}$ with identical features: $\mathbf{p}_{new} = \mathbf{p}$, $\mathbf{c}_{new} = \mathbf{c}$, $\mathbf{d}_{new} = \mathbf{d}$, $\mathbf{q}_{new} = \mathbf{q}$ [S1].
2. Reset gradient momentum: $\mathbf{m}_{new} = \mathbf{0}$ [S1].

**Splitting (for large primitives):**
1. Create two new primitives $\mathbf{p}_{a}, \mathbf{p}_{b}$ with smaller sizes.
2. Sample positions from a PDF based on the original primitive's geometry [S1].

**Sampling Logic:**
- **Octahedra:**
  - Standard deviations $\sigma_i = d_i$ (corresponding distance for each axis) [S1].
  - Sample $\Delta \mathbf{p} \sim \mathcal{N}(\mathbf{0}, \text{diag}(\sigma^2))$ [S1].
  - Apply rotation: $\mathbf{p}_{new} = \mathbf{p} + \mathbf{R}(\mathbf{q}) \Delta \mathbf{p}$ [S1].
  - This spreads primitives along longer dimensions [S1].
- **Tetrahedra:**
  - Set $\sigma = \frac{1}{2} \max(\mathbf{d})$ for all dimensions [S1].
  - Sample $\Delta \mathbf{p} \sim \mathcal{N}(\mathbf{0}, \sigma^2 \mathbf{I})$ [S1].
  - Apply rotation: $\mathbf{p}_{new} = \mathbf{p} + \mathbf{R}(\mathbf{q}) \Delta \mathbf{p}$ [S1].

#### 4. LinPrim-MCMC (Optional Advanced Control)
Adapt GS-MCMC population control by treating primitives as equivalent Gaussians [S1].
1. Scale octahedron distances: $\mathbf{d}' = \mathbf{d} / 2.6$ [S1].
2. Use $\mathbf{d}'$ as Gaussian standard deviations for MCMC sampling [S1].
3. Perform MCMC steps to draw samples from the scene distribution, allowing precise control over final population size [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def prune_primitives(primitives, camera, scene_extent, opacity_thresh=0.005, screen_thresh=100.0):
    """Remove primitives based on opacity, size, and screen space."""
    to_remove = []
    for i, prim in enumerate(primitives):
        # Check opacity
        if prim.opacity < opacity_thresh:
            to_remove.append(i)
            continue
        
        # Calculate size
        if prim.type == 'OCTAHEDRON':
            size = 2.0 * np.max(prim.distances)
        elif prim.type == 'TETRAHEDRON':
            size = np.sqrt(2.0) * np.max(prim.distances)
        else:
            size = 0.0
            
        # Check scene extent size
        if size > 0.1 * scene_extent: # Threshold relative to scene
            to_remove.append(i)
            continue
            
        # Check screen space area (simplified)
        screen_area = calculate_screen_area(prim, camera)
        if screen_area > screen_thresh:
            to_remove.append(i)
            
    # Remove in reverse order to maintain indices
    for i in reversed(to_remove):
        primitives.pop(i)
    return primitives

def clone_and_split(primitives, grad_thresh=0.001):
    """Clone small primitives with high gradients, split large ones."""
    new_primitives = []
    for prim in primitives:
        grad_norm = np.linalg.norm(prim.gradient)
        if grad_norm > grad_thresh:
            # Determine if clone or split based on size
            if prim.type == 'OCTAHEDRON':
                size = 2.0 * np.max(prim.distances)
            else:
                size = np.sqrt(2.0) * np.max(prim.distances)
                
            if size < 0.05: # Arbitrary threshold for "small"
                # Clone
                new_prim = copy(prim)
                new_prim.gradient_momentum = np.zeros(3) # Reset momentum
                new_primitives.append(new_prim)
            else:
                # Split
                # Sample two new positions
                pos_a, pos_b = sample_split_positions(prim)
                new_prim_a = copy(prim)
                new_prim_a.position = pos_a
                new_prim_a.gradient_momentum = np.zeros(3)
                
                new_prim_b = copy(prim)
                new_prim_b.position = pos_b
                new_prim_b.gradient_momentum = np.zeros(3)
                
                # Reduce size for children
                new_prim_a.distances = [d * 0.5 for d in prim.distances]
                new_prim_b.distances = [d * 0.5 for d in prim.distances]
                
                new_primitives.append(new_prim_a)
                new_primitives.append(new_prim_b)
    return new_primitives

def sample_split_positions(prim):
    """Sample positions for splitting based on primitive type."""
    q = prim.rotation # Quaternion
    R = quaternion_to_rotation_matrix(q)
    
    if prim.type == 'OCTAHEDRON':
        # Standard deviations correspond to distances
        sigma = np.array(prim.distances)
        # Ensure symmetry for octahedron (distances are along axes)
        # Sample from diagonal covariance
        delta = np.random.normal(0, sigma)
        pos_offset = R @ delta
    elif prim.type == 'TETRAHEDRON':
        # All standard deviations set to half the largest distance
        max_d = np.max(prim.distances)
        sigma = 0.5 * max_d
        delta = np.random.normal(0, sigma, 3)
        pos_offset = R @ delta
    else:
        pos_offset = np.zeros(3)
        
    pos_a = prim.position + pos_offset
    pos_b = prim.position - pos_offset # Symmetric split
    return pos_a, pos_b

def linprim_mcmc_step(primitives, scale_factor=2.6):
    """Apply LinPrim-MCMC adjustments."""
    for prim in primitives:
        if prim.type == 'OCTAHEDRON':
            # Scale distances to mimic Gaussian std devs
            prim.distances_scaled = [d / scale_factor for d in prim.distances]
        # Perform MCMC sampling using scaled distances as Gaussian parameters
        # [Implementation of specific MCMC kernel omitted, relies on GS-MCMC logic]
        pass
```

### Parameters
- `opacity_thresh`: Default 0.005 [S1].
- `grad_thresh`: Threshold for view-space positional gradient norm [S1].
- `scale_factor` (LinPrim-MCMC): 2.6 [S1].
- `scene_extent`: Bounding box diagonal or max dimension of the scene [S1].

### Edge Cases and Pitfalls
- **Tetrahedron Splitting:** Non-trivial to sample along principal axes due to lack of symmetry; use isotropic scaling with $\sigma = \frac{1}{2} \max(\mathbf{d})$ [S1].
- **Octahedron Anisotropy:** Must sample along local axes before rotating to spread along longer dimensions [S1].
- **Momentum Reset:** Cloned primitives must reset gradient momentum to avoid biased updates [S1].
- **Magnitude Mismatch:** Direct use of octahedron distances as Gaussian std devs is invalid; scaling by 2.6 is required for MCMC compatibility [S1].

## Operating characteristics
- **Time Complexity:** $O(N)$ per frame for pruning and selection, where $N$ is the number of primitives. Cloning/splitting increases $N$ dynamically.
- **Memory Complexity:** $O(N)$, growing as primitives are added.
- **Performance:** Efficiently adapts to scene detail; prevents over-rendering in uniform regions and under-rendering in complex areas [S1].
- **Strengths:** Compatible with 3DGS advancements (e.g., MCMC); handles non-spherical primitives correctly [S1].
- **Limitations:** Requires careful tuning of size thresholds; tetrahedron splitting is less geometrically precise than octahedron splitting [S1].
- **When to Use:** When optimizing linear primitives for novel views or high-fidelity reconstruction where fixed initialization is insufficient [S1].

## Background
LinPrim Population Control originates from the LinPrim paper (2025), which adapts 3D Gaussian Splatting techniques to linear primitives (octahedra and tetrahedra) [S1]. It addresses the need for dynamic population management in differentiable volumetric rendering with non-Gaussian kernels [S1]. The method incorporates insights from GS-MCMC [S1] and Distribution Alignment [S1] to handle magnitude differences between linear primitive distances and Gaussian standard deviations.

## Sources
[S1] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
