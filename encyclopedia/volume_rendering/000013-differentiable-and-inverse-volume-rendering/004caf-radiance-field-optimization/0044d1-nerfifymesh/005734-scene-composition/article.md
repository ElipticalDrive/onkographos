# Scene Composition
Also called: Composed iVR-GS

## Overview
Scene Composition assembles multiple independent radiance-field models (NeRFifyMesh or iVR-GS representations) into a unified global scene without retraining [S2, unofficial][S3]. It solves scalable scene building by allowing pre-converted mesh-based models to be placed in a global coordinate system or integrated with existing NeRF models [S2, unofficial]. It differs from monolithic training approaches by treating each model as an independent, transformable primitive composited at render time [S2, unofficial].

## How it works
**Inputs:**
- $N$ pre-trained local radiance-field models (NeRFifyMesh or iVR-GS) [S2, unofficial][S1].
- For each model $i$, a pose transformation matrix $T_{wi} \in SE(3)$ defining its position and orientation in the global world frame [S2, unofficial].
- For iVR-GS models: quantized attribute data, exact values for mean $\mu$ and normal $n$, shared codebook indices for other attributes, and a global palette color $c_p$ [S1].

**Outputs:**
- A single composed radiance field representation [S1].
- Rendered 2D images with correct occlusion handling between models [S2, unofficial].

**Data Structures:**
- **Local Model Parameters:** NeRFifyMesh stores implicit neural network weights [S2, unofficial]. iVR-GS stores explicit Gaussian primitives with $\mu$ (exact floats), $n$ (exact floats), other attributes (shared codebook indices), and $c_p$ (global palette color) [S1].
- **Global Scene List:** A single parameter list formed by concatenating the decompressed attribute values of all primitives from all basic models [S1].

