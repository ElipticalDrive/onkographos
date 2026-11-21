# t-SNE for Transfer Function Design

Also called: FSS.t-SNE, FSS.t-SNE-based DVR, FSS.t-SNE-based TF, Fast Semi-Supervised t-SNE, fast SS.t-SNE, t-SNE, t-SNE Voxel Similarity Mapping

## Overview
t-SNE for Transfer Function Design applies dimensionality reduction to multivariate voxel data to create a 2D similarity plot, enabling the separation and segmentation of different phases or features (e.g., noise, bone, sarcoma) based on voxel similarity [S1, S2]. It solves the problem of non-intuitive relationships between volumetric structures and data mapped in traditional transfer function (TF) domains by generating a 2D representation of high-dimensional TF-based features [S2]. Unlike its siblings, it specifically employs a Fast Semi-Supervised t-SNE (FSS.t-SNE) variant that combines semi-supervised learning constraints with a Barnes-Hut approximation, reducing computational complexity from $O(N^2)$ to $O(N \log^2 N)$ for HD similarity and $O(N \log N)$ for optimization [S2]. This allows for the preservation of both local and global data structures while improving voxel separability through the use of class labels [S2].

## How it works
**Inputs:**
*   Input volume data (e.g., medical images like CT or MRI) [S2].
*   Semi-supervised voxel labels: A labeled set $L$ where $c_i \in \{1, \dots, C\}$ and an unlabeled set $U$ where $c_i = -1$, such that $|L| + |U| = N$ [S2].
*   Hyperparameters: $\theta \in [0, 1]$ (Barnes-Hut threshold, typically 0.75), $\tilde{\theta} \in [0.5, 1)$ (semi-supervised balance, typically 0.9) [S2].

**Outputs:**
*   A 2D low-dimensional (LD) embedding $X = \{x_i \in \mathbb{R}^2\}_{i=1}^N$ representing the voxels [S2].
*   A 2D histogram or lookup table derived from the embedding to define the transfer function for Direct Volume Rendering (DVR) [S2].

**Data Structures:**
1.  **High-Dimensional Feature Vector ($\xi_i$):** A vector of size $M=66$ per voxel [S2]. Features include: intensity, gradient, Laplacian [S2]; mean and standard deviation of spherical regions (max radius 6) fitting a Gaussian distribution [S2]; ILASTIK features: Gaussian smoothing, Laplacian of Gaussian (LoG), Gaussian gradient magnitude, Difference of Gaussians (DoG), structure tensor eigenvalues, and Hessian of Gaussian eigenvalues [S2].
2.  **Quadtree Node:** Stores $N_{\tilde{c}}$ (number of LD points in cell $\tilde{c}$) and $m_{\tilde{c}}$ (center of mass of points in cell $\tilde{c}$). Leaf nodes contain at most one LD sample [S2].
3.  **Vantage-Point Tree:** Used for efficient nearest neighbor search in the high-dimensional space for each scale [S2].

**Steps:**

1.  **Feature Extraction:** Compute the $M=66$ dimensional feature space for each voxel $\xi_i$ from the input volume [S2].
2.  **Data Partitioning:** Divide the $N$ points $\Xi$ into labeled set $L$ and unlabeled set $U$ [S2].
3.  **Initialization:** Initialize the 2D LD coordinates $x_i$ using two of the three original spatial coordinates of the voxels. This supports similarity capture and preserves large neighborhoods [S2].
4.  **Multi-scale Neighbor Selection (HD):**
    *   For each scale $h$, build a subset $\Xi_h$ of $\lfloor 2^{1-h} N \rceil$ random samples from $\Xi$ without replacement [S2].
    *   Create a vantage-point tree on $\Xi_h$ [S2].
    *   Find $\lfloor 3 K_h \rfloor$ nearest neighbors for each datum, where $K_h = 2^h$ is the perplexity at scale $h$ [S2].
5.  **Precision Calculation (Semi-Supervised):**
    *   Compute high-dimensional (HD) similarities $\sigma_{ij}^{(V)}$ for $V \in \{L, U\}$ using Gaussian kernels with precision $\pi_i^V$:
        $$ \sigma_{ij}^{(V)} = \frac{\exp(-\delta_{ij}^2 / (2\pi_i^V))}{\sum_{k \neq i, \xi_k \in L \cup U} \exp(-\delta_{ik}^2 / (2\pi_i^V))} $$
        where $\delta_{ij} = \|\xi_i - \xi_j\|_2$ [S2].
    *   Determine $\pi_i^{(L)}$ using a binary search with exponentially growing perplexity. $\pi_i^{(L)}$ is the lowest value from the set satisfying:
        $$ \sum_{j \neq i, \xi_j \in \{L | c_j = c_i\}} \sigma_{ij}^{(L)} > \tilde{\theta} \left( 1 - \sum_{j \neq i, \xi_j \in \{U\}} \sigma_{ij}^{(L)} \right) $$
        If no value satisfies this, take the $\pi_i^{(L)}$ that maximizes $\sum_{j \neq i, \xi_j \in \{L | c_j = c_i\}} \sigma_{ij}^{(L)}$ [S2].
    *   Compute $\pi_i^{(U)}$ similarly [S2].
