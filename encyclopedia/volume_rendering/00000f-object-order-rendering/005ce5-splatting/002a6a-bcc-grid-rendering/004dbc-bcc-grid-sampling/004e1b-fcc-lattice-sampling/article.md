# FCC Lattice Sampling

Also called: FCC dataset lattice, FCC lattice, Face Centered Cubic (FCC) Lattice, Face-centered cartesian lattice

## Overview
FCC Lattice Sampling is a 3D sampling method that defines a volume grid as a subset of the Cartesian lattice, offering superior sampling efficiency and optimality for signals sampled at a depressed rate compared to standard Cubic Cartesian (CC) grids [S1]. It is distinguished from its sibling [BCC Grid Sampling](13572-bcc-grid-sampling.md) by its optimality conditions: while BCC is optimal for band-limited and isotropic signals, FCC is optimal when the signal spectrum is spherical or when sampling rates are depressed [S1][S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D signal (volume data) defined over space [S1].
*   **Output:** A set of sample points located on the FCC lattice, and the corresponding scalar values at those points [S1].

**Data Structures:**
The lattice is defined by a generator matrix $G$ or an integer constraint [S1].
*   **Generator Matrix ($G$):** A $3 \times 3$ matrix where columns are the basis vectors of the lattice [S1].
    $$ G = \begin{bmatrix} 0 & 1 & 1 \\ 1 & 0 & 1 \\ 1 & 1 & 0 \end{bmatrix} $$
*   **Integer Constraint:** A point $(x, y, z)$ belongs to the lattice if $x, y, z \in \mathbb{Z}$ and $x + y + z$ is even [S1].

**Parameters:**
*   **Generator Matrix $G$:** Must be non-singular ($\det(G) \neq 0$) [S1]. For the standard FCC lattice, $\det(G) = 2$ [G].
*   **Applicability Condition:** The method is most effective if the frequency spectrum of the signal is spherical [S2, book].

**Steps:**
1.  **Define Lattice Basis:** Choose the generator matrix $G$ for the FCC lattice [S1].
2.  **Generate Sample Coordinates:** For every integer vector $\mathbf{v} = [i, j, k]^T \in \mathbb{Z}^3$, compute the physical coordinate $\mathbf{p} = G \cdot \mathbf{v}$ [S1]. Alternatively, iterate over integer coordinates $(x, y, z)$ and include the point only if $(x + y + z) \pmod 2 = 0$ [S1].
3.  **Sample Signal:** Evaluate the volume signal at each generated coordinate $\mathbf{p}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_fcc_lattice_points(bounding_box_min, bounding_box_max, step_size=1.0):
    """
    Generates coordinates for an FCC lattice within a bounding box.
    Uses the integer constraint definition: x + y + z is even.
    """
    points = []
    
    # Determine integer range covering the bounding box
    min_x = math.floor(bounding_box_min[0] / step_size)
    max_x = math.ceil(bounding_box_max[0] / step_size)
    min_y = math.floor(bounding_box_min[1] / step_size)
    max_y = math.ceil(bounding_box_max[1] / step_size)
    min_z = math.floor(bounding_box_min[2] / step_size)
    max_z = math.ceil(bounding_box_max[2] / step_size)
    
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            for z in range(min_z, max_z + 1):
                # FCC condition: sum of coordinates must be even
                if (x + y + z) % 2 == 0:
                    # Convert integer lattice index to physical space
                    points.append((x * step_size, y * step_size, z * step_size))
                    
    return points
```

**Edge Cases and Pitfalls:**
*   **Boundary Handling:** The integer constraint method requires careful clamping to ensure points fall exactly within the desired bounding box [G].
*   **Precision:** When converting between the generator matrix representation and the integer constraint representation, floating-point errors may occur [G]. The integer constraint method is generally more robust for storage and indexing [G].
*   **Degenerate Input:** If the signal is not band-limited or has a non-spherical spectrum, the sampling efficiency advantage over CC grids may diminish [S1][S2, book].

## Subtopics

### [Normalized Free-Space Water Field](20524-normalized-free-space-water-field.md)
A water field defined by fixed Gaussian centers on an FCC lattice with fixed covariance, optimizing only two non-negative RGB density vectors per center (direct and backscatter attenuation) to prevent transport mass changes via geometric resizing [G].

## Operating characteristics
*   **Time Complexity:** Sampling time is proportional to the number of lattice points within the volume [G]. The density of FCC points is half that of a CC grid with the same spacing, reducing the number of samples by approximately 50% compared to CC for equivalent spatial coverage, though the exact savings depend on the specific implementation and bounding box alignment [G].
*   **Memory Complexity:** Storage is reduced compared to CC grids. The FCC lattice provides the densest packing of equal-sized spheres, making it space-efficient for storage [S2, book].
*   **Performance:** GPU ray casting implementations for FCC datasets have shown superior performance due to efficient empty space skipping and spline value evolution [S1].
*   **Strengths:** Better sampling efficiency than Cartesian lattices; optimal for signals sampled at a depressed rate; densest sphere packing [S1][S2, book].
*   **Limitations:** Not optimal for all signal types (specifically, BCC is preferred for band-limited isotropic signals) [S1].
*   **When to use:** Medical and scientific datasets, GPU ray casting, mobile volume rendering where memory and bandwidth are constrained [S1].

## Background
The FCC lattice was introduced as a more efficient alternative to Cartesian lattices for volume rendering [S1]. Kim implemented a real-time GPU iso-surface ray caster for FCC datasets, demonstrating the practical benefits of this approach [S1]. The theoretical basis for its optimality in specific signal conditions was indicated by Künsch et al. [S1]. The FCC lattice is the dual of the BCC lattice in the spatial domain [S2, book].

## Sources
[S1] Real-Time Volume Rendering of FCC Datasets Using Box-Splines on the Mobile Platforms, https://hal.science/hal-05467237v1, fetched 2026-10-27
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
