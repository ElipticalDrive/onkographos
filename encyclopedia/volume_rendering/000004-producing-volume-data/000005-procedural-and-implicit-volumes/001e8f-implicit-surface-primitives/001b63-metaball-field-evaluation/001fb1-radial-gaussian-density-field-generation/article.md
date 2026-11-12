# Radial Gaussian Density Field Generation
Also called: Caffeine Molecule Volume Generation, Sum of Radial Gaussians

## Overview
Generates a 3D scalar density field for volume rendering by summing normalized radial Gaussian functions centered at predefined atomic positions, then scaling and clamping the result to a $[0,1]$ range [S1, unofficial]. It differs from [Blob Field Generation](2487-blob-field-generation.md) by using smooth Gaussian falloffs rather than polynomial kernels, and from [Fuzzy Metaballs Hyper-parameters](8382-fuzzy-metaballs-hyper-parameters.md) by explicitly defining atom centers and variances for molecular structures rather than abstract fuzzy parameters [S1, unofficial].

## How it works
**Inputs:**
- Volume dimensions: $dimX, dimY, dimZ$ (integers).
- Atom list: A collection of atoms, each defined by center coordinates $(cX, cY, cZ)$ and a variance parameter $\sigma^2$ (or scale).
- Parameters: `postDensityScale` (float, default 2.5), `isoValue` (float, default 0.5, clamped to $[0,1]$).

**Outputs:**
- A 3D volume array of type `uint16_t` (16-bit) or `uint8_t` (8-bit), depending on file size constraints, containing density values in the range $[0, 65535]$ or $[0, 255]$.

**Data Structures:**
The `RadialGaussian` class stores:
- `cX, cY, cZ`: Center coordinates (float).
- `normalization`: Precomputed constant $1.0 / \sqrt{2\pi \sigma^2}$ (float).
- `falloff`: Precomputed constant $-0.5 / \sigma^2$ (float).

**Steps:**
1. Initialize volume dimensions (e.g., $128 \times 128 \times 128$) and allocate memory for the data array.
2. Compute inverse dimensions:
   $$ invDimX = \frac{1.0}{dimX - 1}, \quad invDimY = \frac{1.0}{dimY - 1}, \quad invDimZ = \frac{1.0}{dimZ - 1} $$
3. Define atom centers and their associated variance-derived scales. For the caffeine molecule example, atoms are positioned using a scale factor $s = 0.1$ and offsets $oX=0.5, oY=0.5, oZ=0.5$ applied to normalized coordinates [S1, unofficial].
4. For each voxel $(z, y, x)$:
   a. Compute normalized coordinates:
      $$ nX = x \cdot invDimX, \quad nY = y \cdot invDimY, \quad nZ = z \cdot invDimZ $$
   b. Initialize density $\rho = 0.0$.
   c. For each atom in the list, evaluate its contribution and add to $\rho$:
      $$ \rho \leftarrow \rho + \text{atom.eval}(nX, nY, nZ) $$
   d. Apply global scaling:
      $$ \rho \leftarrow \rho \cdot \text{postDensityScale} $$
   e. Clamp $\rho$ to $[0.0, 1.0]$.
   f. Convert to target bit depth (e.g., `uint16_t`):
      $$ \text{value} = \rho \cdot \text{std::numeric_limits<uint16_t>::max()} $$
   g. Store value in the volume array.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

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
    # Initialize inverse dimensions
    inv_dim_x = 1.0 / (dim_x - 1)
    inv_dim_y = 1.0 / (dim_y - 1)
    inv_dim_z = 1.0 / (dim_z - 1)
    
    # Allocate volume data
    num_voxels = dim_x * dim_y * dim_z
    volume_data = np.zeros(num_voxels, dtype=np.uint16 if bit_depth == 16 else np.uint8)
    
    max_val = 65535 if bit_depth == 16 else 255
    
    for z in range(dim_z):
        for y in range(dim_y):
            for x in range(dim_x):
                # Normalized coordinates
                nx = x * inv_dim_x
                ny = y * inv_dim_y
                nz = z * inv_dim_z
                
                rho = 0.0
                for atom in atoms:
                    rho += atom.eval(nx, ny, nz)
                
                # Scale and clamp
                rho *= post_density_scale
                rho = max(0.0, min(1.0, rho))
                
                # Convert to bit depth
                volume_data[z * dim_y * dim_x + y * dim_x + x] = int(rho * max_val)
                
    return volume_data
```

**Edge Cases and Pitfalls:**
- If `isoValue` is provided outside $[0,1]$, it is clamped to $[0,1]$ (invalid values set to 0) [S1, unofficial].
- Precision issues may arise if variance is extremely small, leading to numerical overflow in the exponential term.
- Boundary conditions: Normalized coordinates range from $0$ to $1$. Atoms placed outside this range will have negligible contribution if the variance is small relative to the distance.

## Subtopics

### [Particle Density Field Computation](4443-particle-density-field-computation.md)
Particle Density Field Computation generates a 3D scalar density field by evaluating a radial kernel function for every voxel in a 3D grid, summing contributions from a set of particles. Unlike [Radial Gaussian Density Field Generation](2081-radial-gaussian-density-field-generation.md), which uses smooth Gaussian falloffs, this method employs a specific polynomial kernel with a sharp cutoff at a normalized squared distance of 0.5, resulting in a compact support that strictly excludes distant particles.

### [Synthetic Active Region Generation](11929-synthetic-active-region-generation.md)
Generates a synthetic bipolar active region magnetogram using Gaussian flux concentrations, satellite flux, PIL shear, diffuse background, and noise. It differs from [Particle Density Field Computation](4443-particle-density-field-computation.md) by explicitly modeling magnetic polarity (positive/negative bipoles) and shear effects rather than scalar particle density.

## Operating characteristics
- **Time Complexity:** $O(V \cdot A)$, where $V$ is the number of voxels and $A$ is the number of atoms. For a $128^3$ grid and ~20 atoms, this is computationally intensive but feasible for pre-computation.
- **Memory Complexity:** $O(V)$ for the volume data. For $128^3$ voxels, this requires $2$ MB (16-bit) or $1$ MB (8-bit).
- **Performance:** The method is deterministic and parallelizable across voxels. The example implementation uses a CPU loop, which may be slow for high-resolution volumes.
- **Strengths:** Smooth, continuous density fields suitable for isosurface extraction (e.g., Dual Marching Cubes). Simple parameterization via atom positions and variances.
- **Limitations:** Not suitable for dynamic scenes with many moving atoms due to recomputation cost. Gaussian falloff has infinite support, though contributions become negligible quickly.
- **When to use:** Static molecular visualization, procedural volume generation for artistic effects.
- **When not to use:** Real-time rendering with dynamic particle systems (consider GPU-based accumulation or [Particle Density Field Computation](4443-particle-density-field-computation.md) with sparse structures).

## Background
Originated from the `dualmc` project by Dominik Wodniok, which demonstrates volume rendering using Dual Marching Cubes [S1, unofficial]. The specific "Caffeine Molecule" example uses atomic coordinates from PubChem to generate a density field representing the caffeine molecule [S1, unofficial]. It serves as a practical example of implicit surface primitives in volume rendering.

## Sources
[S1] dominikwodniok/dualmc, https://github.com/dominikwodniok/dualmc, fetched 2026-10-26 (unofficial)
