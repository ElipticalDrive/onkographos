# Binary Classification

Also called: Binary Classifier

## Overview
Binary classification is a volume rendering method that reduces memory consumption and accelerates rendering by categorizing input volume voxels into two distinct groups: "0-voxels" (excluded) and "1-voxels" (included) based on a threshold [S1, unofficial]. It functions as a hard decision process, creating sharp transitions in the output opacity and color, which distinguishes it from soft classification methods that use smooth transitions [S2, unofficial]. Unlike [Piecewise-Linear Classification](881-piecewise-linear-classification.md) which allows for gradual opacity changes, binary classification produces a binary mask of the object of interest, effectively discarding background data to enable real-time performance on large datasets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume $V$ where each voxel at position $\mathbf{x}$ has an intensity value $v = f(\mathbf{x})$. A threshold value $T$ is required. In specific implementations, a range $[T_{min}, T_{max}]$ is used [S2, unofficial].
*   **Output:** A binary mask or a reduced volume containing only the "1-voxels". Each voxel is assigned an opacity $\alpha$ of either 0 (transparent) or 1 (opaque), and a constant color $C$ if included [G].

**Data Structures:**
The method typically operates on a 3D grid. To achieve speedups, architectures like CUBE use a 3-D cubic frame buffer with skewed memory organization to allow parallel manipulation of data [S1, unofficial]. However, the core logical operation is a simple comparison against a threshold.

**Steps:**
1.  **Thresholding:** For each voxel $\mathbf{x}$ in the input volume, compare its intensity $f(\mathbf{x})$ against the threshold $T$.
    *   If $T_{min} \le f(\mathbf{x}) \le T_{max}$ (or $f(\mathbf{x}) \ge T$ for a single threshold), classify the voxel as a "1-voxel".
    *   Otherwise, classify it as a "0-voxel".
2.  **Assignment:** Assign optical properties based on the classification:
    $$ \alpha(\mathbf{x}) = \begin{cases} 1 & \text{if } \mathbf{x} \text{ is a 1-voxel} \\ 0 & \text{if } \mathbf{x} \text{ is a 0-voxel} \end{cases} $$
    $$ C(\mathbf{x}) = \begin{cases} C_{object} & \text{if } \mathbf{x} \text{ is a 1-voxel} \\ C_{background} & \text{if } \mathbf{x} \text{ is a 0-voxel} \end{cases} $$
3.  **Discarding/Optimization:** 0-voxels are discarded or skipped during rendering to reduce memory bandwidth and computation time [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_classify(volume: List[float], threshold_min: float, threshold_max: float) -> List[bool]:
    """
    Classifies volume voxels into 1-voxels (True) and 0-voxels (False).
    """
    mask = []
    for v in volume:
        # Check if value falls within the object of interest range
        if threshold_min <= v <= threshold_max:
            mask.append(True)
        else:
            mask.append(False)
    return mask

def render_binary(volume: List[float], mask: List[bool], threshold_min: float, threshold_max: float):
    """
    Renders only the 1-voxels.
    """
    for i, v in enumerate(volume):
        if mask[i]:
            # Render voxel with full opacity and object color
            draw_voxel(i, opacity=1.0, color=OBJECT_COLOR)
        else:
            # Skip 0-voxels to save time/memory
            pass
```

**Parameters:**
*   **Threshold ($T$):** The value separating background from foreground. Typical values depend on the data range (e.g., 0.3 to 0.7 for normalized data) [S2, unofficial].
*   **Range ($[T_{min}, T_{max}]$):** Defines the interval of intensities considered part of the object [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Artifacts:** Binary classification introduces sharp transitions in the output function, which generates high frequencies in the output spectrum. This leads to significant rendering artifacts, such as aliasing and jagged edges [S2, unofficial].
*   **Accuracy:** The technique is prone to false positives (including background as object) and false negatives (excluding object as background), which severely affect image quality [S1, unofficial].
*   **Blurring vs. Aliasing:** If pre-classification is used (applying the threshold to voxel values before interpolation), the image may appear blurred if the rendering resolution exceeds the volume resolution. If post-classification is used (interpolating first, then thresholding), sharp edges are preserved but aliasing artifacts are more prominent [S2, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** The classification step is $O(N)$ where $N$ is the number of voxels. When combined with optimized architectures like CUBE, rendering time can be reduced from $O(n^3)$ to $O(n^2 \log n)$ [S1, unofficial].
*   **Memory Complexity:** By discarding 0-voxels, memory consumption is significantly reduced, proportional to the ratio of 1-voxels to total voxels [S1, unofficial].
*   **Strengths:** Enables real-time rendering speeds for massive datasets by eliminating unnecessary computation and memory access for background data [S1, unofficial].
*   **Limitations:** Poor image quality due to aliasing and artifacts from sharp transitions; sensitive to threshold selection [S1, unofficial][S2, unofficial].
*   **When to use:** When real-time performance is critical and the data has a clear separation between background and foreground, or when used as a preprocessing step for further compositing.
*   **When not to use:** When high-fidelity, smooth edges and accurate representation of semi-transparent structures are required; in such cases, volumetric compositing or soft classification is preferred [S1, unofficial].

## Background
Binary classification was one of the earliest techniques adopted to achieve real-time volume rendering on supercomputers and massive clusters [S1, unofficial]. It became a distinguishable feature of early real-time rendering algorithms, with notable architectures like the CUBE (CUbic frame Buffer) system proposed by Kaufman, which utilized binary classification to manipulate data in a 3-D cubic frame buffer [S1, unofficial]. The technique was widely used before volumetric compositing became the standard pipeline, as compositing allowed for semi-transparent rendering without the binary artifacts [S1, unofficial]. The theoretical basis for the artifacts introduced by binary classification (high frequencies in the spectrum) was analyzed by Frieder [S2, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
