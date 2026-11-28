# Caching for SCL Training

Also called: Caching for Efficient Training, Codec Cache, Standard-Codec-in-the-Loop (SCL) caching

## Overview
This method reduces the computational cost of invoking a standard codec round trip on every mini-batch during Standard-Codec-in-the-Loop (SCL) training, particularly for dynamic scenes where video codecs operate on sequences of frames [S1]. It maintains a per-frame cache of decoded feature planes and density volumes, refreshing them only when necessary based on time or parameter drift [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current global training step $g$, cached last refresh step $g_{cache}$, current unquantized TriPlane features $P^{ax}_t$ for each axis $ax \in \{xy, xz, yz\}$, current unquantized density grid $D_t$, and a frame index $t$ with associated rays $\{(o, d)\}$ and camera pose $\pi$ [S1].
*   **Outputs:** Updated cached decoded planes $\widehat{P}^{ax}_t$, cached decoded density $\widehat{D}_t$, and the rendered image $I$ [S1].

**Data Structures:**
*   **Cache:** Stores the decoded TriPlane $\widehat{P}^{ax}_t$ and a snapshot of the raw plane $P^{ax}_t$ at the time of refresh for each axis $ax \in \{xy, xz, yz\}$ [S1].
*   **Cache:** Stores the decoded density grid $\widehat{D}_t$ and a snapshot of the raw density $D_t$ at the time of refresh [S1].
*   **Scalar:** $g_{cache}$, the global step at which the last cache refresh occurred [S1].

**Parameters:**
*   $M$: The maximum interval between step-based refreshes. Typical value: $M=128$ [S1].
*   $\epsilon$: Threshold for relative change (drift) to trigger change-based refresh. Measured by normalized $\ell_2$ difference [S1]. Value not given in the sources.

**Algorithm:**
1.  Sample a frame index $t$, rays $\{(o, d)\}$, and camera pose $\pi$ [S1].
2.  Determine if a refresh is required:
    *   **Step-based refresh:** If the cache is empty or $g - g_{cache} \ge M$, mark `refresh` as true [S1].
    *   **Change-based refresh:** Calculate the relative change between the current $(P^{ax}_t, D_t)$ and their cached snapshots. If this relative change exceeds $\epsilon$ (measured by normalized $\ell_2$ difference), mark `refresh` as true, even if $g - g_{cache} < M$ [S1].
3.  If `refresh` is true:
    *   Run the encode–decode codec round trip for the video segment to update the cache [S1].
    *   Update $g_{cache} \leftarrow g$ [S1].
4.  Perform Straight-Through Estimator (STE) substitution using cached reconstructions [S1]:
    *   For each axis $ax \in \{xy, xz, yz\}$:
        $$ \widehat{P}^{ax}_t \leftarrow \text{cached decoded plane} $$
        $$ \widetilde{P}^{ax}_t \leftarrow \widehat{P}^{ax}_t + (P^{ax}_t - \texttt{detach}(P^{ax}_t)) $$
    *   For density:
        $$ \widehat{D}_t \leftarrow \text{cached decoded density} $$
        $$ \widetilde{D}_t \leftarrow \widehat{D}_t + (D_t - \texttt{detach}(D_t)) $$
    Here, $\widehat{P}$ and $\widehat{D}$ are the quantized/cached values, $\widetilde{P}$ and $\widetilde{D}$ are the values used for rendering/loss, and $P$ and $D$ are the current unquantized values [S1].
5.  Render and compute losses:
    $$ I \leftarrow \mathcal{R}(\widetilde{\mathcal{P}}_t, \widetilde{D}_t, \pi; \phi) $$
    where $\mathcal{R}$ is the rendering function and $\phi$ represents the network parameters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normalized_l2_drift(current, cached):
    diff = current - cached
    norm_diff = sqrt(sum(diff**2))
    norm_current = sqrt(sum(current**2))
    if norm_current == 0:
        return 0.0
    return norm_diff / norm_current

def codec_round_trip(planes, density):
    # Source does not specify codec implementation details [G]
    return planes, density

def volumetric_render(planes, density, rays, pose):
    # Standard volumetric rendering step [G]
    return render_output

def scl_training_step_with_caching(
    global_step, frame_index, rays, camera_pose,
    current_planes, current_density,
    cached_decoded_planes, cached_raw_planes,
    cached_decoded_density, cached_raw_density,
    last_refresh_step, refresh_interval, drift_threshold
):
    refresh_needed = False

    # Step-based refresh
    if last_refresh_step == -1 or (global_step - last_refresh_step) >= refresh_interval:
        refresh_needed = True

    # Change-based refresh
    if not refresh_needed:
        total_drift = 0.0
        for ax in ['xy', 'xz', 'yz']:
            total_drift += compute_normalized_l2_drift(current_planes[ax], cached_raw_planes[ax])
        total_drift += compute_normalized_l2_drift(current_density, cached_raw_density)

        if total_drift > drift_threshold:
            refresh_needed = True

    if refresh_needed:
        new_decoded_planes, new_decoded_density = codec_round_trip(current_planes, current_density)
        cached_decoded_planes = new_decoded_planes
        cached_decoded_density = new_decoded_density
        cached_raw_planes = {ax: current_planes[ax] for ax in current_planes}
        cached_raw_density = current_density
        last_refresh_step = global_step

    rendered_planes = {}
    for ax in ['xy', 'xz', 'yz']:
        rendered_planes[ax] = cached_decoded_planes[ax] + (current_planes[ax] - detach(current_planes[ax]))

    rendered_density = cached_decoded_density + (current_density - detach(current_density))

    rendered_image = volumetric_render(rendered_planes, rendered_density, rays, camera_pose)

    return rendered_image, last_refresh_step, cached_decoded_planes, cached_raw_planes, cached_decoded_density, cached_raw_density
```

**Edge Cases and Pitfalls:**
*   **Precision:** The STE substitution relies on the difference between the current unquantized value and its detached version [S1].
*   **Boundaries:** The change-based refresh depends on the normalized $\ell_2$ difference. If the current planes or density are near zero, the normalization factor becomes small, potentially leading to numerical instability or false triggers [G].
*   **Degenerate Input:** If the cache is empty, a refresh is always triggered [S1].

## Operating characteristics
*   **Time Complexity:** The cost of the codec round trip is amortized over $M$ training steps [S1].
*   **Memory Complexity:** Requires storing additional cached decoded planes and density grids, as well as snapshots of the raw planes and density at the time of refresh [S1].
*   **Measured Performance:** Tab. 4 in [S1] shows that increasing $M$ yields a favorable trade-off between accuracy and training efficiency. $M=128$ captures most of the benefit of SCL training [S1].
*   **Strengths:** Reduces training time significantly for dynamic scenes by avoiding frequent codec invocations. The change-based refresh ensures responsiveness to rapid parameter changes early in training [S1].
*   **Limitations:** The cache may lag behind rapid parameter changes if the drift threshold $\epsilon$ is not set appropriately [S1].
*   **When to use:** When training SCL models on dynamic scenes where codec round trips are computationally expensive [S1].

## Background
This caching mechanism was introduced in the CATRF (Codec-Adaptive TriPlane Radiance Fields) paper [S1]. It addresses the computational bottleneck of invoking standard codecs in the training loop, which is particularly problematic for dynamic scenes [S1].

## Sources
[S1] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
