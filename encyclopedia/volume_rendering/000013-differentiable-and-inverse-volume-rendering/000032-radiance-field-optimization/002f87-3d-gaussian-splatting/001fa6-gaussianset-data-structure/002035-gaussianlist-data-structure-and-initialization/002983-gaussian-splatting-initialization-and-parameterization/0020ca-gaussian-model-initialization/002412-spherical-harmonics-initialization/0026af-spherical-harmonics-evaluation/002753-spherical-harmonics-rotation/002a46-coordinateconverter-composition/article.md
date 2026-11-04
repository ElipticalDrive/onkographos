# CoordinateConverter Composition

Also called: SPZ coordinate conversion pipeline

## Overview
The CoordinateConverter Composition is a composite transformation pipeline used in the SPZ (Splatting Point Cloud) format to convert Gaussian splatting parameters between different coordinate systems, specifically handling the rotation of Spherical Harmonics (SH) coefficients alongside position and quaternion flips. It solves the problem of maintaining visual consistency when the underlying 3D coordinate frame changes (e.g., from a camera-centric view to a world-centric view or vice versa) by applying analytic rotation matrices to SH coefficients and flipping spatial attributes. It is distinct from simple affine transforms because it explicitly manages the coupling between geometric flips and SH band rotations, ensuring that the directional lighting information remains aligned with the rotated geometry.

## How it works
The method constructs a `CoordinateConverter` struct that encapsulates flip vectors for positions, quaternions, and SH coefficients, along with callable functions for applying these transformations. The core logic determines whether a "backward" (rotated-to-standard) or "forward" (standard-to-rotated) conversion is needed, applies specific $\pm \pi/2$ rotations around the X-axis, and composes these with the SH rotation tables.

### Inputs and Outputs
- **Inputs**:
  - `from`: Source coordinate system identifier (integer).
  - `to`: Target coordinate system identifier (integer).
  - `shDegree`: Maximum degree of Spherical Harmonics to process (integer, $0 \le \text{shDegree} \le 4$).
- **Outputs**: A `CoordinateConverter` struct instance containing the necessary flip arrays and function pointers.

