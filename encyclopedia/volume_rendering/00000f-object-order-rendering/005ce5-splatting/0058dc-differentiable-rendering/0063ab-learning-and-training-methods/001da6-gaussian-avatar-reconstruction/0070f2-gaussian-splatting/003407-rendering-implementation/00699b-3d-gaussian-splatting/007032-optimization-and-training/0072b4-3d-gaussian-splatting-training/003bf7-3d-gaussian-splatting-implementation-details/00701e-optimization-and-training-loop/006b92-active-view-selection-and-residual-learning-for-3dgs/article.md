# Active View Selection and Residual Learning for 3DGS

Also called: SA-ResGS (Self-augmented Residual 3D Gaussian Splatting), Self-augmented Residual 3D Gaussian Splatting

## Overview
SA-ResGS is a training framework that combines active view selection with residual learning to improve 3D Gaussian Splatting (3DGS) reconstruction quality under sparse view conditions [S1]. It addresses the limitation that standard training with few views often fails to converge to high-fidelity models due to ambiguous geometry and lack of coverage [S1]. It sets itself apart from sibling methods like [Uncertainty-Guided Residual Learning for 3DGS](20901-uncertainty-guided-residual-learning-for-3dgs.md) by explicitly integrating a self-augmented pre-filtering mechanism (SA-HashGS) to stabilize the view selection process, ensuring that the residual supervision is applied to reliable, high-quality views rather than noisy candidates [S1].

## How it works
SA-ResGS operates as an iterative training loop that alternates between Gaussian optimization and view selection. The method relies on the standard 3DGS optimization principles defined in [Optimization and Training Loop](17791-optimization-and-training-loop.md), specifically the composite loss $\mathcal{L} = (1 - \lambda) \mathcal{L}_{L1} + \lambda \mathcal{L}_{D-SSIM}$ with $\lambda = 0.2$ [S1].

### Inputs and Outputs
*   **Inputs:** A set of candidate training images $V_{cand}$, a COLMAP SfM initialization (positions $\mathbf{p}_i$, scales $\mathbf{s}_i$, rotations $\mathbf{q}_i$), and a fixed iteration budget.
*   **Outputs:** A final set of 3D Gaussians $\mathcal{G}$ optimized for the selected subset of views $V_{sel} \subseteq V_{cand}$.

### Data Structures
The method maintains the standard 3DGS parameter set $\mathcal{G} = \{g_i\}_{i=1}^N$ . Additionally, it utilizes a hash grid or point-based structure for the SA-HashGS component to estimate uncertainty, though the exact internal layout of the hash grid is not specified in the provided sources [S1]. The view selection mechanism relies on a direction-dependent coverage feature map with max aggregation and dropped opacity to guide selection [S1].

### Algorithm Steps
1.  **Initialization:** Initialize the Gaussian model $\mathcal{G}$ using the provided COLMAP SfM points. Set the initial training view set $V_{sel}$ to 4 uniformly distributed views from the candidate set [S1].
2.  **Training Loop:**
    a. Render images from the current views $V_{sel}$ using the current $\mathcal{G}$.
    b. Compute the composite loss $\mathcal{L}$ between rendered images and ground truth .
    c. Update $\mathcal{G}$ using the Adam optimizer ($\beta_1=0.9, \beta_2=0.99$) for a fixed number of iterations (typically 20,000 total iterations are reported for the full schedule) [S1].
3.  **View Selection (Active View Selection):**
    a. Evaluate the remaining candidate views $V_{cand} \setminus V_{sel}$.
    b. Use the SA-HashGS mechanism to estimate uncertainty/coverage for each candidate. This involves a self-augmented pre-filtering step that restricts the candidate set for Fisher uncertainty estimation [S1].
    c. Select the view with the highest uncertainty/lowest coverage metric.
4.  **View Addition:** Add the selected view to $V_{sel}$.
5.  **Iteration:** Repeat steps 2-4 until $V_{sel}$ reaches the maximum limit (typically 20 views) [S1].
6.  **Residual Learning Integration:** Throughout the training, the "ResGS" component applies residual supervision. This involves comparing the current reconstruction against a reference or previous state to guide optimization in sparse or ambiguous regions [S1]. The synergy between SA-HashGS and ResGS ensures that residual updates are based on stable view sequences [S1].

### Parameters
*   **Initial Views:** 4 uniformly distributed views [S1].
*   **View Addition Schedule:** Add 1 view every 100 epochs until reaching 20 training views [S1].
*   **Total Iterations:** 20,000 iterations per experiment [S1].
*   **Random Seed:** Fixed for reproducibility [S1].
*   **SfM Initialization:** Standard COLMAP SfM points [S1].

### Edge Cases and Pitfalls
*   **Uncertainty Quantification Error:** Purely dynamic selection without the SA-HashGS stabilization (†+ResGS) can lead to poor performance if uncertainty estimates are noisy, especially under large uncertainty quantification errors [S1].
*   **Hold-out Views:** Unlike some other active view selection methods (e.g., PRIMU-AVS), SA-ResGS does not require a hold-out view for training a regressor, relying instead on the single direction-dependent coverage feature map [S1].

## Subtopics

### [Active View Selection for 3D Gaussian Splatting](21848-active-view-selection-for-3d-gaussian-splatting.md)
A specific view selection strategy used within SA-ResGS that uses a direction-dependent coverage feature map with max aggregation and dropped opacity, omitting the gradient boosting regressor found in other methods like PRIMU-AVS to avoid needing a hold-out view.

## Operating characteristics
*   **Time Complexity:** The method requires multiple training phases (one for each view addition), significantly increasing total training time compared to single-view-set training. Each phase runs for a portion of the 20,000 iterations.
*   **Memory Complexity:** Standard 3DGS memory usage plus overhead for the SA-HashGS uncertainty estimation structures [S1].
*   **Performance:** SA-ResGS outperforms baselines like FisherRF, ACP, and random selection, as well as variants like †+ResGS (without SA-HashGS) and †+SA-HashGS (without ResGS) [S1]. The full SA-ResGS configuration shows synergy, providing the best reconstruction quality in sparse view scenarios [S1].
*   **Strengths:** Improves reconstruction quality with fewer views; stabilizes view selection; avoids need for hold-out views.
*   **Limitations:** Increased training time due to iterative view selection and re-optimization; sensitivity to the quality of the initial SfM reconstruction.

## Background
SA-ResGS was introduced in the paper "SA-ResGS: Self-augmented Residual 3D Gaussian Splatting for Next Best View Selection" [S1]. It builds upon the 3D Gaussian Splatting framework  and addresses challenges in active view selection for 3D reconstruction. The method integrates concepts from residual learning and self-augmented pre-filtering to enhance the robustness of the view selection process.

## Sources
[S1] SA-ResGS: Self-augmented Residual 3D Gaussian Splatting for Next Best View Selection, https://doi.org/10.1007/978-3-032-37013-6_17, fetched 2026-10-29
