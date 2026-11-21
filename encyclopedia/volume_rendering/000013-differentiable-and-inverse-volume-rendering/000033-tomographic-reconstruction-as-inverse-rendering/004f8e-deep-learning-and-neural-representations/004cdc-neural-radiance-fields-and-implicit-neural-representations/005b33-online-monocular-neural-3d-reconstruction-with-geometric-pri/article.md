# Online Monocular Neural 3D Reconstruction with Geometric Priors

Also called: MonoNeuralFusion, MonoNeuralFusion Online Pipeline, Online Reconstruction System

## Overview
Online Monocular Neural 3D Reconstruction with Geometric Priors (MonoNeuralFusion) is an incremental surface mapping system that reconstructs 3D geometry from monocular RGB sequences by maintaining and refining a sparse feature volume [S1, unofficial]. It solves the problem of real-time, globally coherent 3D reconstruction from a single camera by decoupling the pipeline into a parallel mapping thread for feature volume construction and an optimization thread for volume rendering refinement [S1, unofficial]. Unlike offline dense voxel methods (e.g., Atlas) or purely depth-estimation approaches, MonoNeuralFusion uses a sparse representation guided by geometric priors (predicted normals) to achieve high normal consistency and lower Chamfer distance while operating online [S1, unofficial].

## How it works
The method builds on [Neural Radiance Fields and Implicit Neural Representations](13341-neural-radiance-fields-and-implicit-neural-representations.md) by using a neural radiance field $f_\omega$ to represent the scene, but optimizes it incrementally using geometric constraints.

### Inputs and Outputs
*   **Inputs:** Monocular RGB image sequence, camera intrinsics, and estimated camera extrinsics (poses).
*   **Outputs:** A surface mesh (extracted via Marching Cubes) and a refined neural radiance field representation.

### Data Structures
1.  **Sparse Feature Volume:** A 3D grid storing multi-level image features extracted by a 2D CNN encoder (MnasNet) and processed by a 3D sparse CNN [S1, unofficial]. The volume is sparse, meaning it only stores features for occupied voxels, reducing memory usage compared to dense grids [S1, unofficial].
2.  **Keyframe Database:** Stores selected keyframes, their poses, and associated predicted normal maps [S1, unofficial].
3.  **Neural Radiance Field ($f_\omega$):** A neural network parameterized by $\omega$ that maps 3D coordinates and viewing directions to color and density [S1, unofficial]. Initialized randomly from a zero-mean Gaussian distribution [S1, unofficial].

### Numbered Steps

**1. Mapping Thread (Feature Volume Construction)**
1.  Receive an incoming RGB frame with its pose.
2.  Determine if the frame is a keyframe. A frame is selected as a keyframe if the distance to the last keyframe $> \tau_d = 0.1$ OR the rotation angle from the last keyframe $> \tau_a = 15^\circ$ [S1, unofficial].
3.  If a keyframe is selected:
    a. Predict the normal map for the image using an out-of-the-box Omnidata model [S1, unofficial].
    b. Insert the keyframe and its normal map into the Keyframe Database.
    c. Maintain a sliding window of keyframes. When the window size reaches $K = 9$, construct the Sparse Feature Volume using all images in the window and their camera parameters [S1, unofficial].
    d. Apply Marching Cubes to the current Sparse Feature Volume to extract an initial surface mesh [S1, unofficial].

**2. Optimization Thread (Volume Rendering Refinement)**
1.  Run in parallel with the Mapping Thread.
2.  Select $B = 9$ optimization keyframes from the Keyframe Database [S1, unofficial].
3.  Randomly sample $M = 512$ pixels on each of the selected keyframes [S1, unofficial].
4.  For each sampled pixel, cast a ray through the scene.
5.  Perform volume rendering to compute the rendered color and depth. This involves:
    a. Coarse sampling: Sample $N_c = 16$ points along the ray [S1, unofficial].
    b. Importance sampling: Sample $N_i = 16$ points along the ray based on coarse densities [S1, unofficial].
    c. Compute the final color and depth using the accumulated density and color from $f_\omega$ [S1, unofficial].
