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
    *   $|\rho| = 8$ (feature dimensionality) [S1, unofficial].
    *   $W_c$ is the number of weak classifiers. In a one-versus-one ECOC design for $N$ classes, $W_c = N(N-1)/2$ [S1, unofficial].
    *   **Row 1:** Stores the weight $c_m$ for each weak classifier. The position $i$ in this row corresponds to the feature index $mod(i, |\rho|)$. The next weight for the same feature is at $i + |\rho|$. Positions for features not considered by a specific stump are set to zero [S1, unofficial].
    *   **Row 2:** Stores the polarity $P_m$ for each weak classifier at column $i$ [S1, unofficial].
    *   **Row 3:** Stores the threshold $T_m$ for each weak classifier at column $i$ [S1, unofficial].

**Algorithm Steps:**

**1. Training Stage (Discrete AdaBoost):**
The system trains binary classifiers for each dichotomy in the ECOC framework [S1, unofficial].

1.  Initialize weights $w_i = 1/k$ for $i = 1 \dots k$ [S1, unofficial].
2.  For $m = 1$ to $M$ (number of weak classifiers):
    a. Fit a weak classifier $f_m(\rho) \in \{-1, 1\}$ using the current weights $w_i$ on the training data. The weak classifier is a Decision Stump, fitting a threshold $T_m$ and polarity $P_m$ on a selected feature [S1, unofficial].
    b. Compute the weighted error: $err_m = E_w[1(l(\rho) \neq f_m(\rho))]$ [S1, unofficial].
    c. Compute the classifier weight: $c_m = \log((1 - err_m) / err_m)$ [S1, unofficial].
    d. Update sample weights: $w_i \leftarrow w_i \cdot \exp[c_m \cdot 1(l(\rho_i) \neq f_m(\rho_i))]$ for $i = 1 \dots k$ [S1, unofficial].
    e. Normalize $w_i$ so that $\sum_i w_i = 1$ [S1, unofficial].
3.  Output the final classifier: $\text{sign}[\sum_{m=1}^M c_m f_m(\rho)]$ [S1, unofficial].

**2. LUT Construction:**
For each trained weak classifier (Decision Stump), store $P_m$ and $T_m$ in rows 2 and 3 of $L$ at the column corresponding to the feature index used by that stump. Store $c_m$ in row 1. Positions for features not considered by a specific stump are set to zero [S1, unofficial].

**3. Testing Stage (Classification):**
Applied to all voxels in the volume. The authors parallelized the training stage using a GPU-based implementation for rapid interaction [S1, unofficial].

1.  Initialize $F(\rho) = 0$.
2.  For $m = 1$ to $M$:
    a. Let $\rho_m$ be the value of the feature selected by the $m$-th weak classifier for the test sample $\rho$ [S1, unofficial].
    b. Retrieve $P_m$ and $T_m$ from the LUT.
    c. Update $F(\rho) = F(\rho) + c_m \cdot (P_m \cdot \rho_m < P_m \cdot T_m)$ [S1, unofficial].
3.  Output the label: $\text{sign}(F(\rho))$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def train_adaboost(training_samples, num_classes):
    """
    training_samples: list of tuples (rho, label)
    rho: array of 8 floats [x, y, z, d, gx, gy, gz, |g|]
    label: integer class index
    num_classes: N
    """
    k = len(training_samples)
    weights = [1.0 / k] * k
    Wc = num_classes * (num_classes - 1) // 2
    weak_classifiers = []
    M = 20  # Number of boosting rounds, not given in sources [G]

    for m in range(M):
        best_feature_idx, best_threshold, best_polarity, best_error = find_best_stump(training_samples, weights)

        predictions = []
        for rho, label in training_samples:
            val = rho[best_feature_idx]
            if best_polarity == 1:
                pred = 1 if val < best_threshold else -1
            else:
                pred = 1 if val >= best_threshold else -1
            predictions.append(pred)

        err_m = sum(weights[i] for i in range(k) if predictions[i] != training_samples[i][1])

        if err_m <= 0 or err_m >= 1:
            c_m = 0.0
        else:
            c_m = math.log((1 - err_m) / err_m)

        weak_classifiers.append({
            'feature_idx': best_feature_idx,
            'threshold': best_threshold,
            'polarity': best_polarity,
            'weight': c_m
        })

        for i in range(k):
            if predictions[i] != training_samples[i][1]:
                weights[i] *= math.exp(c_m)

        sum_w = sum(weights)
        weights = [w / sum_w for w in weights]

    return weak_classifiers