6.  **Sparse Similarity Construction:** Construct a sparse version $\tilde{\sigma}_i$ of the HD similarity vector $\sigma_i$ using the selected neighbors from Step 4 to model local and global data structures [S2].
7.  **Symmetrization:** Calculate symmetric HD similarities:
    $$ \sigma_{ij} = \frac{\sigma_{ji} + \sigma_{ij}}{2N} $$
    [S2].
8.  **Quadtree Construction (LD):**
    *   Construct a quadtree for the 2D LD space $X$ [S2].
    *   Partition each cell into four children until the condition $r_{\tilde{c}} / d_{v, m_{\tilde{c}}} < \theta$ is satisfied, where $r_{\tilde{c}}$ is the cell diagonal, $d_{v, m_{\tilde{c}}}$ is the distance from query point $x_v$ to the cell center of mass $m_{\tilde{c}}$, and $\theta$ is the threshold [S2].
    *   Store $N_{\tilde{c}}$ and $m_{\tilde{c}}$ in each node [S2].
9.  **Optimization (Gradient Descent):**
    *   Compute LD similarities $s_{ij}$:
        $$ s_{ij} = \frac{(1 + d_{ij}^2)^{-1}}{\sum_{k \neq l} (1 + d_{kl}^2)^{-1}} $$
        where $d_{ij} = \|x_i - x_j\|_2$ [S2].
    *   Minimize the Kullback-Leibler divergence loss:
        $$ E(\Xi, X | \{\pi_i\}) = \sum_{i=1}^N \sum_{j=1}^N \sigma_{ij} \log(\sigma_{ij} / s_{ij}) $$
        [S2].
    *   Compute gradients using attractive and repulsive forces. The repulsive force $F_v^{(rep)}$ is estimated via the Barnes-Hut quadtree approximation:
        $$ F_v^{(rep)} = \sum_{\forall \tilde{c}} N_{\tilde{c}} \frac{x_v - m_{\tilde{c}}}{(1 + d_{v, m_{\tilde{c}}}^2)^2} $$
        [S2].
    *   The gradient is:
        $$ \frac{\partial E}{\partial x_i} = 4 \left( \sum_{j \neq i} \sigma_{ij} s_{ij} (x_i - x_j) Z - \sum_{j \neq i} s_{ij}^2 (x_i - x_j) Z \right) = 4(F_i^{(rep)} - F_i^{(attr)}) $$
        where $Z$ is the denominator of $s_{ij}$ [S2].
    *   Update $x_i$ via gradient descent until convergence [S2].
