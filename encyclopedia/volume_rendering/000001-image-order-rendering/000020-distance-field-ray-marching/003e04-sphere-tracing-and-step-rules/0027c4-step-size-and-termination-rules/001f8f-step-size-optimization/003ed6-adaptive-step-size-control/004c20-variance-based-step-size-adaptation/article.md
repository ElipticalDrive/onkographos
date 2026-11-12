# Variance-based Step Size Adaptation

Also called: Choosing dt through Variance Bricking, Variance Bricking

## Overview
Variance-based Step Size Adaptation is a preprocessing technique that determines the adaptive ray traversal step size ($\Delta t$) for volume rendering by calculating the statistical variance of voxel data within spatial partitions called "bricks" [S1, unofficial]. It solves the problem of selecting an appropriate step size for sparse or local ray traversal without the computational expense of calculating iso-surface normals at runtime [S1, unofficial]. By mapping brick variance to a scaling coefficient, it balances rendering speed and image quality, allowing larger steps in homogeneous regions and smaller steps in high-frequency areas [S1, unofficial].

## How it works
**Inputs:**
- Volume dataset stored in bricks.
- Voxel dimensions $dx, dy, dz$.
- Precomputed variance $v$ for each brick.
- Global minimum and maximum variances across the dataset, $v_{min}$ and $v_{max}$.
- Constants $\kappa_{min}$ and $\kappa_{max}$.

**Outputs:**
- Adaptive step size $\Delta t_{ijk}$ for each brick $(i,j,k)$.

**Data Structures:**
- **Brick:** A block of voxels. The variance $v$ is stored per brick.
- **Variance Map:** A 3D array or texture storing $v$ for each brick in the volume.

**Formulas:**
1.  **Variance Calculation:**
    The variance $v$ for a brick is calculated as the mean squared deviation of voxel values $x_i$ from the brick mean $\bar{x}$ [S1, unofficial]:
    $$ v = \frac{1}{n} \sum_{i=1}^{n} (x_i - \bar{x})^2 $$
    where $n$ is the number of voxels in the brick.

2.  **Coefficient Mapping:**
    The coefficient $\kappa_{ijk}$ is derived from the brick's variance $v$ using a linear mapping function $f(v, v_{max}, v_{min})$ [S1, unofficial]:
    $$ \kappa_{ijk} = \kappa_{min} + \frac{(v - v_{min})(\kappa_{max} - \kappa_{min})}{v_{max} - v_{min}} $$
    This maps the variance range $[v_{min}, v_{max}]$ to the coefficient range $[\kappa_{min}, \kappa_{max}]$ [S1, unofficial].

3.  **Step Size Determination:**
    The step size $\Delta t_{ijk}$ for a brick is determined by the smallest voxel dimension divided by the coefficient [S1, unofficial]:
    $$ \Delta t_{ijk} = \frac{\min(dx, dy, dz)}{\kappa_{ijk}} $$

**Parameters:**
- $\kappa_{max}$: Maximum kappa value. Typical value: 0.3 [S1, unofficial].
- $\kappa_{min}$: Minimum kappa value. Typical value: 0.1 [S1, unofficial].
- Brick dimensions: Common sizes include $4 \times 4 \times 4$, $8 \times 8 \times 8$, etc. Performance varies by dataset and hardware [S1, unofficial].

**Steps:**
1.  **Preprocessing (CPU):**
    - Load the volume dataset.
    - Partition the volume into bricks.
    - For each brick, calculate the mean $\bar{x}$ and variance $v$ of its voxel values.
    - Determine the global $v_{min}$ and $v_{max}$ across all bricks.
    - Store $v$ for each brick in a variance map.

2.  **Runtime (GPU):**
    - For each ray, determine the current brick.
    - Retrieve the precomputed variance $v$ for that brick.
    - Calculate $\kappa_{ijk}$ using the linear mapping function.
    - Calculate $\Delta t_{ijk}$ using the voxel dimensions and $\kappa_{ijk}$.
    - Use $\Delta t_{ijk}$ for ray traversal steps within that brick.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_variance(brick_voxels):
    n = len(brick_voxels)
    mean = sum(brick_voxels) / n
    variance = sum((x - mean) ** 2 for x in brick_voxels) / n
    return variance

def map_variance_to_kappa(v, v_min, v_max, kappa_min, kappa_max):
    if v_max == v_min:
        return kappa_min
    kappa = kappa_min + ((v - v_min) * (kappa_max - kappa_min)) / (v_max - v_min)
    return kappa

def calculate_step_size(kappa, dx, dy, dz):
    min_dim = min(dx, dy, dz)
    return min_dim / kappa

def preprocess_volume(volume_data, brick_size):
    v_min = float('inf')
    v_max = float('-inf')
    brick_variances = {}
    
    # Calculate variance for each brick
    for brick_id, brick_voxels in volume_data.iter_bricks(brick_size):
        v = calculate_variance(brick_voxels)
        brick_variances[brick_id] = v
        if v < v_min:
            v_min = v
        if v > v_max:
            v_max = v
            
    return brick_variances, v_min, v_max

def render_ray(ray, brick_variances, v_min, v_max, kappa_min, kappa_max, dx, dy, dz):
    t = ray.origin
    while t < ray.max_distance:
        current_brick_id = get_brick_id_at_position(t, brick_size)
        v = brick_variances[current_brick_id]
        kappa = map_variance_to_kappa(v, v_min, v_max, kappa_min, kappa_max)
        dt = calculate_step_size(kappa, dx, dy, dz)
        
        # Sample and check for intersection
        sample_value = sample_volume(ray.position(t))
        if is_intersection(sample_value):
            return refine_intersection(ray, t, dt)
        
        t += dt
    return None
```

**Edge Cases and Pitfalls:**
- **Constant Variance:** If $v_{max} = v_{min}$, the mapping function divides by zero. The pseudocode handles this by returning $\kappa_{min}$ [S1, unofficial].
- **Brick Size Selection:** Performance is sensitive to brick size. Too small bricks increase overhead; too large bricks reduce adaptivity [S1, unofficial].
- **Rotation Artifacts:** The technique may introduce a slight decrease in image correctness during dataset rotation, though it is often negligible [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Preprocessing is $O(N)$ where $N$ is the number of voxels. Runtime step calculation is $O(1)$ per step.
- **Memory Complexity:** Requires storing variance for each brick, adding $O(N_{bricks})$ memory overhead.
- **Performance:** Improves interactivity for rotating datasets when combined with predictive acceleration techniques [S1, unofficial]. Performance varies by dataset and hardware configuration (e.g., thread block size) [S1, unofficial].
- **Strengths:** Avoids expensive runtime normal calculations; effectively utilizes GPU memory hierarchy; balances speed and quality.
- **Limitations:** Preprocessing adds CPU overhead; slight loss of visual quality in dynamic scenarios; requires careful tuning of $\kappa$ values and brick sizes.

## Background
The method was introduced in "GPU Accelerated Voxel Traversal using the Prediction Buffer" (2010) [S1, unofficial]. It was developed to enhance the performance of sparse and local ray traversal techniques by providing an adaptive step size based on data characteristics. Future work suggested refining the mapping function from variance to step size [S1, unofficial].

## Sources
[S1] GPU Accelerated Voxel Traversal using the Prediction Buffer, https://people.cs.vt.edu/yongcao/publication/pdf/Braley10.pdf, fetched 2026-10-25 (unofficial)
