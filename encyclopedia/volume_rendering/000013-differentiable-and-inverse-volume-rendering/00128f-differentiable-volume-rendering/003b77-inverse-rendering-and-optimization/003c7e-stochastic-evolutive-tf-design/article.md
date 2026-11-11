# Stochastic Evolutive TF Design

Also called: Genetic Algorithm TF Design, Genetic Algorithm for TF Design, He et al. Genetic Algorithms, Stochastic Evolutive Transfer Function Design

## Overview
Stochastic Evolutive TF Design is an interactive, genetic algorithm-based method for generating one-dimensional transfer functions (TFs) for volume rendering [S1][S2, unofficial]. It solves the problem of manually specifying TFs by allowing users to iteratively evolve a population of TFs based on visual feedback from low-resolution thumbnails [S1][S2, unofficial]. It is distinguished from other inverse rendering methods by its reliance on evolutionary search and user-in-the-loop selection rather than gradient-based optimization or neural network generation [S1][S2, unofficial]. Unlike [Text-to-Transfer Function](7365-text-to-transfer-function.md) which uses semantic input, or [GAN-based Volume Rendering](8825-gan-based-volume-rendering.md) which uses generative models, this method uses biological-inspired operators (crossover and mutation) on the TF representation itself [S1][S2, unofficial].

## How it works
The method operates in two levels of interaction: a first level using low-resolution thumbnails for rapid iteration, and a second level for high-resolution inspection and manual refinement [S1]. The core mechanism is a genetic algorithm where the TF is the chromosome [S1][S2, unofficial].

### Inputs and Outputs
*   **Input:** A volume dataset $V$, an initial population of TFs (or random generation), and user selections of preferred thumbnails.
*   **Output:** A final TF that reveals desired structures in the volume.

### Data Structures
The TF can be encoded in two ways, depending on the implementation variant [S1][S2, unofficial]:
1.  **Control Point Variant (He et al. [S2, unofficial]):** The TF is a sequence of control points. Each gene is a control point defined by a scalar value $s$, color $\mathbf{c}$, and opacity $\alpha$. The TF is constructed by interpolating these points [S2, unofficial].
2.  **Lookup Table Variant (Kindlmann & Durkin extension [S1]):** The TF is the lookup table (LUT) itself. Each entry in the LUT (indexed by voxel value) is a gene. This allows direct manipulation of the TF curve and easier combination with other 1D TF design methods [S1].

### Parameters
*   **Population Size:** Not specified in sources.
*   **Mutation Rate:** Not specified, but operators are applied "in a random way" [S1][S2, unofficial].
*   **Crossover Points:** Two points are randomly determined for each crossover operation [S1][S2, unofficial].

### Steps
1.  **Initialization:** Generate an initial population of TFs. If using the control point variant, control points have random attribute values but are ordered by scalar value [S2, unofficial]. If using the LUT variant, random values are assigned to the LUT entries [S1][S2, unofficial].
2.  **Rendering:** For each TF in the current population, render a thumbnail of the volume [S1][S2, unofficial].
3.  **Selection:** The user selects the best thumbnails (and their associated TFs) to serve as parents for the next generation [S1][S2, unofficial].
4.  **Evolution:** Generate a new population of descendants from the selected parents using mutation, crossover, or both [S1][S2, unofficial].
    *   **Crossover:** Randomly select two parent TFs. Randomly determine two cutting points (scalar values) [S1][S2, unofficial]. Combine the interval between the two points of one parent with the two outer intervals (defined by the same points) of the other parent [S1][S2, unofficial].
    *   **Mutation:** Apply one or more of the following operators to a TF [S1][S2, unofficial]:
        *   **M1 (Triangular):** Define an opacity value for the center of a mutation region. Apply linear interpolation between this central opacity and the opacity values at the borders of the mutation region [S1][S2, unofficial].
        *   **M2 (Random):** Randomly set the opacities for voxel values within a mutation region around a center point [S1][S2, unofficial].
        *   **M3 (Rectangular):** Randomly set the center, range, color, and opacity of a mutation region, constrained to limited intervals [S1][S2, unofficial].
        *   **M4 (Global Scale):** Apply a random global scale factor to all opacity values in the TF [S1][S2, unofficial].
5.  **Iteration:** Repeat steps 2-4. Users can select and deselect thumbnails from the new set to define parents for subsequent iterations [S1][S2, unofficial].
6.  **Refinement (Optional):** If a thumbnail's quality is insufficient due to low resolution, the user can inspect the corresponding TF in the second level of interaction, which provides a high-resolution rendering and allows manual specification (e.g., drawing lines on the TF plot) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stochastic_evolutive_tf_design(volume, initial_tfs, max_generations):
    """
    volume: The 3D volume dataset.
    initial_tfs: List of initial Transfer Functions.
    max_generations: Maximum number of evolutionary steps.
    """
    population = initial_tfs
    
    for generation in range(max_generations):
        # Step 2: Render thumbnails
        thumbnails = []
        for tf in population:
            # Render low-res thumbnail using the TF
            # Note: Engel's pre-integrated volume rendering is recommended 
            # to handle high-frequency features introduced by genetic operators [S1]
            img = render_thumbnail(volume, tf)
            thumbnails.append(img)
        
        # Step 3: User Selection
        # The user selects indices of the best thumbnails
        selected_indices = user_select_best(thumbnails)
        parents = [population[i] for i in selected_indices]
        
        # Step 4: Evolution
        new_population = []
        
        # Crossover
        for i in range(0, len(parents), 2):
            if i + 1 < len(parents):
                parent1 = parents[i]
                parent2 = parents[i+1]
                # Randomly determine two cutting points
                cut1, cut2 = random_cut_points(parent1.scalar_range)
                # Combine intervals
                child = crossover(parent1, parent2, cut1, cut2)
                new_population.append(child)
        
        # Mutation
        for tf in parents:
            # Apply mutation operators randomly
            if random() < mutation_rate:
                tf = apply_mutation(tf)
            new_population.append(tf)
            
        population = new_population
        
        # Optional: High-res inspection loop could occur here if user requests
        # if user_requests_high_res():
        #     high_res_render(volume, population)
        
    return population[-1] # Return best TF from last generation

