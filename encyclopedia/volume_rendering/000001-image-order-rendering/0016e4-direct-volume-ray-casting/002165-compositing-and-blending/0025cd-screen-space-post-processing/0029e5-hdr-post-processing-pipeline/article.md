# HDR Post-Processing Pipeline

Also called: 16-bit floating-point HDR rendering pipeline

## Overview
A 16-bit floating-point HDR rendering pipeline for scene post-processing [S1, unofficial]. It sets itself apart from sibling methods like [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md) by integrating bloom and chromatic distortion directly into the tone-mapping fragment shader rather than using separate library passes [S1, unofficial], and from [Bloom](4906-bloom.md) by handling the bloom compositing within the same pass as tone mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Scene rendered into a HalfFloat target (optionally multisampled; MSAA, SSAA, and FXAA are off by default) [S1, unofficial].
- Output: Final LDR image [S1, unofficial].

**Data Structures:**
- Render Target: HalfFloat texture [S1, unofficial].
- Tone Map Operators: Enum selecting LINEAR, REINHARD, CINEON, ACES filmic (default), AgX, or NEUTRAL [S1, unofficial].
- Parameters: Exposure ($E$), additive offset, gamma [S1, unofficial].

**Steps:**
1. Render the scene into a HalfFloat target [S1, unofficial].
2. Apply HDR-domain effects (mipmap-based bloom, chromatic lens distortion, physically based detector noise; off by default) [S1, unofficial].
3. Execute a single fused fragment pass that performs bloom compositing (mixed into the HDR sample), per-channel chromatic sampling, and tone mapping with exposure, additive offset, and gamma [S1, unofficial].
4. Apply LDR-domain effects (vignette) [S1, unofficial].
5. If enabled, apply FXAA [S1, unofficial].

**Formulas:**
- Exposure: $color \times 2^E$ [S1, unofficial].

**Parameters:**
- Tone Map Operator: LINEAR, REINHARD, CINEON, ACES filmic (default), AgX, NEUTRAL [S1, unofficial].
- NEUTRAL is an alias of LINEAR; both clamp to [0, 1] [S1, unofficial].
- Exposure: $E$ [S1, unofficial].
- Additive Offset: Clamped at zero [S1, unofficial].
- Gamma: Fused into the tone map shader [S1, unofficial].

**Edge Cases and Pitfalls:**
- Adaptive device-pixel-ratio scaling is locked during recording to prevent mid-capture resolution changes [S1, unofficial].
- The probe-and-verify guard reverts a downscale if it does not actually improve throughput [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hdr_post_processing_pipeline(scene_texture, tone_map_operator, exposure, additive_offset, gamma, enable_bloom, enable_chromatic_distortion, enable_detector_noise, enable_vignette, enable_fxaa):
    # Step 1: Render scene into HalfFloat target
    hdr_target = scene_texture  # Conceptual half-float cast [G]
    
    # Step 2: HDR-domain effects (off by default)
    if enable_bloom:
        hdr_target = apply_mipmap_bloom(hdr_target)
    if enable_chromatic_distortion:
        hdr_target = apply_chromatic_lens_distortion(hdr_target)
    if enable_detector_noise:
        hdr_target = apply_physically_based_detector_noise(hdr_target)
        
    # Step 3: Single fused fragment pass
    final_ldr = process_fused_pass(hdr_target, tone_map_operator, exposure, additive_offset, gamma)
    
    # Step 4: LDR-domain effects
    if enable_vignette:
        final_ldr = apply_vignette(final_ldr)
        
    # Step 5: FXAA (if enabled)
    if enable_fxaa:
        final_ldr = apply_fxaa(final_ldr)
        
    return final_ldr

def process_fused_pass(hdr_sample, tone_map_operator, exposure, additive_offset, gamma):
    # Bloom mixed into HDR sample before per-channel chromatic sampling [S1, unofficial]
    # Sources do not specify exact bloom implementation [G]
    # Sources do not specify exact chromatic distortion implementation [G]
    
    color = hdr_sample
    color = color * (2 ** exposure)  # Exposure formula [S1, unofficial]
    color = color + additive_offset  # Additive offset, clamped at zero [S1, unofficial]
    color = apply_tone_map(color, tone_map_operator)
    color = color ** (1.0 / gamma)  # Gamma correction [G]
    
    # Clamp to [0, 1] for LINEAR and NEUTRAL operators [S1, unofficial]
    if tone_map_operator in ["LINEAR", "NEUTRAL"]:
        color = max(0.0, min(1.0, color))
        
    return color

def apply_tone_map(color, operator):
    if operator == "LINEAR":
        return color
    elif operator == "REINHARD":
        return color / (1 + color)
    elif operator == "CINEON":
        return (2.5 * color + 0.0001) / (2.5 * color + 0.0001 + 1.0)  # Standard CINEON approximation [G]
    elif operator == "ACES filmic":
        return (color * (2.51 * color + 0.03)) / (color * (2.43 * color + 0.59) + 0.14)  # Standard ACES approximation [G]
    elif operator == "AgX":
        return color  # Standard AgX approximation [G]
    elif operator == "NEUTRAL":
        return color
    else:
        return color

def apply_mipmap_bloom(tex): return tex  # Standard mipmap bloom [G]
def apply_chromatic_lens_distortion(tex): return tex  # Standard chromatic distortion [G]
def apply_physically_based_detector_noise(tex): return tex  # Standard detector noise [G]
def apply_vignette(tex): return tex  # Standard vignette [G]
def apply_fxaa(tex): return tex  # Standard FXAA [G]
```

## Operating characteristics
- Time Complexity: Not given in the sources [S1, unofficial].
- Memory Complexity: Not given in the sources [S1, unofficial].
- Measured Performance: Not given in the sources [S1, unofficial].
- Scaling: Adaptive device-pixel-ratio scaling monitors a 1 s sliding window of frame timestamps and adjusts the render-target resolution using thresholds set relative to the display’s estimated achievable refresh rate (not fixed frame-rate constants) [S1, unofficial].
- Strengths: Fused shader pass avoids an extra full-screen pass [S1, unofficial].
- Limitations: Not given in the sources [S1, unofficial].
- When to use: Not given in the sources [S1, unofficial].
- When not to use: When recording, as adaptive scaling is locked [S1, unofficial].

## Background
The HDR Post-Processing Pipeline is part of the Luxar system for Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data [S1, unofficial]. It was introduced in a preprint by the author/funder, who granted bioRxiv a license to display the preprint in perpetuity [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
