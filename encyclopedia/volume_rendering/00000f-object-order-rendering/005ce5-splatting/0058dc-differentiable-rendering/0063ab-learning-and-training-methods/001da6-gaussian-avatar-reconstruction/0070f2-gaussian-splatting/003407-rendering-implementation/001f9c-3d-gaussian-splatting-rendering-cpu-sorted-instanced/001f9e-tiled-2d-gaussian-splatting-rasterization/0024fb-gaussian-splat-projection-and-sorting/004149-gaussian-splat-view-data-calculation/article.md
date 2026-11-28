# Gaussian Splat View Data Calculation

Also called: FGaussianSplatCalcViewDataCS

## Overview
A GPU compute shader that transforms Gaussian splats to screen space, evaluates view-dependent spherical harmonics, and handles 4D temporal or native 4D transformations per frame [S1, unofficial]. It serves as the final preparation step before sorting and rasterization, handling complex 4D temporal variations (keyframe interpolation, temporal marginalization, or native 4D primitives) and advanced culling mechanisms (cluster-based, LOD, and distance-based) [S1, unofficial]. It differs from its parent [Gaussian Splat Projection and Sorting](3369-gaussian-splat-projection-and-sorting.md) by focusing specifically on the evaluation of appearance (SH), 4D state resolution, and the population of the `ViewDataBuffer` with screen-space attributes required for the subsequent tiled rasterization pass [G].

## How it works
### Inputs and Outputs
**Inputs:**
- **Splat Data:**
  - `PackedSplatBuffer`: `ByteAddressBuffer`, 16 bytes/splat. Contains position, rotation (quaternion), scale, and opacity [S1, unofficial].
  - `SHBuffer`: `ByteAddressBuffer`. Stores spherical harmonic coefficients [S1, unofficial].
  - `TemporalBuffer`: `ByteAddressBuffer`, 16 bytes/splat. Used for temporal marginalization (velocity/motion) [S1, unofficial].
  - `KeyframeBuffer`: `ByteAddressBuffer`, 64 bytes/splat/frame. Stores baked splat data for keyframe-based 4D rendering [S1, unofficial].
  - `Native4DBuffer`: `ByteAddressBuffer`, 80 bytes/splat. Stores dual-quaternion rotations and spherindrical harmonic coefficients for native 4D primitives [S1, unofficial].
- **Culling/Visibility Data:**
  - `SplatClusterIndexBuffer`: `StructuredBuffer<uint>`. Maps splat indices to clusters [S1, unofficial].
  - `ClusterVisibilityBitmap`: `StructuredBuffer<uint>`. Bitmask of visible clusters [S1, unofficial].
  - `LODClusterSelectedBitmap`: `StructuredBuffer<uint>`. Bitmask for LOD cluster visibility [S1, unofficial].
  - `SelectedClusterBuffer`: `StructuredBuffer<uint>`. List of selected cluster indices [S1, unofficial].
  - `CompactedSplatIndices`: `StructuredBuffer<uint>`. List of visible splat indices (used if compaction is enabled) [S1, unofficial].
- **Transform Matrices:**
  - `LocalToWorld`: `FMatrix44f` [S1, unofficial].
  - `WorldToPLY`: `FMatrix44f`. Transforms world directions to PLY/SH space for SH evaluation [S1, unofficial].
  - `WorldToClip`: `FMatrix44f` [S1, unofficial].
  - `WorldToView`: `FMatrix44f` [S1, unofficial].
- **Camera/Scene Parameters:**
  - `CameraPosition`: `FVector3f` [S1, unofficial].
  - `ScreenSize`: `FVector2f` [S1, unofficial].
  - `FocalLength`: `FVector2f` [S1, unofficial].
  - `PreViewTranslation`: `FVector3f`. Used for velocity calculation; `TranslatedWorld = World + PreViewTranslation` [S1, unofficial].
