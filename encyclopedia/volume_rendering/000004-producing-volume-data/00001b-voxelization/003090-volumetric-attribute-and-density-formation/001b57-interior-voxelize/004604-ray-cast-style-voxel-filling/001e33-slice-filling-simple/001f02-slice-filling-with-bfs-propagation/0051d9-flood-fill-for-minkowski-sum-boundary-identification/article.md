# Flood Fill for Minkowski Sum Boundary Identification

Also called: FloodFill, Algorithm 1 in

## Overview
This method identifies the outer boundary of a Minkowski sum by propagating a "filled" state from the exterior into the interior volume, stopping when blocked by the object's boundary. It builds on [Slice Filling with BFS Propagation](1909-slice-filling-with-bfs-propagation.md) by using a global iterative flood fill rather than slice-local propagation to determine which voxels are outside the Minkowski sum. Unlike Lien's method [S1], which handles internal holes and generates exact boundary representations, this approach is approximate but significantly faster.

## How it works
**Inputs and Outputs:**
- $T_o$: Input/output 3D texture (boolean or integer). Represents the current state of the flood fill. Value 1 indicates a voxel is filled (exterior), 0 indicates empty (interior).
- $T_b$: Input 3D texture. Represents the boundary of the Minkowski sum object. Value 1 indicates a boundary voxel, 0 otherwise.
- Output: The updated $T_o$ containing the identified outer boundary voxels.

**Data Structures:**
- $T_o$: 3D texture of size $N_x \times N_y \times N_z$.
- $T_b$: 3D texture of size $N_x \times N_y \times N_z$.
- $T_{new}$: 3D texture of size $N_x \times N_y \times N_z$, used to track newly filled voxels in the current iteration.
- $T_{mask}$: 3D texture of size $N_x \times N_y \times N_z$, used to identify neighbors of filled voxels.

**Algorithm Steps:**
1. Initialize $T_o$ by subtracting the intersection with the target boundary: $T_o \leftarrow T_o - (T_o \cap T_b)$. This ensures boundary voxels are not marked as filled exterior.
2. Create $T_{new}$ and $T_{mask}$. Clear all voxels of $T_{mask}$ to 0.
3. Identify initial candidates: For all voxels $(i, j, k)$, if at least one neighbor has value 1 in $T_o$, set $T_{mask}(i, j, k) = 1$.
4. Repeat the following loop until convergence:
    a. Clear all voxels of $T_{new}$ to 0.
    b. For all voxels $(i, j, k)$ satisfying $T_{mask}(i, j, k) = 1$:
        i. If $T_o(i, j, k) = 0$ and $T_b(i, j, k) = 0$, then set $T_o(i, j, k) = 1$ and $T_{new}(i, j, k) = 1$.
    c. Clear all voxels of $T_{mask}$ to 0.
    d. Identify next candidates: For all voxels $(i, j, k)$, if at least one neighbor has value 1 in $T_{new}$, set $T_{mask}(i, j, k) = 1$.
    e. Check termination: If $\forall (i, j, k), T_{new}(i, j, k) = 0$, exit loop.
