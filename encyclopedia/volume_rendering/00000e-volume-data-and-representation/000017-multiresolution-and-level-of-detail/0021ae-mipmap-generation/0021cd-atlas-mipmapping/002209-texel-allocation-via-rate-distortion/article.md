# Texel Allocation via Rate-Distortion

Also called: Atlas Baking, Bake and Texel Allocation, Stage 2, Support-Aware Extent and Rate-Distortion Allocation, Texel Allocation

## Overview
This method optimizes the spatial footprint and texel count of surfels in a texture atlas by excluding zero-support regions and allocating texels based on visual importance rather than strict Nyquist limits [S1, unofficial]. It addresses the inefficiency of uniform atlas sizing where distant surfels consume excessive memory for negligible visual contribution [S1, unofficial]. It differs from its parent [Atlas Mipmapping](2639-atlas-mipmapping.md) by actively reducing the atlas size through rate-distortion optimization, and from its sibling [Anisotropic Texture Atlas Packing](2670-anisotropic-texture-atlas-packing.md) by focusing on per-surfel size allocation via error probing rather than geometric packing constraints [S1, unofficial].

## How it works
The method consists of two phases: baking the neural representation into a high-resolution atlas, and allocating texel counts per surfel using a rate-distortion probe.

### Inputs and Outputs
**Inputs:**
- Trained hash grid and MLP parameters.
- Surfel positions, orientations, and scales.
- Training view data (for accumulated visibility weights).
- Target distortion threshold (set to BC7 codec error).

**Outputs:**
- A shelf-packed texture atlas with BC7 compression.
- Per-surfel rectangle dimensions (width $w$, height $h$).
- Surfel-to-texel mapping.

### Data Structures
- **Surfel Rectangle:** Defined by width $w$ and height $h$ in texels. Dimensions are powers of two, halved from a maximum cap ($r_{max}=64$) down to a minimum of 4, or zero (no texture).
- **Halving Ladder:** For each surfel, a sequence of candidate sizes: $(r_x, r_y) \rightarrow (r_x/2, r_y/2) \rightarrow \dots$, clamped at 4.
- **Visibility Weight ($\alpha_T$):** Accumulated opacity-weighted visibility of the surfel across training views.

### Steps

1. **Support-Aware Extent Reduction:**
   Reduce the baking extent $e$ from 4 to 3. The compact Beta kernel is identically zero for $r > 3\sigma$. This removes texels that are never fetched, saving approximately 50.5% of texels in representative indoor scenes [S1, unofficial].

2. **Baking:**
   Evaluate the trained hash grid and MLP on each surfel's tile.
   - UV coordinates range from $[-3, 3]^2$ at texel centers.
   - Maximum size is $64 \times 64$ texels per surfel.
   - Precision is FP16.
   - This produces a "raw bake" which is band-limited by the hash grid [S1, unofficial].

3. **Rate-Distortion Probe:**
   For every surfel and every size on its halving ladder:
   - Downsample the full-resolution bake to the candidate size.
   - Upsample it back to the full resolution.
   - Calculate the kernel- and opacity-weighted error between the original bake and the reconstructed version.
   - Weight this error by the surfel's accumulated visibility $\alpha_T$.
   - Let $D_i(s)$ be the importance-weighted distortion for surfel $i$ at size $s$ [S1, unofficial].

4. **Allocation via Lagrange Multiplier:**
   - Define a global Lagrange multiplier $\lambda$.
   - For each surfel $i$, select the size $s_i$ that minimizes:
     $$ J_i(s_i) = D_i(s_i) + \lambda \cdot \text{texels}(s_i) $$
   - Bisect $\lambda$ such that the total importance-weighted distortion equals the target distortion (set to the error introduced by BC7 compression).
   - Surfels assigned zero texture render from their spherical Voronoi color alone [S1, unofficial].

