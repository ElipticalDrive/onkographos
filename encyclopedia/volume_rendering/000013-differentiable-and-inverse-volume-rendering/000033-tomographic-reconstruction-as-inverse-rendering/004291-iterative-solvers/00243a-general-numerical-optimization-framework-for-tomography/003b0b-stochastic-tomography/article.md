# Stochastic Tomography

Also called: Random walk tomography, Stochastic solver for tomography, random walk algorithm for tomographic reconstruction

## Overview
Stochastic Tomography is a matrix-free, grid-free iterative solver for tomographic reconstruction that represents the volume estimate as a set of discrete samples rather than a fixed voxel grid [S1]. It minimizes a convex objective function composed of squared residuals and a regularizer by using a random walk to explore the solution space and deposit emissivity [S1]. Unlike grid-based methods like [SIRT](3101-sirt-simultaneous-iterative-reconstruction-technique.md) or [SART](3101-sirt-simultaneous-iterative-reconstruction-technique.md), it avoids spatial discretization artifacts and allows for easy replacement of regularizers without re-implementing the solver core [S1]. It builds on the [General Numerical Optimization Framework for Tomography](3251-general-numerical-optimization-framework-for-tomography.md) by providing a specific mechanism for the data fidelity term via residual tracking and splatting [S1].

## How it works
**Inputs:**
*   $M$ projection images (observations) $o_i$ for $i=1 \dots M$.
*   Camera projection functions $proj_i(\mathbf{x})$ mapping 3D points to 2D image coordinates.
*   Visual hull mask $H(\mathbf{x})$ indicating the valid volume region (samples outside are rejected) [S1].
*   Sample budget $N_{total}$ (total number of mutations) [S1].
*   Gaussian mutation width $\sigma$ (typically 5-10% of the bounding box diagonal) [S1].
*   Emissivity magnitude $e_d$ (amount of density deposited per sample) [S1].
*   Optional regularizer $\Gamma(\mathbf{v})$ [S1].

**Outputs:**
*   A set of discrete samples $\{\mathbf{x}_k\}$ with associated emissivity signs, representing the volume estimate $\tilde{v}(\mathbf{x})$ [S1].