### Data Structures
The `CoordinateConverter` struct layout is as follows:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct CoordinateConverter {
    // Flips for position (x, y, z)
    std::array<float, 3> flipP;
    
    // Flips for quaternion (x, y, z); w is never flipped
    std::array<float, 3> flipQ;
    
    // Flips for 24 SH coefficients (covers degrees l=0..3)
    std::array<float, 24> flipSh;
    
    // Per-band SH rotation functions. 
    // SH_MAX_DEGREE is a constexpr int set to 4.
    std::array<std::function<void(float*)>, SH_MAX_DEGREE> rotFlipShFuncs;
    
    // Function to apply flipP and X-axis rotation to position
    std::function<void(float*)> rotFlipPFunc;
    
    // Function to apply flipQ and quaternion rotation to quaternion
    std::function<void(float*)> rotFlipQFunc;
};
```

### Parameters
- `SH_MAX_DEGREE`: A constant integer set to 4, limiting the maximum SH degree handled by `rotFlipShFuncs`.
- `needRotation`: A boolean flag determined by comparing `from` and `to`. If `from == to`, no rotation is needed.

### Numbered Steps
1. **Check Rotation Need**: Determine if `from` equals `to`. If they are identical, return a default `CoordinateConverter` with identity flips and null functions.
2. **Determine Direction**: Check bit 3 of the `from` identifier. If bit 3 is set, the conversion is "backward" (rotated to standard); otherwise, it is "forward" (standard to rotated).
3. **Inner Family Shift**: If a rotation is needed, shift `from` into the same family as `to` by adding or subtracting 8 (`innerFrom = from \pm 8`). This aligns the coordinate systems before applying the specific rotation.
4. **Recursive Call**: Recursively call `coordinateConverter(innerFrom, to, shDegree)` to obtain a base converter. This handles the basic flip mappings between the aligned families.
5. **Compose Rotations**: Bake the $\pm \pi/2$ rotation around the X-axis into `rotFlipPFunc` and `rotFlipQFunc` based on the direction (backward/forward).
6. **SH Rotation Assignment**: Iterate `b` from 0 to `shDegree - 1`. Select the appropriate SH rotation table:
   - Use the "minus" table for backward conversions.
   - Use the "plus" table for forward conversions.
   - Retrieve the rotation function `rotFn = shTable[b]` and store it in `result.rotFlipShFuncs[b]`.

### Formulas and Transformations

**Position Rotation ($R_x$)**:
For a position vector $p = [x, y, z]^T$:
- **Backward (rotated $\to$ standard)**: Apply flipP, then $R_x(-\pi/2)$.
  $$
  \begin{bmatrix} x' \\ y' \\ z' \end{bmatrix} = \begin{bmatrix} 1 & 0 & 0 \\ 0 & 0 & 1 \\ 0 & -1 & 0 \end{bmatrix} \begin{bmatrix} x \\ y \\ z \end{bmatrix} \implies p[1]=z, \ p[2]=-y
  $$
- **Forward (standard $\to$ rotated)**: Apply $R_x(+\pi/2)$, then flipP.
  $$
  \begin{bmatrix} x' \\ y' \\ z' \end{bmatrix} = \begin{bmatrix} 1 & 0 & 0 \\ 0 & 0 & -1 \\ 0 & 1 & 0 \end{bmatrix} \begin{bmatrix} x \\ y \\ z \end{bmatrix} \implies p[1]=-z, \ p[2]=y
  $$

**Quaternion Rotation**:
For a quaternion $q = [w, x, y, z]$ (note: input array is typically $[x, y, z, w]$ or similar, but the source specifies operations on indices 0-3):
Let $s = \sqrt{2}/2$.
- **Backward**: Apply flipQ, then quaternion rotation.
  $$
  \begin{aligned}
  p[0] &= s \cdot (-w + x) \\
  p[1] &= s \cdot (y + z) \\
  p[2] &= s \cdot (-y + z) \\
  p[3] &= s \cdot (w + x)
  \end{aligned}
  $$
- **Forward**: Apply quaternion rotation, then flipQ.
  $$
  \begin{aligned}
  p[0] &= s \cdot (w + x) \\
  p[1] &= s \cdot (y - z) \\
  p[2] &= s \cdot (y + z) \\
  p[3] &= s \cdot (w - x)
  \end{aligned}
  $$
  *Note: The source code indices `p[0]` to `p[3]` correspond to the quaternion components. The exact mapping of $w,x,y,z$ to array indices depends on the internal representation, but the algebraic relations are as stated.*

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Callable, Array

SH_MAX_DEGREE = 4

def coordinate_converter(from_sys: int, to_sys: int, sh_degree: int) -> CoordinateConverter:
    """
    Constructs a CoordinateConverter for transforming between coordinate systems.
    """
    # Base case: same system
    if from_sys == to_sys:
        return CoordinateConverter(identity=True)

    # Determine direction via bit 3
    is_backward = bool(from_sys & 8)
    
    # Shift into same family
    # Note: The exact +/- 8 logic depends on the specific encoding of 'from'
    # Assuming 'from' needs to be aligned to 'to's family
    inner_from = from_sys + 8 if is_backward else from_sys - 8
    
    # Recursive call for base flips
    base_converter = coordinate_converter(inner_from, to_sys, sh_degree)
    
    # Create result
    result = CoordinateConverter()
    result.flipP = base_converter.flipP
    result.flipQ = base_converter.flipQ
    result.flipSh = base_converter.flipSh
    
    # Define rotation functions
    s = math.sqrt(2) / 2
    
    if is_backward:
        # Backward: flipP then R_x(-pi/2)
        def rot_flip_p(p: Array[float]):
            # Apply flipP
            p[0] *= result.flipP[0]
            p[1] *= result.flipP[1]
            p[2] *= result.flipP[2]
            # Apply R_x(-pi/2): y'=z, z'=-y
            y = p[1]
            z = p[2]
            p[1] = z
            p[2] = -y
            
        def rot_flip_q(q: Array[float]):
            # Apply flipQ (w is not flipped, assuming q=[x,y,z,w] or similar)
            # Source says flipQ is for x,y,z. w is never flipped.
            # Assuming q array layout matches the rotation formula indices
            q[0] *= result.flipQ[0]
            q[1] *= result.flipQ[1]
            q[2] *= result.flipQ[2]
            # w is q[3] or similar, not flipped
            
            # Quaternion rotation
            w = q[3] # Assuming w is at index 3 based on formula usage
            x = q[0]
            y = q[1]
            z = q[2]
            
            q[0] = s * (-w + x)
            q[1] = s * (y + z)
            q[2] = s * (-y + z)
            q[3] = s * (w + x)
            
    else:
        # Forward: R_x(+pi/2) then flipP
        def rot_flip_p(p: Array[float]):
            # Apply R_x(+pi/2): y'=-z, z'=y
            y = p[1]
            z = p[2]
            p[1] = -z
            p[2] = y
            # Apply flipP
            p[0] *= result.flipP[0]
            p[1] *= result.flipP[1]
            p[2] *= result.flipP[2]
            
        def rot_flip_q(q: Array[float]):
            # Quaternion rotation
            w = q[3]
            x = q[0]
            y = q[1]
            z = q[2]
            
            q[0] = s * (w + x)
            q[1] = s * (y - z)
            q[2] = s * (y + z)
            q[3] = s * (w - x)
            
            # Apply flipQ
            q[0] *= result.flipQ[0]
            q[1] *= result.flipQ[1]
            q[2] *= result.flipQ[2]
            
    result.rotFlipPFunc = rot_flip_p
    result.rotFlipQFunc = rot_flip_q
    
    # SH Rotation Assignment
    # shTable is a global or passed-in structure containing rotation functions
    # minus_table for backward, plus_table for forward
    table = sh_minus_table if is_backward else sh_plus_table
    
    for b in range(sh_degree):
        if b < SH_MAX_DEGREE:
            result.rotFlipShFuncs[b] = table[b]
        else:
            result.rotFlipShFuncs[b] = lambda x: None # No-op for higher degrees if sh_degree > 4
            
    return result

class CoordinateConverter:
    def __init__(self, identity=False):
        if identity:
            self.flipP = [1.0, 1.0, 1.0]
            self.flipQ = [1.0, 1.0, 1.0]
            self.flipSh = [1.0] * 24
            self.rotFlipShFuncs = [lambda x: None] * SH_MAX_DEGREE
            self.rotFlipPFunc = lambda x: None
            self.rotFlipQFunc = lambda x: None
        else:
            self.flipP = [0.0, 0.0, 0.0]
            self.flipQ = [0.0, 0.0, 0.0]
            self.flipSh = [0.0] * 24
            self.rotFlipShFuncs = [lambda x: None] * SH_MAX_DEGREE
            self.rotFlipPFunc = None
            self.rotFlipQFunc = None
```

