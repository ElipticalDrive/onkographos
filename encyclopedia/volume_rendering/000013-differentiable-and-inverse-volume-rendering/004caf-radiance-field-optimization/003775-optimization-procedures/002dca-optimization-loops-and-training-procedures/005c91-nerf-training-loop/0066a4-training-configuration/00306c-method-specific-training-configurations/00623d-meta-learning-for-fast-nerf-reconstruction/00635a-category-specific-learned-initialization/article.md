# Category-Specific Learned Initialization

Also called: Category-specific prior initialization, Federated averaging initialization, Learned Initialization, Meta-learned Initialization

## Overview
Category-Specific Learned Initialization uses meta-learning to capture shape and appearance priors from a large collection of objects within a specific category (e.g., cars on ShapeNet) to initialize the weights of instance-specific neural radiance field (NeRF) MLPs. It solves the problem of poor reconstruction quality and slow convergence when training on novel scenes with sparse input views, where standard random initialization fails to complete partial views. It sets itself apart from standard initialization by distilling category-level knowledge into the starting weights, allowing the model to generalize better and converge faster, particularly in data-sparse regimes.

## How it works
**Inputs and Outputs:**
- **Input:** A large dataset of 3D models from a specific category (e.g., ShapeNet cars), rendered into 2D images.
- **Output:** A set of optimized MLP weights $\theta_{init}$ that serve as the initialization for instance-specific "thing" MLPs during novel scene reconstruction.

**Data Structures:**
- $\theta$: The weights of the MLP (typically a "thing" MLP with 4 hidden layers and width 128, using positional encoding with $L=6$ frequencies for object coordinates).
- $D_k$: The dataset for client $k$, defined as a bundle of rays corresponding to every observed pixel of object instance $k$.
- $\mathcal{B}$: A set of local minibatches, where each batch contains $B$ rays.

**Algorithm (Federated Averaging for Meta-Learning):**
The method employs Federated Averaging (FedAvg) to meta-learn the initialization. This process is equivalent to REPTILE meta-learning [S1].

1.  **Server Initialization:** Initialize global weights $\theta_0$.
2.  **Outer Loop (Meta-Training):** For each outer epoch $t = 1, 2, \dots$:
    a.  **Client Updates (Parallel):** For each client $k \in \{1, \dots, K\}$ (where $K$ is the number of unique object instances):
        i.  Receive current weights $\theta_t$ from the server.
        ii. Split $D_k$ into batches of ray bundles of size $B$.
        iii. Perform local optimization for $E$ inner epochs:
            For each inner epoch $i = 1, \dots, E$:
            For each batch $b \in \mathcal{B}$:
            $$ \theta \leftarrow \theta - \eta \nabla_\theta \ell(\theta; b) $$
            *(Note: $\eta$ is the learning rate, $\ell$ is the loss function, typically reconstruction loss. The algorithm assumes simple SGD but can be adapted for other optimizers [S1].)*
        iv. Return the updated local weights $\theta_{t+1}^k$ to the server.
    b.  **Server Aggregation:** Average the weights from all clients:
        $$ \theta_{t+1} = \frac{1}{K} \sum_{k=1}^{K} \theta_{t+1}^k $$
3.  **Final Initialization:** The resulting $\theta_{final}$ is used as the initialization for the thing MLPs when training on a novel scene.

**Usage in Novel Scene Reconstruction:**
When reconstructing a novel scene containing instances of the trained category:
1.  Initialize the MLP for each object instance track $T$ with the meta-learned weights $\theta_{final}$.
2.  Train the inference-time scene representation network on the novel scene data, focusing only on the individual set of object instances present.

**Parameters:**
- $K$: Number of clients (unique object instances in the meta-training set).
- $B$: Local minibatch size (number of rays per batch).
- $E$: Number of inner (local) epochs per client.
- $\eta$: Learning rate.
- $\theta_t$: Network parameters after $t$ outer epochs.

**Edge Cases and Pitfalls:**
- The algorithm description assumes simple SGD; adaptation for other optimizers requires modifying the `CLIENT_UPDATE` step [S1].
- Standard Xavier (Glorot) initialization fails to complete partial views with sparse inputs, whereas this method succeeds [S1].

## Subtopics

### Federated Averaging for Category-Specific Initialization
A decentralized federated training algorithm used to realize category-specific learned initialization for thing MLPs by meta-learning from 2D rendered images of 3D models from ShapeNet. It allows for parallel processing of unique object instances. It is chosen when a large shape collection is available for a specific category (e.g., cars and vans) to distill priors into initialization weights. It is equivalent to REPTILE meta-learning [S1].

## Operating characteristics
- **Time Complexity:** The meta-training phase requires $T$ outer epochs, each involving $K$ parallel client updates of $E$ inner epochs. The total computational cost is proportional to $T \times K \times E$.
- **Memory Complexity:** Requires storing the MLP weights $\theta$ and the datasets $D_k$ for each client.
- **Performance:** On ShapeNet cars, the model reconstructs novel object instances well even with just a single image input, outperforming standard initialization which fails to complete partial views [S1]. With dense input views (at least 50 views over two full epochs), it provides faster convergence compared to standard Xavier initialization [S1].
- **Strengths:** Improves generalization and convergence; enables complete reconstruction from sparse views (1 or 3 images); efficient inference-time training as the network focuses only on present instances [S1].
- **Limitations:** Requires a large shape collection for the specific category; other object categories not covered by the meta-learned prior use biased initialization [S1].
- **When to use:** When reconstructing scenes with sparse input views for categories with available prior data (e.g., cars, vans).
- **When not to use:** For categories without a large shape collection or when dense views are available and standard initialization converges sufficiently (though learned initialization still offers faster convergence [S1]).

## Background
Originated in the Panoptic Neural Fields paper (2022) [S1]. The method leverages meta-learning [11] to capture category-specific shape and appearance priors. It uses the FedAvg algorithm [36], which is known to be equivalent to REPTILE [42] meta-learning [25]. The approach was evaluated on ShapeNet [5] cars (provided by [56]) and applied to cars and vans in dynamic KITTI scenes. It addresses the issue where dense content in "stuff" can suppress gradients from distant objects, making prior knowledge via initialization robust [S1].

## Sources
[S1] Panoptic Neural Fields: A Semantic Object-Aware Neural Scene Representation, https://doi.org/10.1109/cvpr52688.2022.01253, fetched 2026-10-26