**Data Structures:**
*   **Residual Images:** An array of $M$ 2D images $\mathbf{r}_j$, one for each camera. Initialized to the observed intensities $o_j$ [S1].
*   **Sample List:** A collection of 3D coordinates $\mathbf{x}_k$ that have been accepted [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize residual images $\mathbf{r}_j = o_j$ for all $j=1 \dots M$. Start $K$ random walks (chains), each beginning at a uniformly random point $\mathbf{x}_0$ within the bounding box [S1].
2.  **Mutation:** For each walk, generate a candidate sample $\mathbf{x}_k$ by adding a Gaussian offset to the previous sample $\mathbf{x}_{k-1}$:
    $$ \mathbf{x}_k = \mathbf{x}_{k-1} + \mathbf{\epsilon}, \quad \mathbf{\epsilon} \sim \mathcal{N}(0, \sigma^2 I) $$
    The transition PDF $t(\mathbf{x}_k | \mathbf{x}_{k-1})$ is symmetric [S1].
3.  **Visual Hull Check:** If $\mathbf{x}_k$ lies outside the visual hull (i.e., projects to a zero-intensity pixel in any camera), reject the sample and set $\mathbf{x}_k \leftarrow \mathbf{x}_{k-1}$ [S1].
4.  **Objective Change Calculation:** Calculate the change in the objective function $\Delta F(\mathbf{x}_k)$ if the sample were accepted. This involves evaluating the change in squared residuals for both positive and negative emissivity:
    *   For each camera $j$, compute tentative residuals by splatting $\pm e_d$ at $proj_j(\mathbf{x}_k)$ onto $\mathbf{r}_j$:
        $$ \tilde{\mathbf{r}}_j^\pm = splat(\mathbf{r}_j, proj_j(\mathbf{x}_k), \pm e_d) $$
    *   Compute the change in squared residual error for camera $j$:
        $$ \Delta_j^\pm F(\mathbf{x}_k) = \mathbf{r}_j(proj_j(\mathbf{x}_k))^2 - \tilde{\mathbf{r}}_j^\pm(proj_j(\mathbf{x}_k))^2 $$
    *   Sum over all cameras:
        $$ \Delta^\pm F(\mathbf{x}_k) = \sum_{j=1}^M \Delta_j^\pm F(\mathbf{x}_k) $$
    *   Select the best sign:
        $$ \Delta F(\mathbf{x}_k) = \max(\Delta^+ F(\mathbf{x}_k), \Delta^- F(\mathbf{x}_k)) $$
    *   If a regularizer $\Gamma$ is used, add its energy change $\Delta \Gamma(\mathbf{x}_k)$ to $\Delta F(\mathbf{x}_k)$ [S1].
5.  **Acceptance:** Calculate acceptance probability $a$:
    $$ a = \min\left(1, \frac{\Delta F(\mathbf{x}_k)}{\Delta F(\mathbf{x}_{k-1})}\right) $$
    Accept the sample if $\Delta F(\mathbf{x}_{k-1}) < 0$ (initial step or previous step was invalid) OR `random() <= a`. Additionally, the sample is only recorded if $\Delta F(\mathbf{x}_k) > 0$ [S1].
6.  **Update:**
    *   If accepted: Record $\mathbf{x}_k$ in the sample list. Update residual images $\mathbf{r}_j$ by splatting the chosen sign of $e_d$ at $proj_j(\mathbf{x}_k)$ [S1].
    *   If rejected: Keep $\mathbf{x}_k \leftarrow \mathbf{x}_{k-1}$ and continue the walk from this point [S1].
7.  **Iteration:** Repeat steps 2-6 until the total sample budget is exhausted [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stochastic_tomography(observations, cameras, visual_hull, budget, sigma, ed, regularizer=None):
    # Initialize residuals
    residuals = [obs.copy() for obs in observations]
    samples = []
    
    # Start multiple walks
    num_walks = 10000 # Example value
    current_points = [random_point_in_bbox() for _ in range(num_walks)]
    
    # Initialize previous objective change (large positive to ensure first step acceptance)
    prev_delta_F = 1e9 
    
    for k in range(budget):
        new_points = []
        for i in range(num_walks):
            x_prev = current_points[i]
            
            # 1. Mutation
            x_cand = x_prev + gaussian_vector(sigma)
            
            # 2. Visual Hull Check
            if not visual_hull.contains(x_cand):
                new_points.append(x_prev)
                continue
                
            # 3. Calculate Delta F
            # Calculate residual changes for positive and negative emissivity
            delta_F_pos = 0.0
            delta_F_neg = 0.0
            
            for j, cam in enumerate(cameras):
                proj = cam.project(x_cand)
                
                # Simulate splatting for positive ed
                r_j_pos = splat(residuals[j], proj, ed)
                delta_r_pos = residuals[j][proj] ** 2 - r_j_pos[proj] ** 2
                
                # Simulate splatting for negative ed
                r_j_neg = splat(residuals[j], proj, -ed)
                delta_r_neg = residuals[j][proj] ** 2 - r_j_neg[proj] ** 2
                
                delta_F_pos += delta_r_pos
                delta_F_neg += delta_r_neg
            
            # Select best sign
            delta_F = max(delta_F_pos, delta_F_neg)
            
            # Add regularizer if present
            if regularizer:
                delta_F += regularizer.energy_change(x_cand, x_prev)
            
            # 4. Acceptance
            if prev_delta_F < 0 or random() <= min(1.0, delta_F / prev_delta_F):
                if delta_F > 0:
                    # Accept
                    samples.append(x_cand)
                    # Update residuals with best sign
                    if delta_F_pos > delta_F_neg:
                        for j, cam in enumerate(cameras):
                            residuals[j] = splat(residuals[j], cam.project(x_cand), ed)
                    else:
                        for j, cam in enumerate(cameras):
                            residuals[j] = splat(residuals[j], cam.project(x_cand), -ed)
                    prev_delta_F = delta_F
                    new_points.append(x_cand)
                else:
                    # Reject (delta_F <= 0)
                    new_points.append(x_prev)
                    prev_delta_F = prev_delta_F # Keep previous for next step? 
                    # Note: Algorithm 1 in [S1] implies x_k = x_{k-1} if rejected.
                    # The prev_delta_F for the NEXT step should be based on the state of x_{k-1}.
                    # Since x_{k-1} didn't change, prev_delta_F remains valid for the next mutation from x_{k-1}.
            else:
                # Reject (probabilistic)
                new_points.append(x_prev)
                # prev_delta_F remains the same as it was for x_{k-1}
        
        current_points = new_points
        
    return samples
```

## Subtopics

### [Stochastic Tomography Sample Acceptance and Residual Splatting](8890-stochastic-tomography-sample-acceptance-and-residual-splatti.md)
This method defines the acceptance criterion and state update mechanism for the Stochastic Tomography solver. It determines whether a candidate sample improves the reconstruction by evaluating the change in squared residuals across all projections, and if accepted, updates the residual images using bilinear splatting. It distinguishes itself by allowing both positive and negative emissivity deposition to enable "rough-in" and refinement phases, and by rejecting samples outside the visual hull to avoid wasted computation.

### Stochastic Tomography Regularization
Regularization incorporates prior information into the objective function to handle sparse or noisy measurements [S1]. It is added to the energy difference computed during sampling [S1]. Regularizers are evaluated on stencils centered at the sample with spacing equal to the grid spacing [S1]. L2 uses a 7-point Laplacian stencil (squared result), TV uses gradient magnitude via first-order finite differences, and SAD sums absolute differences between the center and 26 neighbors in a 3x3x3 window [S1]. Image-space regularization applies 2D variants of these regularizers directly on generated output images [S1]. Regularizers must be convex and have compact support [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the sample budget and number of cameras per mutation. Computation time varies by dataset: 14 min/frame for 200M mutations, 31 min/frame for 400M mutations, performed on a single core of an Intel i7 desktop [S1].
*   **Memory Complexity:** Low, as it does not store a full voxel grid. Memory is dominated by the residual images ($M \times W \times H$) and the list of accepted samples [S1].
*   **Strengths:** Grid-free representation avoids fixed resolution tradeoffs; easy to swap regularizers; handles sparse views well; captures fine-scale transient features [S1].
*   **Limitations:** Sensitive to camera misalignment (limits resolution); reflections in capture setup can introduce artifacts; requires careful tuning of mutation width [S1].
*   **When to use:** 3D imaging of mixing fluids, sparse view tomography, dynamic fluid capture [S1].
*   **When not to use:** When high precision is required despite minor misalignments, or when reflections cannot be separated from the volume [S1].

## Background
Stochastic Tomography was introduced in 2011 by [S1] as a new random walk algorithm for tomographic reconstruction. It was developed to address the limitations of grid-based methods like SART, which require fixed output grids and complex re-implementation for different regularizers [S1]. The method is applied to 3D imaging of mixing fluids, including fluorescein-sodium dye in water, alcohol-fluorescein-sodium mixtures, and dyepowder [S1]. It is not a true Markov-Chain Monte Carlo method because the target PDF is unknown and the acceptance probability depends on the full sampling history, not just the previous sample [S1].

## Sources
[S1] Stochastic tomography and its applications in 3D imaging of mixing fluids, https://doi.org/10.1145/2185520.2185548, fetched 2026-11-03