- **Control Flags & Counts:**
  - `SplatCount`: Total splats to process [S1, unofficial].
  - `UseKeyframes`, `UseTemporal`, `UseNative4D`: Boolean flags (`uint32`) enabling specific 4D modes [S1, unofficial].
  - `KeyframeSplatCount` (M), `KeyframeFrameCount` (N), `KeyframeFrameTime` [S1, unofficial].
  - `Native4DSplatCount` [S1, unofficial].
  - `TimeDuration` (l) [S1, unofficial].
  - `SHOrder`, `NumSHCoeffs`, `SHBufferCoeffs` [S1, unofficial].
  - `UseSHRendering`, `OpacityScale`, `SplatScale` [S1, unofficial].
  - `MaxDrawDistance`, `FadeOutStartDistance` [S1, unofficial].
  - `UseClusterCulling`, `UseLODRendering`, `UseCompaction`, `UseGlobalCompactionPath` [S1, unofficial].
  - `VisibleSplatCount`, `OriginalSplatCount`, `MaxRenderBudget` [S1, unofficial].
  - `GlobalBaseOffset`, `ProxyIndex` [S1, unofficial].
  - `TimeWeightMinAlpha` [S1, unofficial].

**Outputs:**
- `ViewDataBuffer`: `RWStructuredBuffer<FGaussianSplatViewData>`. Contains screen-space position, covariance, color (evaluated SH), opacity, and depth/visibility flags for each active splat [S1, unofficial].

### Data Structures
The output structure `FGaussianSplatViewData` is not explicitly defined in the source notes [S1, unofficial]. Based on standard Gaussian Splatting rasterization requirements, it contains at least: `float3` or `float4` screen-space position (x, y, z/w) [G]; `float3x3` or `float4x4` 2D covariance matrix (or its Cholesky decomposition/affine parameters) [G]; `float3` or `float4` color (RGB/RGBA) [G]; `float` opacity [G]; `uint` or `float` depth/z-value [G]; `uint` cluster index or visibility flags [G].

### Parameters and Typical Values
Parameters are passed as shader constants or buffers [S1, unofficial]. `THREADGROUP_SIZE` is defined as 256 during shader compilation [S1, unofficial]. `SHOrder` determines the spherical harmonic order, with `NumSHCoeffs` derived as $(L+1)^2$ for order $L$ (commonly 4, 9, or 16) [S1, unofficial]. `SHBufferCoeffs` indicates the number of coefficients actually stored per splat based on asset SHBands [S1, unofficial]. `MaxDrawDistance` sets the distance-based visibility range, where 0 means unlimited [S1, unofficial]. `FadeOutStartDistance` defines the distance at which splats begin to fade out [S1, unofficial]. `MaxRenderBudget` caps the write index, where 0 means no cap [S1, unofficial]. `UseKeyframes`, `UseTemporal`, and `UseNative4D` are `uint32` flags enabling keyframe-based 4D rendering, 4D temporal marginalization, and Fudan native 4D (dual-quaternion rotation + spherindrical harmonics), respectively [S1, unofficial]. `KeyframeSplatCount` (M), `KeyframeFrameCount` (N), and `KeyframeFrameTime` control keyframe interpolation [S1, unofficial]. `Native4DSplatCount` specifies the number of splats covered by the Native4DBuffer [S1, unofficial]. `TimeDuration` (l) is the time duration (TimeEnd - TimeStart) for native 4D SH evaluation [S1, unofficial]. `UseSHRendering`, `OpacityScale`, and `SplatScale` control appearance rendering and scaling [S1, unofficial]. `UseClusterCulling`, `UseLODRendering`, `UseCompaction`, and `UseGlobalCompactionPath` enable cluster culling, LOD rendering, compaction processing, and global base offset reading, respectively [S1, unofficial]. `VisibleSplatCount` and `OriginalSplatCount` track visible splats and the boundary where LOD splats start [S1, unofficial]. `GlobalBaseOffset` and `ProxyIndex` manage buffer offsets for the non-compaction and global compaction paths [S1, unofficial]. `PreViewTranslation` is used for velocity calculation (`TranslatedWorld = World + PreViewTranslation`) [S1, unofficial]. `CameraPosition`, `ScreenSize`, and `FocalLength` define the camera state [S1, unofficial]. `TimeWeightMinAlpha` sets the minimum alpha threshold for temporal weighting [S1, unofficial].

