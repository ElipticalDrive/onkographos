# Primary Sample Space Transformation Layout

Also called: Array of Arrays for Primary Samples, BDPT Primary Sample Layout

## Overview
A data layout for storing the primary sample vector $\bar{u}$ in Bidirectional Path Tracing (BDPT) to ensure that small changes in the sample space $U$ correspond to small changes in the path space $P$ [S1]. It solves the problem of sample correlation in heterogeneous participating media by separating random numbers for eye/light subpaths and between scattering/propagation steps, unlike standard BDPT which uses successive dimensions of $U$ directly [S1].

## How it works
**Inputs:** A primary sample vector $\bar{u}$, typically generated from a low-discrepancy sequence or pseudo-random number generator.
**Outputs:** A structured layout (array of arrays) mapping indices to specific sampling operations within the BDPT algorithm.

**Data Structure:**
The primary sample $\bar{u}$ is stored as an array of arrays (list of lists).
- The $i$-th row vector provides input to the sampling done at the $|i|$-th vertex of the respective subpath.
- Elements within each row are accessed in sequential order.
- Indices are assigned as follows:
  - Positive indices ($i > 0$): Eye subpath vertices.
  - Negative indices ($i < 0$): Light subpath vertices.
  - Index $0$: Transmittance estimation.

**Numbered Steps:**
1. Initialize the primary sample structure as an empty array of arrays.
2. For each vertex generation step in the BDPT algorithm:
   a. Determine the vertex index $i$ based on the subpath (eye or light) and the vertex order.
   b. Retrieve the row vector corresponding to index $i$.
   c. Consume random values from the row vector sequentially for the specific sampling operation (e.g., distance sampling, direction sampling).
   d. If the row is exhausted, append new random values to the row [G].
3. For transmittance estimation between deterministic connections:
   a. Use the row vector at index $0$.
   b. Consume random values sequentially.
   c. No further separation within index $0$ is required due to the smoothness of transmittance [S1].

**Formulas:**
Let $P$ be the path space and $U$ be the primary sample space. The transformation $s: U \rightarrow P$ must satisfy:
$$ \|\Delta U\| \text{ small} \implies \|\Delta P\| \text{ small} $$
This is achieved by ensuring that random numbers driving distinct parts of path generation are separated [S1].

**Parameters:**
- Index mapping: Positive for eye, negative for light, 0 for transmittance.
- Access pattern: Sequential within each row.

**Edge Cases and Pitfalls:**
- **Heterogeneous Media:** The amount of random input needed for distance sampling cannot be determined in advance. The array-of-arrays layout handles this by allowing variable-length rows [S1].
- **Transmittance Estimation:** Each deterministic connection between subpaths requires an additional unknown amount of random values for transmittance estimation. The layout accommodates this by using index $0$ [S1].
- **Transmittance Smoothness:** Large variations in random values driving transmittance estimation have little effect on the result because transmittance is fairly smooth in realistic settings. Therefore, further separation within the transmittance row (index 0) is not necessary [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class PrimarySampleSpaceLayout:
    def __init__(self):
        # Array of arrays to store primary samples
        # Index 0: Transmittance
        # Index > 0: Eye subpath vertices
        # Index < 0: Light subpath vertices
        self.samples = {}

    def get_sample(self, index, count):
        """
        Retrieve 'count' random values for the given index.
        Consumes values sequentially from the row.
        """
        if index not in self.samples:
            self.samples[index] = []
        
        row = self.samples[index]
        if len(row) < count:
            # Generate more random values if needed [G]
            new_values = generate_random_values(count - len(row))
            row.extend(new_values)
        
        # Consume values sequentially
        values = row[:count]
        del row[:count]
        return values

    def sample_eye_vertex(self, vertex_index, count):
        """Sample a vertex on the eye subpath."""
        return self.get_sample(vertex_index, count)

    def sample_light_vertex(self, vertex_index, count):
        """Sample a vertex on the light subpath."""
        # Use negative index for light subpath
        return self.get_sample(-vertex_index, count)

    def sample_transmittance(self, count):
        """Sample transmittance between subpath connections."""
        return self.get_sample(0, count)
```

## Subtopics

### [Bidirectional Path Tracing](6666-bidirectional-path-tracing.md)
The parent method that constructs two independent subpaths and connects them. This layout is a specific data organization technique for BDPT's primary samples.

### [Bidirectional Path-Space Differentiable Rendering](7618-bidirectional-path-space-differentiable-rendering.md)
A sibling method that extends BDPT to differentiable rendering. The primary sample space transformation layout may be relevant for ensuring smooth gradients in differentiable contexts, though specific details are not provided in the sources.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Ensures small changes in $U$ cause small changes in $P$, which is crucial for robustness in participating media. Handles variable-length random number requirements for heterogeneous media.
- **Limitations:** Requires managing variable-length arrays. Not given in the sources for specific performance trade-offs compared to other layouts.
- **When to use:** When using BDPT in scenes with participating media, especially heterogeneous media, to ensure sample space smoothness.
- **When not to use:** Not given in the sources.

## Background
Originated in the paper "Unbiased Global Illumination with Participating Media" [S1]. The method was developed to address the limitations of standard BDPT in participating media, where the fixed mapping of random numbers to path vertices fails to ensure smoothness in path space due to the variable nature of distance sampling in heterogeneous media.

## Sources
[S1] Unbiased Global Illumination with Participating Media, http://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/ugiwpm.pdf, fetched 2026-10-30
