# Radial Gaussian Density Field Generation

Also called: Caffeine Molecule Volume Generation, Sum of Radial Gaussians

## Overview
Generates a 3D scalar density field for volume rendering by summing normalized radial Gaussian functions centered at predefined atomic positions, then scaling and clamping the result to a $[0,1]$ range [S1, unofficial]. It solves the problem of creating smooth, continuous volumetric representations of discrete particle systems (such as molecules) from implicit surface primitives. Unlike [Blob Field Generation](2487-blob-field-generation.md), which may use different falloff characteristics, this method specifically employs smooth Gaussian decay, and unlike [Fuzzy Metaballs Hyper-parameters](8382-fuzzy-metaballs-hyper-parameters.md), it relies on fixed analytical Gaussian kernels rather than tunable fuzzy boundaries for the core density definition.

## How it works
**Inputs:**
- Volume dimensions: $dimX, dimY, dimZ$ (integers).
- Set of atoms, where each atom has a center $(cX, cY, cZ)$ and a variance parameter $\sigma^2$ (or scale).
- `postDensityScale`: Global multiplier (default 2.5) [S1, unofficial].
- `isoValue`: Threshold for surface extraction, clamped to $[0,1]$ (default 0.5) [S1, unofficial].

**Outputs:**
- A 3D volume array of type `uint16_t` (or specified bit depth), storing density values scaled to $[0, 65535]$.

**Data Structures:**
The `RadialGaussian` class stores:
- `cX, cY, cZ`: Center coordinates (float).
- `normalization`: Precomputed constant $1.0 / \sqrt{2\pi \sigma^2}$ (float).
- `falloff`: Precomputed constant $-0.5 / \sigma^2$ (float).

**Steps:**
1. Initialize volume dimensions (e.g., $128 \times 128 \times 128$) and allocate memory for `volume.data`. Set `volume.bitDepth` (e.g., 16) [S1, unofficial].
2. Compute inverse dimensions:
   $$ invDimX = \frac{1.0}{dimX - 1}, \quad invDimY = \frac{1.0}{dimY - 1}, \quad invDimZ = \frac{1.0}{dimZ - 1} $$
   [S1, unofficial]
3. Define atom centers and their associated variance-derived scales. For the caffeine molecule example, a global scale $s = 0.1$ and offsets $oX=0.5, oY=0.5, oZ=0.5$ are applied to atomic coordinates [S1, unofficial].
4. For each voxel $(z, y, x)$:
   a. Compute normalized coordinates:
      $$ nX = x \cdot invDimX, \quad nY = y \cdot invDimY, \quad nZ = z \cdot invDimZ $$
      [S1, unofficial]
   b. Initialize density $\rho = 0.0$ [S1, unofficial].
   c. For each atom $i$:
      i. Calculate squared distance from atom center to normalized voxel position:
         $$ d^2 = (nX - cX_i)^2 + (nY - cY_i)^2 + (nZ - cZ_i)^2 $$
         [G] (Inferred from Gaussian definition and usage of `falloff`)
      ii. Evaluate atom contribution:
          $$ \rho \leftarrow \rho + normalization_i \cdot e^{falloff_i \cdot d^2} $$
          [S1, unofficial] (Derived from `normalization` and `falloff` fields and standard Gaussian form)
   d. Apply global scaling:
      $$ \rho \leftarrow \rho \cdot postDensityScale $$
      [S1, unofficial]
   e. Clamp $\rho$ to $[0.0, 1.0]$ [S1, unofficial].
   f. Convert to target bit depth:
      $$ value = \rho \cdot 65535 $$
      [S1, unofficial] (Assuming 16-bit `uint16_t` max)
   g. Store `value` in `volume.data`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class RadialGaussian:
    def __init__(self, cx, cy, cz, variance):
        self.cx = cx
        self.cy = cy
        self.cz = cz
        self.normalization = 1.0 / math.sqrt(2.0 * math.pi * variance)
        self.falloff = -0.5 / variance

    def eval(self, nx, ny, nz):
        dx = nx - self.cx
        dy = ny - self.cy
        dz = nz - self.cz
        dist_sq = dx*dx + dy*dy + dz*dz
        return self.normalization * math.exp(self.falloff * dist_sq)