### Method Steps
1. **Dispatch Setup:** The shader is dispatched with `THREADGROUP_SIZE = 256`. The thread group count is derived from `SplatCount` or `VisibleSplatCount` [S1, unofficial].
2. **Splat Index Resolution:** If `UseCompaction` is true, the thread ID maps to `CompactedSplatIndices` to retrieve the original splat index [S1, unofficial]. If `UseGlobalCompactionPath` is true, the base offset is read from `GlobalBaseOffsetsBuffer[ProxyIndex]`; otherwise, `GlobalBaseOffset` is used directly [S1, unofficial].
3. **4D State Evaluation:** The active 4D mode is selected by priority: `UseNative4D` > `UseKeyframes` > `UseTemporal` [S1, unofficial]. Native 4D reads dual-quaternion rotations and spherindrical SH from `Native4DBuffer`, interpolating based on `CurrentTime` and `TimeDuration` (l) [S1, unofficial]. Keyframe mode interpolates between `KeyframeFrameTime` and adjacent frames using `KeyframeBuffer`, applying slerp for rotation [S1, unofficial]. Temporal marginalization reads velocity/motion from `TemporalBuffer`, extrapolates position using `CurrentTime` and `PreViewTranslation`, and adjusts covariance for temporal variance [S1, unofficial]. Static mode reads directly from `PackedSplatBuffer` [S1, unofficial].
4. **Visibility Culling:** Distance culling checks `CameraPosition` against `MaxDrawDistance` (0 disables it) and applies a fade-out factor if distance exceeds `FadeOutStartDistance` [S1, unofficial]. Cluster culling (`UseClusterCulling`) checks `SplatClusterIndexBuffer` against `ClusterVisibilityBitmap` and `LODClusterSelectedBitmap` (if `UseLODRendering`) [S1, unofficial]. LOD rendering filters splats with index $\ge$ `OriginalSplatCount` [S1, unofficial].
5. **Projection to Screen Space:** The splat center is transformed to clip space via `WorldToClip`, then projected to screen coordinates using `ScreenSize` and `FocalLength` [S1, unofficial]. The 2D covariance is computed by transforming the 3D covariance with the projection Jacobian [G].
6. **Spherical Harmonics Evaluation:** If `UseSHRendering` is true, the view direction is transformed to PLY/SH space via `WorldToPLY`, and SH coefficients from `SHBuffer` (or `Native4DBuffer`) are evaluated [S1, unofficial]. `OpacityScale` and `SplatScale` are applied to the result [S1, unofficial].
7. **Write View Data:** Computed attributes are written to `ViewDataBuffer` at `base_offset + global_idx` [S1, unofficial]. Writes are skipped if `MaxRenderBudget` is set and the index $\ge$ `MaxRenderBudget` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
THREADGROUP_SIZE = 256

def evaluate_sh(view_dir_ply, sh_coeffs, sh_order, num_coeffs):
    """Evaluates spherical harmonics for a given view direction."""
    # [G] Standard SH evaluation: sum over l, m of Y_l^m(theta, phi) * c_l^m
    color = [0.0, 0.0, 0.0]
    # Standard implementation iterates through coefficients and applies basis functions
    for i in range(num_coeffs):
        # Basis function evaluation depends on sh_order and view_dir_ply
        # color[0] += sh_coeffs[i * 3] * basis[i]
        # color[1] += sh_coeffs[i * 3 + 1] * basis[i]
        # color[2] += sh_coeffs[i * 3 + 2] * basis[i]
        pass
    return color

def slerp(q1, q2, t):
    """Spherical linear interpolation of quaternions."""
    # [G] Standard slerp: normalize(q1 * (1-t) + q2 * t) or dot-product based
    return q1

def interpolate_dual_quaternion(dq1, dq2, t):
    """Interpolates dual quaternions for native 4D."""
    # [G] Standard dual quaternion interpolation: slerp for rotation, lerp for translation
    return dq1