6.  Compute the loss function $\mathcal{L}$, which includes photometric loss and a normal consistency loss weighted by 0.05 [S1, unofficial].
7.  Use the Adam optimizer to refine the scene representation parameters ($\omega$) for $T = 100$ iterations [S1, unofficial].

**3. Per-Scene Fine-Tuning (Optional)**
1.  After the online process, or for specific high-fidelity requirements, optimize the entire scene.
2.  Sample $M = 512$ pixels on each of $B = 9$ images.
3.  Use $N_c = N_i = 32$ sampling points per ray [S1, unofficial].
4.  Optimize for $T = 5,000$ iterations [S1, unofficial].

### Formulas
The optimization minimizes a loss function. While the exact composite loss formula is not fully detailed in the provided text, the components are specified:
*   Photometric loss component weight: $\lambda_{lo} = 0.8$ (using L1 loss) [S1, unofficial].
*   Normal consistency loss weight: 0.05 [S1, unofficial].
*   Clamp threshold $\tau = 0.2$ used in NISR training [S1, unofficial].

The gradient is computed via backpropagation through the volume rendering operator and the neural network $f_\omega$.

### Parameters
*   $\tau_d = 0.1$: Distance threshold for keyframe selection [S1, unofficial].
*   $\tau_a = 15^\circ$: Rotation threshold for keyframe selection [S1, unofficial].
*   $K = 9$: Sliding window size for feature volume construction [S1, unofficial].
*   $B = 9$: Number of optimization keyframes per loop [S1, unofficial].
*   $M = 512$: Number of pixels sampled per keyframe for optimization [S1, unofficial].
*   $N_c = 16$: Coarse sampling points per ray (online) [S1, unofficial].
*   $N_i = 16$: Importance sampling points per ray (online) [S1, unofficial].
*   $T = 100$: Online optimization iterations per loop [S1, unofficial].
*   $N_c = N_i = 32$: Sampling points per ray (fine-tuning) [S1, unofficial].
*   $T = 5,000$: Fine-tuning iterations [S1, unofficial].
*   Normal weight: 0.05 [S1, unofficial].
*   $\lambda_{lo} = 0.8$: L1 loss weight [S1, unofficial].
*   $\tau = 0.2$: Clamp threshold [S1, unofficial].
*   Learning rate: 0.001 (Adam optimizer) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class MonoNeuralFusion:
    def __init__(self):
        # Parameters
        self.tau_d = 0.1
        self.tau_a = 15.0  # degrees
        self.K = 9
        self.B = 9
        self.M = 512
        self.Nc_coarse = 16
        self.Ni_fine = 16
        self.T_online = 100
        self.normal_weight = 0.05
        self.lr = 0.001
        
        # Data Structures
        self.keyframe_database = []  # List of dicts: {'rgb': img, 'pose': pose, 'normal': normal}
        self.sliding_window = []
        self.sparse_feature_volume = None
        self.f_omega = None  # Neural Radiance Field, initialized randomly from zero-mean Gaussian
        
        # Optimizer
        self.optimizer = Adam(self.f_omega.parameters(), lr=self.lr)

    def is_keyframe(self, current_frame, last_keyframe):
        dist = distance(current_frame.pose, last_keyframe.pose)
        rot_angle = rotation_angle(current_frame.pose, last_keyframe.pose)
        return dist > self.tau_d or rot_angle > self.tau_a

    def mapping_thread_step(self, current_frame):
        # 1. Check if keyframe
        if not self.keyframe_database:
            is_kf = True
        else:
            last_kf = self.keyframe_database[-1]
            is_kf = self.is_keyframe(current_frame, last_kf)
            
        if is_kf:
            # 2. Predict Normal
            normal_map = predict_normals(current_frame.rgb) # Using Omnidata
            
            # 3. Update Database
            self.keyframe_database.append({
                'rgb': current_frame.rgb,
                'pose': current_frame.pose,
                'normal': normal_map
            })
            
            # 4. Sliding Window Construction
            self.sliding_window.append(current_frame)
            if len(self.sliding_window) >= self.K:
                # Construct Sparse Feature Volume
                self.sparse_feature_volume = construct_sparse_volume(
                    self.sliding_window, 
                    camera_intrinsics=current_frame.intrinsics
                )
                
                # Extract Mesh
                mesh = marching_cubes(self.sparse_feature_volume)
                
                # Reset window or keep sliding? Text implies "sliding window", 
                # usually means we keep the last K. 
                # For simplicity, we assume we rebuild with the last K.
                self.sliding_window = self.sliding_window[-self.K:]
                
                # Note: Optimization thread runs in parallel, so we don't wait for it here.

    def optimization_thread_step(self):
        if len(self.keyframe_database) < self.B:
            return
            
        # 1. Select B frames
        selected_frames = select_random_frames(self.keyframe_database, self.B)
        
        # 2. Sample Pixels
        # For each selected frame, sample M pixels
        for _ in range(self.T_online):
            loss = 0.0
            gradients = []
            
            for frame in selected_frames:
                # Randomly sample M pixels
                u, v = random_sample_pixels(frame.rgb.shape, self.M)
                
                for i in range(self.M):
                    # 3. Ray Casting and Volume Rendering
                    ray = cast_ray(frame.pose, frame.intrinsics, u[i], v[i])
                    
                    # Coarse Sampling
                    coarse_points = sample_along_ray(ray, self.Nc_coarse)
                    coarse_densities, coarse_colors = self.f_omega(coarse_points, ray.directions)
                    
                    # Importance Sampling
                    fine_points = importance_sample(coarse_densities, ray, self.Ni_fine)
                    fine_densities, fine_colors = self.f_omega(fine_points, ray.directions)
                    
                    # Composite Color and Depth
                    rendered_color, rendered_depth = composite(fine_points, fine_densities, fine_colors, ray)
                    
                    # 4. Loss Computation
                    gt_color = frame.rgb[u[i], v[i]]
                    gt_normal = frame.normal[u[i], v[i]]
                    
                    # Photometric Loss (L1)
                    color_loss = l1_loss(rendered_color, gt_color)
                    
                    # Normal Consistency Loss
                    # Predict normal from rendered depth or volume gradient
                    pred_normal = predict_normal_from_volume(rendered_depth, self.sparse_feature_volume)
                    normal_loss = l1_loss(pred_normal, gt_normal)
                    
                    # Combined Loss
                    batch_loss = color_loss + self.normal_weight * normal_loss
                    loss += batch_loss
                    
                    # Accumulate gradients (conceptual)
                    gradients.append(batch_loss)
            
            # 5. Update Parameters
            loss.backward()
            self.optimizer.step()
            self.optimizer.zero_grad()

    def fine_tuning(self):
        # Select all keyframes or a subset
        selected_frames = self.keyframe_database
        
        Nc_fine = 32
        Ni_fine = 32
        T_fine = 5000
        
        for _ in range(T_fine):
            loss = 0.0
            for frame in selected_frames:
                u, v = random_sample_pixels(frame.rgb.shape, self.M)
                for i in range(self.M):
                    ray = cast_ray(frame.pose, frame.intrinsics, u[i], v[i])
                    
                    # Fine sampling
                    coarse_points = sample_along_ray(ray, Nc_fine)
                    coarse_densities, _ = self.f_omega(coarse_points, ray.directions)
                    
                    fine_points = importance_sample(coarse_densities, ray, Ni_fine)
                    fine_densities, fine_colors = self.f_omega(fine_points, ray.directions)
                    
                    rendered_color, _ = composite(fine_points, fine_densities, fine_colors, ray)
                    
                    gt_color = frame.rgb[u[i], v[i]]
                    color_loss = l1_loss(rendered_color, gt_color)
                    
                    # Normal loss also applied
                    pred_normal = predict_normal_from_volume(rendered_depth, self.sparse_feature_volume)
                    gt_normal = frame.normal[u[i], v[i]]
                    normal_loss = l1_loss(pred_normal, gt_normal)
                    
                    loss += color_loss + self.normal_weight * normal_loss
            
            loss.backward()
            self.optimizer.step()
            self.optimizer.zero_grad()