5. **Re-baking and Compression:**
   - Re-evaluate the hash grid and MLP at the allocated sizes.
   - Shelf-pack the resulting rectangles into a single atlas.
   - Compress the atlas using BC7 [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texel_allocation(surfels, hash_grid, mlp, training_views):
    """
    Allocates texel counts per surfel using rate-distortion optimization.
    """
    # Step 1 & 2: Bake
    # Evaluate full-resolution bake for all surfels
    full_bakes = {}
    for surfel in surfels:
        # UV in [-3, 3]^2, up to 64x64 texels, FP16
        full_bakes[surfel.id] = evaluate_hash_mlp(surfel, hash_grid, mlp, extent=3)

    # Step 3: Rate-Distortion Probe
    # Calculate visibility weights
    visibility_weights = compute_accumulated_alpha(surfels, training_views)

    # Precompute distortion for all candidate sizes
    distortions = {} # surfel_id -> {size: distortion_value}
    for surfel in surfels:
        fid = surfel.id
        full_bake = full_bakes[fid]
        max_size = 64
        min_size = 4
        
        # Generate halving ladder
        sizes = []
        w, h = max_size, max_size
        while w >= min_size and h >= min_size:
            sizes.append((w, h))
            w = max(min_size, w // 2)
            h = max(min_size, h // 2)
        sizes.append((0, 0)) # No texture
        
        surfel_distortions = {}
        for (w, h) in sizes:
            if w == 0 or h == 0:
                # No texture: distortion is effectively infinite or handled separately
                # In practice, this size is only chosen if lambda is very high
                surfel_distortions[(w, h)] = float('inf')
                continue
            
            # Downsample and upsample
            small_bake = downsample(full_bake, w, h)
            reconstructed_bake = upsample(small_bake, full_bake.width, full_bake.height)
            
            # Calculate weighted error
            # Error is kernel- and opacity-weighted MSE between full_bake and reconstructed_bake
            raw_error = calculate_weighted_mse(full_bake, reconstructed_bake)
            
            # Weight by accumulated visibility
            importance_weighted_distortion = raw_error * visibility_weights[fid]
            surfel_distortions[(w, h)] = importance_weighted_distortion
        
        distortions[fid] = surfel_distortions

    # Step 4: Allocation
    # Bisect lambda to meet BC7 distortion target
    target_distortion = get_bc7_floor_error()
    
    lambda_min = 0.0
    lambda_max = 1e9 # Sufficiently large
    
    for _ in range(50): # Bisect iterations
        lambda_mid = (lambda_min + lambda_max) / 2
        total_distortion = 0.0
        allocations = {}
        
        for surfel in surfels:
            fid = surfel.id
            best_size = None
            best_cost = float('inf')
            
            for (w, h), d in distortions[fid].items():
                texel_count = w * h if w > 0 else 0
                cost = d + lambda_mid * texel_count
                if cost < best_cost:
                    best_cost = cost
                    best_size = (w, h)
            
            allocations[fid] = best_size
            # Add distortion of chosen size to total
            total_distortion += distortions[fid][best_size]
        
        if total_distortion > target_distortion:
            lambda_min = lambda_mid
        else:
            lambda_max = lambda_mid
            
    return allocations

def re_bake_and_pack(surfels, allocations, hash_grid, mlp):
    """
    Re-evaluates at allocated sizes and packs into atlas.
    """
    atlas_rects = []
    for surfel in surfels:
        w, h = allocations[surfel.id]
        if w == 0 or h == 0:
            continue # Render from spherical Voronoi color
        
        # Re-evaluate at specific size
        bake = evaluate_hash_mlp(surfel, hash_grid, mlp, extent=3, target_size=(w, h))
        atlas_rects.append({
            'id': surfel.id,
            'width': w,
            'height': h,
            'data': bake
        })
    
    # Shelf-pack into one atlas
    atlas = shelf_pack(atlas_rects)
    
    # BC7 compress
    compressed_atlas = bc7_compress(atlas)
    return compressed_atlas
```

### Parameters and Edge Cases
- **Extent $e$:** Set to 3. Using $e=4$ includes zero-support regions, wasting ~50.5% of texels [S1, unofficial].
- **Target Distortion:** Set to the error floor of BC7 compression. This ensures the allocation cost does not exceed the quantization error [S1, unofficial].
- **Halving Ladder:** Sizes are halved until clamped at 4. Sizes below 4 are treated as "no texture" [S1, unofficial].
- **Probe Source:** Probing the *raw* bake (before finetuning) keeps 45% of texels. Probing the *finetuned* atlas keeps 60% but costs a second finetune, yields 0.05 dB less PSNR, and uses 23% more bytes [S1, unofficial]. The raw bake is preferred as it is band-limited by the hash grid, making downsampling lossless in terms of high-frequency content [S1, unofficial].

## Subtopics

### [Residual Neural Texture Baking](2709-residual-neural-texture-baking.md)
A post-training stage that converts a learned neural appearance model into a static spatial texture atlas. It differs from this method by performing the actual evaluation and storage of the residual field, while this method handles the allocation of texel counts per surfel based on rate-distortion principles.

## Operating characteristics
- **Memory Reduction:** The sized atlas achieves significant memory savings with minimal quality loss. On Mip-NeRF 360, it reduces size from 388 MB to 173 MB (0.44×) with a slight PSNR increase (27.31 vs 27.23). On Tanks & Temples, 284 MB to 132 MB (0.46×) with comparable PSNR (24.11 vs 24.10). On Deep Blending, 228 MB to 73 MB (0.32×) with increased PSNR (30.05 vs 29.87) [S1, unofficial].
- **Quality Trade-offs:** The distortion target is MSE-based. Deepest cuts (e.g., playroom at -52% bytes) may increase LPIPS by 0.002–0.010 [S1, unofficial].
- **Performance:** The method adds a baking and probing phase but eliminates neural network inference costs during rendering [S1, unofficial].
- **Limitations:** The method assumes the use of BC7 compression. The rate-distortion probe requires a full-resolution bake, which can be memory-intensive for large scenes before allocation [S1, unofficial].

## Background
This method is part of the "Bake It Till You Make It" pipeline for ultrafast spatial texture-atlas splatting [S1, unofficial]. It was introduced to address the inefficiencies of Nyquist-based sizing in neural texture atlases, where surfel sizes are determined by the hash grid resolution rather than their visual contribution to the rendered image [S1, unofficial]. The approach combines support-aware extent reduction with a Lagrange multiplier-based allocation strategy to optimize the trade-off between texel count and visual fidelity [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
