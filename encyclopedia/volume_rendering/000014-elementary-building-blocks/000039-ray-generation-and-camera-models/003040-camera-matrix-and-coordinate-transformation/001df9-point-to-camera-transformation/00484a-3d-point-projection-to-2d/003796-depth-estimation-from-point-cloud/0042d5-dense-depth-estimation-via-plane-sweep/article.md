# Dense Depth Estimation via Plane Sweep

Also called: GPU Plane Sweep Depth Estimation, Live Image Capturing and Dense Depth Estimation, WoeKoch04, Woetzel and Koch

## Overview
Dense Depth Estimation via Plane Sweep estimates per-pixel depth by projecting a series of depth planes into multiple camera views and evaluating photometric consistency across views. It solves the problem of recovering dense 3D geometry from multi-view imagery by replacing explicit volumetric storage with a single-pass rendering approach. Unlike [Triangulation for Closest Point](9933-triangulation-for-closest-point.md), which computes depth from point correspondences, this method operates directly on image pixels and is distinct from [Binocular Depth Estimation (Stereo Matching)](15014-binocular-depth-estimation-stereo-matching.md) by generalizing the cost-volume concept to arbitrary numbers of views (though practical implementations may limit this) and combining rendering with reconstruction in a single step without producing an explicit intermediate volume [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: A set of $N$ rectified images $I_1, \dots, I_N$ with known camera projection matrices.
- Outputs: A dense depth map $D$ for a reference view (typically the first camera).

**Data Structures:**
- Depth planes: Defined by a minimum depth $d_{min}$, maximum depth $d_{max}$, and step size $\Delta d$.
- Cost Volume: Implicitly computed per pixel during the sweep; no explicit 3D texture is stored [S1, unofficial].

**Steps:**
1. **Initialization**: For each pixel $(u, v)$ in the reference image, initialize a consistency score $C(u, v) = 0$.
2. **Plane Sweep**: Iterate through depth values $d$ from $d_{min}$ to $d_{max}$ with step $\Delta d$:
   a. Construct a depth plane at distance $d$ in the reference camera coordinate system.
   b. For each other camera $k \in \{2, \dots, N\}$:
      i. Project the 3D point corresponding to $(u, v, d)$ from the reference view into camera $k$'s image plane to obtain coordinates $(u_k, v_k)$.
      ii. Sample the intensity or color value $I_k(u_k, v_k)$ from image $k$.
      iii. Compare $I_k(u_k, v_k)$ with the reference pixel value $I_1(u, v)$.
      iv. Accumulate a photometric consistency cost (e.g., sum of absolute differences or gradient correlation) into $C(u, v)$ for this depth hypothesis.
3. **Silhouette/Background Test**: Optionally, reject depth hypotheses that project outside the image boundaries or fail a background consistency check. Note that in the GPU-accelerated implementation, the photo-consistency check is performed on each fragment regardless of rejection by silhouette tests [S1, unofficial].
4. **Depth Selection**: For each pixel $(u, v)$, select the depth $d$ that minimizes the accumulated cost $C(u, v)$.
5. **Refinement**: Apply local smoothing or winner-takes-all selection to produce the final depth map.

**Formulas:**
- Projection: $\mathbf{x}_k = P_k \mathbf{X}_1$, where $\mathbf{X}_1$ is the 3D point at depth $d$ for pixel $(u,v)$ in camera 1, and $P_k$ is the projection matrix of camera $k$.
- Cost: $C(u, v) = \sum_{d} \sum_{k=2}^{N} \text{Cost}(I_1(u, v), I_k(u_k, v_k))$.

**Parameters:**
- $d_{min}, d_{max}$: Depth range.
- $\Delta d$: Depth resolution.
- $N$: Number of views.

**Edge Cases:**
- **Occlusions**: Points visible in the reference view may be occluded in other views, leading to inconsistent costs.
- **Textureless Regions**: Lack of distinctive features causes ambiguous minima in the cost function.
- **Precision**: Floating-point precision in projection calculations can cause sampling errors at large depths.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plane_sweep_depth_estimation(images, cameras, d_min, d_max, d_step):
    """
    images: list of N images (reference is images[0])
    cameras: list of N camera projection matrices
    d_min, d_max, d_step: depth range parameters
    """
    height, width = images[0].shape[:2]
    depth_map = np.zeros((height, width))
    
    # Initialize cost volume implicitly
    # In GPU implementation, this is done in a shader loop
    
    for d in np.arange(d_min, d_max, d_step):
        for y in range(height):
            for x in range(width):
                # Get 3D point in reference camera
                X_ref = unproject(x, y, d, cameras[0])
                
                cost = 0
                for k in range(1, len(images)):
                    # Project to other camera
                    X_k = cameras[k] @ X_ref
                    u_k, v_k = X_k[0]/X_k[2], X_k[1]/X_k[2]
                    
                    # Check bounds
                    if not (0 <= u_k < width and 0 <= v_k < height):
                        continue
                        
                    # Sample intensity
                    I_ref = images[0][y, x]
                    I_k = images[k][int(v_k), int(u_k)]
                    
                    # Accumulate cost
                    cost += abs(I_ref - I_k)
                
                # Store cost for this depth
                # In practice, keep track of min cost and corresponding depth
                if d == d_min or cost < min_cost[y, x]:
                    min_cost[y, x] = cost
                    depth_map[y, x] = d
                    
    return depth_map
```
*Note: The above pseudocode is a simplified CPU version. The GPU implementation processes these steps in parallel shaders [S1, unofficial].*

## Subtopics

### [Binocular Depth Estimation (Stereo Matching)](15014-binocular-depth-estimation-stereo-matching.md)
Binocular depth estimation estimates scene depth from a pair of rectified left and right images by computing a disparity map, which is inversely proportional to depth. It solves the problem of recovering 3D structure from 2D imagery using geometric constraints between two viewpoints. Unlike monocular depth estimation, which relies on learned priors from single images, this method leverages explicit pairwise multi-view consistency, making it more effective for scenes with weak texture or repetitive patterns where local photometric costs are ambiguous. It builds on [Dense Depth Estimation via Plane Sweep](10850-dense-depth-estimation-via-plane-sweep.md) by providing the foundational cost-volume-based approach that plane sweep generalizes to multiple views.

## Operating characteristics
- **Time Complexity**: $O(H \cdot W \cdot N \cdot D)$, where $H, W$ are image dimensions, $N$ is number of views, and $D$ is number of depth planes.
- **Memory Complexity**: $O(H \cdot W)$ for the cost accumulation per pixel, as no explicit 3D volume is stored [S1, unofficial].
- **Performance**: Dense depth maps can be computed with up to 20 frames per second on GPU hardware [S1, unofficial].
- **Strengths**: High speed due to GPU acceleration; no explicit volume storage reduces memory footprint; works with uncalibrated views if projection matrices are known.
- **Limitations**: Limited to only four camera views in the original implementation [S1, unofficial]; sensitive to occlusions and textureless regions; requires rectified images or known camera poses.
- **When to use**: Real-time dense reconstruction with limited views and GPU resources.
- **When not to use**: Scenes with significant occlusions, very large depth ranges, or when more than four views are available and CPU resources are abundant.

## Background
Woetzel and Koch proposed a live system for image capturing and dense depth estimation in [WoeKoch04]. Their plane-sweep algorithm runs almost entirely on the GPU, leaving the main CPU free for other tasks such as image capture and higher level recognition. This approach was developed to address the computational bottlenecks of traditional volumetric reconstruction methods.

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
