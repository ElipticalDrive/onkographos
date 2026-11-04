# Voxel World Generation with Perlin Noise

Also called: Terrain Generation, World Generation

## Overview
This method generates a 3D voxel-based world by evaluating fractal Brownian motion (fBm) noise against a threshold to determine voxel existence and material type (stone, dirt, grass) [S1, unofficial]. It differs from sibling methods like the [Noise Field Generator](1302-noise-field-generator.md) by explicitly mapping noise values to discrete material layers based on vertical position and local neighborhood checks, rather than producing a continuous scalar field or texture [S1, unofficial]. It builds on [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md) to provide the underlying pseudo-random noise function [S1, unofficial].

## How it works
**Inputs:**
- `WorldGenerationData`: Struct containing `arity` (octaves), `gain` (amplitude multiplier per octave), `frequency` (base frequency), `threshold` (noise value cutoff for existence), and `seed` [S1, unofficial].
- `SceneData`: Struct containing world dimensions (`worldWidth`, `worldHeight`), rendering parameters, and camera data [S1, unofficial].
- Voxel storage: A `texture_storage_3d<r32uint, read_write>` buffer where each 32-bit uint stores data for 4 voxels packed in (x,z) pairs [S1, unofficial].

**Outputs:**
- Populated voxel buffer where each voxel is assigned a type ID (0: empty, 1: stone, 3: dirt, 7: grass) [S1, unofficial].

**Data Structures:**
- `WorldGenerationData`:
  - `arity: f32`
  - `gain: f32`
  - `frequency: f32`
  - `threshold: f32`
  - `seed: f32`
- Voxel Packing Layout (within a single `u32`):
  - Bits 0-7: Voxel at local offset (0,0) [S1, unofficial]
  - Bits 8-15: Voxel at local offset (0,1) [S1, unofficial]
  - Bits 16-23: Voxel at local offset (1,0) [S1, unofficial]
  - Bits 24-31: Voxel at local offset (1,1) [S1, unofficial]
  - Each 8-bit slot stores the voxel type ID [S1, unofficial].

**Algorithm Steps:**
1. The compute shader entry point `ComputeMain` is invoked with a workgroup size of (8, 4, 8) [S1, unofficial].
2. For each workgroup, calculate the base voxel position:
   - `voxelPos.x = texelPos.x * 2`
   - `voxelPos.y = texelPos.y`
   - `voxelPos.z = texelPos.z * 2` [S1, unofficial]
3. Loop over a 2x2 block of voxels in the x and z dimensions:
   ```python
   for x in range(2):
       for z in range(2):
           offsetVoxelPos = (voxelPos.x + x, voxelPos.y, voxelPos.z + z)
           noiseValue = GetVoxelNoise(offsetVoxelPos)
           
           # Check existence
           if not (noiseValue > threshold and offsetVoxelPos.y < worldGenerationData.ceiling):
               SetVoxel(offsetVoxelPos, 0) # Empty
               continue
           
           # Determine material
           voxelColor = 1 # Default: Stone
           
           # Check 5 voxels above (y+1 to y+5)
           hasExposedSide = False
           for dy in range(1, 6):
               checkPos = (offsetVoxelPos.x, offsetVoxelPos.y + dy, offsetVoxelPos.z)
               if not (GetVoxelNoise(checkPos) > threshold and checkPos.y < worldGenerationData.ceiling):
                   hasExposedSide = True
                   break
           
           if hasExposedSide:
               voxelColor = 3 # Dirt
           
           # Check for grass
           immediateAbovePos = (offsetVoxelPos.x, offsetVoxelPos.y + 1, offsetVoxelPos.z)
           immediateAboveExposed = not (GetVoxelNoise(immediateAbovePos) > threshold and immediateAbovePos.y < worldGenerationData.ceiling)
           
           if immediateAboveExposed or offsetVoxelPos.y == worldGenerationData.ceiling - 1:
               voxelColor = 7 # Grass
           
           # Pack and store
           SetVoxel(offsetVoxelPos, voxelColor)
   ``` [S1, unofficial]
