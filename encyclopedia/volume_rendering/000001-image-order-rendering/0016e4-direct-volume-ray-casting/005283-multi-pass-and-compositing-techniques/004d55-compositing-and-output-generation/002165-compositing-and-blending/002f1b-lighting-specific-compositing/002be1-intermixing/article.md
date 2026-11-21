# Intermixing

Also called: Accumulation-level intermixing, Multi-channel intermixing

## Overview
Intermixing is a technique for rendering multi-channel volume data by evaluating the illumination model for each channel at every ray sample and combining the results during the accumulation phase [S1, unofficial]. It solves the problem of correctly visualizing interactions between different data modalities (e.g., CT and MRI) or multiple deformed states of the same object by preserving depth ordering through per-sample combination [S1, unofficial]. It is distinguished from image-level compositing by its ability to handle occlusions between different channels correctly, as it retains depth information that image-level blending discards [S1, unofficial].

## How it works
Intermixing operates on a sequence of ray samples where each sample $i$ consists of color and opacity values for $N$ distinct channels. Let $c_{i,k}$ and $\alpha_{i,k}$ denote the color (RGB) and opacity (A) of channel $k$ at sample $i$, where $k \in \{1, \dots, N\}$.

**Inputs:**
- A list of ray samples sorted front-to-back.
- Each sample contains $N$ tuples of $(c_{i,k}, \alpha_{i,k})$.
- Background color $C_{bg}$.

**Outputs:**
- Final pixel color $C_{final}$.

**Data Structures:**
- Accumulators: $C_{acc}$ (vec3, initialized to 0) and $\alpha_{acc}$ (float, initialized to 0).
- Transmittance: $T$ (float, initialized to 1.0).

**Algorithm:**
The standard front-to-back compositing loop is extended to handle multiple channels. For each sample $i$ along the ray:

1. Initialize transmittance $T \leftarrow 1.0$.
2. For each channel $k$ from 1 to $N$:
   a. Evaluate the illumination model for channel $k$ at the current sample position to obtain $c_{i,k}$ and $\alpha_{i,k}$ [S1, unofficial].
   b. Update the global accumulators using the channel's contribution:
      $$
      C_{acc} \leftarrow C_{acc} + T \cdot \alpha_{i,k} \cdot c_{i,k}
      $$
      $$
      \alpha_{acc} \leftarrow \alpha_{acc} + T \cdot \alpha_{i,k}
      $$
   c. Update transmittance:
      $$
      T \leftarrow T \cdot (1 - \alpha_{i,k})
      $$
3. If $\alpha_{acc} \ge 1.0$, terminate early (early ray termination) [G].
4. After processing all samples, composite over the background:
   $$
   C_{final} = C_{acc} + (1 - \alpha_{acc}) \cdot C_{bg}
   $$

**Pitfalls:**
- **Image-level intermixing failure:** Rendering a complete image for each channel and mixing them using alpha blending fails to handle occlusions between different channels correctly because depth information is lost in the accumulated intermediate results [S1, unofficial].
- **Performance:** Intermixing is computationally more expensive than single-channel rendering or image-level compositing because the illumination model must be evaluated separately for each channel at every sample [S1, unofficial].
- **Hardware Limits:** In implementations relying on GPU texture formats (e.g., Brix et al. [BPH14]), the number of channels $N$ is often limited to 4 due to the 4-component limit of standard texture formats [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intermixing_loop(samples, background_color, num_channels):
    """
    Accumulation-level intermixing for multi-channel volume rendering.
    samples: list of lists of tuples. samples[i] contains N tuples of (color: vec3, opacity: float).
    background_color: vec3 for the scene background.
    num_channels: int, number of channels N.
    """
    C_acc = vec3(0.0)
    alpha_acc = 0.0
    T = 1.0
    
    for sample_channels in samples:
        # sample_channels is a list of (c_k, alpha_k) for k in 0..N-1
        for k in range(num_channels):
            c_k, alpha_k = sample_channels[k]
            
            # Update accumulators
            C_acc = C_acc + T * alpha_k * c_k
            alpha_acc = alpha_acc + T * alpha_k
            
            # Update transmittance
            T = T * (1.0 - alpha_k)
            
            # Early ray termination check
            if alpha_acc >= 1.0:
                break
        if alpha_acc >= 1.0:
            break
            
    # Composite over background
    return C_acc + (1.0 - alpha_acc) * background_color
```

## Subtopics

### [Multi-volume rendering](6407-multi-volume-rendering.md)
Multi-volume rendering refers to the simultaneous rendering of multiple co-registered volumes by evaluating samples along a viewing ray for all visible channels. It solves the problem of visualizing complex interactions between different data modalities (e.g., CT and MRI) or deformed states of the same object. It is distinguished from single-volume rendering by its ability to intermix data sources, and from image-level compositing by its preservation of depth ordering through per-sample combination.

## Operating characteristics
- **Complexity:** Time complexity is $O(R \cdot N \cdot S)$, where $R$ is the number of rays, $S$ is the number of samples per ray, and $N$ is the number of channels. Memory complexity depends on the storage of $N$ channels per sample or brick.
- **Performance:** Intermixing is more flexible than other approaches but computationally more expensive [S1, unofficial].
- **Strengths:** Correctly handles occlusions between different channels; preserves depth ordering.
- **Limitations:** Higher computational cost; potential hardware limits on channel count (e.g., $N \le 4$ in some GPU-based implementations) [S1, unofficial].

## Background
Intermixing, specifically accumulation-level intermixing, was presented by Brix et al. in 2014 [BPH14] as part of an out-of-core rendering approach for multi-channel volume data sets. It builds on Crassin et al.’s octree-based memory virtualization scheme [CNLE09]. The method is used in systems such as Voreen [DLJL22], Viv [MGP+22], and Neuroglancer [Goo22].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