def crossover(tf1, tf2, cut1, cut2):
    """
    Combines interval between cut1 and cut2 of tf1 with outer intervals of tf2.
    Assumes TFs are defined over the same scalar range.
    """
    # Implementation depends on TF representation (LUT or Control Points)
    # For LUT:
    # child_lut = concatenate(tf2[0:cut1], tf1[cut1:cut2], tf2[cut2:end])
    # For Control Points:
    # Interpolate and merge control points based on cut points
    pass

def apply_mutation(tf):
    """
    Applies one of M1, M2, M3, or M4.
    """
    operator = random.choice(['M1', 'M2', 'M3', 'M4'])
    if operator == 'M1':
        return triangular_mutation(tf)
    elif operator == 'M2':
        return random_mutation(tf)
    elif operator == 'M3':
        return rectangular_mutation(tf)
    elif operator == 'M4':
        return global_scale_mutation(tf)
    return tf

def triangular_mutation(tf):
    """M1: Linear interpolation from center to borders."""
    center = random_scalar_value()
    range_left, range_right = get_mutation_range(center)
    center_opacity = random_opacity()
    left_opacity = tf.opacity(range_left)
    right_opacity = tf.opacity(range_right)
    
    # Interpolate between left and center, then center and right
    for s in range(range_left, center):
        t = (s - range_left) / (center - range_left)
        tf.opacity(s) = lerp(left_opacity, center_opacity, t)
    for s in range(center, range_right):
        t = (s - center) / (range_right - center)
        tf.opacity(s) = lerp(center_opacity, right_opacity, t)
    return tf

def random_mutation(tf):
    """M2: Random opacities in a region."""
    center = random_scalar_value()
    range_left, range_right = get_mutation_range(center)
    for s in range(range_left, range_right):
        tf.opacity(s) = random_opacity()
    return tf

def rectangular_mutation(tf):
    """M3: Random center, range, color, opacity in intervals."""
    center = random_scalar_value()
    range_left, range_right = get_mutation_range(center)
    color = random_color()
    opacity = random_opacity()
    for s in range(range_left, range_right):
        tf.opacity(s) = opacity
        tf.color(s) = color
    return tf

def global_scale_mutation(tf):
    """M4: Scale all opacities."""
    scale = random_scale_factor()
    for s in tf.scalar_range:
        tf.opacity(s) = clamp(tf.opacity(s) * scale, 0.0, 1.0)
    return tf
```

### Pitfalls
*   **High-Frequency Artifacts:** Genetic operators (especially M2 and M3) can produce high-frequency features in the TF domain, which may cause artifacts in the rendered images [S1].
*   **Mitigation:** Use Engel's pre-integrated volume rendering method, which handles high frequencies well [S1].

## Operating characteristics
*   **Time Complexity:** Depends on population size, number of generations, and rendering time per thumbnail. Rendering thumbnails is the bottleneck.
*   **Memory Complexity:** Proportional to the number of TFs in the population and the size of the TF representation (LUT size or number of control points) [S1][S2, unofficial].
*   **Strengths:** Intuitive user interface; allows users to guide the search based on visual structure; flexible combination with other 1D TF design methods (LUT variant) [S1][S2, unofficial].
*   **Limitations:** Requires user interaction at each generation; not fully automated; thumbnail resolution may hide details [S1].
*   **When to use:** When users have a good idea of the desired visual outcome but struggle with manual TF specification; for exploratory visualization [S1][S2, unofficial].
*   **When not to use:** When fully automated TF generation is required; when high precision is needed without user iteration [S1].

## Background
The method is based on the technique proposed by He et al. [HE 96] (cited as [S2, unofficial] in the notes, corresponding to He et al. 1996) and extended by Kindlmann and Durkin [KIN 98] (cited as [S1] in the notes, corresponding to Kindlmann and Durkin 1998/2008) [S1][S2, unofficial]. He et al. developed a technique for specifying 1D TFs directed at images, using genetic algorithms where TFs are encoded as sequences of control points [S2, unofficial]. Kindlmann and Durkin proposed a more flexible implementation where the TF lookup table itself is the chromosome, allowing combination with other 1D TF design methods [S1]. The method is used in volume visualization tools for interactive exploration [S1][S2, unofficial].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
[S2] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
