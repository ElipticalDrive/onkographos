# Stochastic Evolutive TF Design

Also called: Genetic Algorithm TF Design, Genetic Algorithm for TF Design, He et al. Genetic Algorithms, Stochastic Evolutive Transfer Function Design

## Overview
Stochastic Evolutive TF Design is an interactive, genetic algorithm-based method for generating one-dimensional transfer functions (TFs) for volume rendering [S1][S2, unofficial]. It solves the problem of manual TF specification by allowing users to evolve a population of TFs through selection, crossover, and mutation, guided by visual feedback in the form of thumbnails [S1][S2, unofficial]. It differs from gradient-based inverse rendering methods by relying on user-driven selection rather than differentiable loss minimization, and from purely algorithmic TF generation by incorporating explicit genetic operators that manipulate the TF structure directly [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset, an initial population of TFs, and user selections of preferred TFs from rendered thumbnails.
*   **Output:** An evolved TF (or set of TFs) that reveals desired structures in the volume.

**Data Structures:**
The method supports two encoding schemes for the TF chromosome:
1.  **Lookup Table (LUT) Encoding:** The TF is represented as a fixed-size array of entries, where each entry corresponds to a scalar value bin. Each entry contains color and opacity values. This is the implementation described in [S1][S2, unofficial] as the primary method, allowing combination with other 1D TF design methods.
2.  **Control Point Encoding:** The TF is represented as an ordered sequence of control points. Each point is defined by a scalar value and respective color and opacity. The TF used for rendering is obtained by interpolating between these control points. This is the original approach by He et al. [S2, unofficial].

*Note: The following steps describe the LUT-based implementation detailed in [S1][S2, unofficial], as it provides specific operator definitions.*

**Numbered Steps:**

1.  **Initialization:** Generate an initial population of $P$ TFs. Each TF is a lookup table of size $N$ (number of scalar bins). For each TF, populate the entries with random color and opacity values, ensuring the scalar values are ordered [S2, unofficial].
2.  **Rendering:** For each TF in the current population, render a thumbnail of the volume dataset.
3.  **Selection:** The user inspects the thumbnails and selects the best ones to serve as parents for the next generation. The selected TFs are preserved in the selection pool [S1][S2, unofficial].
4.  **Reproduction:** Generate a new population of descendants from the selected parents using crossover and mutation operators [S1][S2, unofficial].
    *   **Crossover:**
        1.  Randomly select two parent TFs.
        2.  Randomly determine two cutting points, $c_1$ and $c_2$, within the range $[0, N-1]$, such that $c_1 < c_2$.
        3.  Create two offspring. For the first offspring, take the interval $[c_1, c_2]$ from Parent A and the outer intervals $[0, c_1)$ and $[c_2, N-1]$ from Parent B. For the second offspring, take the interval $[c_1, c_2]$ from Parent B and the outer intervals from Parent A [S1][S2, unofficial].
    *   **Mutation:** Apply one or more of the following mutation operators to a TF [S1][S2, unofficial]:
        *   **M1 (Triangular):** Define a mutation region $[m_1, m_2]$. Define a center $m_c$ within this region. Define a target opacity $o_{center}$ for $m_c$. Apply linear interpolation between $o_{center}$ and the opacity values at $m_1$ and $m_2$.
        *   **M2 (Random):** Define a mutation center $m_c$ and a range. Randomly set the opacities for the voxel values (bins) around $m_c$.
        *   **M3 (Rectangular):** Randomly select a center and range for the mutation. Randomly set the colors and opacities for the bins within this range, constrained by limited intervals.
        *   **M4 (Global Scale):** Apply a random global scale factor to all opacity values in the TF.
5.  **Iteration:** Render thumbnails for the new population. Repeat steps 3-4 until the user is satisfied with the visualization. High-resolution rendering can be used for detailed inspection if thumbnail quality is insufficient [S1][S2, unofficial].

**Formulas:**
*   **Rendering:** The thumbnails are rendered using Engel's pre-integrated volume rendering method to handle high-frequency features introduced by genetic operators [S1][S2, unofficial]. The standard emission-absorption model is used for the final high-resolution rendering if needed [G].
*   **Interpolation (Control Point Variant):** If using control points, the TF value $T(v)$ for a scalar value $v$ is obtained by interpolating between adjacent control points $(v_i, c_i, o_i)$ and $(v_{i+1}, c_{i+1}, o_{i+1})$ [S2, unofficial].

**Parameters:**
*   Population size: Not given in the sources.
*   Mutation rate: Not given in the sources.
*   Crossover rate: Not given in the sources.
*   LUT size ($N$): Not given in the sources.

**Edge Cases and Pitfalls:**
*   **High-Frequency Artifacts:** Genetic operators can produce high-frequency features in the TF domain, which might cause artifacts in the resulting images. This is mitigated by using Engel's pre-integrated volume rendering method, which handles high frequencies well [S1][S2, unofficial].
*   **Low-Resolution Evaluation:** Thumbnail quality may be insufficient for proper evaluation. The method provides a second level of interaction for high-resolution inspection of specific TFs [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

class TransferFunction:
    def __init__(self, size, num_bins):
        self.size = size
        self.num_bins = num_bins
        # Initialize with random color and opacity for each bin
        self.colors = [[random.random(), random.random(), random.random()] for _ in range(num_bins)]
        self.opacities = [random.random() for _ in range(num_bins)]

    def crossover(self, other):
        """
        Crossover operator: randomly takes two TFs and cuts them at two points.
        The interval between the two points of one TF is combined with the two outer intervals
        of the other TF.
        """
        c1 = random.randint(0, self.num_bins - 2)
        c2 = random.randint(c1 + 1, self.num_bins - 1)
        
        # Offspring 1: Interval from self, outer from other
        child1_colors = self.colors.copy()
        child1_colors[c1:c2+1] = other.colors[c1:c2+1]
        child1_opacities = self.opacities.copy()
        child1_opacities[c1:c2+1] = other.opacities[c1:c2+1]
        
        # Offspring 2: Interval from other, outer from self
        child2_colors = other.colors.copy()
        child2_colors[c1:c2+1] = self.colors[c1:c2+1]
        child2_opacities = other.opacities.copy()
        child2_opacities[c1:c2+1] = self.opacities[c1:c2+1]
        
        return [TransferFunction(self.size, self.num_bins) for _ in range(2)] # Placeholder for actual object creation

    def mutate_triangular(self):
        """
        M1: Triangular mutation. Define an opacity value for the center of the mutation region
        and apply linear interpolation between the central opacity value and the opacity values
        at each border of the mutation.
        """
        m1 = random.randint(0, self.num_bins - 2)
        m2 = random.randint(m1 + 1, self.num_bins - 1)
        mc = random.randint(m1, m2)
        o_center = random.random()
        
        # Interpolate from m1 to mc
        for i in range(m1, mc + 1):
            t = (i - m1) / (mc - m1) if mc != m1 else 0
            self.opacities[i] = self.opacities[m1] * (1 - t) + o_center * t
            
        # Interpolate from mc to m2
        for i in range(mc, m2 + 1):
            t = (i - mc) / (m2 - mc) if m2 != mc else 0
            self.opacities[i] = o_center * (1 - t) + self.opacities[m2] * t

    def mutate_random(self):
        """
        M2: Random mutation. Randomly set the opacities for the voxel values around the mutation center.
        """
        mc = random.randint(0, self.num_bins - 1)
        range_size = random.randint(1, self.num_bins // 2)
        start = max(0, mc - range_size)
        end = min(self.num_bins - 1, mc + range_size)
        
        for i in range(start, end + 1):
            self.opacities[i] = random.random()

    def mutate_rectangular(self):
        """
        M3: Rectangular mutation. Randomly set the center and range of the mutation,
        as well as the colors and opacities, between limited intervals.
        """
        center = random.randint(0, self.num_bins - 1)
        range_size = random.randint(1, self.num_bins // 2)
        start = max(0, center - range_size)
        end = min(self.num_bins - 1, center + range_size)
        
        for i in range(start, end + 1):
            self.opacities[i] = random.random()
            self.colors[i] = [random.random(), random.random(), random.random()]

    def mutate_global_scale(self):
        """
        M4: Global Scale. Apply a random global scale to all opacity values.
        """
        scale = random.random()
        for i in range(self.num_bins):
            self.opacities[i] = min(1.0, self.opacities[i] * scale)

def evolve_tf(volume_data, initial_population, num_generations):
    """
    Main evolution loop.
    """
    population = initial_population
    
    for gen in range(num_generations):
        # Render thumbnails for current population
        thumbnails = [render_thumbnail(tf, volume_data) for tf in population]
        
        # User selects parents (simulated here by selecting top K based on some metric)
        # In practice, this is an interactive step
        selected_parents = select_parents_interactively(thumbnails)
        
        new_population = []
        while len(new_population) < len(population):
            # Crossover
            if random.random() < 0.8: # Crossover rate
                p1, p2 = random.sample(selected_parents, 2)
                children = p1.crossover(p2)
                new_population.extend(children)
            
            # Mutation
            if len(new_population) < len(population):
                parent = random.choice(selected_parents)
                child = TransferFunction(parent.size, parent.num_bins)
                child.colors = parent.colors.copy()
                child.opacities = parent.opacities.copy()
                
                # Apply random mutation
                mutation_type = random.choice(['triangular', 'random', 'rectangular', 'global_scale'])
                if mutation_type == 'triangular':
                    child.mutate_triangular()
                elif mutation_type == 'random':
                    child.mutate_random()
                elif mutation_type == 'rectangular':
                    child.mutate_rectangular()
                elif mutation_type == 'global_scale':
                    child.mutate_global_scale()
                
                new_population.append(child)
        
        population = new_population
    
    return population
```

## Subtopics

### Control Point Encoding
An alternative encoding where TFs are sequences of control points interpolated for rendering. It is more compact but requires interpolation logic. Choose when memory is constrained or smooth TFs are preferred. [S2, unofficial]

### Manual Specification
A complementary method where users draw directly on the TF plot. It offers precise control but lacks the stochastic exploration of genetic algorithms. Choose when a specific TF shape is known. [S1][S2, unofficial]

## Operating characteristics
*   **Time Complexity:** Depends on the number of generations, population size, and rendering time per thumbnail. Rendering thumbnails is the dominant cost.
*   **Memory Complexity:** Proportional to the population size and TF size (LUT or control points).
*   **Strengths:** Intuitive user interface; allows exploration of diverse TFs; handles high-frequency TF features via pre-integrated rendering.
*   **Limitations:** Requires user interaction; not fully automated; performance depends on rendering speed.
*   **When to use:** When the user has a vague idea of the desired visualization and wants to explore the space of possible TFs.
*   **When not to use:** When a fully automated, gradient-based optimization is required, or when real-time feedback is critical and rendering is slow.

## Background
The method is based on the work of He et al. [HE 96], who proposed a genetic algorithm-based technique for specifying 1D TFs directed at images. The implementation described in [S1][S2, unofficial] is a more flexible version that uses the TF lookup table itself as the chromosome, allowing combination with other 1D TF design methods. The method was developed to provide a more intuitive domain for TF specification, where the distance to the edge is mapped to optical attributes [S2, unofficial].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
[S2] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
