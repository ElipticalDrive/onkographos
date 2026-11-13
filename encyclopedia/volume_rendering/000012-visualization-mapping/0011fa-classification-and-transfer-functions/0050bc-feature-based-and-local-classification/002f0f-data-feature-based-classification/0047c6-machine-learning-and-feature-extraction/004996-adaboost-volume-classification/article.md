# AdaBoost Volume Classification

Also called: ECOC-AdaBoost Volume Classification, GPU-Parallelized AdaBoost Volume Classification, Intelligent Interactive Volume Classification

## Overview
AdaBoost Volume Classification is an interactive, data-driven method for multi-class segmentation of volumetric data using a framework of Error Correcting Output Codes (ECOC) and Discrete AdaBoost classifiers [S1, unofficial]. It solves the problem of classifying multiple regions of interest on demand without requiring preprocessing or complex optimization procedures [S1, unofficial]. Unlike graph cuts or random walker methods, it avoids segmentation dependencies among neighboring voxels, making it suitable for real-time applications and handling thinned or complex structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel model $V^M$ of size $dim_x \times dim_y \times dim_z$ and a set of $k$ training samples labeled by the user. Each training sample consists of a feature vector $\rho$ and a ground truth label $l(\rho)$ [S1, unofficial].
*   **Output:** A classification for every voxel in the volume, determining its optical properties (opacity and color) for rendering [S1, unofficial].

**Data Structures:**
1.  **Feature Vector ($\rho$):** An 8-dimensional vector for each voxel containing:
    *   Coordinates: $x, y, z$
    *   Density: $d$
    *   Gradient components: $g_x, g_y, g_z$
    *   Gradient magnitude: $|g|$ [S1, unofficial]
2.  **Look Up Table (LUT) ($L$):** A matrix of size $3 \times (|\rho| \times W_c)$ used to store the trained weak classifiers for GPU parallelization [S1, unofficial].
    *   $|\rho| = 8$ (feature dimensionality).
    *   $W_c$ is the number of weak classifiers. In a one-versus-one ECOC design for $N$ classes, $W_c = N(N-1)/2$ [S1, unofficial].
    *   **Row 1:** Stores the weight $c_m$ for each weak classifier. The position $i$ in this row corresponds to the feature index $mod(i, |\rho|)$. The next weight for the same feature is at $i + |\rho|$. Positions for features not considered by a specific stump are set to zero [S1, unofficial].
    *   **Row 2:** Stores the polarity $P_m$ for each weak classifier at column $i$ [S1, unofficial].
    *   **Row 3:** Stores the threshold $T_m$ for each weak classifier at column $i$ [S1, unofficial].

**Algorithm Steps:**

**1. Training Stage (Discrete AdaBoost):**
The system trains binary classifiers for each dichotomy in the ECOC framework [S1, unofficial].

1.  Initialize weights $w_i = 1/k$ for $i = 1 \dots k$.
2.  For $m = 1$ to $M$ (number of weak classifiers):
    a. Fit a weak classifier $f_m(\rho) \in \{-1, 1\}$ using the current weights $w_i$ on the training data. The weak classifier is a Decision Stump, fitting a threshold $T_m$ and polarity $P_m$ on a selected feature [S1, unofficial].
    b. Compute the weighted error: $err_m = E_w[1(l(\rho) \neq f_m(\rho))]$ [S1, unofficial].
    c. Compute the classifier weight: $c_m = \log((1 - err_m) / err_m)$ [S1, unofficial].
    d. Update sample weights: $w_i \leftarrow w_i \cdot \exp[c_m \cdot 1(l(\rho_i) \neq f_m(\rho_i))]$ for $i = 1 \dots k$.
    e. Normalize $w_i$ so that $\sum_i w_i = 1$ [S1, unofficial].
3.  Output the final classifier: $\text{sign}[\sum_{m=1}^M c_m f_m(\rho)]$ [S1, unofficial].

