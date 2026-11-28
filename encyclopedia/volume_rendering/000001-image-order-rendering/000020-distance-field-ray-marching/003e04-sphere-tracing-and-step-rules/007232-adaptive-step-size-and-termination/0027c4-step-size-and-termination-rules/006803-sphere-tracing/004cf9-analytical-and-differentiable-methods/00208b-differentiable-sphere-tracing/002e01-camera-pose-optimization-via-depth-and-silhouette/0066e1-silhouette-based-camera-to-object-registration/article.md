# Silhouette-based Camera-to-Object Registration

Also called: Coarse Pose Initialization, Multi-view Consensus Registration, Silhouette Matching for CAD Registration, Silhouette-Based CAD Matching

## Overview
This method retrieves a CAD model from a library and registers relative camera extrinsics to its canonical frame by matching segmented object silhouettes against rendered CAD silhouettes [S1, unofficial]. It solves the problem of initializing camera-to-object poses for sparse-view reconstruction without requiring per-object pre-training or dense feature matching [S1, unofficial]. Unlike single-image methods that process frames individually, this method uses existing relative calibration to retain mutually consistent proposals for the whole capture [S1, unofficial].

## How it works
**Inputs:**
- A set of $N$ input images $I = \{I_1, \dots, I_N\}$ with known relative calibration (poses relative to one another) [S1, unofficial].
- A library of candidate CAD models [S1, unofficial].

**Outputs:**
- A selected CAD model $M$ [S1, unofficial].
- An initial transformation $T_{init}$ registering the camera trajectory to the object's canonical frame [S1, unofficial].
- Initial per-camera poses $T_{init}^{(i)}$ [S1, unofficial].

**Data Structures:**
- **Pre-rendered Library:** A static lookup table mapping each CAD model to a set of 2D binary silhouettes. Each entry contains:
    - `model_id`: Identifier for the CAD mesh.
    - `viewpoint_grid`: A dense grid of viewpoints sampled over the viewing sphere.
    - `silhouettes`: Binary masks rendered from the `viewpoint_grid` using white textures, no environment lighting, and a black background [S1, unofficial].
- **Observed Masks:** Binary foreground masks $M_{GT}$ extracted from input images [S1, unofficial].

**Parameters:**
- **IoU Threshold:** 0.7 [S1, unofficial].
- **Top-K Hypotheses:** 50 viewpoints per mask [S1, unofficial].

**Steps:**

1.  **Extract Observed Masks:**
    For each input image $I_i$, extract a binary foreground mask $M_{GT}^{(i)}$ using a segmentation model (e.g., SAM3) [S1, unofficial].

2.  **Normalize Masks:**
    For every mask (both observed $M_{GT}^{(i)}$ and pre-rendered library silhouettes):
    - Crop to the tight bounding box.
    - Resize to a fixed-size square canvas while preserving aspect ratio [S1, unofficial].

3.  **Generate Pose Hypotheses:**
    For each observed mask $M_{GT}^{(i)}$:
    - Compare against all pre-rendered silhouettes from the library.
    - Compute Intersection-over-Union (IoU) for each candidate silhouette.
    - Retain the top 50 viewpoints with IoU $\ge 0.7$ as pose hypotheses for view $i$ [S1, unofficial].

4.  **Multi-view Consensus via RANSAC:**
    Perform RANSAC over Sim(3) to find the most consistent registration:
    - **Iteration:**
        - Randomly sample two images, $I_a$ and $I_b$.
        - Select one pose hypothesis from the hypotheses of $I_a$ and one from $I_b$.
        - Compute a candidate transformation $T$ that registers the relative calibration of the cameras to the object frame:
            - Rotation and translation are derived from the first hypothesis [S1, unofficial].
            - Scale is derived from the ratio of inter-camera distances [S1, unofficial].
        - **Scoring:**
            - Render the candidate CAD model's silhouette in all $N$ views using $T$.
            - Accumulate per-view IoU scores between rendered silhouettes and observed masks $M_{GT}^{(j)}$ for all $j=1 \dots N$.
    - **Selection:**
        - Select the CAD model $M$ and transformation $T_{init}$ with the highest aggregate support (sum of IoU scores) [S1, unofficial].

