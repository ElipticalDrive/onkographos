# Additive Metaball Modification for Marching Cubes

Also called: Meta-ball addition to Perlin noise, User-defined function modification

## Overview
This method enables dynamic, local modification of a Marching Cubes terrain (such as digging or placing blocks) by superimposing user-defined density functions, typically metaballs, onto a base procedural noise field [S1, unofficial]. It solves the problem of maintaining terrain integrity during interactive editing by avoiding direct alteration of the underlying pseudo-random noise sequence, which would cause catastrophic global artifacts [S1, unofficial]. Unlike siblings that modify the grid topology directly or require complex data structures for dynamic updates, this approach relies on additive mixing of scalar fields, allowing smooth integration of changes without rebuilding the base noise [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A base noise function $N(x, y, z)$ (e.g., Perlin noise), a list of user-defined ball functions, and an isovalue threshold for material selection [S1, unofficial].
- **Output**: A scalar value at any point $(x, y, z)$ used to determine material type or surface presence [S1, unofficial].

**Data Structures**
- `UserBallFunctions`: A class representing a single metaball.
  - Fields:
    - `x0, y0, z0`: Integer coordinates defining the center of the metaball.
    - `ballDensity`: Double-precision floating-point value representing the density parameter.
- `userDefinedFunctions`: A list (or array) of `UserBallFunctions` objects [S1, unofficial].

**Numbered Steps**
1. Initialize an empty list `userDefinedFunctions` to store active metaballs [S1, unofficial].
2. When a user action occurs (e.g., digging), create a new `UserBallFunctions` instance with the specified center $(x_0, y_0, z_0)$ and density $d$ [S1, unofficial].
3. Add the new instance to `userDefinedFunctions` [S1, unofficial].
4. To evaluate the material at a point $(x, y, z)$:
   a. Calculate the base terrain value: $terrainValue = N(x, y, z)$ [S1, unofficial].
   b. Initialize `userModification = 0` [S1, unofficial].
   c. Iterate through each `func` in `userDefinedFunctions`: calculate $v = func.valueAt(x, y, z)$ and add $v$ to `userModification` [S1, unofficial].
   d. Compute the final scalar: $finalValue = terrainValue + userModification$ [S1, unofficial].
   e. Select the material based on $finalValue$ relative to the isovalue [S1, unofficial].

**Formulas**
The value of a single user-defined ball function at point $(x, y, z)$ is calculated as [S1, unofficial]:
$$value(x, y, z) = \frac{ballDensity}{(x-x_0)^2 + (y-y_0)^2 + (z-z_0)^2}$$
The total evaluation function combines the base terrain and user modifications [S1, unofficial]:
$$totalValue(x, y, z) = N(x, y, z) + \sum_{i} value_i(x, y, z)$$
Where $N(x, y, z)$ is the base noise value and the sum iterates over all functions in `userDefinedFunctions` [S1, unofficial].

**Parameters**
- `ballDensity`: The numerator in the ball function formula, controlling the strength of the modification [S1, unofficial].
- `x0, y0, z0`: Integer coordinates for the ball center [S1, unofficial].
- Typical values: Not given in the sources.

**Edge Cases and Pitfalls**
- **High Base Density**: If the base Perlin noise density is very high, subtracting a metaball's density might not drop the total value below the isovalue immediately, resulting in a gradual material transition rather than an immediate void [S1, unofficial]. To fix this, increase the `ballDensity` magnitude, use higher-order powers in the denominator, or change the mixing function from additive (`+`) to a conditional maximum/minimum function [S1, unofficial].
- **Chunk Boundaries**: Metaballs may influence voxels beyond their assigned chunk boundary, causing side-effects in neighboring chunks [S1, unofficial]. Mitigation strategies include using ball functions with bounded support or carefully managing the persistence of metaball definitions per-chunk [S1, unofficial].
- **Center Evaluation**: Evaluating exactly at $(x_0, y_0, z_0)$ results in division by zero [G]. The pseudocode handles this by returning infinity, which effectively guarantees the material is selected at the exact center [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
MATERIAL_SOLID = 1
MATERIAL_AIR = 0

class UserBallFunctions:
    def __init__(self, x0: int, y0: int, z0: int, ballDensity: float):
        self.x0 = x0
        self.y0 = y0
        self.z0 = z0
        self.ballDensity = ballDensity

    def valueAt(self, x: int, y: int, z: int) -> float:
        dx = x - self.x0
        dy = y - self.y0
        dz = z - self.z0
        distSq = dx*dx + dy*dy + dz*dz
        if distSq == 0:
            return float('inf') if self.ballDensity > 0 else float('-inf')
        return self.ballDensity / distSq

def selectAppropriateMaterial(value: float, isovalue: float) -> int:
    return MATERIAL_SOLID if value > isovalue else MATERIAL_AIR

def evaluate_material(x: int, y: int, z: int, isovalue: float, 
                      userDefinedFunctions: list, 
                      perlin_noise_func) -> int:
    terrainValue = perlin_noise_func(x, y, z)
    userModification = 0.0
    for func in userDefinedFunctions:
        userModification += func.valueAt(x, y, z)
    totalValue = terrainValue + userModification
    return selectAppropriateMaterial(totalValue, isovalue)

def add_metaball(x0: int, y0: int, z0: int, density: float, 
                 userDefinedFunctions: list) -> None:
    ball = UserBallFunctions(x0, y0, z0, density)
    userDefinedFunctions.append(ball)
```

## Subtopics

### Bounded Metaballs
Not given in the sources.

### Non-Additive Mixing
Not given in the sources.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: The method is lightweight for sparse modifications, but evaluating many metaballs per voxel can degrade performance [S1, unofficial].
- **Strengths**: Preserves the base terrain structure and allows persistent, dynamic editing without regenerating noise [S1, unofficial].
- **Limitations**: Additive mixing can lead to gradual transitions, and metaballs have infinite support unless bounded [S1, unofficial].
- **When to Use**: Ideal for voxel-based games or simulations where players can dig or place objects in a procedurally generated world [S1, unofficial].
- **When Not to Use**: When strict topological changes are required without gradual material fading, or when performance constraints prevent evaluating many metaballs per voxel [S1, unofficial].

## Background
This technique was discussed in the context of implementing destructible environments in voxel-based games using Marching Cubes [S1, unofficial]. It was proposed as an alternative to modifying the Perlin noise sequence directly, which was noted to be disastrous on a large scale due to the deterministic nature of pseudo-random number generators [S1, unofficial]. The method leverages the additive property of scalar fields to allow user-defined modifications to coexist with procedural generation [S1, unofficial].

## Sources
[S1] Marching cubes with destructible environment, https://gamedev.stackexchange.com/questions/136510/marching-cubes-with-destructible-environment, fetched 2026-10-27 (unofficial)
