# K-means++ Subsampling

Also called: k-means++ clustering

## Overview
K-means++ Subsampling is a data reduction technique that selects a representative subset of raw data points to serve as the initial cluster centers for k-means clustering. It addresses the scalability bottleneck in high-dimensional volume data exploration by limiting the number of points processed during the iterative clustering phase, thereby allowing the algorithm to terminate within a reasonable time limit. It differs from standard k-means++ initialization [K-Means++ Initialization](18746-k-means-initialization.md) by treating the initialization step as a permanent subsampling operation for the entire clustering process, rather than just a setup phase for using all data points.

## How it works
**Inputs:** A dataset of $n$ points in a high-dimensional space; a target number of clusters $k$; a sampling rate $\alpha \in (0, 1]$.
**Outputs:** A subset of $m = \lfloor n \cdot \alpha \rfloor$ points from the original dataset, which serve as the initial centroids for the subsequent k-means clustering.

**Data Structures:**
- `points`: List of $n$ vectors, each of dimension $d$.
- `centers`: List of $k$ vectors, initialized from the subsampled points.
- `distances`: Array of size $n$, storing the squared Euclidean distance from each point to its nearest selected center.

**Parameters:**
- **Sampling rate ($\alpha$):** The fraction of raw data points retained. Typical values are 50% for hyperspectral data, 30% for hurricane data, and 25% for ionization data [S1].
- **Number of clusters ($k$):** Fixed at 3000 in the referenced implementations [S1].

**Algorithm Steps:**
1. **Calculate Total Points:** Determine the number of points to retain: $m = \lfloor n \cdot \alpha \rfloor$.
2. **Initialize First Center:** Select one point $c_1$ uniformly at random from the $n$ input points.
3. **Compute Distances:** For all $n$ points, compute the squared Euclidean distance to $c_1$. Store these in `distances`.
4. **Iterative Selection:** For $i$ from 2 to $k$:
   a. **Normalize Probabilities:** Compute the probability $P_j$ for each unselected point $j$ as:
      $$ P_j = \frac{\text{distances}[j]}{\sum_{l \in \text{unselected}} \text{distances}[l]} $$
      If the sum of distances is zero, select uniformly at random among unselected points [G].
   b. **Select Center:** Choose a point $c_i$ from the unselected points based on probabilities $P_j$.
   c. **Update Distances:** For each unselected point $j$, update `distances[j]` to be the minimum of its current value and the squared distance to the newly selected center $c_i$.
