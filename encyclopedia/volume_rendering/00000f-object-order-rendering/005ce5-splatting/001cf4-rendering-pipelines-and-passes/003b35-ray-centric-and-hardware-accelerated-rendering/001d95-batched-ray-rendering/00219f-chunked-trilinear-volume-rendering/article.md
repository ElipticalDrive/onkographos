# Chunked Trilinear Volume Rendering
## Overview
Chunked Trilinear Volume Rendering evaluates a neural field or trilinear volume by processing rays in fixed-size batches [S1, unofficial]. It prevents memory overflow by iterating over subsets of rays rather than evaluating the full batch at once [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
- `rays`: Tensor of shape `(N_rays_all, ...)` containing ray origins and directions [S1, unofficial].
- `chunk`: Integer specifying the number of rays per processing batch [S1, unofficial].
- `tensorf`: The neural field or trilinear volume evaluation function [S1, unofficial].
- `device`: The compute device (e.g., CUDA) [S1, unofficial].
- `is_train`: Boolean flag for gradient tracking mode [S1, unofficial].
- `white_bg`: Boolean flag for background color assumption [S1, unofficial].
- `ndc_ray`: Boolean flag for NDC coordinate transformation [S1, unofficial].
- `N_samples`: Integer for samples per ray [S1, unofficial].

**Outputs:**
- `rgbs`: Concatenated RGB color maps [S1, unofficial].
- `depth_maps`: Concatenated depth maps [S1, unofficial].
- `None`: Placeholder for `alphas`, `weights`, and `uncertainties` to maintain interface compatibility [S1, unofficial].

### Data Structures
- `rgbs`: List of tensors, each of shape `(chunk, 3)` or `(remaining_rays, 3)` [S1, unofficial].
- `depth_maps`: List of tensors, each of shape `(chunk, 1)` or `(remaining_rays, 1)` [S1, unofficial].
- `rays_chunk`: Tensor of shape `(chunk, ...)` or `(remaining_rays, ...)` [S1, unofficial].

### Steps
1. Initialize empty lists for output tensors: `rgbs`, `alphas`, `depth_maps`, `weights`, `uncertainties` [S1, unofficial].
2. Calculate total rays: $N_{\text{rays\_all}} = \text{rays.shape}[0]$ [S1, unofficial].
3. Determine the number of chunks:
   $$ N_{\text{chunks}} = \lfloor N_{\text{rays\_all}} / \text{chunk} \rfloor + \mathbb{I}(N_{\text{rays\_all}} \bmod \text{chunk} > 0) $$
   where $\mathbb{I}$ is the indicator function [S1, unofficial].
4. Iterate `chunk_idx` from $0$ to $N_{\text{chunks}} - 1$:
   a. Extract and move chunk to device: $\text{start} = \text{chunk\_idx} \times \text{chunk}$, $\text{end} = (\text{chunk\_idx} + 1) \times \text{chunk}$, $\text{rays\_chunk} = \text{rays}[\text{start}:\text{end}]$ copied to device [S1, unofficial].
   b. Perform the forward pass through the neural field: $\text{rgb\_map}, \text{depth\_map} = \text{tensorf}(\text{rays\_chunk}, \text{is\_train}=\text{is\_train}, \text{white\_bg}=\text{white\_bg}, \text{ndc\_ray}=\text{ndc\_ray}, N\_samples=N\_samples)$ [S1, unofficial].
   c. Append results to lists: $\text{rgbs}.\text{append}(\text{rgb\_map})$, $\text{depth\_maps}.\text{append}(\text{depth\_map})$ [S1, unofficial].
5. Concatenate along ray dimension and return: $\text{concatenate}(\text{rgbs})$, $\text{None}$, $\text{concatenate}(\text{depth\_maps})$, $\text{None}$, $\text{None}$ [S1, unofficial].

### Parameters
- `N_samples`: Default `-1`. Number of samples per ray [S1, unofficial].
- `white_bg`: Default `True`. Background color assumption for alpha compositing [S1, unofficial].
- `ndc_ray`: Default `False`. Enables NDC coordinate transformation [S1, unofficial].
- `is_train`: Default `False`. Enables gradient tracking mode [S1, unofficial].

### Edge Cases and Pitfalls
- Explicitly returns `None` for `alphas`, `weights`, and `uncertainties` [S1, unofficial]. Pipelines expecting full volumetric outputs will crash or produce invalid data if they unpack all five return values [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def concatenate(tensors, dim=0):
    """Concatenates a list of tensors along the specified dimension."""
    if not tensors:
        return None
    import numpy as np
    return np.concatenate(tensors, axis=dim)

def copy_to_device(tensor, device):
    """Copies a tensor to the specified compute device."""
    return tensor.to(device)

def chunked_trilinear_volume_rendering(rays, chunk, tensorf, device, is_train=False, white_bg=True, ndc_ray=False, N_samples=-1):
    rgbs = []
    alphas = []
    depth_maps = []
    weights = []
    uncertainties = []

    N_rays_all = rays.shape[0]
    num_chunks = N_rays_all // chunk + int(N_rays_all % chunk > 0)

    for chunk_idx in range(num_chunks):
        start = chunk_idx * chunk
        end = (chunk_idx + 1) * chunk
        rays_chunk = copy_to_device(rays[start:end], device)

        rgb_map, depth_map = tensorf(
            rays_chunk,
            is_train=is_train,
            white_bg=white_bg,
            ndc_ray=ndc_ray,
            N_samples=N_samples
        )

        rgbs.append(rgb_map)
        depth_maps.append(depth_map)

    final_rgbs = concatenate(rgbs, dim=0)
    final_depths = concatenate(depth_maps, dim=0)

    return final_rgbs, None, final_depths, None, None
```
## Operating characteristics
Not given in the sources.
## Background
The method originates from implementations in the Collaborative Neural Radiance Fields for Novel View Synthesis repository [S1, unofficial].
## Sources
[S1] Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