4. `GetVoxelNoise(pos)` evaluates the fBm noise function using the `WorldGenerationData` parameters [S1, unofficial].
5. `SetVoxel(pos, type)` updates the packed 32-bit value in the storage texture at the appropriate texel and bit position [S1, unofficial].

**Pseudocode for Noise Evaluation:**
The noise function `snoise` is a 3D Perlin-like gradient noise implementation [S1, unofficial]. The fBm summation is defined as:

$$ \text{fbm}(p) = \sum_{i=0}^{n-1} \text{amplitude}_i \cdot \text{snoise}(p \cdot \text{frequency}_i) $$

Where:
- $n$ is `arity` (octaves) [S1, unofficial].
- $\text{frequency}_i = \text{base\_frequency} \cdot \text{lacunarity}^i$ [S1, unofficial].
- $\text{amplitude}_i = \text{gain}^i$ [S1, unofficial].

```python
import math

class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, o): return Vec3(self.x+o.x, self.y+o.y, self.z+o.z)
    def __sub__(self, o): return Vec3(self.x-o.x, self.y-o.y, self.z-o.z)
    def __mul__(self, s): return Vec3(self.x*s, self.y*s, self.z*s)
    def dot(self, o): return self.x*o.x + self.y*o.y + self.z*o.z
    def floor(self): return Vec3(math.floor(self.x), math.floor(self.y), math.floor(self.z))
    def step(self, o): return Vec3(1.0 if self.x>o.x else 0.0, 1.0 if self.y>o.y else 0.0, 1.0 if self.z>o.z else 0.0)
    def min(self, o): return Vec3(min(self.x,o.x), min(self.y,o.y), min(self.z,o.z))
    def max(self, o): return Vec3(max(self.x,o.x), max(self.y,o.y), max(self.z,o.z))
    def abs(self): return Vec3(abs(self.x), abs(self.y), abs(self.z))
    def xzyw(self): return (self.x, self.z, self.y, self.w if hasattr(self,'w') else 0.0)
    def yyy(self): return (self.y, self.y, self.y)
    def xxx(self): return (self.x, self.x, self.x)
    def zxy(self): return (self.z, self.x, self.y)
    def xyz(self): return (self.x, self.y, self.z)

def snoise(v):
    C = (1.0 / 6.0, 1.0 / 3.0)
    D = (0.0, 0.5, 1.0, 2.0)
    i = v.floor()
    x0 = v - i + i * C[0]
    g = x0.step(Vec3(x0.y, x0.x, x0.z))
    l = Vec3(1.0) - g
    i1 = g.min(Vec3(l.z, l.x, l.y))
    i2 = g.max(Vec3(l.z, l.x, l.y))
    x1 = x0 - i1 + C[0]
    x2 = x0 - i2 + C[1]
    x3 = x0 - D[1]
    i = Vec3(i.x % 289, i.y % 289, i.z % 289)
    p = permute(permute(permute(i.z + Vec3(0.0, i1.z, i2.z, 1.0)) + i.y + Vec3(0.0, i1.y, i2.y, 1.0)) + i.x + Vec3(0.0, i1.x, i2.x, 1.0))
    n_ = 0.142857142857
    ns = n_ * D[3] - D[0]
    j = p - 49.0 * math.floor(p * ns * ns)
    x_ = math.floor(j * ns)
    y_ = math.floor(j - 7.0 * x_)
    x = x_ * ns + ns
    y = y_ * ns + ns
    h = Vec3(1.0) - x.abs() - y.abs()
    b0 = Vec3(x[0], x[1], y[0], y[1])
    b1 = Vec3(x[2], x[3], y[2], y[3])
    s0 = math.floor(b0) * 2.0 + Vec3(1.0)
    s1 = math.floor(b1) * 2.0 + Vec3(1.0)
    sh = -h.step(Vec3(0.0))
    a0 = b0.xzyw() + s0.xzyw() * sh.xxyy()
    a1 = b1.xzyw() + s1.xzyw() * sh.zzww()
    p0 = Vec3(a0[0], a0[1], h[0])
    p1 = Vec3(a0[2], a0[3], h[1])
    p2 = Vec3(a1[0], a1[1], h[2])
    p3 = Vec3(a1[2], a1[3], h[3])
    norm = taylorInvSqrt(Vec3(p0.dot(p0), p1.dot(p1), p2.dot(p2), p3.dot(p3)))
    p0 *= norm[0]; p1 *= norm[1]; p2 *= norm[2]; p3 *= norm[3]
    m = max(Vec3(0.5) - Vec3(x0.dot(x0), x1.dot(x1), x2.dot(x2), x3.dot(x3)), Vec3(0.0))
    m = m * m
    return 105.0 * m.dot(Vec3(p0.dot(x0), p1.dot(x1), p2.dot(x2), p3.dot(x3)))

def fbm(pos, octaves, lacunarity, gain, base_frequency, seed):
    amplitude = 1.0
    frequency = base_frequency
    sum_val = 0.0
    p = pos
    for i in range(octaves):
        sum_val += amplitude * snoise_seeded(p * frequency, seed)
        frequency *= lacunarity
        amplitude *= gain
    return sum_val

def SetVoxel(voxelPos, voxelType, texture):
    texelPos = (voxelPos[0] >> 1, voxelPos[1], voxelPos[2] >> 1)
    packedVoxel = texture[texelPos]
    mask = 0xFF
    xOdd = voxelPos[0] & 0x1
    zOdd = (voxelPos[2] & 0x1) << 1
    shiftCount = (xOdd | zOdd) * 8
    mask = mask << shiftCount
    mask = ~mask
    packedVoxel &= mask
    packedVoxel |= voxelType << shiftCount
    texture[texelPos] = packedVoxel
``` [S1, unofficial]