def select_random_frames(database, B):
    return random.sample(database, B)

def random_sample_pixels(shape, M):
    h, w = shape[:2]
    u = np.random.randint(0, h, M)
    v = np.random.randint(0, w, M)
    return u, v

def l1_loss(pred, gt):
    return np.mean(np.abs(pred - gt))
```

### Edge Cases and Pitfalls
*   **Degenerate Input:** If the camera does not move significantly (distance $< 0.1$ and rotation $< 15^\circ$), no keyframes are selected, and the feature volume is not updated. This requires sufficient camera motion for reconstruction [S1, unofficial].
*   **Precision:** The method relies on accurate pose estimation. Errors in extrinsic parameters will propagate into the feature volume construction and volume rendering [S1, unofficial].
*   **Memory:** While sparse, the feature volume can still grow large for extensive scenes. The sliding window approach limits the volume size to $K=9$ frames at a time, but the optimization thread may need to access a larger database [S1, unofficial].

## Subtopics

### Neural Adaptive Tomography (NeAT)
Uses adaptive sampling and neural fields for tomography. Chooses this when high-resolution internal structure is needed from limited angles. [Neural Adaptive Tomography (NeAT)](1862-neural-adaptive-tomography-neat.md)

### HoloINR
Uses holographic principles for neural implicit representations. Chooses this when dealing with light field or holographic data. [HoloINR](4745-holoinr.md)

### Canonical Neural Radiance Field for X-ray Tomography
Maps views to a canonical space for X-ray tomography. Chooses this for X-ray specific applications with rigid objects. [Canonical Neural Radiance Field for X-ray Tomography](11451-canonical-neural-radiance-field-for-x-ray-tomography.md)

### VolumeNeRF
Uses volume rendering with NeRF for general 3D reconstruction. Chooses this when dense, offline reconstruction is acceptable and no online constraint exists. [VolumeNeRF](13110-volumenerf.md)

## Operating characteristics
*   **Time Complexity:** The online system runs in parallel threads. Mapping thread complexity depends on the number of keyframes and the cost of normal prediction and feature volume construction. Optimization thread complexity is $O(T \cdot B \cdot M \cdot (N_c + N_i))$ per loop, where $T$ is iterations, $B$ is keyframes, $M$ is pixels, and $N_c, N_i$ are sampling points [S1, unofficial].
*   **Memory Complexity:** Sparse feature volume size is proportional to the number of occupied voxels in the sliding window ($K=9$ frames). This is significantly less than dense voxel grids used by methods like Atlas [S1, unofficial].
*   **Measured Performance:** On the TUM RGB-D dataset, the method achieves Chamfer Distance = 0.107, F-score = 0.375, Normal Consistency = 0.806, Abs Rel = 0.062, Abs Diff = 0.108, and $\delta < 1.25 = 0.959$ [S1, unofficial].
*   **Runtime:** Normal prediction takes ~61.48ms per keyframe. Feature volume construction takes ~34.35ms per local fragment (9 keyframes) on an Nvidia RTX 3090 GPU [S1, unofficial].
*   **Fine-tuning Time:** Per-scene fine-tuning typically takes less than 9 minutes [S1, unofficial].
*   **Strengths:** High normal consistency, globally coherent reconstruction, online capability, lower Chamfer distance compared to other online methods [S1, unofficial].
*   **Limitations:** Requires accurate camera poses. Performance depends on the quality of the normal prediction model. Recall is comparable to offline dense methods but may be lower due to sparsity [S1, unofficial].

## Background
MonoNeuralFusion was introduced in the paper "MonoNeuralFusion: Online Monocular Neural 3D Reconstruction with Geometric Priors" [S1, unofficial]. It builds upon previous work in neural surface reconstruction such as NeuralRecon and TransformerFusion, aiming to improve normal consistency and online efficiency. The method uses MnasNet for 2D feature extraction and a 3D sparse CNN for feature processing [S1, unofficial]. It was evaluated on the ScanNet and TUM RGB-D datasets [S1, unofficial].

## Sources
[S1] MonoNeuralFusion: Online Monocular Neural 3D Reconstruction with Geometric Priors, https://doi.org/10.48550/arxiv.2209.15153, fetched 2026-10-29 (unofficial)