**Steps:**
1. **Normalization:** Each NeRFifyMesh model is normalized to fit within a unit cube and defined in a local coordinate frame [S2, unofficial].
2. **Storage (iVR-GS specific):** Save exact values of $\mu$ and $n$, and indices with shared codebooks for other attributes for each editable Gaussian. Store palette color $c_p$ as a global parameter for each basic scene [S1].
3. **Ray Casting:** Cast rays from a virtual camera in the global frame [S2, unofficial]. The ray is defined as $r_w(t) = o_w + t\vec{d}_w$, where $o_w$ is ray origin and $\vec{d}_w$ is ray direction [S2, unofficial].
4. **Coordinate Transformation:** Transform the world-frame ray into the local coordinate frame of each model $i$ using the inverse transformation: $r_i(t) = T^{-1}_{wi} r_w(t)$, where $T_{wi} \in SE(3)$ is the pose of model $i$ in the world frame [S2, unofficial].
5. **Local Evaluation:** Evaluate each local NeRF model using the transformed ray $r_i(t)$. Each model returns RGB and depth values for the transformed rays [S2, unofficial].
6. **Decompression and Composition (iVR-GS specific):** Decompress each model from the quantized format and append attribute values of primitives from basic iVR-GS models into a single parameter list to form the composed model [S1].
7. **Depth-Based Blending:** Composite per-model renderings in the 2D image plane using depth-based ordering to resolve occlusions. For each pixel, blend RGB outputs from different models back-to-front using the corresponding depth values [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def invert_se3(T):
    # T is a 4x4 list of lists representing an SE(3) matrix.
    # Returns the inverse matrix.
    R = [T[i][:3] for i in range(3)]
    t = [T[i][3] for i in range(3)]
    Rt = [[R[j][i] for j in range(3)] for i in range(3)]
    inv_t = [-sum(Rt[i][j] * t[j] for j in range(3)) for i in range(3)]
    inv = [[0.0] * 4 for _ in range(4)]
    for i in range(3):
        inv[i][:3] = Rt[i]
        inv[i][3] = inv_t[i]
    inv[3][3] = 1.0
    return inv

def transform_point(M, p):
    x, y, z = p
    return (M[0][0]*x + M[0][1]*y + M[0][2]*z + M[0][3],
            M[1][0]*x + M[1][1]*y + M[1][2]*z + M[1][3],
            M[2][0]*x + M[2][1]*y + M[2][2]*z + M[2][3])

def transform_vector(M, v):
    x, y, z = v
    return (M[0][0]*x + M[0][1]*y + M[0][2]*z,
            M[1][0]*x + M[1][1]*y + M[1][2]*z,
            M[2][0]*x + M[2][1]*y + M[2][2]*z)

def compose_and_render(models, camera_origin, camera_direction, poses):
    # Step 3 & 4: Cast ray and transform to local frames
    results = []
    for i, model in enumerate(models):
        T_inv = invert_se3(poses[i])
        local_origin = transform_point(T_inv, camera_origin)
        local_dir = transform_vector(T_inv, camera_direction)
        # Step 5: Local evaluation
        # The sources do not specify the exact evaluation routine; standard radiance field sampling is used.
        rgb, depth = [0.0, 0.0, 0.0], 0.0  # Placeholder for model.evaluate()
        results.append({'rgb': rgb, 'depth': depth})
    
    # Step 7: Depth-based blending
    results.sort(key=lambda r: r['depth'], reverse=True)
    
    final_rgb = [0.0, 0.0, 0.0]
    for res in results:
        rgb = res['rgb']
        alpha = res.get('alpha', 1.0)
        final_rgb = [final_rgb[j] + rgb[j] * alpha * (1.0 - sum(final_rgb[k] for k in range(3))) for j in range(3)]
    return final_rgb

def compose_ivrgs_models(basic_models):
    # Step 6: Decompress and append primitives
    composed_primitives = []
    for model in basic_models:
        decompressed = model.decompress()
        composed_primitives.extend(decompressed.primitives)
    return composed_primitives
```

**Parameters:**
- $T_{wi}$: Pose matrix. Not given in the sources. [G]
- Depth threshold: Not given in the sources. [G]
- Early termination threshold: Not given in the sources. [G]

**Edge Cases and Pitfalls:**
- **Occlusion:** Incorrect depth sorting leads to visual artifacts where models intersect incorrectly. Depth-based ordering is critical [S2, unofficial].
- **Coordinate Systems:** Failure to correctly invert $T_{wi}$ results in models appearing in wrong positions or orientations [S2, unofficial].
- **Quantization Loss (iVR-GS):** While VQ provides ~4× compression, decompression must be exact for $\mu$ and $n$ to maintain geometric fidelity [S1].
- **Boundary Artifacts:** Models normalized to a unit cube may have discontinuities at their boundaries if not handled by the underlying radiance field [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Rendering time scales linearly with the number of models $N$ and the number of primitives per model. Each ray must be evaluated against all models before blending [S2, unofficial].
- **Memory Complexity:** Memory usage is the sum of the memory required for all individual models. For iVR-GS, this includes the decompressed primitives and codebooks [S1].
- **Performance:** The composition pipeline itself is lightweight; the cost is dominated by the individual model evaluations. NeRFifyMesh models capture interior features due to geometry-based sampling [S2, unofficial].
- **Strengths:** Enables real-time scene editing for iVR-GS by adjusting editable Gaussian attributes [S1]. Allows modular scene building without retraining [S2, unofficial][S3].
- **Limitations:** Requires pre-computation/normalization of each model. Occlusion handling is simplified to depth-based blending, which may not account for complex inter-model lighting or shadows [S2, unofficial].

## Background
Scene Composition was introduced as part of the NeRFifyMesh pipeline [S2, unofficial] and the iVR-GS framework [S1]. NeRFifyMesh was developed to convert textured mesh models into NeRF representations for robotics scene building, addressing the need for scalable scene assembly [S2, unofficial]. iVR-GS extended this concept to editable 3D Gaussian Splatting, allowing for composability and real-time editing [S1].

## Sources
[S1] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
[S2] NeRFifyMesh: Optimizing Neural Radiance Fields from Textured Meshes for Robotics Scene Building, https://doi.org/10.48550/arxiv.2610.10387, fetched 2026-10-26 (unofficial)
[S3] Frequency-Modulated Point Cloud Rendering with Easy Editing, https://doi.org/10.1109/cvpr52729.2023.00020, fetched 2026-10-29
