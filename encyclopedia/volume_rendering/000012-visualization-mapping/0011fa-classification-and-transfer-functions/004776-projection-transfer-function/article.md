# Projection Transfer Function
Also called: Summed Projection

## Overview
The Projection Transfer Function is a classification method that generates a 2D image by summing scalar activity values along rays passing through a 3D dataset [S1, unofficial][S2]. It serves as a comparative baseline to maximum intensity projection (MIP) by aggregating total signal rather than peak signal [S1, unofficial]. Unlike methods that modulate opacity based on local density gradients or probabilistic models, this method applies a uniform integration weight, effectively treating the volume as a transparent medium where all samples contribute linearly to the final pixel value [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset containing scalar field values (e.g., activity, density) and a transfer function definition specifying integration bounds and field IDs.
*   **Output:** A 2D image where each pixel value represents the sum of field values along the corresponding ray.

**Data Structures:**
The method utilizes a `MultiVariateTransferFunction` container which holds a list of `TransferFunction` tables. Each table defines the integration behavior for a specific field.
*   `x_bounds`: A tuple `(min_val, max_val)` defining the truncation range for input values. Values outside this range are clipped [S2]. Default: `(-1e+60, 1e+60)`.
*   `n_fields`: An integer specifying the number of distinct fields to project [S2]. Default: `1`.
*   `field_id`: An integer identifier linking a `TransferFunction` table to a specific data field in the volume [S2].
*   `channels`: A list of integers (0-5) linking the table's output to image channels (Red, Green, Blue, Red Absorption, Green Absorption, Blue Absorption) [S2].

**Steps:**
1.  Initialize a `MultiVariateTransferFunction` object.
2.  Define a `TransferFunction` table with a value range (`x_bounds`) and a shape (e.g., Gaussian). For a simple projection, the table typically outputs a constant weight (e.g., 1.0) within the active range to sum values [S2].
3.  Add the transfer function table to the multi-variate container using `add_field_table`, associating it with a specific `field_id` [S2].
4.  Link the table to one or more output channels using `link_channels` [S2].
5.  During ray casting, for each sample point $\mathbf{x}$ along a ray:
    a. Retrieve the scalar value $v$ from the specified field.
    b. Clamp $v$ to `x_bounds`.
    c. Evaluate the transfer function to get a weight $w$. For a pure projection, $w$ is typically 1.0 for valid values [S2].
    d. Accumulate the weighted value to the ray's integral: $I = \sum w \cdot v$.
6.  After ray casting completes, scale the accumulated integral $I$ by the total path length (width) through which rays were cast to normalize the density [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ProjectionTransferFunction:
    def __init__(self, x_bounds=(-1e60, 1e60), n_fields=1):
        self.x_bounds = x_bounds
        self.n_fields = n_fields
        self.field_tables = []
        self.channel_links = {}

    def add_field_table(self, table, field_id, weight_field_id=-1, weight_table_id=-1):
        """
        Adds a transfer function table for a specific field.
        For simple projection, the table should return a weight of 1.0.
        """
        self.field_tables.append({
            'table': table,
            'field_id': field_id,
            'weight_field_id': weight_field_id,
            'weight_table_id': weight_table_id
        })

    def link_channels(self, table_id, channels):
        """
        Links a field table to output channels.
        """
        self.channel_links[table_id] = channels

    def evaluate(self, value, table_id):
        """
        Evaluates the transfer function for a given value.
        Returns the weight to be added to the sum.
        """
        # Clamp value to bounds
        min_val, max_val = self.x_bounds
        if value < min_val:
            value = min_val
        elif value > max_val:
            value = max_val
        
        # Retrieve table
        table = self.field_tables[table_id]['table']
        
        # Evaluate table (e.g., Gaussian or linear)
        # For simple projection, this should ideally return 1.0
        weight = table.evaluate(value)
        return weight

    def cast_ray(self, ray_samples, field_data):
        """
        Casts a single ray and returns the summed projection.
        """
        total_sum = 0.0
        # Iterate through samples along the ray
        for sample in ray_samples:
            # Get value from field_data at sample position
            # This depends on volume interpolation implementation
            val = field_data.get_value(sample.position, field_id=0)
            
            # Evaluate transfer function
            weight = self.evaluate(val, 0)
            
            # Accumulate
            total_sum += weight * val
            
        return total_sum
```

**Parameters:**
*   `x_bounds`: Tuple of floats. Range for truncation. Default: `(-1e+60, 1e+60)` [S2].
*   `n_fields`: Integer. Number of fields to project. Default: `1` [S2].

**Edge Cases and Pitfalls:**
*   **Scaling:** The raw sum depends on the number of samples, which is determined by the ray length and step size. The final image must be scaled by the total width through which rays were cast, a value inaccessible to the transfer function itself [S2].
*   **Logging:** When using this function, explicitly disable logging of fields to avoid performance overhead or incorrect state tracking [S2].
*   **Unlinked Tables:** Not all added tables must be linked to channels; unlinked tables contribute values but do not affect the final image unless linked [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays (pixels) and $S$ is the number of samples per ray. The evaluation is $O(1)$ per sample.
*   **Memory Complexity:** $O(1)$ for the transfer function structure itself, plus $O(V)$ for the volume data.
*   **Strengths:** Computationally simple; highlights regions of high total integrated activity rather than just local peaks [S1, unofficial]. Useful for comparing against MIP [S1, unofficial].
*   **Limitations:** Lacks depth cueing; foreground and background structures overlap additively, potentially obscuring details [S1, unofficial]. Requires post-processing scaling for quantitative accuracy [S2].
*   **When to use:** When the total integrated signal is more important than the maximum intensity, or as a baseline comparison for MIP [S1, unofficial].

## Background
The summed projection method was introduced as a comparative technique to Maximum Intensity Projection (MIP) in the context of 3D display of SPECT images [S1, unofficial]. It was implemented in FORTRAN on a microVAX-II minicomputer for cine display rotation [S1, unofficial]. The method sums activity values along columns (rays) to form a 2D image, providing a different visual emphasis than MIP, which selects the maximum value [S1, unofficial]. The `ProjectionTransferFunction` class in the yt project implements this logic for volume rendering [S2].

## Sources
[S1] "Volume rendering in three-dimensional display of SPECT images", http://jnm.snmjournals.org/cgi/pmidlookup?view=long&pmid=2384811, fetched 2026-10-29 (unofficial)
[S2] ColorTransferFunction yt.visualization.volume_rendering.transfer_functions.ColorTransferFunction, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.transfer_functions.html, fetched 2026-10-29
