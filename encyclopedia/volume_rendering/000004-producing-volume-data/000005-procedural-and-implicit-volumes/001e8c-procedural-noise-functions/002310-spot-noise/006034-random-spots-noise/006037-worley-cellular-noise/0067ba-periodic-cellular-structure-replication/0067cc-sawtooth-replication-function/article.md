# Sawtooth Replication Function
Also called: FRep Cellular Structure Replication
## Overview
The Sawtooth Replication Function is a basic periodic replication function used to generate infinite cellular structures from a unit cell defined by a continuous real function [S1]. It is a core component of the [Periodic Cellular Structure Replication](19927-periodic-cellular-structure-replication.md) method [S1]. Due to its nature, the resulting cellular model exhibits geometric discontinuity on cell faces if the unit cell lacks connectivity, and can be $C^0$-discontinuous even with geometric connectivity; full unit cell connectivity is required to obtain a $C^0$-continuous function [S1].
## How it works
**Inputs and Outputs:**
- Input: A real-valued coordinate $t$.
- Output: A real-valued periodic coordinate $g(t)$.

**Data Structures:**
- Not given in the sources.

**Parameters:**
- $a$: The period of the sawtooth wave [S1].

**Formula:**
The replication function $g(t)$ is defined as:
$$g(t) = \frac{1}{2} + \frac{1}{a} \left( t - \text{floor}\left(\frac{t}{a} + \frac{1}{2}\right) \right)$$
where $\text{floor}(x)$ returns the greatest integer less than or equal to $x$ [S1].

**Steps:**
1. Compute the shifted normalized position: $u = \frac{t}{a} + \frac{1}{2}$ [S1].
2. Compute the integer part: $k = \text{floor}(u)$ [S1].
3. Compute the remainder: $r = t - k \cdot a$ [S1].
4. Compute the output: $g(t) = \frac{1}{2} + \frac{r}{a}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sawtooth_replication(t: float, a: float) -> float:
    """
    Computes the sawtooth replication function value.
    
    Args:
        t: Input coordinate.
        a: Period of the sawtooth wave.
        
    Returns:
        The replicated coordinate value.
    """
    shifted = t / a + 0.5
    k = math.floor(shifted)
    remainder = t - k * a
    result = 0.5 + remainder / a
    return result
```

**Edge Cases and Pitfalls:**
- **Discontinuity:** The resulting cellular model has geometric discontinuity on the faces of the cells if the unit cell has no connectivity property [S1].
- **C0 Discontinuity:** Even if the unit cell has geometric connectivity, the resulting function can be $C^0$-discontinuous on the faces of each cell [S1]. Full unit cell connectivity is required to obtain $C^0$-continuous functions [S1].
- **Precision:** The use of `floor` can introduce precision issues near boundaries where $t/a + 1/2$ is an integer. Standard floating-point arithmetic should be used, but care must be taken with boundary conditions in high-precision rendering [G].
## Subtopics

### [Triangle Wave Replication Function](20002-triangle-wave-replication-function.md)
The Triangle Wave Replication Function is a mathematical tool used to create $C^1$-continuous cellular structures. It sets itself apart from the Sawtooth Replication Function by ensuring smooth first-derivative continuity, provided the unit cell geometry is symmetric with respect to planes parallel to the cell faces and passing through their centers. It is chosen when higher continuity is required compared to the basic sawtooth function. The formula is not given in the sources.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Provides a basis for generating infinite periodic structures [S1].
- **Limitations:** Results in geometric discontinuity on cell faces if the unit cell lacks connectivity, and can be $C^0$-discontinuous even with geometric connectivity [S1].
- **When to use:** When generating basic cellular structures where $C^0$ continuity is acceptable or when full connectivity is present [S1].
- **When not to use:** When $C^1$ continuity is required or when the unit cell lacks connectivity [S1].
## Background
The Sawtooth Replication Function is described in the context of procedural function-based modelling of volumetric microstructures [S1]. It is part of a family of replication functions used to create infinite cellular structures by applying periodic replication to a unit cell [S1].
## Sources
[S1] Procedural function-based modelling of volumetric microstructures, https://doi.org/10.1016/j.gmod.2011.03.001, fetched 2026-10-26
