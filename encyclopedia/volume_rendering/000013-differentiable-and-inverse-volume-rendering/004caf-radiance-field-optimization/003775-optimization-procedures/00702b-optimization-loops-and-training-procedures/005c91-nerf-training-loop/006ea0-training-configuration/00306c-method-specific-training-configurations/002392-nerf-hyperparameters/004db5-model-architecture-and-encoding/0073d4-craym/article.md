# CRAYM
Also called: Neural Field Optimization via Camera RAY Matching

## Overview
CRAYM is a method for 3D neural field optimization that jointly optimizes camera poses and an implicit field to handle noisy camera poses [S1]. It constructs a 3D feature volume $V$ and optimizes it by matching rays across different images, using key rays for structural constraints and auxiliary rays for local context [S1]. Unlike standard NeRF approaches that assume known poses, CRAYM parameterizes transformation matrices as variables in the joint optimization, allowing it to recover geometry and appearance even when initial pose estimates are inaccurate [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of input images, initial camera poses (intrinsics and extrinsics), and initial sparse keypoint matches between image pairs.
*   **Outputs:** A 3D feature volume $V$, optimized camera poses, and a trained geometry network $\Phi_g$ and texture network $\Phi_t$ for novel view synthesis or 3D reconstruction [S1].

**Data Structures**
1.  **Feature Volume $V$:** Represented using multi-resolution hash encoding [S1]. It stores features for 3D points.
2.  **Progressive Feature Mask $M$:** A mask used to filter out fine-level features during early iterations of coarse-to-fine training [S1].
3.  **Key Rays $\{r_k\}$:** Rays issued from cameras through pixel centers associated with surface points having rich textures and sharp features. Obtained by detecting keypoints using SuperPoint and performing point-to-point matching using SuperGlue [S1].
4.  **Auxiliary Rays $\{r_a\}$:** Rays sampled around keypoints to provide contextual or local structural information [S1].
5.  **Geometry Network $\Phi_g$:** Takes a 3D point $p$ and its feature as input to produce an SDF value and opaque density $\sigma$ [S1].
6.  **Texture Network $\Phi_t$:** Takes output features from $\Phi_g$, ray directions, and normal at $p$ as inputs to predict color $\mathbf{c}(p)$ [S1].

**Steps**
1.  **Initialization:** Initialize the feature volume $V$ using multi-resolution hash encoding. Initialize camera poses as variables for joint optimization. Initialize networks $\Phi_g$ and $\Phi_t$ [S1].
2.  **Ray Selection:**
    *   Detect keypoints on input images using SuperPoint and match them using SuperGlue to obtain key rays $\{r_k\}$ [S1].
    *   Sample auxiliary rays $\{r_a\}$ around the keypoints [S1].
3.  **Feature Extraction:** For a 3D point $p$, extract its feature $f(p)$ from the volume:
    $$ f(p) = M(V(p)) $$
    where $V(p)$ is the value in the feature volume at $p$ and $M$ is the progressive feature mask [S1].
4.  **Key Rays Enrichment (KRE):** For each point $p_k$ along a key ray $r_k$:
    *   Sample points $q_j$ around $p_k$ along the $j$-th auxiliary ray [S1].
    *   Fuse features using function $g$ to get the enriched feature $f'(p_k)$:
        $$ f'(p_k) = \sum_j g(f(p_k), f(q_j)) $$
        where $g$ fuses the feature of the key ray point and the auxiliary ray points [S1].
5.  **Geometry Prediction:** Pass the enriched feature $f'(p_k)$ and point $p_k$ to the geometry network $\Phi_g$ to predict the SDF value and feature vector $f''(p_k)$, from which opaque density $\sigma$ is derived [S1].
6.  **Color Prediction:** Pass the output features from $\Phi_g$, ray direction, and normal at $p_k$ to the texture network $\Phi_t$ to predict color $\mathbf{c}(p_k)$ [S1].
7.  **Joint Optimization:** Optimize the feature volume $V$, camera poses, and networks $\Phi_g, \Phi_t$ end-to-end. The Matched Rays Coherency (MRC) module is used to enhance rendering quality by maintaining coherency between matched key rays and disambiguating imprecise matchings [S1].

**Parameters**
*   **Multi-resolution hash encoding:** Used for $V$ [S1].
*   **Progressive feature mask $M$:** Filters fine-level features in early training [S1].
*   **Camera poses:** Parameterized as variables [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CRAYM Training Step
def craysm_step(images, initial_poses, superpoint_matches):
    # 1. Initialize structures
    V = MultiResolutionHashEncoding()
    M = ProgressiveFeatureMask()
    Phi_g = GeometryNetwork()
    Phi_t = TextureNetwork()
    poses = Parameterize(initial_poses) # Variables for joint optimization
    
    # 2. Ray Selection
    key_points = SuperPoint(images)
    key_ray_matches = SuperGlue(key_points)
    key_rays = [Ray(origin=pose.o, direction=pose.d) for pose, match in key_ray_matches]
    aux_rays = SampleAuxiliaryRays(key_rays)
    
    # 3. Joint Optimization Loop (simplified)
    for iteration in range(num_iterations):
        loss = 0
        for r_k in key_rays:
            # Sample points along key ray
            p_k = SamplePointAlongRay(r_k)
            
            # Feature Extraction
            f_p_k = M(V(p_k))
            
            # Key Rays Enrichment
            f_aux = []
            for r_a in aux_rays_around(r_k, p_k):
                q_j = SamplePointAlongRay(r_a)
                f_q_j = M(V(q_j))
                f_aux.append(f_q_j)
            
            # Fuse features (g is a fusion function, e.g., concatenation + MLP)
            f_prime_p_k = FuseFeatures(f_p_k, f_aux)
            
            # Geometry Prediction
            sdf, f_double_prime_p_k = Phi_g(p_k, f_prime_p_k)
            sigma = DensityFromSDF(sdf)
            
            # Color Prediction
            normal = NormalFromSDF(sdf)
            color = Phi_t(f_double_prime_p_k, r_k.direction, normal)
            
            # Compute loss (rendering loss + coherency loss + pose regularization)
            loss += ComputeLoss(color, gt_color)
            
        # Update all parameters
        Update(poses, V, Phi_g, Phi_t, loss)
        
    return V, poses, Phi_g, Phi_t
```

## Operating characteristics
*   **Strengths:** Robust to noisy camera poses; handles occlusion and unreliable matching better than methods assuming perfect poses [S1].
*   **Limitations:** Requires keypoint detection and matching (SuperPoint/SuperGlue), which may fail in textureless regions [S1]. Joint optimization increases computational complexity and training time compared to fixed-pose NeRF [S1].
*   **Performance:** Not given in the sources.
*   **When to use:** When camera poses are noisy or unknown, and the scene has sufficient texture for keypoint matching [S1].
*   **When not to use:** When poses are known with high accuracy, or the scene is textureless [S1].

## Background
CRAYM was proposed to address the failure of existing approaches at high noise levels in camera poses [S1]. It builds on the idea of mapping rays to density and color implicitly but introduces ray matching across images to formulate geometric priors [S1]. The method uses multi-resolution hash encoding for the feature volume and joint optimization of poses and the implicit field [S1].

## Sources
[S1] CRAYM: Neural Field Optimization via Camera RAY Matching, https://core.ac.uk/works/318675119, fetched 2026-10-31
