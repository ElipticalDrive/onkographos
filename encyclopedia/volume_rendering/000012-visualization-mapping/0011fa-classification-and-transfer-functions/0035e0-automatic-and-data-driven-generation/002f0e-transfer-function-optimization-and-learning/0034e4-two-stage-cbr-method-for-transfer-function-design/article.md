# Two-stage CBR Method for Transfer Function Design

Also called: CBR-TF, Retrieval based Transfer Function Design, Two-stage CBR-TF approach, two-stage method

## Overview
The Two-stage CBR Method is an automatic transfer function (TF) design technique for volume rendering that retrieves similar visualization cases from a knowledge database and optimizes the resulting TF for visibility [S1, unofficial]. It solves the problem of manually defining TFs for specific Structures of Interest (SOIs) by using a user-specified ray to generate a Target Intensity Query (TIQ), which is matched against pre-labeled data [S1, unofficial]. It sets itself apart from its siblings by combining deep image feature matching (Stage 1) with intensity profile similarity (Stage 2) to improve retrieval accuracy over single-stage methods like the ED baseline [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1.  **Input Image Volume:** A 3D scalar field (e.g., CT scan).
2.  **Knowledge Database:** A collection of pre-computed ray elements derived from labeled image volumes. Each element consists of:
    *   A pair of co-planar images aligned with the primary axes relative to the ray.
    *   An intensity profile along the ray.
    *   The corresponding SOI label profile.
    *   Entry and exit coordinates of the ray.
3.  **User Interaction:** A line (ray) drawn by the user on a 2D cross-sectional view (axial, coronal, or sagittal) or a point clicked on a 3D Direct Volume Rendering (DVR) visualization [S1, unofficial].

**Outputs:**
1.  **Optimized Transfer Function:** A 1D intensity-based TF composed of tent-shaped components for each identified SOI.
2.  **Volume Rendering:** The final DVR visualization emphasizing the retrieved SOIs [S1, unofficial].

### Data Structures

**Ray Element (Database Entry):**
*   `images`: Array of 2 images (co-planar views).
*   `intensity_profile`: Array of float values representing intensity along the ray.
*   `label_profile`: Array of integer labels corresponding to SOIs along the ray.
*   `coordinates`: Tuple of (entry_point, exit_point).

**Target Intensity Query (TIQ):**
*   `qi`: Pair of images extracted from the input volume along the user-specified ray.
*   `qp`: Intensity profile along the user-specified ray.

**CNN Feature Vector:**
*   4096-dimensional float vector extracted from the FC7 layer of AlexNet [S1, unofficial].

### Numbered Steps

1.  **Offline Knowledge Database Construction:**
    *   Construct a database using labeled image volumes where every voxel has an intensity value and an SOI label [S1, unofficial].
    *   Uniformly extract multiple rays along each of the three primary axes (x, y, z).
    *   Exclude empty backgrounds.
    *   Use 192 rays total (64 per axis) per volume to reduce computational complexity and avoid highly similar rays [S1, unofficial].
    *   Store the ray elements (images, profiles, labels) in the database [S1, unofficial].

2.  **Ray Extraction and TIQ Generation:**
    *   For the input image volume, extract the images and intensity profile corresponding to the user-selected ray [S1, unofficial].
    *   Form the TIQ consisting of the image pair $q_i$ and intensity profile $q_p$ [S1, unofficial].

3.  **Stage 1: Image Matching (CNN-based):**
    *   Extract CNN features from the TIQ images $q_i$ and database images $c_i$ using a pre-trained AlexNet [S1, unofficial].
    *   Calculate the Image Similarity Distance $D_{img}(q_i, c_i)$ using Euclidean Distance (ED) between the 4096-dimensional feature vectors [S1, unofficial]:
        $$ D_{img}(q_i, c_i) = \sum_{j=1}^{2} \sum_{b=1}^{4096} (F^b_{qij} - F^b_{cij})^2 $$
        where $q_{ij}$ is the $j$-th image from the input pair, $c_{ij}$ is the $j$-th image from the database pair, and $F^b$ is the $b$-th CNN feature [S1, unofficial].
    *   Retrieve the top $N$ candidates from the database based on this distance [S1, unofficial].

4.  **Stage 2: Intensity Profile Matching (DTW-based):**
    *   For each of the top $N$ candidates, calculate the Intensity Profile Similarity Distance $D_{prof}(q_p, c_p)$ using Dynamic Time Warping (DTW) [S1, unofficial].
    *   Create a $L_{q_p} \times L_{c_p}$ distance matrix of pairwise distances, where $L_{q_p}$ and $L_{c_p}$ are the lengths of the profiles [S1, unofficial].
    *   Find the optimal warping path $P^*$ with minimum cost subject to:
        *   (i) Boundary condition: Path starts and ends in diagonally opposite corners.
        *   (ii) Continuity: Adjacent cells.
        *   (iii) Monotonicity: No backward steps [S1, unofficial].
    *   Calculate the distance:
        $$ D_{prof}(q_p, c_p) = \min \sqrt{\sum_{k=1}^{K} p_k / K} $$
        where $K$ is the length of the sequence $P^*$ and $p_k$ are the points in the path [S1, unofficial].
    *   Re-rank the top $N$ candidates based on this distance [S1, unofficial].

5.  **Ranking and Retrieval:**
    *   Combine the image retrieval and intensity profile matching scores. The final retrieval result is the case with the smallest overall similarity distance [S1, unofficial].
    *   Extract the SOI labels from the retrieved case [S1, unofficial].

6.  **TF Generation:**
    *   Generate a 1D intensity-based TF for the retrieved SOIs [S1, unofficial].
    *   For each SOI $S_m$ ($m=1..M$), create a tent-shaped component $T_m$:
        $$ T_m = \{(I_{l,m}, 0), (I_m, \sigma_m), (I_{h,m}, 0)\} $$
        where:
        *   $I_{l,m}$ is the lowest intensity in $S_m$.
        *   $I_{h,m}$ is the highest intensity in $S_m$.
        *   $I_m$ is the average intensity in $S_m$.
        *   $\sigma_m$ is the initial opacity of the tent peak, set to 0.3 [S1, unofficial].
    *   The initial TF is the union of all tent shapes: $\cup_{m=1}^{M} T_m$ [S1, unofficial].
    *   Assign unique colors to tent apexes and a single color (e.g., black) to end points for depth perception [S1, unofficial].

7.  **TF Optimization:**
    *   Iteratively optimize TF parameters using the downhill simplex method to minimize the energy function $E(\theta)$ based on visibility tolerance [S1, unofficial]:
        $$ \min_\theta E(\theta) = \sum_{m=1}^{M} (V^T_m - V^R_m)^2 $$
        where:
        *   $M$ is the total number of SOIs.
        *   $\theta$ is the set of opacity parameters of tent apexes.
        *   $V^T_m$ is the target visibility for SOI $m$ (auto-uniform or user-defined).
        *   $V^R_m$ is the resultant visibility for SOI $m$ [S1, unofficial].
    *   Calculate visibility $V_{SOI}$ for each SOI:
        $$ V_{SOI} = \sum_{p \in SOI} O(p)F(p) $$
        where:
        *   $O(p)$ is the opacity defined by the TF at voxel $p$.
        *   $F(p)$ is the front-to-back opacity composition from viewpoint $h$ to $p$:
            $$ F(p) = e^{-\int_{h}^{p} A(g) dg} $$
        *   $A(p)$ is the composited opacity at voxel $p$:
            $$ A(p) = A(p - \Delta p) + (1.0 - A(p - \Delta p))O(p) $$
            where $\Delta p$ is the sampling step size [S1, unofficial].
    *   Continue iteration until convergence [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_cbr_tf(input_volume, user_ray, knowledge_db):
    """
    Implements the Two-stage CBR Method for Transfer Function Design.
    
    Args:
        input_volume: 3D array of intensity values.
        user_ray: List of 3D coordinates defining the user-selected ray.
        knowledge_db: List of RayElement objects.
        
    Returns:
        optimized_tf: List of tent parameters for each SOI.
    """
    
    # Step 2: Ray Extraction and TIQ Generation
    tiq_images = extract_co_planar_images(input_volume, user_ray)
    tiq_profile = extract_intensity_profile(input_volume, user_ray)
    tiq = (tiq_images, tiq_profile)
    
    # Step 3: Stage 1 - Image Matching
    # Extract CNN features using pre-trained AlexNet
    tiq_features = extract_cnn_features(tiq_images) # 4096-dim vector
    
    # Calculate Image Similarity Distance for all database entries
    candidate_distances = []
    for db_entry in knowledge_db:
        db_features = extract_cnn_features(db_entry.images)
        dist_img = euclidean_distance(tiq_features, db_features)
        candidate_distances.append((db_entry, dist_img))
    
    # Sort by distance and get top N
    candidate_distances.sort(key=lambda x: x[1])
    top_n_candidates = candidate_distances[:N]
    
    # Step 4: Stage 2 - Intensity Profile Matching (DTW)
    ranked_candidates = []
    for candidate, dist_img in top_n_candidates:
        # Calculate DTW distance between profiles
        dist_prof = dynamic_time_warping(tiq_profile, candidate.intensity_profile)
        # Combine scores (simple sum as per source description of "overall similarity distance")
        # Note: Source implies re-ranking based on profile similarity, 
        # effectively using dist_prof as the primary ranker for the top N.
        ranked_candidates.append((candidate, dist_prof))
    
    # Sort by profile distance
    ranked_candidates.sort(key=lambda x: x[1])
    best_match = ranked_candidates[0][0]
    
    # Step 5: Retrieve SOI Labels
    retrieved_labels = best_match.label_profile
    unique_sois = get_unique_labels(retrieved_labels)
    
    # Step 6: TF Generation
    initial_tf = []
    for so_id in unique_sois:
        # Get intensity statistics for this SOI from the retrieved labels/profile
        so_indices = get_indices_for_label(retrieved_labels, so_id)
        so_intensities = [tiq_profile[i] for i in so_indices]
        
        il_m = min(so_intensities)
        ih_m = max(so_intensities)
        im_m = mean(so_intensities)
        sigma_m = 0.3 # Initial opacity
        
        tent = {
            'label': so_id,
            'intensity_low': il_m,
            'intensity_peak': im_m,
            'intensity_high': ih_m,
            'opacity_peak': sigma_m
        }
        initial_tf.append(tent)
        
    # Step 7: TF Optimization
    optimized_tf = optimize_tf_visibility(initial_tf, input_volume, unique_sois)
    
    return optimized_tf

def extract_cnn_features(images):
    """
    Extracts 4096-dimensional feature vector from FC7 layer of AlexNet.
    """
    # Assuming images is a list of 2 images
    features = []
    for img in images:
        # Preprocess image for AlexNet (resize, normalize, etc.)
        processed_img = preprocess_for_alexnet(img)
        # Forward pass through AlexNet up to FC7
        feature_vector = alexnet_fc7(processed_img)
        features.append(feature_vector)
    
    # Concatenate features from both images? 
    # Source formula sums over j=1 to 2, implying concatenation or separate calculation.
    # Formula: sum_j sum_b (F_qij - F_cij)^2
    # This implies we treat the two images as a single feature vector of 8192 dims 
    # or calculate distance per image and sum. 
    # The pseudocode below calculates distance per image and sums, 
    # matching the formula structure.
    return features # Return list of 2 vectors

def euclidean_distance(v1, v2):
    return sum((a - b) ** 2 for a, b in zip(v1, v2))

def dynamic_time_warping(seq1, seq2):
    """
    Calculates DTW distance between two sequences.
    """
    len1 = len(seq1)
    len2 = len(seq2)
    
    # Initialize distance matrix
    dtw_matrix = [[0.0] * (len2 + 1) for _ in range(len1 + 1)]
    
    # Fill with infinity
    for i in range(len1 + 1):
        for j in range(len2 + 1):
            dtw_matrix[i][j] = float('inf')
            
    dtw_matrix[0][0] = 0.0
    
    # Fill matrix
    for i in range(1, len1 + 1):
        for j in range(1, len2 + 1):
            cost = abs(seq1[i-1] - seq2[j-1])
            dtw_matrix[i][j] = cost + min(
                dtw_matrix[i-1][j],     # Deletion
                dtw_matrix[i][j-1],     # Insertion
                dtw_matrix[i-1][j-1]    # Match
            )
            
    # Calculate final distance metric as per source: min sqrt(sum(pk/K))
    # The source formula is slightly ambiguous. It uses min over paths P.
    # Standard DTW returns the cost of the optimal path.
    # The source formula: min sqrt( sum(pk/K) )
    # This looks like the root mean square of the path costs.
    # However, standard DTW cost is sum of costs. 
    # Let's implement standard DTW cost and then apply the source's specific metric if needed.
    # Given the source says "Intensity Profile Similarity Distance ... = min sqrt(sum(pk/K))",
    # and pk are points in the path.
    # We will return the standard DTW cost, as the square root and division by K 
    # are likely normalization factors applied post-calculation or specific to their implementation.
    # For implementation purposes, the DTW cost is the core similarity measure.
    
    optimal_cost = dtw_matrix[len1][len2]
    
    # Reconstruct path to calculate the specific metric if required
    # But for ranking, the cost itself is sufficient.
    return optimal_cost

def optimize_tf_visibility(initial_tf, volume, sois):
    """
    Optimizes TF parameters using downhill simplex method.
    """
    # Define energy function
    def energy_function(params):
        # params: list of opacities for tent peaks
        current_tf = build_tf_from_params(initial_tf, params)
        total_energy = 0.0
        for so_id in sois:
            # Calculate resultant visibility VR_m
            vr_m = calculate_visibility(volume, current_tf, so_id)
            # Target visibility VT_m (assume uniform 1.0/M or user-defined)
            vt_m = 1.0 / len(sois) # Default uniform
            total_energy += (vt_m - vr_m) ** 2
        return total_energy

    # Initial simplex vertices
    num_params = len(initial_tf)
    simplex = []
    for i in range(num_params + 1):
        vertex = [0.3] * num_params # Start with initial opacity 0.3
        if i > 0:
            vertex[i-1] = 0.3 + 0.1 # Perturb one parameter
        simplex.append(vertex)
        
    # Downhill simplex iteration
    # ... (Standard Nelder-Mead implementation) ...
    # Converge when energy change is small
    
    best_params = simplex[0]
    optimized_tf = build_tf_from_params(initial_tf, best_params)
    return optimized_tf

def calculate_visibility(volume, tf, so_id):
    """
    Calculates visibility V_SOI for a specific SOI.
    """
    visibility = 0.0
    # Iterate over all voxels in the volume
    for p in volume.voxels:
        if p.label == so_id:
            # Calculate opacity O(p) from TF
            o_p = tf.get_opacity(p.intensity)
            # Calculate front-to-back composition F(p)
            f_p = calculate_front_to_back_opacity(volume, p)
            visibility += o_p * f_p
    return visibility

def calculate_front_to_back_opacity(volume, p):
    """
    Calculates F(p) = exp(- integral A(g) dg)
    A(p) = A(p-dp) + (1 - A(p-dp)) * O(p)
    """
    # Simplified ray casting from viewpoint to p
    # This is a placeholder for the actual ray traversal logic
    # which depends on the specific renderer implementation.
    # The source defines the mathematical relationship.
    pass
```

### Parameters
*   **CNN Architecture:** AlexNet, pre-trained on ImageNet (1.2 million images). 7 trainable layers: 5 convolutional, 2 fully connected. FC7 output is 4096 dimensions [S1, unofficial].
*   **Tent Parameters:**
    *   $I_{l,m}$: Lowest intensity in SOI $S_m$.
    *   $I_{h,m}$: Highest intensity in SOI $S_m$.
    *   $I_m$: Average intensity in SOI $S_m$.
    *   $\sigma_m$: Initial opacity of tent peak, set to 0.3 [S1, unofficial].
*   **Visibility Tolerance:**
    *   $V$: Visibility proportion among all SOIs ($V=1.0$ means exclusive priority).
    *   $V^T$: Target visibility (auto-uniform or user-defined).
    *   $V^R$: Resultant visibility [S1, unofficial].
*   **Ray Extraction:** 192 rays total (64 per axis) per volume [S1, unofficial].
*   **Knowledge Database:** 3840 ray elements derived from 20 CT image volumes [S1, unofficial].
*   **Tracked SOIs:** Artery (1058), Bone (1824), Kidney (506), Liver (1645), Lung (1101), Spleen (306) [S1, unofficial].
*   **Evaluation Protocol:** 10-fold cross-validation, 18 CT volumes as database, 2 as queries, rotated 9 times [S1, unofficial].
*   **Hardware:** Intel i7 CPU, Nvidia GTX 980 Ti GPU, Windows 7 (x64), Voreen volume rendering engine [S1, unofficial].

### Edge Cases and Pitfalls
*   **ED Baseline Failure:** The Euclidean Distance (ED) baseline incorrectly labels arteries and kidneys along the Target Intensity Query (TIQ). The two-stage method correctly retrieves all SOIs by combining image and profile matching [S1, unofficial].
*   **Fine-tuning:** Fine-tuning AlexNet did not consistently improve recall; only spleen showed a meaningful improvement (0.044), suggesting fine-tuning may not be optimal for this application [S1, unofficial].
*   **CNN Choice:** Pre-trained AlexNet achieved the highest average recall across all SOIs except for lung, where GoogleNet provided a marginal improvement (0.007) [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Retrieval Computation:**
    *   ED Baseline: 0.84 seconds [S1, unofficial].
    *   DTW Compartment: 1.14 seconds [S1, unofficial].
    *   AlexNet Compartment: 15.70 seconds [S1, unofficial].
    *   Two-stage Method: 15.93 seconds [S1, unofficial].
*   **TF Generation:** < 1 second [S1, unofficial].
*   **Memory:** Depends on the size of the knowledge database (3840 ray elements for 20 volumes) and the CNN model weights (AlexNet has 60 million parameters) [S1, unofficial].

### Measured Performance
*   **Recall (All SOIs):**
    *   Two-stage Method (Pre-trained AlexNet): 0.719 [S1, unofficial].
    *   ED Baseline: 0.518 [S1, unofficial].
    *   GoogleNet: 0.707 [S1, unofficial].
    *   ResNet: 0.620 [S1, unofficial] [S1, unofficial].
*   **Average Recall Improvement:** 0.201 over ED baseline across all SOIs; largest improvement for kidney: 0.304 [S1, unofficial].
*   **Precision:** Not explicitly listed in the main text, but included in Appendix (Table A1) of the source [S1, unofficial].

### Strengths and Limitations
*   **Strengths:**
    *   Superior retrieval accuracy compared to single-stage methods (ED baseline, DTW only, Image only) [S1, unofficial].
    *   Effective for complex SOIs with similar intensity and shape (e.g., kidney, liver, spleen) [S1, unofficial].
    *   Simple and intuitive user interaction (drawing a line) [S1, unofficial].
*   **Limitations:**
    *   Computationally expensive due to CNN feature extraction (15.70 seconds) [S1, unofficial].
    *   Fine-tuning CNNs does not consistently improve performance [S1, unofficial].
    *   Relies on a pre-labeled knowledge database [S1, unofficial].

### When to Use
*   When automatic TF generation is needed for medical volumes with known SOIs.
*   When high retrieval accuracy is critical, especially for SOIs with similar intensity profiles but different spatial structures (e.g., arteries vs. kidneys) [S1, unofficial].

### When Not to Use
*   When real-time performance is required and the CNN computation time is prohibitive [S1, unofficial].
*   When the volume contains SOIs not present in the knowledge database [S1, unofficial].

## Background

### Origin
The Two-stage CBR Method was introduced by Y. Jung, J. Kong, and J. Kim in the paper "A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval" [S1, unofficial].

### History and Development
The method builds upon Case-Based Reasoning (CBR) techniques in visualization. It addresses the limitations of previous CBR approaches that relied solely on intensity profile matching using Euclidean Distance, which failed to distinguish between SOIs with similar intensity profiles but different spatial structures [S1, unofficial]. The integration of deep learning (AlexNet) for image feature extraction and Dynamic Time Warping (DTW) for profile matching was proposed to improve retrieval accuracy [S1, unofficial].

### Usage
The method is implemented in the Voreen volume rendering engine and evaluated on the 3D-IRCADb-01 dataset, which consists of 20 CT image volumes of the upper-abdomen [S1, unofficial]. It is used for automatic TF design in medical visualization, allowing users to quickly identify and visualize specific anatomical structures [S1, unofficial].

## Sources

[S1] A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval, https://arxiv.org/abs/2209.06421v1, fetched 2026-10-26 (unofficial)
