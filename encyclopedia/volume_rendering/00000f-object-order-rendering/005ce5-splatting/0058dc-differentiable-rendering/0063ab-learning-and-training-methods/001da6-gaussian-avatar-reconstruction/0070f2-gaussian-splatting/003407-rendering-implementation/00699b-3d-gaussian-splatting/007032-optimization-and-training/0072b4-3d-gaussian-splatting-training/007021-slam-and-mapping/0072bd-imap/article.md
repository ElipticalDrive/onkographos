# iMAP

Also called: iMAP SLAM

## Overview
iMAP is the first NeRF-based dense online Simultaneous Localization and Mapping (SLAM) model, jointly optimizing camera pose and an implicit scene representation in the form of a Neural Radiance Field (NeRF) [S1, S2]. It solves the problem of simultaneous tracking and mapping by treating the scene geometry and appearance as a continuous function optimized via gradient descent [S1]. Unlike its siblings which may use explicit structures or hybrid representations, iMAP relies on a single Multilayer Perceptron (MLP) for the entire map [S2]. It sets itself apart by introducing an iterative two-step approach where a fast tracking step runs in parallel with a slower mapping step, allowing pose estimation to approach camera framerate [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A stream of RGB images from a moving camera. Initial camera poses are not required; the system initializes from the first frame.
*   **Outputs:** A sequence of estimated camera poses and a NeRF model representing the scene's geometry and appearance.

**Data Structures:**
*   **NeRF Model:** A single Multilayer Perceptron (MLP) [S2].
    *   **Architecture:** 4 hidden layers, each with 256 neurons [S2].
    *   **Input:** 3D spatial coordinates $(x, y, z)$ [S2].
    *   **Output:** Volume density $\sigma$ and RGB color $(r, g, b)$ [S1, S2].
    *   **Note:** The model does not take viewing direction as input, meaning it assumes isotropic appearance and does not model reflections or view-dependent effects [S2].
*   **Memory Bank:** A set of selected keyframes used for continuous backend map optimization [S2].
*   **Pose Buffer:** Stores the estimated camera poses for the current frame and keyframes [S1].

**Parameters:**
*   **Learning Rate:** Not given in the sources.
*   **Batch Size:** Not given in the sources.
*   **Keyframe Selection Criteria:** Not given in the sources, other than "sparse and incrementally selected" [S1].

**Algorithm Steps:**
The method operates in an iterative loop for each new frame, consisting of two parallel or sequential stages:

1.  **Initialization:**
    *   Initialize the NeRF MLP with random weights [G].
    *   Set the initial camera pose for the first frame to identity or a known starting position [G].
    *   Initialize an empty memory bank for keyframes [S2].

2.  **Tracking (Pose Optimization):**
    *   For the current incoming RGB image $I_t$:
    *   Sample rays through the current NeRF model using the current estimated camera pose $P_t$.
    *   Render the predicted image $\hat{I}_t$ using volume rendering [S1].
    *   Compute the photometric loss $L_{track} = || I_t - \hat{I}_t ||^2$.
    *   Optimize the camera pose $P_t$ by minimizing $L_{track}$ with respect to the pose parameters (rotation and translation) [S1].
    *   This step is designed to be fast, aiming for near-camera-framerate speed [S1].

3.  **Mapping (Bundle Adjustment):**
    *   Select a subset of keyframes from the memory bank and the current frame to form a training batch [S1].
    *   Add the current frame to the memory bank if it meets selection criteria (e.g., sufficient motion or error) [S1].
    *   Sample rays through the NeRF model using the poses of the selected keyframes.
    *   Render the predicted images for these keyframes.
    *   Compute the photometric loss $L_{map} = \sum || I_k - \hat{I}_k ||^2$ for all keyframes $k$ in the batch [S1].
    *   Perform joint optimization (bundle adjustment) to minimize $L_{map}$ with respect to both the NeRF MLP weights and the camera poses of the keyframes in the batch [S1].
    *   This step is slower and runs less frequently or in parallel with tracking [S1].

4.  **Loop:**
    *   Repeat for the next frame.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class iMAP:
    def __init__(self, hidden_layers=4, neurons_per_layer=256):
        # Initialize single MLP for NeRF
        # Input: 3D coordinates (x, y, z)
        # Output: Density (sigma) and Color (RGB)
        # No viewing direction input
        self.neural_network = MLP(hidden_layers=hidden_layers, 
                                  neurons_per_layer=neurons_per_layer)
        
        # Memory bank for keyframes
        self.keyframe_bank = []
        
        # Current camera pose (4x4 transformation matrix)
        self.current_pose = np.eye(4)
        
        # Initial pose for the first frame
        self.initial_pose = np.eye(4)

    def track(self, current_image):
        """
        Fast tracking step: Optimize camera pose for the current frame.
        """
        # Sample rays through the scene using current_pose
        # Render predicted image
        predicted_image = render_volume(self.neural_network, self.current_pose)
        
        # Compute photometric loss
        loss = np.sum((current_image - predicted_image) ** 2)
        
        # Optimize current_pose to minimize loss
        # Gradient descent on pose parameters
        self.current_pose = optimize_pose(self.current_pose, loss, self.neural_network)
        
        return self.current_pose

    def map(self, current_image, current_pose):
        """
        Slower mapping step: Jointly optimize NeRF weights and keyframe poses.
        """
        # Add current frame to keyframe bank if selected
        if self.should_add_keyframe(current_image, current_pose):
            self.keyframe_bank.append({
                'image': current_image,
                'pose': current_pose
            })
        
        # Select a sparse subset of keyframes for optimization
        # This includes recent keyframes and potentially the current one
        selected_keyframes = self.select_keyframes(self.keyframe_bank)
        
        # Joint optimization (Bundle Adjustment)
        # Minimize photometric loss over all selected keyframes
        # Variables: NeRF weights, Poses of selected keyframes
        for keyframe in selected_keyframes:
            predicted_image = render_volume(self.neural_network, keyframe['pose'])
            loss = np.sum((keyframe['image'] - predicted_image) ** 2)
            
            # Update NeRF weights and keyframe pose
            self.neural_network = optimize_network(self.neural_network, loss)
            keyframe['pose'] = optimize_pose(keyframe['pose'], loss, self.neural_network)
            
        return self.neural_network

    def should_add_keyframe(self, image, pose):
        # Placeholder for keyframe selection logic
        # Sources mention "sparse and incrementally selected" [S1]
        # Implementation detail: [G] Check for sufficient motion or photometric error
        return True 

    def select_keyframes(self, bank):
        # Placeholder for keyframe selection logic
        # Sources mention optimizing on a "sparse and incrementally selected set" [S1]
        # Implementation detail: [G] Return a subset of the bank
        return bank[-10:] # Example: last 10 keyframes

    def process_frame(self, current_image):
        # Step 1: Tracking (Fast)
        self.current_pose = self.track(current_image)
        
        # Step 2: Mapping (Slow, can be parallelized or deferred)
        self.map(current_image, self.current_pose)
        
        return self.current_pose, self.neural_network

def render_volume(nerf_model, camera_pose):
    """
    Standard volume rendering using the NeRF model and camera pose.
    """
    # Sample points along rays
    # Query NeRF for density and color
    # Composite colors and densities
    pass

def optimize_pose(pose, loss, nerf_model):
    """
    Gradient descent on camera pose parameters.
    """
    # Compute gradient of loss w.r.t pose
    # Update pose
    pass

def optimize_network(nerf_model, loss):
    """
    Gradient descent on NeRF MLP weights.
    """
    # Compute gradient of loss w.r.t MLP weights
    # Update weights
    pass
```

**Edge Cases and Pitfalls:**
*   **Forgetting:** The single MLP suffers from severe catastrophic forgetting as new data is added [S2]. iMAP addresses this partially by organizing keyframes into a memory bank for continuous backend optimization [S2].
*   **Scene Size:** The representation capability of a single MLP is limited, restricting iMAP to small scenes [S2]. It struggles with large-scale unobserved features like walls and floors [S1].
*   **Real-time Performance:** While tracking is fast, the combined system is far from real-time due to the slow NeRF-based mapping step [S1].
*   **View Dependence:** The model does not consider viewing direction, so it cannot model reflections or view-dependent effects [S2].

## Subtopics

### [NICER-SLAM](12004-nicer-slam.md)
NICER-SLAM is an end-to-end dense Simultaneous Localization and Mapping (SLAM) system that performs simultaneous tracking and mapping using only RGB inputs. It improves upon Nice-SLAM by introducing a hierarchical neural implicit representation based on Signed Distance Functions (SDFs), enabling detailed 3D geometry reconstruction and photorealistic novel view synthesis. Unlike its predecessor, which relied on mixed explicit/implicit representations with pre-allocated voxels, NICER-SLAM leverages monocular geometric cues, optical flow, and a locally adaptive SDF-to-density transformation to guide optimization without depth supervision.

## Operating characteristics
*   **Time Complexity:** Tracking is fast, close to camera framerate [S1]. Mapping is slow, as it involves optimizing a NeRF model [S1].
*   **Memory Complexity:** Depends on the size of the memory bank and the NeRF model. The single MLP is relatively small (4 layers, 256 neurons) [S2], but the memory bank grows with the number of keyframes [S2].
*   **Measured Performance:** iMAP achieves lower pose estimation errors and better scene reconstruction than its predecessor (Nice-SLAM is better, but iMAP is the baseline) [S1]. However, it is far from real-time performance [S1].
*   **Scaling:** Limited to small scenes due to the single MLP's representation capability and forgetting problem [S2].
*   **Strengths:** First NeRF-based dense online SLAM; joint optimization of pose and map; fast tracking [S1, S2].
*   **Limitations:** Severe forgetting; limited to small scenes; not real-time; no view-dependent effects [S1, S2].
*   **When to use:** Small-scale indoor scenes where real-time performance is not critical and high-quality dense reconstruction is desired [S2].
*   **When not to use:** Large-scale outdoor scenes, real-time applications, or scenes with strong view-dependent effects [S1, S2].

## Background
iMAP was published in March 2021 [S1]. It is a pioneering work in the field of NeRF-based SLAM, being the first to use a single MLP for implicit map representation in a SLAM context [S2]. It was introduced by Sucar et al. [S1]. The method builds on the concept of Neural Radiance Fields (NeRF) [S1]. It was followed by improvements such as NICE-SLAM (December 2021) and NeRF-SLAM (October 2022) [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advancement in Neural Radiance Fields Propulsion for SLAM: A Novel Upgrade, https://www.atlantis-press.com/proceedings/icaic-24/126003439, fetched 2026-10-28
