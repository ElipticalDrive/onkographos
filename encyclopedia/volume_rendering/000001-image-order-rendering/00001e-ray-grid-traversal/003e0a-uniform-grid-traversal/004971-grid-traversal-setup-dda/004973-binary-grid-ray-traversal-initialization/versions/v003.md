- `step_index`: Integer step direction $(s_x, s_y, s_z)$ for index updates [S1, unofficial].
          **Data Structures:**
          - Vectors are 3-component floats [G].
          - `step_index` is a 3-component integer vector [G].
          **Steps:**
          1. **Calculate Start Index:**
             Compute the index of the next voxel to enter:
             $$ \text{start\_index} = \text{current\_index} + \text{index\_delta} $$
             [S1, unofficial]
          2. **Calculate Distance to Next Voxel Start (`tmax_xyz`):**
             Compute the distance from the ray origin to the start of the next voxel in each dimension [S1, unofficial]. The formula accounts for the grid origin (`aabb_min`), the voxel size, and the current ray position relative to `tmin`:
             $$ \text{tmax\_xyz} = \left( (\text{aabb\_min} + (\text{start\_index} \times \text{voxel\_size}) - \text{ray\_start}) \times \text{ray\_inv\_dir} \right) + \text{tmin} $$
             [S1, unofficial]
             *Note: Multiplication is component-wise.*
          3. **Initialize `tdist` (Distance to Next Boundary):**
             For each axis $c \in \{x, y, z\}$:
             - If $\text{ray\_inv\_dir}_c = 0$ (ray parallel to axis):
               $$ \text{tdist}_c = \text{tmax} $$
             - Else:
               $$ \text{tdist}_c = \text{tmax\_xyz}_c $$
             [S1, unofficial]
          4. **Initialize `step_float` (Direction Vector):**
             For each axis $c \in \{x, y, z\}$:
             - If $\text{ray\_inv\_dir}_c = 0$:
               $$ \text{step\_float}_c = 0.0 $$
             - Else if $\text{ray\_inv\_dir}_c > 0$ (equivalent to direction $> 0$):
               $$ \text{step\_float}_c = 1.0 $$
             - Else:
               $$ \text{step\_float}_c = -1.0 $$
             [S1, unofficial]
          5. **Convert to Integer Step:**
             $$ \text{step\_index} = \text{round}(\text{step\_float}) $$
             [S1, unofficial]
          6. **Calculate `delta_temp` (Voxel Crossing Distance):**
             Compute the distance traveled along the ray to cross one voxel in each dimension:
             $$ \text{delta\_temp} = \text{voxel\_size} \times \text{ray\_inv\_dir} \times \text{step\_float} $$
             [S1, unofficial]
             *Note: Multiplication is component-wise.*
          7. **Initialize `delta` (Step Size):**
             For each axis $c \in \{x, y, z\}$:
             - If $\text{ray\_inv\_dir}_c = 0$:
               $$ \text{delta}_c = \text{tmax} $$
             - Else:
               $$ \text{delta}_c = \text{delta\_temp}_c $$
             [S1, unofficial]
          **Pseudocode:**
          ```python
          def binary_grid_ray_traversal_initialization(
              ray_start, ray_inv_dir, voxel_size, aabb_min,
              current_index, index_delta, tmin, tmax
          ):
              # Step 1: Calculate start index
              start_index = current_index + index_delta
      
              # Step 2: Calculate tmax_xyz
              # Component-wise operations
              voxel_offset = aabb_min + (start_index * voxel_size)
              diff = voxel_offset - ray_start
              tmax_xyz = (diff * ray_inv_dir) + tmin
      
              # Step 3: Initialize tdist
              tdist = [0.0, 0.0, 0.0]
              for c in range(3):
                  if ray_inv_dir[c] == 0.0:
                      tdist[c] = tmax
                  else:
                      tdist[c] = tmax_xyz[c]
      
              # Step 4: Initialize step_float
              step_float = [0.0, 0.0, 0.0]
              for c in range(3):
                  if ray_inv_dir[c] == 0.0:
                      step_float[c] = 0.0
                  elif ray_inv_dir[c] > 0.0:
                      step_float[c] = 1.0
                  else:
                      step_float[c] = -1.0
      
              # Step 5: Convert to integer step
              step_index = [int(round(s)) for s in step_float]
      
              # Step 6: Calculate delta_temp
              delta_temp = [0.0, 0.0, 0.0]
              for c in range(3):
                  delta_temp[c] = voxel_size * ray_inv_dir[c] * step_float[c]
      
              # Step 7: Initialize delta
              delta = [0.0, 0.0, 0.0]
              for c in range(3):
                  if ray_inv_dir[c] == 0.0:
                      delta[c] = tmax
                  else:
                      delta[c] = delta_temp[c]
      
              return tdist, delta, step_index
          ```
          **Edge Cases and Pitfalls:**
          - **Parallel Rays:** If `ray_inv_dir` is 0 for any axis, the ray never crosses a boundary in that dimension [S1, unofficial]. The code clamps `tdist` and `delta` to `tmax` and sets steps to 0 to avoid division by zero and infinite loops [S1, unofficial].
          ## Subtopics
          ### [Binary Grid Ray Traversal (Single Voxel Step)](12504-binary-grid-ray-traversal-single-voxel-step.md)
          Advances the ray through a single voxel by updating the index and checking boundaries. It is the core iterative step following initialization. It differs from parent methods by performing the actual index update and boundary check rather than just setting up initial parameters. Choose it when traversing a binary occupancy grid where only presence/absence matters [G].
          ## Operating characteristics
          Not given in the sources.
          ## Background
          This method is part of the Digital Differential Analyzer (DDA) traversal technique for uniform grids [G]. It is implemented in the CNC (Compressed Neural Fields) project, which uses binary voxel mix encoding for efficient neural field representation [S1, unofficial].
          ## Sources
          [S1, unofficial] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