def find_best_stump(training_samples, weights):
    """Finds the decision stump (feature, threshold, polarity) minimizing weighted error [G]."""
    best_error = float('inf')
    best_feature_idx = 0
    best_threshold = 0.0
    best_polarity = 1

    for feat_idx in range(8):
        values = sorted(set(rho[feat_idx] for rho, _ in training_samples))
        if len(values) < 2:
            continue
        thresholds = [(values[i] + values[i+1]) / 2.0 for i in range(len(values) - 1)]

        for thresh in thresholds:
            for pol in [1, -1]:
                error = 0.0
                for i, (rho, label) in enumerate(training_samples):
                    val = rho[feat_idx]
                    pred = 1 if (pol * val < pol * thresh) else -1
                    if pred != label:
                        error += weights[i]
                if error < best_error:
                    best_error = error
                    best_feature_idx = feat_idx
                    best_threshold = thresh
                    best_polarity = pol
    return best_feature_idx, best_threshold, best_polarity, best_error

def classify_volume(volume_data, weak_classifiers):
    """
    volume_data: list of voxel objects with .features attribute (8 floats)
    weak_classifiers: list of dicts from training
    """
    classified_volume = []

    for voxel in volume_data:
        rho = voxel.features
        F_rho = 0.0

        for clf in weak_classifiers:
            P_m = clf['polarity']
            T_m = clf['threshold']
            c_m = clf['weight']
            rho_m = rho[clf['feature_idx']]

            condition = (P_m * rho_m < P_m * T_m)
            contribution = c_m * condition
            F_rho += contribution

        if F_rho > 0:
            label = 1
        elif F_rho < 0:
            label = -1
        else:
            label = 1  # Tie-breaking, not specified [G]

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
*   **Time Complexity:** Not given in the sources. Standard AdaBoost training scales with $O(M \cdot k \cdot |\rho|)$ per binary problem, and testing scales with $O(M)$ per voxel [G].
*   **Memory Complexity:** The LUT requires $3 \times |\rho| \times W_c$ storage. For $N$ classes, $W_c = N(N-1)/2$ [S1, unofficial].
*   **Performance:** The authors parallelized the training stage using a GPU-based implementation for rapid interaction and classification [S1, unofficial].
*   **Strengths:** No preprocessing required; interactive; handles thinned and complex structures better than probabilistic segmentation methods; no dependency on neighboring voxels [S1, unofficial].
*   **Limitations:** Requires user-labeled training samples; performance depends on the quality and quantity of training data; multi-class extension relies on ECOC which increases the number of classifiers [S1, unofficial].
*   **When to use:** Interactive volume visualization where users paint voxels on 2D slices to label samples [S1, unofficial].
*   **When not to use:** When automatic, unsupervised segmentation is required; when real-time performance is critical and the number of classes is very large (increasing $W_c$) [S1, unofficial].

## Background
The method was proposed by S. Grau, A. Puig, S. Escalera, and M. Salamó from the Universitat Politècnica de Catalunya, Universitat de Barcelona, and Centre de Visió per Computador, Universitat Autònoma de Barcelona [S1, unofficial]. It was published in 2013 as part of the Eurographics conference proceedings [S1, unofficial]. The work extends previous data-driven and image-driven classification methods and addresses limitations of graph cuts and random walker methods which involve complex optimization and segmentation dependencies [S1, unofficial].

## Sources
[S1] Intelligent Interactive Volume Classification, https://doi.org/10.2312/pe.pg.pg2013short.023-028, fetched 2026-10-26 (unofficial)