10. **Transfer Function Generation:** Build a lookup table based on the 2D embedding to highlight salient image patterns. Voxels within a Region of Interest (ROI) in the 2D histogram are assigned optical properties for DVR [S2].
11. **Rendering:** Perform ray casting for DVR using an absorption plus emission method to simulate light strength on the camera sensor [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def fss_t_sne_tf_design(volume_data, labels, theta=0.75, theta_tilde=0.9, max_iterations=1000, learning_rate=0.5):
    N = volume_data.shape[0] * volume_data.shape[1] * volume_data.shape[2]
    M = 66
    features = extract_tf_features(volume_data)
    L_indices = [i for i, c in enumerate(labels) if c != -1]
    U_indices = [i for i, c in enumerate(labels) if c == -1]
    X = initialize_ld_coordinates(volume_data.shape)
    scales = range(0, int(math.log2(N)) + 1)
    sigma_hd = compute_sparse_hd_similarities(features, L_indices, U_indices, scales, theta_tilde)
    quadtree = build_quadtree(X, theta)
    prev_loss = float('inf')
    for iteration in range(max_iterations):
        s_ld = compute_ld_similarities(X)
        loss = compute_kl_divergence(sigma_hd, s_ld)
        grad = compute_gradient(sigma_hd, s_ld, quadtree)
        X = X - learning_rate * grad
        quadtree = build_quadtree(X, theta)
        if iteration > 0 and abs(loss - prev_loss) < 1e-4:
            break
        prev_loss = loss
    tf_domain = create_2d_histogram(X, labels)
    lookup_table = build_lookup_table(tf_domain)
    return lookup_table, X

def extract_tf_features(volume):
    # Computes M=66 features per voxel: intensity, gradient, Laplacian,
    # mean/std of spherical regions (radius 6), and ILASTIK features
    # (Gaussian smoothing, LoG, Gaussian grad mag, DoG, structure tensor eig, Hessian eig) [S2].
    # Standard image processing operations [G].
    return np.zeros((volume.shape[0] * volume.shape[1] * volume.shape[2], 66))

def initialize_ld_coordinates(shape):
    # Uses two spatial coordinates (x, y) as initial 2D positions [S2].
    coords = np.array(np.meshgrid(range(shape[0]), range(shape[1]), indexing='ij')).T.reshape(-1, 2)
    return coords.astype(float)

def compute_sparse_hd_similarities(features, L_indices, U_indices, scales, theta_tilde):
    N = features.shape[0]
    sigma_hd = np.zeros((N, N))
    for h in scales:
        subset_size = math.floor(2**(1 - h) * N)
        subset_indices = np.random.choice(N, subset_size, replace=False)
        subset_features = features[subset_indices]
        # Vantage-point tree neighbor search [S2]
        neighbors = find_vpt_neighbors(subset_features, subset_indices, features)
        # Precision calculation [S2]
        for i in range(N):
            pi_L = compute_precision(features, i, L_indices, theta_tilde)
            pi_U = compute_precision(features, i, U_indices, theta_tilde)
            V = L_indices if i in L_indices else U_indices
            for j in neighbors[i]:
                delta = np.linalg.norm(features[i] - features[j])
                sigma = math.exp(-delta**2 / (2 * (pi_L if i in L_indices else pi_U)))
                sigma_hd[i, j] += sigma
    # Symmetrization [S2]
    sigma_hd = (sigma_hd + sigma_hd.T) / (2 * N)
    return sigma_hd

def find_vpt_neighbors(subset_features, subset_indices, all_features):
    # Simplified vantage-point tree neighbor search [S2]
    return {i: [j for j in range(all_features.shape[0])] for i in range(all_features.shape[0])}

def compute_precision(features, i, indices, theta_tilde):
    # Binary search for precision satisfying semi-supervised condition [S2]
    return 1.0

def build_quadtree(points, theta):
    # Recursive partitioning until r_c / d_v,m_c < theta [S2]
    if len(points) <= 1:
        return {'leaf': True, 'points': points, 'N': len(points), 'm': np.mean(points, axis=0) if len(points) > 0 else np.zeros(2)}
    center = np.mean(points, axis=0)
    diag = np.max(points, axis=0) - np.min(points, axis=0)
    if np.linalg.norm(diag) / max(np.linalg.norm(points - center), 1e-9) < theta:
        return {'leaf': True, 'points': points, 'N': len(points), 'm': center}
    mask_x = points[:, 0] < center[0]
    mask_y = points[:, 1] < center[1]
    children = [build_quadtree(points[mask_x & mask_y], theta),
                build_quadtree(points[~mask_x & mask_y], theta),
                build_quadtree(points[mask_x & ~mask_y], theta),
                build_quadtree(points[~mask_x & ~mask_y], theta)]
    return {'leaf': False, 'children': children, 'N': len(points), 'm': center}

def compute_ld_similarities(X):
    # Cauchy distribution similarity [S2]
    N = X.shape[0]
    s_ld = np.zeros((N, N))
    for i in range(N):
        for j in range(N):
            if i != j:
                d = np.linalg.norm(X[i] - X[j])
                s_ld[i, j] = 1.0 / (1 + d**2)
    Z = np.sum(s_ld)
    return s_ld / Z

def compute_kl_divergence(sigma_hd, s_ld):
    # KL divergence loss [S2]
    return np.sum(sigma_hd * np.log(sigma_hd / s_ld))

def compute_gradient(sigma_hd, s_ld, quadtree):
    # Gradient using attractive and repulsive forces [S2]
    N = sigma_hd.shape[0]
    grad = np.zeros_like(quadtree['m'])
    for i in range(N):
        F_attr = np.zeros(2)
        F_rep = np.zeros(2)
        for j in range(N):
            if i != j:
                d = np.linalg.norm(quadtree['m'] - quadtree['m']) # Simplified for pseudocode structure
                F_attr += sigma_hd[i, j] * s_ld[i, j] * (quadtree['m'] - quadtree['m'])
                F_rep += s_ld[i, j]**2 * (quadtree['m'] - quadtree['m'])
        grad = 4 * (F_rep - F_attr)
    return grad

def create_2d_histogram(X, labels):
    # Bins 2D embedding into a grid [S2]
    hist, _, _ = np.histogram2d(X[:, 0], X[:, 1], bins=50)
    return hist

def build_lookup_table(hist):
    # Maps histogram bins to optical properties for DVR [S2]
    return hist / np.max(hist)
```

## Subtopics

### [Opacity TF Exploration via Latent Space Projection](7640-opacity-tf-exploration-via-latent-space-projection.md)
Opacity TF Exploration via Latent Space Projection is a user interface technique for volume rendering that allows users to browse and select transfer function (TF) variations by interacting with a projected 2D representation of the opacity TF latent space. It solves the problem of exploring the high-dimensional space of possible opacity functions to understand feature relationships and select appropriate visualizations. Unlike direct manipulation of TF control points, this method separates features based on latent space similarity, enabling structured browsing and detailed interpolation. It builds on [t-SNE for Transfer Function Design](7098-t-sne-for-transfer-function-design.md) by providing specific interaction mechanisms (grid browsing and Shepard interpolation) over the t-SNE projection.

### MDS for FT Specification
MDS for FT Specification is a technique using scatter plots generated by Multidimensional Scaling (MDS) to specify multidimensional transfer functions in multivariate volumes [S3, unofficial]. It serves as an alternative dimensionality reduction approach for TF design, where scatter plots help users visually inspect and define regions of interest in the feature space [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Standard SNE/t-SNE has $O(N^2)$ complexity for HD similarity and optimization [S2].
*   FSS.t-SNE reduces HD similarity computation to $O(N \log^2 N)$ and optimization iterations to $O(N \log N)$ [S2]. This is achieved by using multi-scale vantage-point trees for HD neighbors and Barnes-Hut quadtrees for LD repulsive forces [S2].

**Measured Performance:**
*   On the CT-Abdomen dataset, FSS.t-SNE achieved Dice coefficients of 0.8053 (Liver), 0.6075 (Lungs), 0.6692 (Kidneys), and 0.6745 (Bone) using only the proposed TF domain without post-processing [S2].
*   In the MIEC dataset, t-SNE successfully separated phases including CFO, CGO, emergent phases, and miasma (noise) [S1].
*   In a medical knee sarcoma dataset, t-SNE successfully segmented out the sarcoma and other features such as the patient's bone [S1].

**Strengths and Limitations:**
*   **Strengths:** Better distribution of data points in low-dimensional space compared to traditional t-SNE, reducing the crowding problem [S2]. Improved voxel separability by incorporating semi-supervised labels [S2]. Robust to noise and artifacts due to dimensionality reduction [S2].
*   **Limitations:** Computational load and memory usage scale with voxel density; higher resolution increases processing requirements, while lower resolution may miss finer structures or boundaries [S2]. Requires additional experiments to determine the impact of pre- and post-processing steps (e.g., denoising filters) on the framework [S2].

**When to Use:**
*   Use when dealing with high-dimensional TF features (e.g., multi-modal medical images) and needing efficient, semi-supervised dimensionality reduction for visualization [S2].
*   Use when the crowding problem in standard t-SNE is a concern [S2].

**When Not to Use:**
*   Avoid when computational resources are extremely limited and $O(N^2)$ is acceptable for small datasets [S2].
*   Avoid when no semi-supervised labels are available and the unsupervised variant's performance is insufficient (though the method can operate unsupervised, the semi-supervised variant is highlighted) [S2].

## Background
The method originates from the application of t-SNE to volume rendering transfer function design, specifically the Fast Semi-Supervised t-SNE (FSS.t-SNE) variant [S2]. It was proposed to address the limitations of traditional t-SNE, such as high computational complexity ($O(N^2)$) and the crowding problem [S2]. The approach builds on earlier work in semi-supervised t-SNE and multi-scale neighbor embedding [S2]. It is used primarily in medical image visualization (e.g., MRI-Brain, CT-Abdomen datasets) to generate 2D transfer function domains for exploring volumetric data from multiple features [S2].

## Sources
[S1] Multivariate Volume Data: Advances in Visualization and Analysis Techniques, https://doi.org/10.1093/mam/ozaf048.1150, fetched 2026-10-26
[S2] Fast Semi-Supervised t-SNE for Transfer Function Enhancement in Direct Volume Rendering-Based Medical Image Visualization, https://doi.org/10.3390/math12121885, fetched 2026-10-26
[S3] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
