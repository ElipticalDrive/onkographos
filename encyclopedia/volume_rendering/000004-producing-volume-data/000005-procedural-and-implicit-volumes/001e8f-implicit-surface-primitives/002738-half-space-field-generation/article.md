# Half-Space Field Generation

Also called: Binary Half-Volume, Level Set Domain Definition, Level Set Masking, Step Function Field

## Overview
Half-Space Field Generation creates a binary volume representation by assigning a value of 1 to voxels within a defined region (typically the fluid domain) and 0 to those outside, effectively masking simulation or rendering computations. It differs from continuous implicit primitives by discretizing the domain boundary into a hard binary mask rather than evaluating a smooth scalar field for isosurface extraction. This method is primarily used to restrict physical simulation (such as pressure solving or force application) to active regions, ignoring empty space like air [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D grid domain defined by dimensions $(xSize, ySize, zSize)$ and a logical condition defining the "inside" region (e.g., $y > ySize/2$ or $\phi < 0$).
- Output: A 3D array (or texture) of floats or integers with the same dimensions, containing binary values (0 or 1).

**Data Structures:**
- `volume`: A 3D array of size $(xSize, ySize, zSize)$.
- Data type: Typically 32-bit float or 16-bit float for GPU compatibility, though binary logic implies integer or boolean semantics [S1, S2, unofficial].

**Steps:**
1. Initialize the 3D float array `volume` of size $(xSize, ySize, zSize)$ to zero [S1, unofficial].
2. Iterate through the voxel coordinates $(x, y, z)$ where $0 \le x < xSize$, $0 \le z < zSize$.
3. Apply the half-space condition. In the canonical example, the domain is split by the Y-axis:
   - If $y \ge ySize/2$, set `volume[x, y, z] = 1` [S1, unofficial].
   - If $y < ySize/2$, set `volume[x, y, z] = 0` (already initialized) [S1, unofficial].
4. For general level-set based generation, evaluate the level set function $\phi(\vec{x})$. If $\phi(\vec{x}) < 0$ (inside fluid), set the value to 1; otherwise 0 [S2, unofficial].
5. Use the resulting binary mask in shaders or solvers to gate computations. For example, set pressure to zero outside the liquid ($\phi \ge 0$) before solving [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_half_space_volume(xSize, ySize, zSize, split_axis='y', split_threshold=0.5):
    """
    Generates a binary 3D volume where voxels are 1 if they satisfy the half-space condition, else 0.
    """
    # Initialize 3D float array of size (xSize, ySize, zSize) with zeros
    volume = [[[0.0 for _ in range(xSize)] for _ in range(zSize)] for _ in range(ySize)]
    
    # Iterate through all voxels
    for y in range(ySize):
        for z in range(zSize):
            for x in range(xSize):
                # Determine if the voxel is inside the half-space
                # Example condition: y >= ySize / 2
                if split_axis == 'y':
                    is_inside = y >= (ySize * split_threshold)
                else:
                    # Placeholder for other axes or general level set checks
                    # In a general level set context, this would be: phi(x,y,z) < 0
                    is_inside = False 
                
                if is_inside:
                    volume[y][z][x] = 1.0
                else:
                    volume[y][z][x] = 0.0
                    
    return volume

def apply_mask_to_pressure(pressure_field, mask_field):
    """
    Masks pressure field using the half-space volume.
    Sets pressure to zero where mask is 0.
    """
    for y in range(len(pressure_field)):
        for z in range(len(pressure_field[0])):
            for x in range(len(pressure_field[0][0])):
                if mask_field[y][z][x] == 0:
                    pressure_field[y][z][x] = 0.0
    return pressure_field
```

**Parameters:**
- `split_threshold`: The fractional position of the boundary (e.g., 0.5 for the middle) [S1, unofficial].
- `threshold` for level sets: A small epsilon value to determine $\phi \ge 0$ vs $\phi < 0$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Using 32-bit floating-point textures for storage yields roughly half the performance of 16-bit textures due to bandwidth issues [S2, unofficial].
- **Boundary Conditions:** Forces such as gravity must not be applied outside the liquid domain to avoid numerical artifacts [S2, unofficial].
- **GPU Flow Control:** Checking the level set value in shaders requires dynamic flow control. If the GPU lacks this, performance deares [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N = xSize \times ySize \times zSize$, as every voxel must be visited to set the binary value.
- **Memory Complexity:** $O(N)$ for storing the 3D array.
- **Performance:** Bandwidth is a major factor. Storing the mask in 16-bit textures improves performance compared to 32-bit [S2, unofficial].
- **Strengths:** Simple to implement; effectively culls empty space in simulations (e.g., air) [S2, unofficial].
- **Limitations:** Hard boundaries can lead to aliasing if not handled carefully; less flexible than continuous fields for blending [S2, unofficial].
- **Alternatives:** For GPUs without dynamic flow control, use z-culling to eliminate cells or sparse data structures (e.g., OpenVDB) instead of a dense binary mask [S2, unofficial].

## Background
The technique is widely used in real-time fluid simulation to define the domain of the solver. It allows the simulation to ignore air ($\phi \ge 0$) and only compute pressure and forces within the liquid ($\phi < 0$) [S2, unofficial]. The approach is detailed in *GPU Gems 3* regarding real-time 3D fluid simulation [S2, unofficial]. The binary generation code snippet is also found in voxel generation tutorials [S1, unofficial].

## Sources
[S1] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