def main(thread_id_x, UseCompaction, CompactedSplatIndices, SplatCount,
         UseGlobalCompactionPath, GlobalBaseOffsetsBuffer, ProxyIndex, GlobalBaseOffset,
         UseNative4D, Native4DBuffer, CurrentTime, TimeDuration, UseKeyframes,
         KeyframeFrameTime, KeyframeBuffer, UseTemporal, TemporalBuffer, PreViewTranslation,
         PackedSplatBuffer, SplatScale, OpacityScale, MaxDrawDistance, FadeOutStartDistance,
         CameraPosition, UseClusterCulling, SplatClusterIndexBuffer, ClusterVisibilityBitmap,
         UseLODRendering, LODClusterSelectedBitmap, OriginalSplatCount, WorldToClip,
         ScreenSize, FocalLength, WorldToPLY, UseSHRendering, SHBuffer, Native4DSplatCount,
         SHOrder, NumSHCoeffs, SHBufferCoeffs, MaxRenderBudget, GlobalBaseOffset):
    global_idx = thread_id_x
    splat_idx = CompactedSplatIndices[global_idx] if UseCompaction else global_idx
    if splat_idx >= SplatCount:
        return

    base_offset = GlobalBaseOffsetsBuffer[ProxyIndex] if UseGlobalCompactionPath else GlobalBaseOffset

    pos = [0.0, 0.0, 0.0]
    rot = [0.0, 0.0, 0.0, 0.0]
    scale = [0.0, 0.0, 0.0]
    opacity = 0.0
    color = [0.0, 0.0, 0.0]

    if UseNative4D:
        # [G] Native 4D interpolation logic not detailed in sources
        pos = Native4DBuffer[splat_idx].pos
        rot = interpolate_dual_quaternion(Native4DBuffer[splat_idx].dq1, Native4DBuffer[splat_idx].dq2, CurrentTime / TimeDuration)
        scale = Native4DBuffer[splat_idx].scale
        opacity = Native4DBuffer[splat_idx].opacity
        view_dir = [CameraPosition[i] - pos[i] for i in range(3)]
        view_dir_ply = [sum(WorldToPLY[i][j] * view_dir[j] for j in range(3)) for i in range(3)]
        color = evaluate_sh(view_dir_ply, Native4DBuffer[splat_idx].sh_coeffs, SHOrder, NumSHCoeffs)
    elif UseKeyframes:
        # [G] Keyframe interpolation logic not detailed in sources
        pos = KeyframeBuffer[splat_idx].pos
        rot = slerp(KeyframeBuffer[splat_idx].rot1, KeyframeBuffer[splat_idx].rot2, KeyframeFrameTime)
        scale = KeyframeBuffer[splat_idx].scale
        opacity = KeyframeBuffer[splat_idx].opacity
        view_dir = [CameraPosition[i] - pos[i] for i in range(3)]
        view_dir_ply = [sum(WorldToPLY[i][j] * view_dir[j] for j in range(3)) for i in range(3)]
        color = evaluate_sh(view_dir_ply, SHBuffer[splat_idx].sh_coeffs, SHOrder, NumSHCoeffs)
    elif UseTemporal:
        # [G] Temporal marginalization logic not detailed in sources
        pos = [TemporalBuffer[splat_idx].pos[i] + PreViewTranslation[i] * CurrentTime for i in range(3)]
        rot = TemporalBuffer[splat_idx].rot
        scale = TemporalBuffer[splat_idx].scale
        opacity = TemporalBuffer[splat_idx].opacity
        view_dir = [CameraPosition[i] - pos[i] for i in range(3)]
        view_dir_ply = [sum(WorldToPLY[i][j] * view_dir[j] for j in range(3)) for i in range(3)]
        color = evaluate_sh(view_dir_ply, SHBuffer[splat_idx].sh_coeffs, SHOrder, NumSHCoeffs)
    else:
        pos = PackedSplatBuffer[splat_idx].pos
        rot = PackedSplatBuffer[splat_idx].rot
        scale = PackedSplatBuffer[splat_idx].scale
        opacity = PackedSplatBuffer[splat_idx].opacity
        view_dir = [CameraPosition[i] - pos[i] for i in range(3)]
        view_dir_ply = [sum(WorldToPLY[i][j] * view_dir[j] for j in range(3)) for i in range(3)]
        color = evaluate_sh(view_dir_ply, SHBuffer[splat_idx].sh_coeffs, SHOrder, NumSHCoeffs)

    scale = [s * SplatScale for s in scale]
    opacity *= OpacityScale

    dist = sum((pos[i] - CameraPosition[i])**2 for i in range(3))**0.5
    if MaxDrawDistance > 0.0 and dist > MaxDrawDistance:
        return

    if dist > FadeOutStartDistance:
        fade = (dist - FadeOutStartDistance) / (MaxDrawDistance - FadeOutStartDistance) if MaxDrawDistance > FadeOutStartDistance else 0.0
        opacity *= max(0.0, 1.0 - min(1.0, fade))

    if UseClusterCulling:
        cluster_idx = SplatClusterIndexBuffer[splat_idx]
        if not (ClusterVisibilityBitmap[cluster_idx // 32] & (1 << (cluster_idx % 32))):
            return
        if UseLODRendering and splat_idx >= OriginalSplatCount:
            if not (LODClusterSelectedBitmap[cluster_idx // 32] & (1 << (cluster_idx % 32))):
                return

    clip_pos_w = sum(WorldToClip[3][j] * (pos[j] if j < 3 else 1.0) for j in range(4))
    screen_pos = [(sum(WorldToClip[i][j] * (pos[j] if j < 3 else 1.0) for j in range(4)) / clip_pos_w) * ScreenSize[i] for i in range(2)]
    depth = (sum(WorldToClip[2][j] * (pos[j] if j < 3 else 1.0) for j in range(4)) / clip_pos_w)

    write_idx = base_offset + global_idx
    if MaxRenderBudget == 0 or write_idx < MaxRenderBudget:
        # [G] ViewDataBuffer write operation
        pass
```

### Edge Cases and Pitfalls
- **Precision:** Floating-point precision issues may arise during 4D interpolation, especially for large time durations or extreme scales [G].
- **Boundaries:** Ensure `SplatCount` and `VisibleSplatCount` are correctly synchronized with the input buffers [G].
- **Degenerate Input:** If `MaxDrawDistance` is 0, distance culling is disabled. If `SHOrder` is 0, SH evaluation simplifies to a constant color [G].
- **Buffer Overflows:** `MaxRenderBudget` is critical to prevent writing beyond the allocated `ViewDataBuffer` size [S1, unofficial].

## Operating characteristics
- **Time Complexity:** O(N), where N is the number of splats, as each splat is processed independently in parallel [G].
- **Memory Complexity:** O(N) for the `ViewDataBuffer`, with input buffers being read-only [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of splats and 4D evaluation complexity, with native 4D being more expensive than keyframe or temporal modes [G].
- **Strengths:** Handles complex 4D variations efficiently on the GPU and supports advanced culling mechanisms [S1, unofficial].
- **Limitations:** Requires DirectX 12 / Shader Model 6, crashes on non-ASCII paths, and has limited support for proprietary 4D formats [S1, unofficial].
- **When to Use:** Real-time rendering of 3D and 4D Gaussian Splatting scenes in Unreal Engine 5 [S1, unofficial].
- **When Not to Use:** Offline rendering, non-UE5 environments, or unsupported proprietary 4D formats [S1, unofficial].

## Background
The method is part of the OpenSplat4D plugin for Unreal Engine 5, developed by YuanBaoSMadLab [S1, unofficial]. It supports various 4D Gaussian Splatting techniques, including Spacetime Gaussians (STG), Fudan 4DGS, and keyframe-based 4DGS [S1, unofficial]. The plugin aims to provide real-time rendering capabilities for 4D Gaussian Splatting in game engines [S1, unofficial].

## Sources
[S1] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
