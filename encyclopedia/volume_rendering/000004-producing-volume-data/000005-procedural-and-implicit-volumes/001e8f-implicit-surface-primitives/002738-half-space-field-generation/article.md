# Half-Space Field Generation

Also called: Binary Half-Volume, Level Set Domain Definition, Level Set Masking, Step Function Field

## Overview
Half-Space Field Generation creates a binary scalar field that defines a simulation domain (typically a fluid) by partitioning space into "inside" and "outside" regions based on a threshold. It solves the problem of restricting physical simulation and rendering computations to a specific sub-volume, ignoring empty space or background media. Unlike smooth implicit primitives (e.g., SDFs or metaballs) that define continuous geometry, this method produces a discontinuous step function, effectively acting as a mask for the simulation grid.

## How it works
**Inputs and Outputs:**
- Input: A 3D grid of dimensions $X \times Y \times Z$ (denoted as `xSize`, `ySize`, `zSize`).
- Input: A threshold value $\phi_{thresh}$ (typically 0).
- Output: A 3D float array `field` of size $(X, Y, Z)$ where each voxel contains either 0 or 1.

**Data Structures:**
- `field`: A 3D array of 32-bit floating-point numbers.
- Layout: Indexed as `field[x, y, z]` where $0 \le x < X$, $0 \le y < Y$, $0 \le z < Z$.

**Algorithm:**
The method partitions the grid along the Y-axis (vertical) into two half-spaces. This specific implementation assumes a coordinate system where $Y=0$ is the bottom and $Y=YSize$ is the top, with the fluid occupying the upper half.

1.  Initialize the 3D array `field` with zeros.
2.  Iterate through all voxels $(x, y, z)$.
3.  If $y \ge \lfloor YSize / 2 \rfloor$, set `field[x, y, z] = 1`.
4.  If $y < \lfloor YSize / 2 \rfloor$, set `field[x, y, z] = 0`.

**Formulas:**
The value $v$ of a voxel at $(x, y, z)$ is defined as:
$$
v(x, y, z) = \begin{cases} 
1 & \text{if } y \ge \frac{YSize}{2} \\
0 & \text{if } y < \frac{YSize}{2}
\end{cases}
$$

**Parameters:**
- `ySize`: The vertical resolution of the grid.
- Threshold: Implicitly 0.5 of the grid height in this binary step implementation.

**Edge Cases and Pitfalls:**
- **Boundary Precision:** The division `ySize / 2` must be handled consistently with integer arithmetic. If `ySize` is odd, the split occurs between the two central slices.
- **Simulation Masking:** In fluid simulation contexts, this field is used to mask pressure solves and external forces (like gravity). Forces are only applied where `field == 1` [S2, unofficial].
- **GPU Optimization:** On GPUs lacking dynamic flow control, branching on this field can be inefficient. Alternatives include using Z-culling to eliminate cells or sparse data structures [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_half_space_field(x_size, y_size, z_size):
    """
    Generates a binary 3D field where the upper half (y >= y_size/2) is 1,
    and the lower half is 0.
    """
    # Initialize 3D array with zeros
    field = [[[0.0 for _ in range(z_size)] for _ in range(y_size)] for _ in range(x_size)]
    
    # Calculate split point
    split_y = y_size // 2
    
    # Set upper half to 1
    for y in range(split_y, y_size):
        for z in range(z_size):
            for x in range(x_size):
                field[x][y][z] = 1.0
                
    # Lower half remains 0 (initialized above)
    
    return field
```

## Subtopics

### [Implicit Surface Definition](2798-implicit-surface-definition.md)
Defines the general mathematical framework for scalar fields. Half-Space Field is a specific, discontinuous instance of this.

### [Implicit Geometry Sampling](6685-implicit-geometry-sampling.md)
Discusses how to sample these fields. Half-Space fields are trivial to sample but require careful handling of the discontinuity for marching cubes or ray marching.

## Operating characteristics
- **Time Complexity:** $O(X \cdot Y \cdot Z)$ to initialize the grid. Subsequent queries are $O(1)$.
- **Memory Complexity:** $O(X \cdot Y \cdot Z)$ for the full grid. Can be reduced to $O(X \cdot Z \cdot Y_{active})$ if using sparse structures [S2, unofficial].
- **Strengths:** Extremely fast to generate; computationally cheap to evaluate; ideal for masking simulations to ignore empty space (e.g., air in water simulations) [S2, unofficial].
- **Limitations:** Produces a sharp, discontinuous boundary which may alias poorly in ray marching without supersampling or specific anti-aliasing techniques; does not represent curved surfaces natively.
- **When to use:** When the domain is a simple geometric partition (like a half-space) and the primary goal is simulation masking rather than visual surface representation.

## Background
This technique is primarily documented in the context of real-time 3D fluid simulation. It is used to define the fluid domain, ensuring that simulation steps (pressure solve, gravity) are only performed on liquid cells ($\phi < 0$ or in this binary case, the "active" half) and not on the surrounding air [S2, unofficial]. The approach leverages the level set function's sign to control computational load.

## Sources
[S1] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