### Edge Cases and Pitfalls
- **Precision**: The use of $\sqrt{2}/2$ introduces floating-point approximation. Ensure consistent precision (float32 vs float64) across the rotation and flip steps.
- **Quaternion W Component**: The source explicitly states `w` is never flipped. The pseudocode must ensure `w` is not multiplied by `flipQ`.
- **SH Degree Limit**: `SH_MAX_DEGREE` is 4. If `shDegree` exceeds this, the behavior for bands $b \ge 4$ is undefined in the source. The pseudocode assumes a no-op or error handling.
- **Bit 3 Ambiguity**: The determination of "backward" relies on bit 3 of `from`. If `from` does not follow the expected encoding, the direction may be misinterpreted.

## Subtopics

### CoordinateConverter Composition
- **What it is**: The composite transformation pipeline described above.
- **What sets it apart**: It combines geometric flips with analytic SH rotations in a single callable structure, optimized for the SPZ format's specific coordinate system requirements.
- **When to choose it**: Use when loading or saving SPZ files that require coordinate system conversion, or when integrating Gaussian splatting data from different sources with varying coordinate frames.
- **Link**: [CoordinateConverter Composition](52-coordinate-converter-composition.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ for the construction of the converter, as it involves a fixed number of recursive calls and array initializations. The application of the converter is $O(1)$ per Gaussian, as it involves a fixed number of arithmetic operations and function calls.
- **Memory Complexity**: $O(1)$ per converter instance, storing fixed-size arrays and function pointers.
- **Measured Performance**: Not given in the sources.
- **Strengths**: Efficient, avoids matrix multiplications by using precomputed analytic rotations and flips.
- **Limitations**: Limited to SH degree 4. Specific to the SPZ coordinate system encoding.
- **When to use**: When working with SPZ files or Gaussian splatting data that requires coordinate system alignment.
- **When not**: When using different SH degrees or coordinate systems not covered by the SPZ extensions.

## Background
The CoordinateConverter Composition is part of the SPZ (Splatting Point Cloud) format developed by Niantic. It is used to handle the conversion of Gaussian splatting parameters between different coordinate systems, ensuring that the visual appearance remains consistent across different viewing contexts. The method is documented in the Niantic Labs SPZ repository.

## Sources
[S1] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