**2. LUT Construction:**
For each trained weak classifier (Decision Stump), store $P_m$ and $T_m$ in rows 2 and 3 of $L$ at the column corresponding to the feature index used by that stump. Store $c_m$ in row 1. Positions for features not considered by a specific stump are set to zero [S1, unofficial].

**3. Testing Stage (Classification):**
Applied to all voxels in the volume, typically parallelized on the GPU [S1, unofficial].

1.  Initialize $F(\rho) = 0$.
2.  For $m = 1$ to $M$:
    a. Let $\rho_m$ be the value of the feature selected by the $m$-th weak classifier for the test sample $\rho$.
    b. Retrieve $P_m$ and $T_m$ from the LUT.
    c. Update $F(\rho) = F(\rho) + c_m \cdot (P_m \cdot \rho_m < P_m \cdot T_m)$ [S1, unofficial].
3.  Output the label: $\text{sign}(F(\rho))$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_adaboost(training_samples, num_classes):
    """
    training_samples: list of tuples (rho, label)
    rho: array of 8 floats [x, y, z, d, gx, gy, gz, |g|]
    label: integer class index
    num_classes: N
    """
    k = len(training_samples)
    # Initialize weights
    weights = [1.0 / k] * k
    
    # ECOC one-versus-one design
    # Wc = N(N-1)/2 binary problems
    Wc = num_classes * (num_classes - 1) // 2
    
    # Store weak classifiers parameters
    # Each weak classifier is a Decision Stump on one feature
    weak_classifiers = [] 
    
    M = 20 # Number of boosting rounds, not given in sources [G]
    
    for m in range(M):
        # Fit weak classifier fm using weights
        # In practice, this involves finding the best feature, threshold, and polarity
        # that minimizes weighted error.
        best_feature_idx, best_threshold, best_polarity, best_error = find_best_stump(training_samples, weights)
        
        # Predictions of the weak classifier
        predictions = []
        for rho, label in training_samples:
            val = rho[best_feature_idx]
            if best_polarity == 1:
                pred = 1 if val < best_threshold else -1
            else:
                pred = 1 if val >= best_threshold else -1
            predictions.append(pred)
            
        # Compute error
        err_m = sum(weights[i] for i in range(k) if predictions[i] != training_samples[i][1])
        
        # Compute weight
        if err_m == 0 or err_m >= 1:
            # Handle degenerate cases to avoid division by zero or log(0)
            c_m = 0.0 
        else:
            c_m = math.log((1 - err_m) / err_m)
            
        weak_classifiers.append({
            'feature_idx': best_feature_idx,
            'threshold': best_threshold,
            'polarity': best_polarity,
            'weight': c_m
        })
        
        # Update weights
        for i in range(k):
            if predictions[i] != training_samples[i][1]:
                weights[i] *= math.exp(c_m)
                
        # Normalize weights
        sum_w = sum(weights)
        weights = [w / sum_w for w in weights]
        
    return weak_classifiers

def classify_volume(volume_data, weak_classifiers):
    """
    volume_data: 3D array of voxels, each voxel has feature vector rho (8 floats)
    weak_classifiers: list of dicts from training
    """
    # Construct LUT for GPU or efficient CPU access
    # LUT size: 3 x (8 x Wc)
    # However, we can just iterate if M is small, but LUT is specified for GPU parallelization [S1, unofficial]
    
    # For implementation clarity, we compute F(rho) directly
    # In GPU, this would be a kernel reading from LUT
    
    classified_volume = []
    
    for voxel in volume_data:
        rho = voxel.features # [x, y, z, d, gx, gy, gz, |g|]
        F_rho = 0.0
        
        for clf in weak_classifiers:
            m_idx = weak_classifiers.index(clf) # In LUT, this maps to column
            # Retrieve parameters from LUT or clf dict
            P_m = clf['polarity']
            T_m = clf['threshold']
            c_m = clf['weight']
            
            # Get feature value used by this stump
            rho_m = rho[clf['feature_idx']]
            
            # Decision stump evaluation
            # (Pm * rho_m < Pm * Tm) evaluates to 1 if true, 0 if false
            # Note: The source formula uses indicator function logic implicitly
            # If P=1: rho < T -> 1 (class 1), else -1? 
            # The source says: F(rho) = F(rho) + cm * (Pm * rho_m < Pm * Tm)
            # And output is sign(F(rho)). 
            # Standard Discrete AdaBoost weak classifier f_m(rho) is +/- 1.
            # The term (Pm * rho_m < Pm * Tm) is 1 if condition met, 0 otherwise.
            # This implies the weak classifier output in the sum is effectively 
            # 2*indicator - 1? Or the source formula assumes a specific mapping.
            # Let's stick strictly to the source formula:
            
            condition = (P_m * rho_m < P_m * T_m)
            # In Python, bool is 0 or 1
            contribution = c_m * condition
            F_rho += contribution
            
        # Final decision
        if F_rho > 0:
            label = 1
        elif F_rho < 0:
            label = -1
        else:
            label = 1 # Tie-breaking, not specified [G]
            
        classified_volume.append(label)
        
    return classified_volume