5. Delete $T_{new}$ and $T_{mask}$. Return $T_o$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flood_fill(T_o, T_b, Nx, Ny, Nz):
    # Step 1: Initialize To by removing boundary intersection
    # To <- To - (To ∩ Tb)
    for k in range(Nz):
        for j in range(Ny):
            for i in range(Nx):
                if T_o[i, j, k] == 1 and T_b[i, j, k] == 1:
                    T_o[i, j, k] = 0

    # Step 2: Create Tnew and Tmask, clear Tmask
    T_new = [[0] * Nx for _ in range(Ny)] # Placeholder for 3D, assuming list of lists for brevity in logic
    T_mask = [[0] * Nx for _ in range(Ny)]
    # Note: In implementation, these are 3D arrays. 
    # Clearing T_mask is implicit by initialization to 0.
    
    # Step 3: Initialize Tmask based on neighbors in To
    # For all voxel(i,j,k), if at least one neighbor has value 1 in To, set Tmask(i,j,k) = 1
    for k in range(Nz):
        for j in range(Ny):
            for i in range(Nx):
                has_filled_neighbor = False
                # Check 6-connectivity neighbors
                for di, dj, dk in [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1)]:
                    ni, nj, nk = i + di, j + dj, k + dk
                    if 0 <= ni < Nx and 0 <= nj < Ny and 0 <= nk < Nz:
                        if T_o[ni, nj, nk] == 1:
                            has_filled_neighbor = True
                            break
                if has_filled_neighbor:
                    T_mask[i, j, k] = 1

    # Step 4: Repeat loop
    while True:
        # Clear Tnew
        for k in range(Nz):
            for j in range(Ny):
                for i in range(Nx):
                    T_new[i, j, k] = 0
        
        # Propagate fill
        # For all voxel(i,j,k) satisfying Tmask(i,j,k) = 1, if To(i,j,k) = 0 and Tb(i,j,k) = 0, 
        # then set To(i,j,k) = 1 and Tnew(i,j,k) = 1
        for k in range(Nz):
            for j in range(Ny):
                for i in range(Nx):
                    if T_mask[i, j, k] == 1:
                        if T_o[i, j, k] == 0 and T_b[i, j, k] == 0:
                            T_o[i, j, k] = 1
                            T_new[i, j, k] = 1
        
        # Clear Tmask
        for k in range(Nz):
            for j in range(Ny):
                for i in range(Nx):
                    T_mask[i, j, k] = 0
        
        # Identify next candidates from Tnew
        # For all voxel(i,j,k), if at least one neighbor has value 1 in Tnew, set Tmask(i,j,k) = 1
        for k in range(Nz):
            for j in range(Ny):
                for i in range(Nx):
                    has_new_neighbor = False
                    for di, dj, dk in [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1)]:
                        ni, nj, nk = i + di, j + dj, k + dk
                        if 0 <= ni < Nx and 0 <= nj < Ny and 0 <= nk < Nz:
                            if T_new[ni, nj, nk] == 1:
                                has_new_neighbor = True
                                break
                    if has_new_neighbor:
                        T_mask[i, j, k] = 1
        
        # Termination check
        # Until ∀(i, j, k), Tnew(i, j, k) = 0
        all_zero = True
        for k in range(Nz):
            for j in range(Ny):
                for i in range(Nx):
                    if T_new[i, j, k] != 0:
                        all_zero = False
                        break
                if not all_zero: break
            if not all_zero: break
        
        if all_zero:
            break

    # Step 5: Cleanup and return
    # Delete Tnew and Tmask
    del T_new
    del T_mask
    return T_o
```

**Parameters:**
- Resolution: Typically $512^3$ or $1024^3$ [S1].

**Edge Cases and Pitfalls:**
- Performance is determined by shape complexity. If a large portion of the boundary surface is "invisible" along all orthogonal directions from outside, the flood fill takes more time due to deeper propagation [S1].
- The method is approximate. It does not handle internal holes as accurately as Lien's method [S1].

## Subtopics

### Surface Voxelization from Outer Boundary
A post-processing step to compute the correct surface voxelization by identifying primitive voxels adjacent to the identified outer voxels. A primitive voxel in $T_b$ is part of the final outer boundary surface if it has at least one outer voxel (from $T_o$) as a neighbor [S1].

## Operating characteristics
- **Time Complexity:** Determined by shape complexity and volumetric resolution. For a resolution of $512^3$, it usually takes less than one second for most models [S1]. For a resolution of $1024^3$, the time ranges from a few to tens of seconds, depending on the number of iterations required [S1].
- **Memory Complexity:** Requires storage for $T_o$, $T_b$, $T_{new}$, and $T_{mask}$, all of size $N_x \times N_y \times N_z$.
- **Performance:** The algorithm runs completely on the GPU. It is at least one order of magnitude faster than Lien's method [S1].
- **Strengths:** Fast execution, simple implementation, suitable for high-resolution approximations.
- **Limitations:** Approximate result; does not handle internal holes as well as exact methods; performance degrades with complex shapes having large "invisible" boundary surfaces [S1].

## Background
The method was proposed in a GPU-based voxelization approach to 3D Minkowski sum computation [S1]. It is used in applications including geometric modeling (e.g., offsetting and sweeping), mathematical morphological operations, assembly/disassembly, motion planning, and penetration depth computation [S1]. The authors compared their approach with Lien's method, noting that while Lien's method handles internal holes and generates exact boundary representations, this voxelization approach is faster and achieves relatively high accuracy at $1024^3$ resolution [S1].

## Sources
[S1] A GPU-based voxelization approach to 3D Minkowski sum computation, https://doi.org/10.1145/1839778.1839783, fetched 2026-11-05