5. **Final Subsampling:** The set of selected centers $\{c_1, \dots, c_k\}$ constitutes the subsampled data. Note that if $k > m$, the algorithm effectively selects $k$ centers from the full dataset, but the *subsampled* set used for the main clustering loop is typically limited to $m$ points or the $k$ centers themselves depending on implementation. In the cited context, the subsampling rate limits the *input* to the clustering, implying the $k$ centers are drawn from the $\alpha n$ subset, or the subsampling *is* the selection of $k$ centers from the full set with a rate constraint on the effective complexity. The source states "maximize the sampling rate... until clustering terminates," implying the subsampled set size is the variable controlled by $\alpha$. The complexity $O(nk^2)$ refers to the cost on the *original* $n$ points if not subsampled, or the cost on the subsampled set. The text links $O(nk^2)$ to the subsampling method's complexity, suggesting $n$ here refers to the number of points considered in the distance updates. Given "maximize sampling rate", the algorithm likely runs k-means++ on the full set to pick $k$ centers, but the *subsampled* data for further processing (like subspace clustering) is the result. However, the complexity note $O(nk^2)$ is explicitly attributed to "k-means++ as subsampling". This implies the cost is proportional to the full dataset size $n$ times $k^2$ iterations/steps.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def k_means_pp_subsampling(points, k, alpha):
    """
    Selects k initial centers using k-means++ initialization.
    The 'subsampling' aspect refers to using this initialization 
    to define the cluster centers, potentially limiting the 
    dataset size for subsequent heavy operations.
    """
    n = len(points)
    # Determine how many points to actually consider if subsampling 
    # is applied *before* initialization. 
    # Source implies subsampling rate controls the data size for clustering.
    # If alpha < 1, we first subsample the data, then run k-means++ on it?
    # Or does k-means++ *perform* the subsampling by selecting k points?
    # Source: "k-means++ as subsampling has a complexity of O(nk^2)... 
    # We maximize the sampling rate...". 
    # This suggests the method IS the selection of k centers from n points.
    # The "sampling rate" might refer to the ratio k/n or a pre-filter.
    # However, typical k-means++ selects k centers. 
    # Let's assume the standard k-means++ initialization which selects k centers.
    
    centers = []
    distances = [float('inf')] * n
    
    # Step 1: Choose first center randomly
    idx = random.randint(0, n - 1)
    centers.append(points[idx])
    
    # Update distances to the first center
    for i in range(n):
        distances[i] = squared_distance(points[i], centers[0])
        
    # Step 2: Choose remaining k-1 centers
    for _ in range(1, k):
        # Calculate probabilities proportional to squared distance
        # Sum of distances for normalization
        total_dist = sum(distances)
        
        if total_dist == 0:
            # Fallback if all points are already covered (e.g., duplicates)
            # Select randomly from remaining unselected points
            unselected = [i for i in range(n) if points[i] not in centers]
            if not unselected:
                break
            idx = random.choice(unselected)
        else:
            # Weighted random choice
            probs = [d / total_dist for d in distances]
            idx = weighted_choice(points, probs)
            
        centers.append(points[idx])
        
        # Update distances
        new_center = points[idx]
        for i in range(n):
            d = squared_distance(points[i], new_center)
            if d < distances[i]:
                distances[i] = d
                
    return centers

def squared_distance(p1, p2):
    return sum((a - b) ** 2 for a, b in zip(p1, p2))

def weighted_choice(items, weights):
    # Simple implementation of weighted random choice
    total = sum(weights)
    r = random.uniform(0, total)
    upto = 0
    for item, weight in zip(items, weights):
        upto += weight
        if upto >= r:
            return item
    return items[-1]
```

## Subtopics

### MiniBatch K-means
A variant that updates codewords using subsets of input vectors to reduce computational cost. It initializes the first center randomly and samples each subsequent center with probability proportional to its squared distance from the nearest selected center. This approach is used in hybrid compression methods for 3D Gaussian Splatting models to train VQ codebooks efficiently [S2].

## Operating characteristics
**Time Complexity:** $O(nk^2)$ per iteration, where $n$ is the number of data points and $k$ is the number of clusters [S1].
**Memory Complexity:** Not explicitly given for the subsampling step alone, but subspace clustering (a related step in the pipeline) incurs $O(m^2)$ memory where $m$ is the number of samples [S1].
**Performance:** The method allows processing of large datasets (e.g., 1.2GB hyperspectral images) by limiting the number of points used for clustering. The sampling rate is maximized until the algorithm terminates in a reasonable time [S1].
**Strengths:** Improves scalability for high-dimensional data exploration; ensures initial centers are well-spread.
**Limitations:** Complexity grows quadratically with $k$ and linearly with $n$; hardware capacities still limit the size of raw volumetric data that can be processed for smooth animated transitions [S1].

## Background
The method is used in multivariate volume visualization through dynamic projections, particularly for earth remote sensing and hyperspectral imaging. It was introduced as part of a system for handling multivariate TF (Transfer Function) design and volume rendering, where data reduction via sampling and clustering is necessary to manage system scalability and hardware limits [S1].

## Sources
[S1] Multivariate volume visualization through dynamic projections, https://doi.org/10.1109/ldav.2014.7013202, fetched 2026-10-26
[S2] Hybrid Compression Method for Trained 3D Gaussian Splatting Models Based on VQ and HEVC., https://doi.org/10.3390/s26134125, fetched 2026-10-27
