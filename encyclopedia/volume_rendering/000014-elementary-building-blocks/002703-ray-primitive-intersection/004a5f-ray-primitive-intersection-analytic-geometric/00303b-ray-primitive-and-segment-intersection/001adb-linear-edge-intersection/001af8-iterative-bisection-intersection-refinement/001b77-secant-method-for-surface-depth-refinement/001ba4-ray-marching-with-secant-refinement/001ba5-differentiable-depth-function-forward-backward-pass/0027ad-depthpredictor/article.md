# DepthPredictor
## Overview
The `DepthPredictor` is a PyTorch `nn.Module` that implements a depth prediction pipeline by integrating a distribution sampler with conversion functions [S1, unofficial]. It transforms raw feature tensors containing probability density function (PDF) and offset channels into probabilistic depth estimates and associated opacities, differing from sibling methods by explicitly handling discrete distribution sampling and transmittance-based opacity computation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `features` tensor of shape `Float[Tensor, "batch view ray surface depth 2"]` [S1, unofficial]. The last dimension contains two channels: raw PDF values and raw offset values [S1, unofficial].
*   **Outputs:**
    *   `depth`: `Float[Tensor, "batch view ray surface sample"]` [S1, unofficial].
    *   `opacity`: `Float[Tensor, "batch view ray surface sample"]` [S1, unofficial].
    *   `index`: `Int64[Tensor, "batch view ray surface sample"]` [S1, unofficial].

**Data Structures:**
The method uses a sampler object that provides `sample` and `gather` methods to handle discrete sampling from the PDF and retrieval of corresponding offset values [S1, unofficial].

**Parameters:**
*   `num_depths`: Integer specifying the number of discrete depth buckets (size of the `depth` dimension in input) [S1, unofficial].
*   `near`: Near plane distance for depth conversion [S1, unofficial].
*   `far`: Far plane distance for depth conversion [S1, unofficial].
*   `deterministic`: Boolean flag controlling sampling mode [S1, unofficial].
*   `gaussians_per_pixel`: Integer specifying the number of samples to draw per pixel/ray [S1, unofficial].
*   `use_transmittance`: Boolean flag controlling the opacity calculation method [S1, unofficial].

**Steps:**
1.  **Unbind Features:** Separate the last dimension of `features` into `pdf_raw` and `offset_raw` tensors [S1, unofficial].
2.  **Normalize PDF:** Apply a Softmax function to `pdf_raw` to obtain the probability density function `pdf` [S1, unofficial].
3.  **Normalize Offsets:** Apply a Sigmoid function to `offset_raw` to obtain bounded offsets `offset` [S1, unofficial].
4.  **Sample:** Draw `index` and corresponding PDF densities `pdf_i` from `pdf` using the sampler, with the sample count determined by `gaussians_per_pixel` [S1, unofficial].
5.  **Gather Offsets:** Retrieve the offsets corresponding to the sampled indices: `offset = sampler.gather(index, offset)` [S1, unofficial].
6.  **Calculate Relative Disparity:** Compute relative disparity using the sampled indices and offsets:
    $$ \text{relative\_disparity} = \frac{\text{index} + \text{offset}}{\text{num\_depths}} $$
7.  **Convert to Depth:** Map relative disparity to absolute depth using the near and far planes:
    $$ \text{depth} = \text{relative\_disparity\_to\_depth}(\text{relative\_disparity}, \text{near}, \text{far}) $$
8.  **Compute Opacity:**
    *   If `use_transmittance` is true:
        a. Compute cumulative sum of PDF: `partial = pdf.cumsum(dim=-1)` [S1, unofficial].
        b. Shift cumulative sum to align with transmittance: `partial = torch.cat((torch.zeros_like(partial[..., :1]), partial[..., :-1]), dim=-1)` [S1, unofficial].
        c. Compute opacity per bucket: `opacity_raw = pdf / (1 - partial + 1e-10)` [S1, unofficial].
        d. Gather opacity values: `opacity = sampler.gather(index, opacity_raw)` [S1, unofficial].
    *   If `use_transmittance` is false:
        a. Set opacity directly to the sampled PDF density: `opacity = pdf_i` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class DepthPredictor(nn.Module):
    def __init__(self, sampler, num_depths, near, far, use_transmittance=True):
        super().__init__()
        self.sampler = sampler
        self.num_depths = num_depths
        self.near = near
        self.far = far
        self.use_transmittance = use_transmittance

    def forward(self, features, deterministic=False, gaussians_per_pixel=1):
        pdf_raw, offset_raw = features.unbind(dim=-1)
        pdf = torch.softmax(pdf_raw, dim=-1)
        offset = torch.sigmoid(offset_raw)
        index, pdf_i = self.sampler.sample(pdf, deterministic, gaussians_per_pixel)
        offset = self.sampler.gather(index, offset)
        relative_disparity = (index + offset) / self.num_depths
        depth = relative_disparity_to_depth(relative_disparity, self.near, self.far)
        if self.use_transmittance:
            partial = pdf.cumsum(dim=-1)
            partial = torch.cat((torch.zeros_like(partial[..., :1]), partial[..., :-1]), dim=-1)
            opacity = pdf / (1 - partial + 1e-10)
            opacity = self.sampler.gather(index, opacity)
        else:
            opacity = pdf_i
        return depth, opacity, index

def relative_disparity_to_depth(relative_disparity, near, far):
    # Standard linear mapping from normalized disparity to depth [G]
    return near + relative_disparity * (far - near)
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The opacity calculation includes a small epsilon (`1e-10`) to prevent division by zero when `1 - partial` approaches zero [S1, unofficial].
*   **Near/Far Plane Boundaries:** The `near` and `far` parameters define the valid depth range; values outside this range produce invalid depth estimates [S1, unofficial].

## Subtopics

### Monocular Depth Predictor with Distribution Sampling
A sibling method that also uses distribution sampling but may differ in the specific architecture of the predictor or the sampling strategy [S1, unofficial]. Choose this when a monocular prior is integrated directly into the sampling process [Link](3879-monocular-depth-predictor-with-distribution-sampling.md).

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