5.  **Initialize Poses:**
    Apply $T_{init}$ to the relatively calibrated cameras to yield initial per-camera poses $T_{init}^{(i)}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_registration(input_images, relative_calib, cad_library, iou_threshold=0.7, top_k=50):
    """
    input_images: List of images I_1...I_N
    relative_calib: List of relative poses between cameras
    cad_library: Pre-rendered silhouettes for CAD models
    """
    # Step 1 & 2: Extract and Normalize Masks
    observed_masks = []
    for img in input_images:
        mask = extract_mask(img) # e.g., SAM3
        normalized_mask = normalize_mask(mask) # Crop to bbox, resize to square
        observed_masks.append(normalized_mask)

    # Step 3: Generate Hypotheses
    # hypotheses[i] is a list of (cad_model_id, viewpoint_index, score)
    hypotheses = []
    for i, mask in enumerate(observed_masks):
        view_hypotheses = []
        for model_id, rendered_silhouettes in cad_library.items():
            for view_idx, rendered_mask in enumerate(rendered_silhouettes):
                iou = compute_iou(mask, rendered_mask)
                if iou >= iou_threshold:
                    view_hypotheses.append((model_id, view_idx, iou))
        # Sort by score descending and keep top K
        view_hypotheses.sort(key=lambda x: x[2], reverse=True)
        hypotheses.append(view_hypotheses[:top_k])

    # Step 4: RANSAC over Sim(3)
    best_score = -1
    best_model = None
    best_T_init = None
    
    # Number of RANSAC iterations (not specified, assume sufficient for convergence)
    num_iterations = 1000 
    
    for _ in range(num_iterations):
        # Sample two random images
        idx_a, idx_b = random.sample(range(len(observed_masks)), 2)
        
        # Sample one hypothesis from each
        hyp_a = random.choice(hypotheses[idx_a])
        hyp_b = random.choice(hypotheses[idx_b])
        
        model_id_a, view_idx_a, _ = hyp_a
        model_id_b, view_idx_b, _ = hyp_b
        
        # Consistency check: hypotheses must come from the same CAD model
        if model_id_a != model_id_b:
            continue
            
        # Compute candidate transformation T
        # Rotation/Translation from first hypothesis (hyp_a)
        # Scale from ratio of inter-camera distances
        T_candidate = compute_sim3_transform(hyp_a, hyp_b, relative_calib)
        
        # Score the candidate
        total_iou = 0
        for j, mask in enumerate(observed_masks):
            # Render silhouette for model_id_a using T_candidate and view j
            rendered_mask = render_silhouette(model_id_a, T_candidate, view_index=j)
            normalized_rendered = normalize_mask(rendered_mask)
            total_iou += compute_iou(mask, normalized_rendered)
            
        if total_iou > best_score:
            best_score = total_iou
            best_model = model_id_a
            best_T_init = T_candidate

    # Step 5: Initialize Poses
    # Apply best_T_init to relative cameras
    initial_poses = apply_transform(relative_calib, best_T_init)
    
    return best_model, best_T_init, initial_poses

def normalize_mask(mask):
    bbox = get_tight_bbox(mask)
    cropped = crop(mask, bbox)
    # Preserve aspect ratio, resize to fixed square (e.g., 256x256)
    return resize_preserve_aspect(cropped, 256)

def compute_sim3_transform(hyp_a, hyp_b, relative_calib):
    # Rotation and translation from the first hypothesis
    R, t = get_pose_from_hypothesis(hyp_a)
    # Scale from the ratio of inter-camera distances
    # Note: relative_calib provides distances between cameras
    # Scale is derived from the geometric consistency of the two sampled views
    scale = compute_scale_from_inter_camera_distance(hyp_a, hyp_b, relative_calib)
    return Sim3(R, t, scale)
```

## Operating characteristics
- **Time Complexity:** The offline pre-rendering of the CAD library is performed once in parallel [S1, unofficial]. Matching a new capture involves comparing masks against stored silhouettes, which is fast [S1, unofficial]. RANSAC iterations depend on the number of views and hypotheses but are generally efficient due to the discrete nature of the library [S1, unofficial].
- **Memory Complexity:** Requires storing pre-rendered silhouettes for all CAD models in the library [S1, unofficial].
- **Strengths:**
    - Works for texture-less or reflective objects, as silhouette matching relies on shape rather than appearance [S1, unofficial].
    - Does not require per-category predictors or hand-annotated alignments [S1, unofficial].
    - Gracefully degrades to as few as 3 views [S1, unofficial].
- **Limitations:**
    - Silhouettes do not uniquely determine pose for symmetric objects [S1, unofficial].
    - The true viewing angle generally falls between discrete library samples, potentially leading to incorrect poses scoring higher IoU than the correct one [S1, unofficial].
    - Requires existing relative calibration to filter inconsistent proposals [S1, unofficial].
- **When to use:** Sparse-view reconstruction where relative calibration is available, and per-object pre-training is not feasible [S1, unofficial].
- **When not to use:** Dense-view captures where analysis-by-synthesis methods (e.g., iNeRF, BARF) might be more appropriate [S1, unofficial].

## Background
This method was introduced in the context of CADSplat, a framework for sparse-view 3D Gaussian Splatting aided by CAD models [S1, unofficial]. It addresses the limitations of classical mesh texturing and per-object model-based trackers, which require separate pre-training for every CAD object and often fail with textureless objects [S1, unofficial]. It shares practical properties with NOCS (Normalized Object Coordinate Space) but avoids the need for a learned per-pixel predictor [S1, unofficial]. The method is used to anchor 3D Gaussian primitives to the surface of a retrieved model for joint optimization of reconstruction and pose [S1, unofficial].

## Sources
[S1] CADSplat: Sparse-View 3D Gaussian Splatting Aided by CAD Models for Robust, Photorealistic Digital-Twin Reconstruction, https://arxiv.org/abs/2609.18473, fetched 2026-10-30 (unofficial)
