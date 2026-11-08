# Two-stage CBR Method for Transfer Function Design

Also called: CBR-TF, Retrieval based Transfer Function Design, Two-stage CBR-TF approach, two-stage method

## Overview
The Two-stage CBR Method is an automatic transfer function (TF) design technique that retrieves similar visualization cases from a knowledge database based on a user-selected ray's image and intensity profile features, then optimizes the retrieved TF for visibility [S1, unofficial]. It solves the problem of manually defining transfer functions for specific Structures of Interest (SOIs) by combining high-level semantic matching with precise intensity profile alignment [S1, unofficial]. Unlike single-stage retrieval methods that rely solely on intensity profiles or image features, this approach uses a two-stage pipeline: first matching co-planar images using Convolutional Neural Network (CNN) features, and second re-ranking the results using Dynamic Time Warping (DTW) on intensity profiles [S1, unofficial]. This combination prevents misclassification of SOIs with similar intensities but different spatial structures, such as arteries and kidneys, which single-stage methods often confuse [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1.  **Knowledge Database:** A collection of labeled image volumes where every voxel has an intensity value and an SOI label. Each volume is pre-processed into ray elements [S1, unofficial].
2.  **Input Image Volume:** The target volume for which the TF is to be designed [S1, unofficial].
3.  **User Interaction:** A line (ray) drawn by the user on a 2D cross-sectional view (axial, coronal, or sagittal) or a point clicked on a 3D DVR visualization to specify the SOIs of interest [S1, unofficial].

**Outputs:**
1.  **Optimized Transfer Function:** A 1D intensity-based TF defined by tent-shaped components for each identified SOI, optimized for visibility [S1, unofficial].
2.  **Volume Rendering:** The final DVR visualization emphasizing the identified SOIs [S1, unofficial].

### Data Structures

**Ray Element:**
Each ray in the knowledge database and query consists of four components:
1.  **Image Pair ($q_i$):** A pair of co-planar 2D images aligned with the primary axes (x, y, z) perpendicular to the ray.
2.  **Intensity Profile ($q_p$):** A 1D array of intensity values along the ray.
3.  **SOI Label Profile:** A 1D array of SOI labels corresponding to the intensity values.
4.  **Coordinates:** Entry and exit point coordinates of the ray [S1, unofficial].

**Knowledge Database Layout:**
*   Derived from $N$ image volumes.
*   Each volume contains 192 rays (64 per axis: x, y, z).
*   Empty background rays are excluded.
*   Total elements = $N \times 192$ [S1, unofficial].

**CNN Feature Vector:**
*   Extracted from the FC7 layer of AlexNet.
*   Dimension: 4096 [S1, unofficial].

### Numbered Steps

1.  **Offline Construction:**
    *   Construct the knowledge database from labeled image volumes [S1, unofficial].
    *   Extract 192 rays per volume (64 along x, 64 along y, 64 along z).
    *   Exclude rays passing only through empty background [S1, unofficial].

2.  **User Input & TIQ Generation:**
    *   The user draws a line (ray) on a 2D view or clicks a point in 3D [S1, unofficial].
    *   Generate a Target Intensity Query (TIQ) comprising:
        *   The image pair $q_i$ (two co-planar images to the ray).
        *   The intensity profile $q_p$ along the ray [S1, unofficial].

3.  **Stage 1: Image Matching (CNN):**
    *   Extract CNN features from the TIQ image pair $q_i$ and all database image pairs $c_i$.
    *   Compute the Image Similarity Distance using Euclidean Distance (ED) between the 4096-dimensional feature vectors [S1, unofficial]:
        $$ \text{Image Similarity Distance}(q_i, c_i) = \sum_{j=1}^{2} \sum_{b=1}^{4096} (F^b_{qij} - F^b_{cij})^2 $$
        Where $F^b$ is the $b$-th CNN feature, $j$ indexes the two images in the pair, and $i$ indexes the database case [S1, unofficial].
    *   Retrieve the top $N$ candidates based on this distance [S1, unofficial].

4.  **Stage 2: Intensity Profile Matching (DTW):**
    *   For the top $N$ candidates from Stage 1, compute the Intensity Profile Similarity Distance using Dynamic Time Warping (DTW) between $q_p$ and the candidate's profile $c_p$ [S1, unofficial].
    *   Create a distance matrix of size $L_{q_p} \times L_{c_p}$, where $L$ is the length of the profile [S1, unofficial].
    *   Find the optimal warping path $P^*$ with minimum cost subject to:
        1.  **Boundary Condition:** Path starts at $(1,1)$ and ends at $(L_{q_p}, L_{c_p})$.
        2.  **Continuity:** Adjacent cells in the path must be neighbors.
        3.  **Monotonicity:** Path does not move backward [S1, unofficial].
    *   Calculate the distance:
        $$ \text{Intensity Profile Similarity Distance}(q_p, c_p) = \min \sqrt{\sum_{k=1}^{K} p_k / K} $$
        Where $K$ is the length of the warping path sequence $P$, and $p_k$ is the cost of the $k$-th step [S1, unofficial].
    *   Re-rank the top $N$ candidates based on this DTW distance. The final retrieval result is the candidate with the smallest overall similarity distance (combining image and profile scores) [S1, unofficial].

