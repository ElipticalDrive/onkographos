# Octahedral Normal Decoding
## Overview
Decodes a 2D octahedral coordinate into a 3D unit vector [S1, unofficial]. It serves as the inverse operation for octahedral normal encoding, reconstructing surface normals from compressed 2D representations [S1, unofficial]. It differs from [Octahedral Normal Encoding](5029-octahedral-normal-encoding.md) by performing the inverse mapping rather than the forward mapping, and from [Data-Dependent Normal Vector Quantizer (Simulated Annealing)](5865-data-dependent-normal-vector-quantizer-simulated-annealing.md) by using a fixed geometric distribution rather than a data-driven one [S1, unofficial].
## How it works
**Inputs:** A 2D vector `e` representing the octahedral coordinate [S1, unofficial].
**Outputs:** A 3D unit vector `v` [S1, unofficial].

**Data Structures:**
- `e`: `vec2` (float), the encoded coordinate [S1, unofficial].
- `v`: `vec3` (float), the decoded normal [S1, unofficial].

**Steps:**
1. Construct an intermediate vector `v` using the formula: $v = \text{vec3}(e, 1.0 - |e.x| - |e.y|)$ [S1, unofficial].
2. If $v.z < 0$: update $v.xy$ using $v.xy = (1.0 - |e.yx|) \cdot \text{vec2}(\text{sign}(e.x), \text{sign}(e.y))$, where $\text{sign}(a) = 1$ if $a \ge 0$ else $-1$ [S1, unofficial].
3. Calculate the length `len` of `v` [S1, unofficial].
4. If `len > 0`, return `v / len`. Otherwise, return `vec3(0, 0, 1)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def oct_decode(e_x, e_y):
    v_z = 1.0 - abs(e_x) - abs(e_y)
    v = (e_x, e_y, v_z)
    if v[2] < 0:
        sign_x = 1.0 if e_x >= 0 else -1.0
        sign_y = 1.0 if e_y >= 0 else -1.0
        v = ((1.0 - abs(e_y)) * sign_x, (1.0 - abs(e_x)) * sign_y, v[2])
    length = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    if length > 0:
        return (v[0] / length, v[1] / length, v[2] / length)
    return (0.0, 0.0, 1.0)
```

**Parameters:**
- None.

**Edge Cases:**
- If the resulting vector has zero length, the method returns `(0, 0, 1)` [S1, unofficial].
## Subtopics

### [Octahedral Normal Encoding](5029-octahedral-normal-encoding.md)
Compresses a 3D unit normal vector into a single 8-bit integer by identifying the dominant axis and its sign. Choose when memory bandwidth is critical and six discrete octahedral directions are sufficient.

### [Data-Dependent Normal Vector Quantizer (Simulated Annealing)](5865-data-dependent-normal-vector-quantizer-simulated-annealing.md)
Generates a normal vector quantizer optimized for a specific volume dataset using simulated annealing. Choose when shading quality is paramount and hardware limits allow for custom quantization tables.

## Operating characteristics
Time complexity: $O(1)$. Memory complexity: $O(1)$. The method involves basic arithmetic operations and a single normalization step [S1, unofficial]. Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] ReconWorldLab/godot-gaussian-splatting, https://github.com/ReconWorldLab/godot-gaussian-splatting, fetched 2026-10-27 (unofficial)