def generate_radial_gaussian_volume(dim_x, dim_y, dim_z, atoms, post_density_scale=2.5, bit_depth=16):
    # 1. Initialize dimensions and memory
    num_voxels = dim_x * dim_y * dim_z
    volume_data = [0] * num_voxels
    
    # 2. Compute inverse dimensions
    inv_dim_x = 1.0 / (dim_x - 1)
    inv_dim_y = 1.0 / (dim_y - 1)
    inv_dim_z = 1.0 / (dim_z - 1)
    
    # 3. Determine max value for bit depth conversion
    max_val = 65535 if bit_depth == 16 else 255
    
    # 4. Iterate over voxels
    for z in range(dim_z):
        for y in range(dim_y):
            for x in range(dim_x):
                # Normalize coordinates
                nx = x * inv_dim_x
                ny = y * inv_dim_y
                nz = z * inv_dim_z
                
                rho = 0.0
                
                # Sum contributions from all atoms
                for atom in atoms:
                    rho += atom.eval(nx, ny, nz)
                
                # Scale and clamp
                rho *= post_density_scale
                if rho < 0.0:
                    rho = 0.0
                if rho > 1.0:
                    rho = 1.0
                
                # Convert to bit depth
                volume_data[z * dim_y * dim_x + y * dim_x + x] = int(rho * max_val)
                
    return volume_data
```

**Edge Cases and Pitfalls:**
- If `dimX`, `dimY`, or `dimZ` is 1, `invDim` becomes undefined (division by zero). The source assumes dimensions $>1$ [S1, unofficial].
- Invalid `isoValue` (NaN or out of bounds) is clamped to 0.0 [S1, unofficial].
- Precision issues may arise if variance is extremely small, leading to numerical overflow in the exponential term [G].

## Subtopics

### [Particle Density Field Computation](4443-particle-density-field-computation.md)
Particle Density Field Computation generates a 3D scalar density field by evaluating a radial kernel function for every voxel in a 3D grid, summing contributions from a set of particles. Unlike [Radial Gaussian Density Field Generation](2081-radial-gaussian-density-field-generation.md), which uses smooth Gaussian falloffs, this method employs a specific polynomial kernel with a sharp cutoff at a normalized squared distance of 0.5, resulting in a compact support that strictly excludes distant particles.

### [Synthetic Active Region Generation](11929-synthetic-active-region-generation.md)
Synthetic Active Region Generation creates a synthetic bipolar active region magnetogram by summing Gaussian flux concentrations, satellite flux, polarity inversion line (PIL) shear, diffuse background, and instrument noise. It solves the problem of generating realistic, volumetric magnetic field data for visualization by converting parametric tube geometry into a discrete set of 3D points with associated vector field values and intensity. It differs from direct field evaluation methods by explicitly tracing a path with a restoring force and random walk, rather than sampling a static analytical field at grid points.

## Operating characteristics
- **Time Complexity:** $O(V \cdot A)$, where $V$ is the number of voxels and $A$ is the number of atoms. Each voxel requires evaluating the Gaussian function for every atom.
- **Memory Complexity:** $O(V)$ for storing the volume data.
- **Performance:** The example uses a $128^3$ grid, resulting in 2,097,152 voxels. The performance depends heavily on the number of atoms and the efficiency of the exponential function evaluation.
- **Strengths:** Produces smooth, continuous density fields suitable for isosurface extraction (e.g., Dual Marching Cubes). The Gaussian kernel is infinitely differentiable, leading to smooth surfaces.
- **Limitations:** Computationally expensive for large numbers of atoms or high-resolution volumes due to the $O(V \cdot A)$ complexity. The support is theoretically infinite, though contributions decay rapidly.
- **When to use:** Suitable for molecular visualization or any scenario requiring smooth, overlapping density fields from discrete sources.
- **When not to use:** When computational resources are limited and a compact support kernel (like the one in [Particle Density Field Computation](4443-particle-density-field-computation.md)) is preferred for faster culling.

## Background
Originated from the `dualmc` project by Dominik Wodniok, which demonstrates volume rendering and isosurface extraction using the Dual Marching Cubes algorithm [S1, unofficial]. The specific "Caffeine Molecule" example serves as a canonical test case for generating implicit surfaces from atomic coordinates. The method is part of a broader family of procedural volume generation techniques used in scientific visualization and computer graphics.

## Sources
[S1] dominikwodniok/dualmc, https://github.com/dominikwodniok/dualmc, fetched 2026-10-26 (unofficial)
