# Parallel Binary-Swap Compositing
## Overview
Parallel Binary-Swap Compositing is a method for combining independently rendered image tiles from multiple processors in a massively parallel environment [S1]. It solves the synchronization and communication bottlenecks of naive parallel compositing by reducing the data volume at each step, ensuring all processors remain busy with useful work [S1]. Unlike approaches where communication dominates or processors idle, this method maintains full processor utilization by pairing processors in a binary-swap fashion [S1].
## How it works
**Inputs and Outputs:**
- Input: $N$ independent rendered image tiles, one per processor [S1].
- Output: A single composite image [S1].

**Data Structures:**
- The method utilizes a k-D tree structure to manage the compositing tree and data distribution [S1].

**Steps:**
1. **Initialization:** Each processor holds its own independently rendered tile [S1].
2. **Pairing:** Processors pair based on a binary-swap pattern [S1]. Processor $i$ pairs with processor $i \oplus 2^k$, where $k$ is the current step index starting from $0$ [G].
3. **Compositing and Reduction:**
   - The receiver composites its current image with the image received from the sender [G].
   - The image size for sending and compositing decreases at each step [S1].
   - The sender transmits its tile to the receiver [G].
   - The sender becomes inactive for this step [G].
4. **Iteration:** The number of active processors is halved ($N \leftarrow N/2$) [G]. The step index increments. The process repeats until only one processor remains [S1].
5. **Termination:** The single remaining processor holds the final composite image [S1].

**Parameters:**
- $N$: Number of processors.
- Initial tile dimensions.

**Edge Cases and Pitfalls:**
- The sources do not specify handling for non-power-of-two processor counts or precision limits during accumulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Each processor executes this loop.
# local_image: initial rendered tile for this processor
# proc_id: unique integer ID in [0, N-1]
# N: total number of processors

current_image = local_image
active_count = N
step = 0

while active_count > 1:
    swap_dist = 1 << step
    partner = proc_id ^ swap_dist
    
    # Check if partner exists in the current active set.
    # active_count holds the number of active processors at the start of this step.
    if partner < active_count:
        if proc_id < partner:
            # Receiver: composite with partner's image
            partner_image = receive_data(partner)
            current_image = alpha_composite(current_image, partner_image)
        else:
            # Sender: transmit current image
            transmit_data(partner, current_image)

    active_count >>= 1
    step += 1
    # The image size for sending and compositing decreases at each step [S1]

final_image = current_image

# Helper definitions (standard operations [G]):
# receive_data(dest): blocks until a message arrives from dest, returns the payload.
# transmit_data(dest, payload): sends payload to dest.
# alpha_composite(src, dst): performs front-to-back alpha compositing: dst = src + (1 - src_alpha) * dst.
```
## Subtopics

### [2-3 Swap Compositing](12223-2-3-swap-compositing.md)
A parallel image compositing technique designed to combine independently rendered image tiles from multiple processors in a distributed-memory environment. It solves the limitation of standard Parallel Binary-Swap Compositing, which requires the number of processors $N$ to be a power of two, by allowing any integer number of processors to participate. Unlike its parent method, it does not enforce a strict binary tree structure, instead grouping processors into pairs or triplets to facilitate data exchange. Choose this variant when $N$ is not a power of two or when a distributed-memory architecture requires flexible grouping.

## Operating characteristics
- **Time Complexity:** The method reduces active processors and image size at each step, completing in $\log_2 N$ iterations [S1, G].
- **Memory Complexity:** Each processor stores its local image tile [G]. Total memory scales with $N$ and the initial tile size [G].
- **Performance:** Efficient because it decreases the image size for sending and compositing at each step, reducing communication overhead [S1]. It keeps all processors busy, avoiding idle time [S1].
- **Strengths:** Simple implementation using a k-D tree structure [S1]. High efficiency in massively parallel environments [S1].
- **Limitations:** The sources do not specify constraints on $N$ or precision limits [S1].
## Background
The method was developed for parallel volume ray-tracing algorithms [S1]. It was implemented on the CM-5 and a network of scientific workstations [S1]. The tests used 32 high-performance workstations, including IBM RS/6000-550, HP9000/730, and Sun Sparc-10/30 machines [S1]. The algorithm divides both computation and memory load across all processors, allowing it to render data sets too large for a single uniprocessor [S1].
## Sources
[S1] A data distributed, parallel algorithm for ray-traced volume ..., https://www.cs.ucdavis.edu/~ma/papers/PRS93.pdf, fetched 2026-10-27
