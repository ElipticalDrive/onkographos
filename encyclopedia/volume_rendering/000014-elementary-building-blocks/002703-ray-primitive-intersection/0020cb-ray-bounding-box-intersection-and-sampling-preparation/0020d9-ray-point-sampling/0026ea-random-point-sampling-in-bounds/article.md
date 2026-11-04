# Random Point Sampling in Bounds

Also called: Uniform Random Point Sampling, UniformSampler, sampleCell

## Overview
Generates uniformly distributed random points within a specified 3D bounding box or voxel volume [S1, S2, S3, unofficial]. It differs from structured sampling by providing stochastic coverage for Monte Carlo integration [G]. Unlike spherical sampling, it assumes an axis-aligned domain, making it computationally efficient but potentially non-uniform if spherical symmetry is required [S3, unofficial].

## How it works
**Inputs:**
- `num_points`: Integer specifying the number of random points to generate [S1, unofficial].
- `bounds`: A tuple or list containing `min_bound` and `max_bound` arrays of 3 floats defining the axis-aligned bounding box [S1, unofficial].
- `device`: The computational device (e.g., 'cpu', 'cuda') for tensor operations [S1, unofficial].

**Outputs:**
- A tensor of shape `(num_points, 3)` containing the generated random points [S1, unofficial].

**Data Structures:**
- Tensors are used for `min_bound`, `max_bound`, and the output points [S1, unofficial]. The bounds are converted to tensors on the specified device and expanded with a batch dimension if necessary for broadcasting [S1, unofficial].

**Steps:**
1. Convert `min_bound` and `max_bound` to tensors on the specified `device` [S1, unofficial].
2. Generate `num_points` random values from a uniform distribution $\mathcal{U}[0, 1)$ for each of the 3 dimensions, resulting in a tensor of shape `(num_points, 3)` [S1, unofficial].
3. Calculate the volume span: $\Delta = \text{max\_bound} - \text{min\_bound}$ [S1, unofficial].
4. Scale the random values by $\Delta$ to map the range $[0, 1)$ to $[0, \Delta)$ [S1, unofficial].
5. Add `min_bound` to the scaled random values to shift the range to $[\text{min\_bound}, \text{max\_bound})$ [S1, unofficial].

**Alternative Voxel-Centric Formulation:**
Some implementations define the sampling relative to a voxel center $c$ and size $s$ [S3, unofficial]. In this case:
1. Generate three independent uniform random values $\mathbf{r} \sim \mathcal{U}(-1, 1)^3$ [S3, unofficial].
2. Scale $\mathbf{r}$ by the voxel size vector $s$ [S3, unofficial].
3. Add the result to the voxel center $c$ to obtain the point $p$ [S3, unofficial].
   $$ p = c + \mathbf{r} \cdot s $$
   where $\mathbf{r}$ is a vector of random values in $(-1, 1)$ and $s$ is the voxel dimension vector [S3, unofficial].

**Parameters:**
- `num_points`: Typical values depend on the training batch size or ray count [G].
- `bounds`: Defined by the specified 3D bounding box or voxel grid [S1, S3, unofficial].

**Edge Cases and Pitfalls:**
- **Non-Cubic Voxels:** If the voxel is not a cube (i.e., $s_x \neq s_y \neq s_z$), the sampling density remains uniform in the axis-aligned box, but the geometric representation may distort if spherical symmetry is assumed [S3, unofficial].
- **Precision:** Using floating-point arithmetic for large bounding boxes may introduce precision errors; ensure the device supports sufficient precision for the coordinate range [G].
- **Boundary Conditions:** The distribution is typically half-open $[min, max)$, meaning points exactly on the maximum boundary are excluded [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def random_point_sampling_in_bounds(num_points, bounds, device):
    """
    Generate uniformly distributed random points within a 3D bounding box.
    
    Args:
        num_points (int): Number of points to sample.
        bounds (tuple): (min_bound, max_bound) as arrays of shape (3,).
        device (str): Target device ('cpu', 'cuda', etc.).
        
    Returns:
        torch.Tensor: Sampled points of shape (num_points, 3).
    """
    min_bound, max_bound = bounds
    
    # Convert to tensors on the specified device and add batch dimension
    min_tensor = torch.tensor(min_bound, dtype=torch.float32, device=device).unsqueeze(0)
    max_tensor = torch.tensor(max_bound, dtype=torch.float32, device=device).unsqueeze(0)
    
    # Generate random numbers in [0, 1)
    # Shape: (num_points, 3)
    random_vals = torch.rand(num_points, 3, device=device)
    
    # Calculate bounds span
    span = max_tensor - min_tensor
    
    # Scale and shift
    # Broadcasting: span (1, 3) scales random_vals (num_points, 3)
    points = random_vals * span + min_tensor
    
    return points
```

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is `num_points`, as it involves generating $3N$ random numbers and basic arithmetic operations [G].
- **Memory Complexity**: $O(N)$ to store the output tensor [G].
- **Performance**: Highly efficient on GPUs due to parallel random number generation and vectorized operations [G].
- **Strengths**: Simple to implement and provides uniform coverage of the axis-aligned volume [G].
- **Limitations**: Does not account for non-cubic voxel shapes or spherical symmetry, which may lead to biased sampling in those contexts [S3, unofficial].
- **When to Use**: Monte Carlo integration and volume rendering where axis-aligned bounding boxes are sufficient [G].
- **When Not to Use**: When spherical symmetry is required or when the domain is not axis-aligned [S3, unofficial].

## Background
The method is implemented in neural surface and volume rendering pipelines [S1, unofficial]. It is also employed in level-of-detail training schemes for 3D Gaussian Splatting variants, where random points within voxel bounds are sampled for densification or pruning decisions [S3, unofficial]. The technique relies on basic uniform random number generation, a standard primitive in computer graphics and scientific computing [G].

## Sources
[S1] Neural Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S3] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
