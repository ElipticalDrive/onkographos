# Random Packed-Sphere Porous Geometry Generator


## Overview
This method generates voxelized representations of porous media by packing spheres into a volume, either with fixed radii or variable radii to match a target porosity. It differs from standard binary voxelization by incorporating a distance-rejection check for sphere placement and a subsequent grid-averaging step to smooth the discrete sphere boundaries, producing a more physically realistic porous structure. It builds on [Porous Voxelization](4361-porous-voxelization.md) by providing a specific geometric primitive (spheres) and placement algorithm.

## How it works
**Inputs:**
- Domain dimensions ($L_x, L_y, L_z$) in voxels.
- Voxel size ($\Delta x$).
- Method selection (1: Constant Size, 2: Variable Size).
- For Method 1: Number of spheres ($N$), fixed radius ($R$).
- For Method 2: Radius range $[r_{min}, r_{max}]$, overlap range $[o_{min}, o_{max}]$, target solid volume fraction ($\phi_{target}$).

**Outputs:**
- Voxel grid: 3D array where value 0 represents solid (inside sphere) and value 1 represents pore space [S1, unofficial].
- Metadata: List of sphere centers and radii.
- File: Legacy VTK `STRUCTURED_POINTS` volume format [S1, unofficial].

**Data Structures:**
- `Grid`: 3D array of integers, size $L_x \times L_y \times L_z$.
- `Spheres`: List of tuples `(x, y, z, r)`.
- `C`: Distance-rejection array (implementation detail for Method 1).

**Steps:**

1. **Initialize Grid:**
   Set all voxels in `Grid` to 1 (pore space).

2. **Sphere Placement:**
   - **Method 1 (Constant Sphere Size):**
     1. Repeat $N$ times:
        a. Generate random center $(x, y, z)$ within domain bounds.
        b. Check for overlap with all previously placed spheres using a distance-rejection check against array `C`. If distance between new center and any existing sphere center is less than $2R$, reject and retry.
        c. If valid, add sphere to `Spheres` and update `C`.
   - **Method 2 (Variable Sphere Size):**
     1. Initialize `current_solid_volume = 0`.
     2. Loop indefinitely:
        a. Generate random radius $r \in [r_{min}, r_{max}]$.
        b. Generate random center $(x, y, z)$.
        c. Check for allowed overlap with existing spheres. Overlap $o$ must be in $[o_{min}, o_{max}]$. Overlap is defined as the intersection volume or distance metric (specific formula not given in sources) [S1, unofficial].
        d. If valid, add sphere to `Spheres`.
        e. Update `current_solid_volume` by adding volume of new sphere minus overlap.
        f. If `current_solid_volume / total_domain_volume >= phi_target`, break loop.

3. **Voxelization:**
   For each sphere in `Spheres`:
   1. Determine bounding box of sphere in voxel coordinates.
   2. For each voxel $(i, j, k)$ in bounding box:
      a. Calculate center of voxel $(x_v, y_v, z_v)$.
      b. Calculate distance $d$ from $(x_v, y_v, z_v)$ to sphere center.
      c. If $d \le r$, set `Grid[i, j, k] = 0` (solid).

4. **Grid Averaging:**
   Apply a smoothing filter to `Grid` to reduce stair-stepping artifacts. The specific averaging kernel or number of iterations is not given in the sources [S1, unofficial].

5. **Output:**
   Write `Grid` to VTK `STRUCTURED_POINTS` format. Write sphere centers and radii to separate text files.

**Parameters:**
- Voxel Convention: 0 = solid, 1 = pore [S1, unofficial].
- Build Command: `gfortran -O2 -o porous_geometry porous_geometry.f90 && ./porous_geometry` [S1, unofficial].

**Edge Cases:**
- **Method 1:** If $N$ is too large for the domain, placement may fail to converge. Not given in the sources.
- **Method 2:** If target porosity is too low, the loop may run indefinitely. Not given in the sources.
- **Precision:** Floating-point precision in distance calculations may cause slight overlaps or gaps. Not given in the sources.

## Operating characteristics
- **Time Complexity:** Method 1 is $O(N^2)$ due to pairwise distance checks. Method 2 is unbounded in worst case, depending on target porosity.
- **Memory Complexity:** $O(L_x L_y L_z)$ for the grid, plus $O(N)$ for sphere metadata.
- **Strengths:** Generates realistic porous structures; supports variable porosity.
- **Limitations:** Slow for large $N$; grid averaging may blur fine details.
- **When to use:** When simulating flow through porous media with spherical grains.
- **When not to use:** When exact sphere positions are required without smoothing.

## Background
This method is implemented in a Fortran-based algorithm described in a GitHub pull request [S1, unofficial]. It is designed for volume rendering and simulation of porous geometries. The output is compatible with ParaView and VisIt [S1, unofficial].

## Sources
[S1] Add Fortran generator for random packed-sphere porous geometry, https://github.com/AMMU1986/AMMAN/pull/144, fetched 2026-10-27 (unofficial)
