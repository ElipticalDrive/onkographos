# RNR-Map Rendering (Decoding)

Also called: Decoding and Rendering (Fdec), Locally Conditioned Radiance Fields Rendering

## Overview
RNR-Map Rendering (Decoding) is the process of reconstructing an RGBD image from a 2D latent map by sampling latent codes along camera rays and converting them into locally conditioned radiance fields using modulation linear layers [S1]. It solves the problem of generalizable neural rendering by allowing image synthesis from arbitrary, unseen environments without per-scene optimization, distinguishing it from standard NeRF which requires continuous field optimization for each specific scene [S1].

## How it works
**Inputs:**
*   Query pose $P_q$ (camera parameters).
*   RNR-Map $m$ (2D latent map of size $U \times V$).
*   Decoder parameters $\theta_{dec}$ (pretrained).

**Outputs:**
*   Reconstructed RGBD image $I$.

**Data Structures:**
*   **RNR-Map ($m$):** A 2D grid of latent codes.
    *   Dimensions: $U \times V$ (height and width).
    *   Element $m(u, v)$: A latent code vector of dimension $D$ ($m(u, v) \in \mathbb{R}^D$).
    *   Mask ($n$): A parallel grid of integers recording the number of averaged features at each position, $n(u, v) = |X_{(u,v)}|$ [S1].

**Steps:**
1.  **Ray Definition:** For each pixel in the target image, define a camera ray corresponding to the query pose $P_q$ [S1].
2.  **Latent Code Sampling:** Sample latent codes from the RNR-Map $m$ along the camera ray. The specific sampling mechanism (e.g., interpolation, nearest neighbor) is not detailed in the provided text, but the codes are retrieved based on the ray's intersection with the map coordinates [S1].
3.  **Modulation:** Convert the sampled latent codes into modulation linear layer-based locally conditioned radiance fields. This involves using the latent codes to modulate the weights or biases of the decoder's linear layers [S1].
4.  **Rendering:** Render the RGBD image from these locally conditioned radiance fields using the pretrained decoder $\theta_{dec}$ [S1].

**Formulas:**
The training loss for the encoder and decoder is defined as the L1 norm between the original observation $I_i$ and the rendered image [S1]:

$$
\mathrm{Loss}(\theta_{enc}, \theta_{dec}) = \frac{1}{N} \sum_{i=1}^N ||I_i - F_{dec}(m_N^g, p_i; \theta_{dec})||_1
$$

Where:
*   $N$ is the number of images.
*   $I_i$ are the original observations.
*   $m_N^g$ is the final global map.
*   $p_i$ are the poses.
*   $F_{dec}$ is the decoding function.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_rnr_map(query_pose, rnr_map, decoder):
    """
    Renders an RGBD image from an RNR-Map given a query pose.
    
    Args:
        query_pose: Camera parameters (intrinsics, extrinsics).
        rnr_map: 2D grid of latent codes m of shape (U, V, D).
        decoder: Pretrained decoder network theta_dec.
        
    Returns:
        rendered_image: Reconstructed RGBD image.
    """
    # 1. Define camera rays for each pixel
    # Assuming standard pinhole camera model
    height, width = rnr_map.shape[0], rnr_map.shape[1]
    rays = define_rays_from_pose(query_pose, height, width)
    
    rendered_image = []
    
    for ray in rays:
        # 2. Sample latent codes from the RNR-Map along the ray
        # The text states sampling occurs along the camera ray corresponding to pixel location.
        # Specific interpolation method is not given in sources.
        # [G] Assuming bilinear interpolation for continuous sampling from discrete grid.
        latent_code = sample_latent_code_from_map(rnr_map, ray)
        
        # 3. Convert sampled latent codes into modulation linear layer-based 
        # locally conditioned radiance fields
        # [G] The decoder uses the latent code to modulate its internal layers.
        # This step is encapsulated in the decoder call below.
        
        # 4. Render the pixel color/intensity
        pixel_value = decoder.decode(latent_code, ray)
        rendered_image.append(pixel_value)
        
    return assemble_image(rendered_image, height, width)

def sample_latent_code_from_map(rnr_map, ray):
    """
    Samples a latent code from the RNR-Map based on the ray's projection.
    [G] Implementation detail: Map ray intersection to 2D grid coordinates (u, v)
    and interpolate m(u, v).
    """
    # Project ray to 2D map coordinates (u, v)
    u, v = project_ray_to_map(ray, rnr_map.shape)
    
    # Clamp coordinates to map boundaries
    u = max(0, min(u, rnr_map.shape[0] - 1))
    v = max(0, min(v, rnr_map.shape[1] - 1))
    
    # [G] Bilinear interpolation
    return interpolate(rnr_map, u, v)
```

**Edge Cases and Pitfalls:**
*   **Unseen Environments:** The method is designed to handle unseen environments by embedding useful information from images into the map, but performance depends on the encoder's ability to generalize [S1].
*   **Precision:** The text does not specify precision issues, but standard floating-point precision applies to the L1 loss and interpolation [S1].

## Operating characteristics
*   **Time Complexity:** Real-time capable. The majority of present NeRF-based navigation methods require significant time for inference due to computation-heavy image rendering, whereas RNR-Map avoids per-scene optimization [S1].
*   **Memory Complexity:** Depends on the size of the RNR-Map ($U \times V \times D$). The map stores discrete latent codes, allowing direct lookup [S1].
*   **Strengths:** Generalizable to arbitrary scenes without additional optimization; real-time rendering capability; handles unseen environments [S1].
*   **Limitations:** Requires a pretrained encoder and decoder; relies on the quality of the latent map construction [S1].
*   **When to use:** Visual navigation tasks in unseen environments where real-time rendering is required [S1].
*   **When not to use:** When per-scene optimization is acceptable and higher fidelity is needed than what the latent map can provide [S1].

## Background
RNR-Map was introduced in the paper "Renderable Neural Radiance Map for Visual Navigation" [S1]. It addresses the limitation of standard NeRF, which optimizes a continuous field and is not generalizable to various scenes, a serious problem for visual navigation tasks in unseen environments [S1]. The method uses a pair of encoder and decoder networks to embed visual information from images into a latent map and reconstruct images using neural radiance fields conditioned on these latent codes [S1].

## Sources
[S1] Renderable Neural Radiance Map for Visual Navigation, https://doi.org/10.1109/cvpr52729.2023.00878, fetched 2026-10-26
