# PCP from 2D Histograms

Also called: Novotny and Hauser PCP Generation

## Overview
This method separates the number of polylines rendered in a Parallel Coordinate Plot (PCP) from the number of raw data samples by generating the plot from 2D histograms of adjacent variable pairs [S1, unofficial]. It addresses the over-plotting and prohibitively expensive rendering costs associated with large datasets [S1, unofficial]. Unlike standard PCPs where each sample is a polyline, this approach aggregates data into histograms to define the visual representation, while also enabling outlier identification through histogram analysis [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset with $N$ samples and $D$ attributes. Adjacent attribute pairs $(x_k, x_{k+1})$ for $k=1 \dots D-1$ [S1, unofficial].
*   **Output:** A set of polylines (or quads) representing the PCP.

**Data Structures:**
*   **2D Histogram:** For each adjacent pair of attributes $(x, y)$, a 2D histogram $H$ is computed [S1, unofficial]. The histogram bins represent the joint frequency distribution of the two attributes [G].
    *   Let $B_x$ be the number of bins for attribute $x$.
    *   Let $B_y$ be the number of bins for attribute $y$.
    *   The histogram is a matrix $H \in \mathbb{Z}^{B_x \times B_y}$, where $H(i, j)$ stores the count of samples falling into bin $i$ of $x$ and bin $j$ of $y$ [G].

**Steps:**
1.  **Histogram Generation:** For each adjacent pair of attributes in the dataset, compute the 2D joint histogram [S1, unofficial].
2.  **Quad Mapping:** Identify all non-zero pixels $P(i, j)$ in the joint histogram of an attribute pair $(x, y)$ [Parent]. Each non-zero pixel corresponds to a bin intersection containing data samples [G].
3.  **Polylines Construction:** For each non-zero pixel $P(i, j)$ in the histogram of the pair $(x_k, x_{k+1})$, generate a line segment (or quad as per parent definition) connecting position $i$ on the PC axis for $x_k$ to position $j$ on the PC axis for $x_{k+1}$ [Parent].
    *   The position $i$ on axis $x_k$ corresponds to the center or representative value of bin $i$ [G].
    *   The position $j$ on axis $x_{k+1}$ corresponds to the center or representative value of bin $j$ [G].
4.  **Aggregation:** Repeat for all adjacent attribute pairs. The resulting set of segments forms the PCP [S1, unofficial].
5.  **Outlier Identification:** Analyze the histogram values $H(i, j)$ [S1, unofficial]. Samples or bin combinations with extremely low counts relative to the distribution may be identified as outliers [S1, unofficial].

**Parameters:**
*   **Bin Count:** The number of bins $B_x, B_y$ for each attribute. Not given in the sources.
*   **Normalization:** Whether histogram counts are normalized. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparsity:** If the histogram is very sparse, many bins may be zero, reducing the number of polylines significantly [G].
*   **Outlier Detection:** The definition of an outlier based on histogram analysis is not explicitly defined in the sources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_pcp_from_histograms(dataset, num_bins):
    # dataset: list of tuples, each tuple has D attribute values
    # num_bins: number of bins for each attribute
    polylines = []
    D = len(dataset[0])
    for k in range(D - 1):
        x_attr = [sample[k] for sample in dataset]
        y_attr = [sample[k+1] for sample in dataset]
        # Compute 2D histogram [G]
        H = compute_2d_histogram(x_attr, y_attr, num_bins)
        # Map non-zero bins to polylines [Parent]
        for i in range(num_bins):
            for j in range(num_bins):
                if H[i, j] > 0:
                    # Position corresponds to bin center [G]
                    x_pos = bin_center(i, num_bins, min(x_attr), max(x_attr))
                    y_pos = bin_center(j, num_bins, min(y_attr), max(y_attr))
                    polylines.append((x_pos, y_pos))
    return polylines

def compute_2d_histogram(x, y, bins):
    # Standard histogram computation [G]
    hist = [[0] * bins for _ in range(bins)]
    x_min, x_max = min(x), max(x)
    y_min, y_max = min(y), max(y)
    for xi, yi in zip(x, y):
        ix = int((xi - x_min) / (x_max - x_min) * bins)
        iy = int((yi - y_min) / (y_max - y_min) * bins)
        ix = min(ix, bins - 1)
        iy = min(iy, bins - 1)
        hist[ix][iy] += 1
    return hist

def bin_center(idx, bins, min_val, max_val):
    # Center of bin idx [G]
    bin_width = (max_val - min_val) / bins
    return min_val + (idx + 0.5) * bin_width
```

## Subtopics

### Visual Clustering on PCPs
A method to perform visual clustering on the PCPs by drawing curved edges instead of polylines and optimizing the arrangement of these curved edges [S1, unofficial]. Zhou et al. [102] propose this approach to enhance clustering visualization [S1, unofficial].

### Parallel Coordinate Plot Clustering
A method to address over-plotting in PCPs by clustering the data values and rendering only the representing polylines of each cluster [S1, unofficial]. Fua et al. [26] propose clustering data values and rendering only representative polylines [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Separates rendering cost from data size; reduces over-plotting; enables outlier identification [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** Large datasets where over-plotting is a concern and global trends are more important than individual samples [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
Novotny and Hauser [70] introduced the method of generating PCPs from 2D histograms to separate the number of polylines from the number of data samples [S1, unofficial]. This approach is part of the broader effort to address over-plotting in multivariate visualization [S1, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