5.  **TF Generation:**
    *   Retrieve the SOI labels from the best-matching database case [S1, unofficial].
    *   For each SOI $S_m$ ($m=1..M$), generate a tent-shaped TF component $T_m$:
        $$ T_m = \{(I_{l,m}, 0), (I_m, \sigma_m), (I_{h,m}, 0)\} $$
        Where:
        *   $I_{l,m}$: Lowest intensity in $S_m$.
        *   $I_{h,m}$: Highest intensity in $S_m$.
        *   $I_m$: Average intensity in $S_m$.
        *   $\sigma_m$: Initial opacity of the tent peak, set to 0.3 [S1, unofficial].
    *   The initial TF is the union of all tent components:
        $$ \text{Initial TF} = \bigcup_{m=1}^{M} T_m $$
    *   Assign unique colors to tent apexes and a single color (e.g., black) to end points for depth perception [S1, unofficial].

6.  **TF Optimization:**
    *   Optimize the TF parameters $\theta$ (opacity values of tent apexes) using the Downhill Simplex method to minimize the energy function $E(\theta)$ [S1, unofficial]:
        $$ \min_\theta E(\theta) = \sum_{m=1}^{M} (V^T_m - V^R_m)^2 $$
        Where:
        *   $V^T_m$: Target visibility for SOI $m$ (uniformly set or user-defined).
        *   $V^R_m$: Resultant visibility for SOI $m$ [S1, unofficial].
    *   **Visibility Calculation:**
        *   Compute front-to-back opacity composition $F(p)$ for each voxel $p$:
            $$ F(p) = e^{-\int_{g}^{p} A(g) dg} $$
            Where $g$ is the viewpoint, and $A(p)$ is the composited opacity at voxel $p$ [S1, unofficial]:
            $$ A(p) = A(p - \Delta p) + (1.0 - A(p - \Delta p))O(p) $$
            Where $O(p)$ is the opacity defined by the TF, and $\Delta p$ is the sampling step size [S1, unofficial].
        *   Calculate visibility for SOI $m$:
            $$ V_{SOI} = \sum_{p \in SOI} O(p)F(p) $$
            Where the sum is over all voxels $p$ belonging to the SOI [S1, unofficial].
    *   Iterate until convergence [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_cbr_tf_design(knowledge_db, input_volume, user_ray):
    """
    knowledge_db: List of RayElement objects.
    input_volume: The 3D volume to visualize.
    user_ray: User-defined line/point in the volume.
    """
    
    # 1. Generate Target Intensity Query (TIQ)
    tiq_images = extract_co_planar_images(user_ray) # Returns 2 images
    tiq_profile = extract_intensity_profile(user_ray) # Returns 1D array
    
    # 2. Stage 1: Image Matching using CNN
    # Extract features using pre-trained AlexNet (FC7 layer, 4096 dims)
    tiq_features = extract_cnn_features(tiq_images)
    
    candidate_distances = []
    for db_ray in knowledge_db:
        db_features = extract_cnn_features(db_ray.images)
        # Euclidean Distance between 4096-dim vectors
        dist = sum((tiq_features[b] - db_features[b])**2 for b in range(4096))
        candidate_distances.append((dist, db_ray))
    
    # Sort by distance and take top N
    candidate_distances.sort(key=lambda x: x[0])
    top_n_candidates = candidate_distances[:N]
    
    # 3. Stage 2: Intensity Profile Matching using DTW
    ranked_candidates = []
    for dist_img, db_ray in top_n_candidates:
        # Dynamic Time Warping distance
        dist_prof = compute_dtw_distance(tiq_profile, db_ray.profile)
        # Combine scores (e.g., weighted sum or just use profile distance for re-ranking)
        # The source implies re-ranking based on profile similarity, 
        # then selecting the one with smallest overall similarity.
        ranked_candidates.append((dist_img + dist_prof, db_ray))
        
    ranked_candidates.sort(key=lambda x: x[0])
    best_ray = ranked_candidates[0][1]
    
    # 4. Generate Initial TF from retrieved SOI labels
    sois = best_ray.soil_labels
    initial_tf = []
    for m, so in enumerate(sois):
        intensities = get_intensity_range(so, input_volume)
        il = min(intensities)
        ih = max(intensities)
        im = mean(intensities)
        sigma = 0.3 # Initial opacity
        # Tent shape: (low_intensity, 0), (avg_intensity, sigma), (high_intensity, 0)
        tent = [(il, 0), (im, sigma), (ih, 0)]
        initial_tf.append(tent)
        
    # 5. Optimize TF for Visibility
    # Define energy function based on target vs resultant visibility
    # Use Downhill Simplex method to minimize E(theta)
    optimized_tf = optimize_visibility(initial_tf, input_volume, sois)
    
    return optimized_tf
```

### Parameters
*   **CNN Architecture:** AlexNet, pre-trained on ImageNet (1.2 million images). 7 trainable layers (5 convolutional, 2 fully connected). FC7 output is 4096 dimensions [S1, unofficial].
*   **Tent Parameters:**
    *   $\sigma_m$ (initial opacity): 0.3 [S1, unofficial].
*   **Ray Extraction:** 192 rays per volume (64 per axis). Empty backgrounds excluded [S1, unofficial].
*   **Visibility Tolerance:** $V^T_m$ can be auto-uniform or user-defined. $V=1.0$ means exclusive priority [S1, unofficial].
*   **Optimization Method:** Downhill Simplex [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** The ED baseline incorrectly labels arteries and kidneys along the TIQ due to similar intensity profiles but different spatial structures. The two-stage method correctly retrieves these by combining image and profile matching [S1, unofficial].
*   **Fine-tuning:** Fine-tuning the CNN (AlexNet) did not consistently improve recall compared to pre-trained weights; only spleen showed meaningful improvement. Pre-trained AlexNet is preferred [S1, unofficial].
*   **Empty Backgrounds:** Rays passing only through empty background must be excluded during extraction to reduce noise and computational complexity [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Retrieval Time:**
    *   ED Baseline: 0.84 seconds [S1, unofficial].
    *   DTW Compartment: 1.14 seconds [S1, unofficial].
    *   AlexNet Compartment: 15.70 seconds [S1, unofficial].
    *   Two-stage Method: 15.93 seconds [S1, unofficial].
*   **TF Generation Time:** < 1 second [S1, unofficial].
*   **Memory:** Depends on the size of the knowledge database (3840 ray elements for 20 volumes in the study) and the CNN model weights [S1, unofficial].

### Measured Performance
*   **Hardware:** Intel i7 CPU, Nvidia GTX 980 Ti GPU, Windows 7 (x64), Voreen engine [S1, unofficial].
*   **Dataset:** 20 CT image volumes (3D-IRCADb-01), 6 SOIs (Artery, Bone, Kidney, Liver, Lung, Spleen) [S1, unofficial].
*   **Recall (All SOIs):**
    *   Two-stage Method: 0.719 [S1, unofficial].
    *   ED Baseline: 0.518 [S1, unofficial].
    *   Pre-trained AlexNet (Stage 1 only): 0.679 [S1, unofficial].
    *   DTW (Stage 2 only): 0.672 [S1, unofficial].
*   **Improvement:** Average recall improvement of 0.201 over ED baseline; largest improvement for Kidney (0.304) [S1, unofficial].
*   **CNN Comparison:** Pre-trained AlexNet (0.719) outperformed GoogleNet (0.707) and ResNet (0.620) [S1, unofficial].

### Strengths and Limitations
*   **Strengths:**
    *   Combines semantic (image) and structural (profile) information for robust retrieval.
    *   Handles SOIs with similar intensities but different shapes (e.g., artery vs. kidney) better than single-stage methods.
    *   Simple user interaction (draw a line) [S1, unofficial].
*   **Limitations:**
    *   Computationally expensive due to CNN feature extraction (15.7s) compared to intensity-only methods (0.84s).
    *   Requires a pre-labeled knowledge database.
    *   Fine-tuning CNNs did not yield consistent improvements [S1, unofficial].

## Background

### Origin
The Two-stage CBR Method was proposed by Jung, Kong, and Kim in the paper "A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval" [S1, unofficial].

### History and Development
*   The method builds upon Case-Based Reasoning (CBR) for visualization, where previous works relied on intensity profile matching using Euclidean Distance (ED) [S1, unofficial].
*   It integrates deep learning (AlexNet) for image feature extraction to overcome the limitations of ED-based intensity matching, which fails to distinguish SOIs with similar intensity profiles but different spatial structures.
*   The approach was implemented in the Voreen volume rendering engine [S1, unofficial].

### Usage
*   Used in medical image visualization, specifically for CT volumes of the upper abdomen.
*   Applied to identify and visualize complex SOIs like arteries, bones, kidneys, liver, lungs, and spleens [S1, unofficial].

## Sources

[S1] A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval, https://arxiv.org/abs/2209.06421v1, fetched 2026-10-26 (unofficial)
