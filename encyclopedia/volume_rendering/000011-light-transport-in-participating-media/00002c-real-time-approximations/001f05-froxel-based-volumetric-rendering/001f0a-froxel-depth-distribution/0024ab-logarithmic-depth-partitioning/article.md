# Logarithmic Depth Partitioning

Also called: Voxelization Layer Distribution

## Overview
Logarithmic Depth Partitioning is a depth partitioning scheme that enforces a minimum angle resolution for surface normals by using logarithmic spacing, keeping cell extent ratios and minimum angle alpha constant throughout the perspective grid [S1]. It solves the problem of ensuring continuous appearance in the final renders generated from the extracted boundary surface depths by requiring more depth precision near the camera than farther away during voxelization [S1]. It sets itself apart from linear depth partitioning by maintaining a constant minimum angle step between reconstructed surfaces across the entire depth range [S1].

## How it works
**Inputs:**
- $d_0$: Near plane depth [S1].
- $\alpha$: Minimum angle step for surface normal resolution (dimensionless) [S1].
- $\theta_y$: Vertical field of view (radians) [S1].
- $R_y$: Vertical grid resolution (number of cells in the vertical axis) [S1].

**Outputs:**
- A mapping from layer index $i$ to depth $d_i$ [S1].
- A mapping from depth $d_i$ to layer index $i$ [S1].

**Data Structures:**
The method defines a set of depth layers [S1]. The depth $d_i$ of layer $i$ is determined by the logarithmic growth formula [S1]. No complex data structures are required for the partitioning definition itself, though precomputed logarithms are recommended for performance [S1].

**Parameters:**
- $K_d$: A constant determining the logarithmic growth rate of depth layers [S1]. It is derived from the condition $c_d = c_w \arctan \alpha$, where $c_d$ is cell depth and $c_w$ is column width [S1].
- $\alpha$: The minimum angle step between a reconstructed surface coplanar to the viewer and an adjacent surface bent towards the viewer [S1]. This parameter, along with $d_0$, fully determines the partitioning [S1].

**Formulas:**
1.  **Vertical Cell Width ($c_w$):**
    $$c_w = 2\tan\left(\frac{\theta_y}{2}\right) \frac{d}{R_y}$$
    Where $d$ is the depth of the cell [S1].

2.  **Growth Constant ($K_d$):**
    Derived from the constraint that the minimum depth step $c_d$ satisfies $\alpha = \tan(c_d/c_w)$ [S1]. Substituting $c_d = K_d d$ and $c_w/d = 2\tan(\theta_y/2)/R_y$ yields:
    $$K_d = \frac{2\tan(\theta_y/2)}{R_y} \arctan \alpha$$
    [S1]

3.  **Layer Depth ($d_i$):**
    The depth of layer $i$ is given by:
    $$d_i = (1 + K_d)^i d_0$$
    [S1]

4.  **Layer Index ($i$):**
    The layer index for a given depth $d_i$ is:
    $$i = \left\lfloor \frac{\log d_i - \log d_0}{\log(1 + K_d)} \right\rfloor$$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_log_depth_partitioning(d0, alpha, theta_y, R_y):
    """
    Computes the logarithmic depth partitioning parameters.
    
    Args:
        d0 (float): Near plane depth.
        alpha (float): Minimum angle step for surface normal resolution.
        theta_y (float): Vertical field of view in radians.
        R_y (int): Vertical grid resolution.
        
    Returns:
        K_d (float): The logarithmic growth constant.
    """
    # K_d = (2 * tan(theta_y / 2) / R_y) * atan(alpha)
    K_d = (2.0 * math.tan(theta_y / 2.0) / R_y) * math.atan(alpha)
    return K_d

def get_layer_index(d, d0, K_d):
    """
    Computes the layer index i for a given depth d.
    
    Args:
        d (float): The depth value.
        d0 (float): Near plane depth.
        K_d (float): The logarithmic growth constant.
        
    Returns:
        i (int): The layer index.
    """
    if d < d0:
        return 0  # Boundary condition [G]
    if K_d == 0.0:
        return 0  # Degenerate case: alpha=0 implies K_d=0, log(1)=0 causes division by zero [G]
    numerator = math.log(d) - math.log(d0)
    denominator = math.log(1.0 + K_d)
    return math.floor(numerator / denominator)

def get_layer_depth(i, d0, K_d):
    """
    Computes the depth d_i for a given layer index i.
    
    Args:
        i (int): The layer index.
        d0 (float): Near plane depth.
        K_d (float): The logarithmic growth constant.
        
    Returns:
        d_i (float): The depth of the layer.
    """
    return (1.0 + K_d) ** i * d0
```

**Edge Cases and Pitfalls:**
- **Precision:** Computing distances repetitively during surface extraction can be avoided by precomputing a lower bound of the number of voxels corresponding to a threshold distance $g$ for each part of a voxel column [S1].
- **Boundary Conditions:** The formula assumes $d \ge d_0$ [G]. Depths less than $d_0$ are typically clamped to the first layer [G].
- **Degenerate Input:** If $\alpha = 0$, then $K_d = 0$, and the logarithmic term $\log(1 + K_d)$ becomes zero, causing division by zero in the index calculation [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
Logarithmic Depth Partitioning, also referred to as Voxelization Layer Distribution, was introduced in the context of on-the-fly voxelization of particle data [S1]. It addresses the need for continuous appearance in rendered boundary surfaces by enforcing minimum angle resolution during voxelization [S1]. The method is particularly relevant for SPH (Smoothed Particle Hydrodynamics) simulations and other particle-based rendering techniques where surface extraction from voxelized data is required [S1].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