**Parameters:**
- `threshold`: Not given in the sources. [S1, unofficial]
- `ceiling`: Not given in the sources. [S1, unofficial]
- `lacunarity`: Not given in the sources. [S1, unofficial]
- `base_frequency`: Not given in the sources. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Boundaries:** The world generation is bounded by `worldGenerationData.ceiling`. Voxels above this height are always empty [S1, unofficial].

## Subtopics

### [Noise Field Generator](1302-noise-field-generator.md)
Generates a continuous noise field without discrete voxelization. Choose when you need smooth scalar fields for other purposes.

### [3D Perlin-Worley Noise Texture Generation](2355-3d-perlin-worley-noise-texture-generation.md)
Combines Perlin and Worley noise for texture synthesis. Choose when you need hybrid noise patterns for surface details.

## Operating characteristics
- **Time Complexity:** Not given in the sources. [S1, unofficial]
- **Memory Complexity:** Not given in the sources. [S1, unofficial]
- **Measured Performance:** Not given in the sources. [S1, unofficial]
- **Scaling:** Performance scales linearly with world volume. The use of compute shaders allows for parallel generation [S1, unofficial].
- **Strengths:** Efficient generation of large voxel worlds; simple material logic; parallelizable [G].
- **Limitations:** Not given in the sources. [S1, unofficial]
- **When to use:** Not given in the sources. [S1, unofficial]
- **When not to use:** Not given in the sources. [S1, unofficial]

## Background
This method is implemented in the `webgpu-voxel-raymarching` project by James Kiernan [S1, unofficial]. It uses a compute shader to generate the voxel data, which is then rendered using ray marching [S1, unofficial]. The noise function is a 3D Perlin-like gradient noise, and the world generation uses fBm to create natural-looking terrain [S1, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