```

**Parameters:**
*   **Feature Vector $\rho$:** 8 features ($x, y, z, d, g_x, g_y, g_z, |g|$) [S1, unofficial].
*   **Number of Weak Classifiers ($W_c$):** Determined by the ECOC design. For $N$ classes, $W_c = N(N-1)/2$ in one-versus-one design [S1, unofficial].
*   **Number of Boosting Rounds ($M$):** Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Feature Indexing in LUT:** The first row of $L$ stores weights for features $s \in [1..8]$. Position $i$ in row 1 corresponds to feature $mod(i, |\rho|)$. The next weight for the same feature is at $i + |\rho|$. Positions for features not considered by a specific stump are set to zero [S1, unofficial].
*   **Precision:** Floating point precision in the accumulation of $F(\rho)$ may affect the sign decision in edge cases where $F(\rho) \approx 0$ [G].
*   **Degenerate Input:** If $err_m = 0$ or $err_m = 1$, the log term becomes undefined. The pseudocode handles this by setting $c_m = 0$ [G].

## Operating characteristics
*   **Time Complexity:** Training is $O(M \cdot k \cdot |\rho|)$ per binary problem, where $M$ is the number of boosting rounds, $k$ is the number of training samples, and $|\rho|$ is the feature dimensionality. Testing is $O(M)$ per voxel.
*   **Memory Complexity:** The LUT requires $3 \times |\rho| \times W_c$ storage. For $N$ classes, $W_c = N(N-1)/2$.
*   **Performance:** The method is parallelized on the GPU for both training and testing, enabling rapid interaction and real-time classification [S1, unofficial].
*   **Strengths:** No preprocessing required; interactive; handles thinned and complex structures better than probabilistic segmentation methods; no dependency on neighboring voxels [S1, unofficial].
*   **Limitations:** Requires user-labeled training samples; performance depends on the quality and quantity of training data; multi-class extension relies on ECOC which increases the number of classifiers [S1, unofficial].
*   **When to use:** Interactive volume visualization where users paint voxels on 2D slices to label samples [S1, unofficial].
*   **When not to use:** When automatic, unsupervised segmentation is required; when real-time performance is critical and the number of classes is very large (increasing $W_c$) [S1, unofficial].

## Background
The method was proposed by S. Grau, A. Puig, S. Escalera, and M. Salamó from the Universitat Politècnica de Catalunya, Universitat de Barcelona, and Centre de Visió per Computador, Universitat Autònoma de Barcelona [S1, unofficial]. It was published in 2013 as part of the Eurographics conference proceedings [S1, unofficial]. The work extends previous data-driven and image-driven classification methods and addresses limitations of graph cuts and random walker methods which involve complex optimization and segmentation dependencies [S1, unofficial].

## Sources
[S1] Intelligent Interactive Volume Classification, https://doi.org/10.2312/pe.pg.pg2013short.023-028, fetched 2026-10-26 (unofficial)
