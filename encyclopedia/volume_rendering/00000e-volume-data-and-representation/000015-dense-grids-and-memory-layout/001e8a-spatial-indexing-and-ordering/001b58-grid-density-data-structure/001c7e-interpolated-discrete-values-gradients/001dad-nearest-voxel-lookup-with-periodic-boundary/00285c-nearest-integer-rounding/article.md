# Nearest Integer Rounding
Also called: Round, Round to nearest integer
## Overview
Nearest Integer Rounding rounds a 3D vector to the nearest integer coordinates [S1]. It serves as the fundamental coordinate quantization step in volume rendering, mapping sample points to discrete grid indices [G]. Unlike floor-based indexing which truncates towards zero or negative infinity, this method minimizes spatial quantization error by selecting the nearest grid node [G]. It differs from [Ceil](1574-ceil.md) which always rounds up, and [Floor](1575-floor.md) which always rounds down, by balancing bias in both directions [G].

## How it works
**Inputs:** A 3D vector `xyz` of type `Vec3<double>`, `Vec3<float>`, or `Vec3<RealT>` [S1].
**Outputs:** A coordinate type `CoordT` representing the rounded integer coordinates [S1].

**Data Structures:**
The input is a 3-element vector [S1]. The output is a scalar or vector of integer type, depending on the template specialization [S1].

**Steps:**
1. For each component $c \in \{x, y, z\}$ of the input vector $xyz$:
   - Let $v = xyz.c$.
   - Compute the rounded integer $r = \text{round}(v)$.
   - Cast $r$ to `CoordT` [S1].

**Formulas:**
$$ r_c = \text{round}(xyz_c) $$
where $r_c$ is the integer result for component $c$ [S1].

**Parameters:**
- `xyz`: The input floating-point coordinates [S1].
- `CoordT`: The target integer type for the output [S1].
- `RealT`: The floating-point type of the input (double, float, etc.) [S1].

**Edge Cases and Pitfalls:**
- The source does not specify tie-breaking rules, overflow handling, or negative zero behavior [S1]. Standard floating-point rounding applies [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def Round(xyz, CoordT=int):
    """
    Rounds a 3D vector to the nearest integer coordinates.
    Args:
        xyz: Input vector of type Vec3<double>, Vec3<float>, or Vec3<RealT>.
        CoordT: Target integer type for the output.
    Returns:
        A tuple of three integers representing the rounded coordinates.
    """
    return (CoordT(round(xyz[0])), CoordT(round(xyz[1])), CoordT(round(xyz[2])))
```

## Subtopics

### Ceil
Returns the ceiling of a floating-point value as an `int32_t` [S1]. It always rounds up to the nearest integer greater than or equal to the input [S1]. Defined for both `double` and `float` inputs [S1]. Use when you need to ensure coverage of the entire voxel space without gaps, such as in bounding box calculations or when sampling requires including the upper boundary [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per component [G].
- **Memory Complexity:** $O(1)$ [G].
- **Performance:** Extremely fast, typically mapping to a single CPU instruction or a highly optimized library call [G].
- **Strengths:** Minimal quantization error compared to floor/ceil; simple implementation [G].
- **Limitations:** Does not handle out-of-bounds indices (that is handled by the parent [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)); tie-breaking behavior may vary across platforms if not explicitly standardized [G].

## Background
The `Round` and `Ceil` functions are part of the `nanovdb::math` namespace in OpenVDB, a library for sparse volumetric data [S1]. They are designed for high-performance GPU and CPU execution, indicated by the `__hostdev__` qualifiers [S1]. The implementation is templated to support various floating-point precisions and coordinate types, ensuring flexibility in volume rendering pipelines that may use single or double precision [S1].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
