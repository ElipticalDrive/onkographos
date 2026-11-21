# Nearest Integer Rounding

Also called: Round, Round to nearest integer

## Overview
Nearest Integer Rounding rounds a 3D vector to the nearest integer coordinates [S1]. It provides the fundamental coordinate quantization step used by [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) to map continuous sample points to discrete grid indices.

## How it works
**Inputs and Outputs:**
- Input: A 3D vector `xyz` of type `Vec3T<RealT>`, where `RealT` is `float`, `double`, or a generic real type [S1].
- Output: A coordinate structure `CoordT` containing the rounded integer components [S1].

**Data Structures:**
- `Vec3T<RealT>`: A template class representing a 3D vector with components `x`, `y`, `z` [S1].
- `CoordT`: A coordinate type returned by the function, typically representing integer grid indices [S1].

**Algorithm:**
The operation is performed component-wise on the input vector [S1].
1. Extract each component $v_c$ from `xyz` for $c \in \{x, y, z\}$.
2. Apply the standard rounding function to $v_c$ to obtain the nearest integer $n_c$.
3. Assemble the three rounded components into a `CoordT` structure and return it [S1].

**Formula:**
$$ n_c = \text{round}(v_c) $$
where $n_c$ is the integer result for component $c$, and $v_c$ is the input floating-point value [S1].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import NamedTuple

class Coord(NamedTuple):
    x: int
    y: int
    z: int

def Round(xyz: tuple[float, float, float]) -> Coord:
    """
    Rounds a 3D vector to the nearest integer coordinates.
    Supports float and double precisions.
    """
    i = round(xyz[0])
    j = round(xyz[1])
    k = round(xyz[2])
    return Coord(i, j, k)
```

## Subtopics

### Ceil
Returns the ceiling of a floating-point value as an `int32_t` [S1]. Defined for both `float` and `double` inputs [S1].

### Fract
Returns the fractional part of a floating-point value [S1]. Defined for both `float` and `double` inputs [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
This method is defined in the `nanovdb::math` namespace of OpenVDB [S1]. The function is implemented as a template accepting a coordinate type `CoordT` and a vector type `Vec3T<RealT>`, with overloads for `float`, `double`, and generic `RealT` types [S1].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
