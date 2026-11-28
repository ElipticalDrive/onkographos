# Unity Mobile Implementation

Also called: SplattingAvatar Unity Renderer

## Overview
This method implements the rendering phase of Gaussian Avatar Reconstruction within the Unity engine, optimized for mobile devices. It solves the problem of real-time, high-fidelity rendering of dynamic 3D Gaussian Splatting avatars by relying on a warped mesh for rigging and standard alpha blending for compositing. It sets itself apart from its siblings by prioritizing maximum compatibility with Unity's standard rendering pipeline and mobile performance constraints, using a simple front-to-back sorting and quad-based rasterization rather than tiled or sort-free GPU-native approaches.

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of 3D Gaussians defined by position, covariance, color, and opacity; a rigged mesh (FBX) defining the avatar's pose; camera parameters.
- **Outputs:** A rendered 2D image with alpha channel.

**Data Structures:**
- **Gaussian Data:** Stored in a `.ply` file. Contains attributes for each Gaussian: position ($\mathbf{x}$), 3D covariance matrix ($\Sigma$), spherical harmonics coefficients (for color), and opacity ($\alpha$).
- **Embedding Data:** Stored in a `.json` file. Describes the mapping between static anchor features and dynamic Gaussian attributes.
- **Mesh Data:** Stored in an `.fbx` file. Contains the rigged mesh topology and bone hierarchy. Triangle order must be maintained for correct rendering.

**Steps:**
1. **Export and Import:**
   - Export the canonical mesh as `.obj` to Mixamo for autorigging [S1, unofficial].
   - Export one `.ply` file of the Gaussians, one `.json` file describing the embedding, and the rigged `.fbx` file to Unity [S1, unofficial].
   - Note: The `.fbx` file can be rigged by other software if triangle order is preserved [S1, unofficial].

2. **Pose and Projection:**
   - Apply the bone transformations from the rigged mesh to the Gaussian positions to obtain world-space positions.
   - Project the 3D Gaussians into camera space.
   - Calculate the 2D covariance matrix $\Sigma'$ in screen space for each Gaussian based on the camera projection and the 3D covariance [S1, unofficial].

3. **Sorting:**
   - Sort all Gaussians by their z-axis coordinate in camera coordinates from near to far [S1, unofficial]. This is performed on the CPU or via a compute shader before rendering.

4. **Rasterization:**
   - For each visible Gaussian, draw one front-parallel quad primitive centered at its projected position. The quad's size and shape are determined by the 2D covariance $\Sigma'$ [S1, unofficial].
   - This "one-primitive-one-Gaussian" strategy ensures proper occlusion handling with other regular objects in the scene [S1, unofficial].

5. **Fragment Shader:**
   - For each pixel covered by a quad, compute the color and alpha.
   - Emit color with alpha pre-multiplication.
   - Set the blend function to `(ONE, ONE_MINUS_SRC_ALPHA)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the rendering loop in Unity
def render_gaussian_avatar(gaussians, camera, mesh_pose):
    # 1. Transform Gaussians
    transformed_gaussians = []
    for g in gaussians:
        # Apply bone transformation from mesh_pose to g.position
        world_pos = apply_rig(g.position, mesh_pose)
        # Project to camera space
        cam_pos = project(world_pos, camera)
        # Calculate 2D covariance in screen space
        cov_2d = project_covariance(g.cov_3d, camera)
        transformed_gaussians.append({
            'pos': cam_pos,
            'cov_2d': cov_2d,
            'color': g.color,
            'alpha': g.alpha
        })

    # 2. Sort by depth (z-axis in camera coordinates, near to far)
    transformed_gaussians.sort(key=lambda g: g['pos'].z)

    # 3. Render
    for g in transformed_gaussians:
        # Draw a front-parallel quad centered at g.pos with size from g.cov_2d
        draw_quad(g.pos, g.cov_2d)

    # Fragment shader logic (conceptual)
    # For each pixel in the quad:
    #   Compute distance from center using cov_2d
    #   Calculate alpha = exp(-0.5 * dist^2) * g.alpha
    #   Color = g.color * alpha
    #   Blend with existing color using ONE, ONE_MINUS_SRC_ALPHA
```

**Edge Cases and Pitfalls:**
- **Triangle Order:** If the `.fbx` file is rigged by external software, the triangle order must be maintained to ensure correct rendering [S1, unofficial].
- **Occlusion:** The one-primitive-one-Gaussian strategy is critical for handling occlusion with other scene objects [S1, unofficial].
- **Mobile Performance:** On mobile devices, the sorting and drawing overhead can be significant. The method aims for steady 30 FPS on iPhone 13 [S1, unofficial].

## Subtopics

### [Unity Sentis Implementation for Gaussian Splats](10691-unity-sentis-implementation-for-gaussian-splats.md)
This method implements the rendering phase of Gaussian Avatar Reconstruction by leveraging the Unity Sentis framework to execute Multi-Layer Perceptrons (MLPs) as custom compute shaders on the GPU. It solves the problem of dynamically predicting 3D Gaussian attributes (position, scale, rotation, opacity, and spherical harmonics) from static anchor features and a time-varying context encoding $\Psi(x)$ without CPU-GPU synchronization bottlenecks. Unlike static splatting, this approach allows for real-time appearance changes by invoking the MLP with different encodings, while maintaining a fixed memory footprint since the number of predicted Gaussians remains constant.

## Operating characteristics
- **Performance:** Over 300 FPS on a modern GPU; steady 30 FPS on an iPhone 13 [S1, unofficial].
- **Strengths:** High compatibility with Unity's standard pipeline; proper occlusion handling with other objects; good rendering quality for head avatars, especially in eye and hair regions [S1, unofficial].
- **Limitations:** Relies on CPU sorting which may not scale as well as tiled or sort-free methods on very high-resolution scenes; performance on mobile is lower than on modern desktop GPUs [S1, unofficial].
- **When to use:** When targeting mobile devices with Unity and requiring compatibility with standard rendering pipelines and occlusion handling.
- **When not to use:** When maximum performance on high-end desktop GPUs is required, or when dynamic attribute prediction via MLPs is needed (use Sentis implementation instead).

## Background
The method is part of the SplattingAvatar project, which focuses on Gaussian Avatar Reconstruction. It was developed to provide a practical implementation for mobile devices, leveraging Unity's capabilities. The approach builds on the general 3D Gaussian Splatting rendering principles but adapts them for the constraints and features of the Unity engine and mobile hardware.

## Sources
[S1] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
