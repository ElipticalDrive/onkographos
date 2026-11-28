# Granularity Calculation Formula

Also called: Adaptive granularity calculation, Screen cell size calculation

## Overview
A method to calculate the optimal screen cell size ($g$) for parallel volume rendering by approximating the 3D volume projection as an $n \times n$ square and dividing it into $g \times g$ pixel cells to achieve a target total number of screen cells ($C$) [S1, unofficial]. It differs from static partitioning by dynamically recalculating the workload distribution based on the current camera view and volume projection, using hypergraph partitioning to minimize communication overhead during cluster migration [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** $n$, the side length of the square approximation of the projection area in pixels; $C$, the target total number of screen cells.
*   **Output:** $g$, the side length of each screen cell in pixels.

**Data Structures:**
No complex data structures are required. The method operates on scalar values.

**Steps:**
1.  Approximate the projection area of the 3D volume on the screen as a square of size $n \times n$ pixels [S1, unofficial].
2.  Determine the best-case number of screen cells required to cover the $n \times n$ area using $g \times g$ cells, which is $\lceil n/g \rceil^2$ [S1, unofficial].
3.  Determine the worst-case number of screen cells, calculated as $\lceil n/g \rceil^2 + \lceil n/g \rceil + \lceil n/g \rceil + 4$ [S1, unofficial].
4.  Calculate the average of the best-case and worst-case cell counts and equate it to the fixed total screen cell number $C$ [S1, unofficial].
5.  Solve the resulting second-order equation for $g$ [S1, unofficial].

**Formulas:**
The relationship between $n$, $g$, and $C$ is defined by the second-order equation [S1, unofficial]:

$$ (2 - C)g^2 + (4n - 8)(n + 2g) - 4n^2 + 4 = 0 $$

Note: The source text contains garbled symbols where 'C' appears as '(7' or 'C', 'n' as 'n' or 'n/', and coefficients may be misprinted (e.g., '471' likely '4n', '47?' likely '4n^2'). The equation is derived from averaging the best/worst case cell counts [S1, unofficial].

Solving the quadratic equation for $g$ yields [S1, unofficial]:

$$ g = \frac{\sqrt{1 - 2C + \sqrt{(C - 1)A - 2C}}}{C - 2} + \frac{2C}{C - 2} $$

Substitutions used: $n = \sqrt{A}$ and $C = A$. Note: The source text is garbled; 'A' is used as a substitution variable for $n^2$ or similar, and the formula structure is ambiguous. The text states: 'substituted, $n = \sqrt{A}$ and $= A$ into that equation' [S1, unofficial].

**Parameters:**
*   $C$: The target total number of screen cells. Typical value: 400 [S1, unofficial].
*   $n$: The side length of the square approximation of the projection area in pixels [S1, unofficial].
*   $g$: The side length of each screen cell in pixels, calculated as the output of the formula [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The formula is an approximation that assumes a square projection area, whereas actual projection areas are usually non-square and complex regions [S1, unofficial].
*   The source text contains garbled symbols and ambiguous variable substitutions in the final formula for $g$. Implementers should verify the derivation from the averaged cell count equation if precision is critical [S1, unofficial].
*   Division by zero occurs if $C = 2$ in the final formula.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_granularity(n, C):
    """
    Calculate the optimal screen cell size g.
    
    Args:
        n (float): Side length of the square approximation of the projection area in pixels.
        C (float): Target total number of screen cells.
        
    Returns:
        float: The side length of each screen cell in pixels.
    """
    # Note: The source formula for g is garbled. 
    # The derivation starts from averaging best/worst case cells equated to C.
    # Best case cells: ceil(n/g)^2
    # Worst case cells: ceil(n/g)^2 + 2*ceil(n/g) + 4
    # Average: ceil(n/g)^2 + ceil(n/g) + 2 = C
    # This leads to a quadratic in terms of k = ceil(n/g).
    # However, the source provides a direct quadratic in g:
    # (2 - C)g^2 + (4n - 8)(n + 2g) - 4n^2 + 4 = 0
    
    # Coefficients for quadratic equation a*g^2 + b*g + c = 0
    # Expanding the source equation:
    # (2 - C)g^2 + (4n - 8)n + (4n - 8)2g - 4n^2 + 4 = 0
    # (2 - C)g^2 + (4n - 8)2g + (4n^2 - 8n - 4n^2 + 4) = 0
    # (2 - C)g^2 + (8n - 16)g + (4 - 8n) = 0
    
    a = 2 - C
    b = 8 * n - 16
    c = 4 - 8 * n
    
    # Handle edge case where a is 0 (C=2)
    if abs(a) < 1e-9:
        # Linear equation: b*g + c = 0
        if abs(b) < 1e-9:
            return 0 # No solution or infinite solutions
        return -c / b
    
    # Quadratic formula: g = (-b + sqrt(b^2 - 4ac)) / (2a)
    # We take the positive root for physical size
    discriminant = b**2 - 4 * a * c
    
    if discriminant < 0:
        # No real solution, return a default or handle error
        # Based on source, this shouldn't happen for valid inputs
        return 0 
        
    sqrt_disc = math.sqrt(discriminant)
    
    # Two roots
    g1 = (-b + sqrt_disc) / (2 * a)
    g2 = (-b - sqrt_disc) / (2 * a)
    
    # Select the positive root
    if g1 > 0:
        return g1
    elif g2 > 0:
        return g2
    else:
        # Both roots negative, return positive of the one with smaller magnitude?
        # Or return 0. Source doesn't specify.
        return 0

# Example usage from source:
# For screen size 400x400, target C=400, n=400
# g = 15, resulting in actual cells C'=401 with error 0.25
print(calculate_granularity(400, 400))
```

## Subtopics

### [View Dependent Preprocessing](23471-view-dependent-preprocessing.md)
View Dependent Preprocessing is a parallel volume rendering optimization technique that dynamically partitions the screen and volume data to balance computational load across processors for each visualization instance. It differs from static partitioning by recalculating the workload distribution based on the current camera view and volume projection, using hypergraph partitioning to minimize communication overhead during cluster migration.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the calculation of $g$ given $n$ and $C$.
*   **Memory Complexity:** $O(1)$.
*   **Measured Performance:** For a screen size of 400x400, target $C=400$, the calculated $g$ is 15, resulting in actual cells $C'=401$ with error 0.25 [S1, unofficial]. For a screen size of 800x800, target $C=400$, the calculated $g$ is 31, resulting in actual cells $C'=375$ with error 6.25 [S1, unofficial]. For a screen size of 1200x1200, target $C=400$, the calculated $g$ is 46, resulting in actual cells $C'=388$ with error 3.00 [S1, unofficial].
*   **Scaling:** Linear increase in $g$ with increasing screen size [S1, unofficial].
*   **Strengths:** Produces good results despite being an approximation [S1, unofficial].
*   **Limitations:** Assumes a square projection area, whereas actual projection areas are usually non-square and complex regions [S1, unofficial].
*   **When to use:** When dynamic partitioning of the screen is required for parallel volume rendering and a target number of screen cells is desired [S1, unofficial].
*   **When not to use:** When high precision in cell count is required or when the projection area is significantly non-square [S1, unofficial].

## Background
The method is described in Appendix A of a source related to hypergraph-partitioning based remapping models for image-space parallel volume rendering [S1, unofficial]. The source references Par.sytec GmbH, Germany, Embedded Parix (EPX) version 1.9.2 User's Guide and Programmers Reference Manual, 1996 [S1, unofficial].

## Sources
[S1] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
