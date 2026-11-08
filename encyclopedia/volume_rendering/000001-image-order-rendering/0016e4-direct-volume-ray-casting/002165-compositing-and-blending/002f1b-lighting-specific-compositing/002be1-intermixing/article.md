# Intermixing

Also called: Accumulation-level intermixing

## Overview
Intermixing is a technique for rendering multi-channel volume data by separately evaluating the illumination model for each channel at every ray sample and combining the results during the accumulation phase [S1, unofficial]. It solves the problem of visualizing complex interactions between distinct volumetric data sources (e.g., different tissue types or simulation states) within a single unified view [S1, unofficial]. Unlike image-level intermixing, which renders complete images per channel and fails to handle occlusions correctly due to missing depth information, accumulation-level intermixing handles inter-channel occlusions naturally by integrating contributions into the standard ray-casting accumulator [S1, unofficial]. It is more flexible than other approaches but computationally more expensive [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ co-registered volume datasets (channels), where each dataset provides scalar values $v_{i,c}$ for sample $i$ and channel $c$. A lighting model $L(v, \mathbf{p}, \mathbf{v})$ that computes color and opacity contributions based on value, position, and view direction.
*   **Outputs:** Final pixel color $C_{final}$ and accumulated opacity $\alpha_{acc}$.

**Data Structures:**
The method operates on the standard ray-casting accumulator state $(C_{acc}, \alpha_{acc})$, initialized to $(0, 0)$ [S1, S51, unofficial]. For each ray sample $i$, the method computes intermediate contributions for each channel $c \in \{1, \dots, N\}$.

**Numbered Steps:**
1.  Initialize accumulators: $C_{acc} = (0, 0, 0)$, $\alpha_{acc} = 0$.
2.  For each sample $i$ along the ray:
    a. For each channel $c$ from $1$ to $N$:
        i. Sample the volume value $v_{i,c}$ from channel $c$ at position $\mathbf{p}_i$.
        ii. Evaluate the lighting model for channel $c$ to obtain the raw color contribution $c_{raw, i,c}$ and opacity contribution $\alpha_{raw, i,c}$.
        iii. Compute the lighting-modified color contribution $\Delta C_{i,c}$ and opacity contribution $\Delta \alpha_{i,c}$. In standard volume rendering, $\Delta \alpha_{i,c} = \alpha_{raw, i,c}$ and $\Delta C_{i,c} = c_{raw, i,c} \cdot \alpha_{raw, i,c}$ [S1, S51, unofficial].
    b. Sum the contributions from all channels for the current sample:
       $$ \Delta C_i = \sum_{c=1}^{N} \Delta C_{i,c} $$
       $$ \Delta \alpha_i = \sum_{c=1}^{N} \Delta \alpha_{i,c} $$
    c. Update the accumulators using the standard front-to-back compositing equations:
       $$ C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \cdot \Delta C_i $$
       $$ \alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \cdot \Delta \alpha_i $$
    d. Apply early ray termination if $\alpha_{acc}^{(i)}$ exceeds a threshold [S10, S12, unofficial].
3.  Composite over background $C_{bg}$:
    $$ C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg} $$

**Parameters:**
*   $N$: Number of channels. Implementation-defined, often limited by hardware texture formats (e.g., $N \le 4$ for standard GPU textures) [S1, unofficial].
*   Threshold for early ray termination: Not given in the sources.

**Pitfalls:**
*   **Computational Cost:** Evaluating the illumination model separately for each channel increases the number of lighting calculations per sample by a factor of $N$ [S1, unofficial].
*   **Hardware Limits:** The number of channels may be constrained by available texture formats (e.g., 4 components) if not using specialized multi-texture or multi-buffer strategies [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intermixing_render(ray, volumes, lighting_model, bg_color):
    # volumes is a list of N volume accessors
    # lighting_model(v, p, view_dir) -> (color, opacity)
    
    acc_color = vec3(0.0)
    acc_alpha = 0.0
    transmittance = 1.0
    
    for sample in ray.samples:
        # Sum contributions from all channels
        delta_color = vec3(0.0)
        delta_alpha = 0.0
        
        for c in range(len(volumes)):
            # Sample value from channel c
            v = volumes[c].sample(sample.position)
            
            # Evaluate lighting for channel c
            c_raw, a_raw = lighting_model(v, sample.position, ray.direction)
            
            # Compute contributions
            # Standard assumption: delta_alpha = a_raw, delta_color = c_raw * a_raw
            delta_alpha += a_raw
            delta_color += c_raw * a_raw
            
        # Update accumulators
        acc_color += transmittance * delta_color
        acc_alpha += transmittance * delta_alpha
        transmittance *= (1.0 - delta_alpha)
        
        # Early ray termination
        if acc_alpha > 0.99:
            break
            
    final_color = acc_color + transmittance * bg_color
    return final_color
```

## Subtopics

### [Multi-volume rendering](6407-multi-volume-rendering.md)
Multi-volume rendering refers to the simultaneous rendering of multiple co-registered or deformed volume datasets. It solves the problem of visualizing complex interactions between distinct volumetric data sources (e.g., different tissue types, simulation states, or anatomical atlases) within a single unified view. Unlike single-volume rendering, it requires specific strategies for combining data from multiple textures, categorized into classification-level, accumulation-level, and image-level intermixing. It builds on [Intermixing](5139-intermixing.md) by providing the structural mechanisms to handle multiple data channels during the ray-casting process.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of channels and $S$ is the number of samples per ray. Lighting evaluation is performed $N$ times per sample [S1, unofficial].
*   **Memory Complexity:** Depends on the storage strategy for multi-channel data. If stored in separate textures, memory scales with $N$ [S1, unofficial]. If stored in interleaved bricks/nodes, memory is proportional to the total voxel count times $N$ [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Handles inter-channel occlusions correctly; flexible lighting models per channel [S1, unofficial].
*   **Limitations:** Computationally expensive due to multiple lighting evaluations; potential hardware texture format limitations [S1, unofficial].
*   **When to use:** When accurate occlusion between different volume channels is required and performance allows [S1, unofficial].
*   **When not to use:** When performance is critical and channels do not occlude each other significantly, or when hardware limits prevent efficient multi-channel sampling [S1, unofficial].

## Background
Intermixing, specifically accumulation-level intermixing, was presented by Brix et al. in 2014 [BPH14] as part of an out-of-core rendering approach for multi-channel volume data sets. It builds on Crassin et al.’s octree-based memory virtualization scheme [CNLE09]. The method is used in systems such as Voreen [DLJL22], Viv [MGP+22], and Neuroglancer [Goo22] to support multi-channel data rendering.

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
