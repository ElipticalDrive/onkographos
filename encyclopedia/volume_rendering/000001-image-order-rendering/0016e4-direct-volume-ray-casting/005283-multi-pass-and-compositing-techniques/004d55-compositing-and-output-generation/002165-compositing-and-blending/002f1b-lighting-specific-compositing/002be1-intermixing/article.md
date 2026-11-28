# Intermixing


## Overview
Intermixing is a technique for rendering multi-channel volume data by separately evaluating the illumination model for each channel at every ray sample and combining the results during the accumulation phase [S1, unofficial]. It distinguishes itself from image-level compositing by correctly handling occlusions between different channels, as it retains depth information that image-level blending discards [S1, unofficial]. It is more flexible than other approaches but computationally more expensive [S1, unofficial].

## How it works
Inputs: A sequence of ray samples sorted front-to-back, where each sample $i$ contains color and opacity values for $N$ distinct channels [S1, unofficial]. Background color $C_{bg}$ [G].
Outputs: Final pixel color $C_{final}$ [G].
Data Structures: Accumulators $C_{acc}$ (vec3, initialized to 0) and $\alpha_{acc}$ (float, initialized to 0) [G]. Transmittance $T$ (float, initialized to 1.0) [G].
Algorithm:
The standard front-to-back compositing loop is extended to handle multiple channels at the same spatial position [G]. For each sample $i$ along the ray:

1. Initialize transmittance $T \leftarrow 1.0$ before the sample loop [G].
2. For each channel $k$ from $1$ to $N$:
   a. Evaluate the illumination model for channel $k$ at the current sample position to obtain $c_{i,k}$ and $\alpha_{i,k}$ [S1, unofficial].
   b. Update the global color accumulator using the channel's contribution scaled by the incoming transmittance:
      $$C_{acc} \leftarrow C_{acc} + T \cdot \alpha_{i,k} \cdot c_{i,k}$$ [G]
   c. Compute the combined transmittance for the current sample by multiplying the individual channel transmittances:
      $$T_{sample} \leftarrow T_{sample} \cdot (1 - \alpha_{i,k})$$ [G]
3. Update the global opacity accumulator and transmittance for the next sample using the combined sample transmittance:
   $$\alpha_{acc} \leftarrow \alpha_{acc} + T \cdot (1 - T_{sample})$$ [G]
   $$T \leftarrow T \cdot T_{sample}$$ [G]
4. If $\alpha_{acc} \ge 1.0$, terminate early (early ray termination) [G].
5. After processing all samples, composite over the background:
   $$C_{final} = C_{acc} + (1 - \alpha_{acc}) \cdot C_{bg}$$ [G]

**Pitfalls:**
- **Image-level intermixing failure:** Rendering a complete image for each channel and mixing them using alpha blending fails to handle occlusions between different channels correctly because depth information is lost in the accumulated intermediate results [S1, unofficial].
- **Performance:** Intermixing is computationally more expensive than single-channel rendering or image-level compositing because the illumination model must be evaluated separately for each channel at every sample [S1, unofficial].
- **Hardware Limits:** In implementations relying on GPU texture formats (e.g., Brix et al. [BPH14]), the number of channels $N$ is often limited to 4 due to the 4-component limit of standard texture formats [S1, unofficial].

```python
def intermixing_loop(samples, background_color):
    """
    Accumulation-level intermixing for multi-channel volume rendering.
    samples: list of lists of tuples. samples[i] contains N tuples of (color: vec3, opacity: float).
    background_color: vec3 for the scene background.
    """
    C_acc = vec3(0.0)
    alpha_acc = 0.0
    T = 1.0
    
    for sample_channels in samples:
        T_sample = 1.0
        for c_k, alpha_k in sample_channels:
            C_acc = C_acc + T * alpha_k * c_k
            T_sample = T_sample * (1.0 - alpha_k)
        
        alpha_acc = alpha_acc + T * (1.0 - T_sample)
        T = T * T_sample
        
        if alpha_acc >= 1.0:
            break
            
    return C_acc + (1.0 - alpha_acc) * background_color
```

## Subtopics

### [Multi-volume rendering](6407-multi-volume-rendering.md)
Multi-volume rendering refers to the simultaneous rendering of multiple co-registered volumes by evaluating samples along a viewing ray for all visible channels. It is distinguished from single-volume rendering by its ability to intermix data sources, and from image-level compositing by its preservation of depth ordering through per-sample combination.

## Operating characteristics
- **Complexity:** Not given in the sources.
- **Performance:** Intermixing is more flexible than other approaches but computationally more expensive [S1, unofficial].
- **Strengths:** Correctly handles occlusions between different channels; preserves depth ordering.
- **Limitations:** Higher computational cost; potential hardware limits on channel count (e.g., $N \le 4$ in some GPU-based implementations) [S1, unofficial].

## Background
Intermixing, specifically accumulation-level intermixing, was presented by Brix et al. in 2014 [BPH14] as part of an out-of-core rendering approach for multi-channel volume data sets [S1, unofficial]. It builds on Crassin et al.’s octree-based memory virtualization scheme [CNLE09] [S1, unofficial]. The method is used in systems such as Voreen [DLJL22], Viv [MGP+22], and Neuroglancer [Goo22] [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
