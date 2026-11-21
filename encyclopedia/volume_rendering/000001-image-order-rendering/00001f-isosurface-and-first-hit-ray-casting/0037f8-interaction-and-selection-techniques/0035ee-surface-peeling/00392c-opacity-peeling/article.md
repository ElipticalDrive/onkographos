# Opacity Peeling

Also called: Rezk-Salama and Kolb method

## Overview
Opacity peeling is an interaction technique that reveals internal volumetric structures by discarding the first $n$ layers of tissue along each ray before contributing to the final image [S1, S2]. Unlike view-independent methods such as [Brain Peeling](13890-brain-peeling.md) which rely on precomputed distance fields, opacity peeling is view-dependent and operates by accumulating alpha values until specific thresholds are met, allowing users to interactively select which occluded layer to visualize [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function mapping scalar values to color and opacity, and user-defined parameters: `wantedLayer` (integer, number of layers to skip), `T_high` (float, upper alpha threshold), and `T_low` (float, lower sample alpha threshold) [S1].
*   **Output:** A 2D image where the first `wantedLayer` layers of occluding tissue are effectively removed [S1].

**Data Structures:**
The method is implemented within a single-pass volume shader (e.g., fragment shader). The following variables are maintained per ray:
*   `currentLayer`: Integer, initialized to 0. Tracks the number of layers currently peeled [S1].
*   `layerAlpha`: Float, initialized to 0.0. Accumulates the alpha contribution of the current layer being peeled [S1].
*   `sampleRGBA`: Float4, initialized to (0,0,0,0). Accumulates the final color and alpha for the visible layer [S1].

**Algorithm Steps:**
1.  **Initialization:** Set `currentLayer = 0`, `layerAlpha = 0.0`, and `sampleRGBA = (0,0,0,0)` [S1].
2.  **Ray Marching:** For each sample position along the ray:
    a.  Sample the volume at the current position to obtain color and opacity: `sample = SampleVolumeWithTF(volIndex, pos)` [S1].
    b.  Check if the current layer count is less than the user-specified `wantedLayer`:
        i.  If `currentLayer < wantedLayer`:
            1.  Update the accumulated alpha for the current layer using the standard alpha compositing update rule:
                $$layerAlpha = layerAlpha + (1.0 - layerAlpha) \times sample.a$$
            2.  Check for layer transition. If `layerAlpha > T_high` AND `sample.a < T_low`:
                1.  Increment the layer counter: `currentLayer = currentLayer + 1` [S1].
                2.  Reset the accumulator: `layerAlpha = 0.0` [S1].
        ii. If `currentLayer >= wantedLayer`:
            1.  Perform standard front-to-back ray integration for the visible layer:
                $$sampleRGBA = sampleRGBA + (1.0 - sampleRGBA.a) \times sample$$
                *(Note: The source code snippet uses `sampleRGBA += sample`, which implies a simplified accumulation or assumes the alpha term is handled differently in the specific PRISM implementation context, but standard volume rendering requires the $(1-a)$ factor for correct blending. However, strictly following the provided pseudocode from [S1]: `sampleRGBA += sample`)* [S1].

**Parameters:**
*   `T_high`: The threshold for accumulated alpha that triggers a potential layer peel. Must be defined in an Init shader [S1].
*   `T_low`: The threshold for the current sample's alpha. A peel only occurs if the accumulated alpha exceeds `T_high` AND the current sample's alpha falls below `T_low`. This prevents peeling during continuous high-opacity regions. Must be defined in an Init shader [S1].
*   `wantedLayer`: The number of layers to discard. Defined in an Init shader [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The condition `layerAlpha > T_high` relies on floating-point comparison. If `T_high` is set too close to 1.0, precision issues may prevent the threshold from being reached [G].
*   **Boundary Conditions:** If the ray exits the volume before `currentLayer` reaches `wantedLayer`, no visible layer is rendered for that ray, potentially resulting in black pixels or background color depending on the clear color [G].
*   **Degenerate Input:** If `T_low` is greater than `T_high`, the condition `layerAlpha > T_high && sample.a < T_low` may never be satisfied if the sample alpha is always high, or may be satisfied too easily if sample alpha is always low, leading to incorrect layer counts [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization (per ray)
currentLayer = 0
layerAlpha = 0.0
sampleRGBA = (0.0, 0.0, 0.0, 0.0)

# Ray marching loop
for pos in ray_positions:
    # Sample volume intensity and map to color/alpha via transfer function
    sample = SampleVolumeWithTF(volIndex, pos)
    
    if currentLayer < wantedLayer:
        # Accumulate alpha for the layer to be peeled
        layerAlpha = layerAlpha + (1.0 - layerAlpha) * sample.a
        
        # Check if we have accumulated enough alpha and the current sample is transparent
        if layerAlpha > T_high and sample.a < T_low:
            currentLayer = currentLayer + 1
            layerAlpha = 0.0
    else:
        # Peeling done, integrate the visible layer
        # Note: Source [S1] uses simple addition. Standard compositing uses (1-a)*sample.
        # We follow the source exactly.
        sampleRGBA = sampleRGBA + sample
```

## Subtopics

### [Brain Peeling](13890-brain-peeling.md)
A view-independent technique that uses a precomputed distance field to discard voxels based on proximity to the brain surface. It requires fewer parameters and is more reliable for exploring surface-adjacent features like vessels, whereas opacity peeling is view-dependent and relies on alpha thresholds.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray. The method adds a constant amount of logic per sample (alpha accumulation and threshold checks) to standard volume rendering.
*   **Memory Complexity:** $O(1)$ additional memory per ray for the state variables (`currentLayer`, `layerAlpha`). No additional volume data structures (like distance fields) are required [S1].
*   **Performance:** As a single-pass solution, it is more efficient than multi-pass rendering techniques that might require multiple render passes to peel layers [S2].
*   **Strengths:** Simple to implement in a shader; allows interactive selection of the depth layer to view; does not require precomputation of auxiliary data structures [S1, S2].
*   **Limitations:** The quality of the peel depends heavily on the choice of `T_high` and `T_low`, which may need tuning for different datasets or transfer functions. It is view-dependent, meaning the peeled layers change as the camera moves [S1, S3].
*   **When to use:** When interactive exploration of internal structures is needed without precomputing auxiliary data, and the user can tolerate view-dependent results [S1, S3].
*   **When not to use:** When view-independent consistency is required, or when the volume has complex, non-layered structures where alpha thresholds do not correspond to meaningful anatomical layers [S1].

## Background
Opacity peeling was proposed by Rezk-Salama and Kolb as a solution to the occlusion problem in volume visualization [S1]. It is employed in the PRISM framework for interactive design of GPU volume rendering shaders [S1]. The technique allows users to reveal interior hidden spatial structures by discarding the first $n$ layers of tissue, where $n$ is defined by the user [S1, S2]. It is related to other peeling techniques such as those discussed in the WYSIWYP method, but specifically uses opacity accumulation thresholds rather than extrema detection or stroke-based segmentation [S3].

## Sources
[S1] PRISM: An open source framework for the interactive design of GPU volume rendering shaders., https://doi.org/10.1371/journal.pone.0193636, fetched 2026-10-26
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S3] WYSIWYP: What You See Is What You Pick, https://doi.org/10.1109/tvcg.2012.292, fetched 2026-11-05
