# Render State Machine


## Overview
The Render State Machine is a threaded control mechanism that manages rendering resolution and interruption based on camera movement and user actions to maintain a target frame rate [S1, unofficial]. It operates as a finite state machine that transitions between resolution tiers ('high', 'low_move', 'low_static') in response to specific actions ('move', 'static', 'rerender', 'step') [S1, unofficial]. It distinguishes itself from sibling methods by explicitly handling interruption via Python trace functions to discard incomplete outputs, ensuring viewer consistency [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Camera state (position, orientation), user actions (move, static, rerender, step), current render state, target FPS, maximum resolution, visibility ray budget.
- **Outputs**: Rendered image (uint8), updated render state, interrupt flags.

**Data Structures**
- **Transitions Table**: A dictionary mapping current states to action-based next states.
  - Type: `Dict[RenderStates, Dict[RenderActions, RenderStates]]`
  - Layout: Nested dictionaries. Keys are strings from `RenderStates` and `RenderActions`.
  - Default: All transitions map to the current state (self-transition).
  - Overrides:
    - `low_move` + `static` -> `low_static`
    - `low_static` + `static` -> `high`
    - `low_static` + `step` -> `high`
    - `low_static` + `move` -> `low_move`
    - `high` + `move` -> `low_move`
    - `high` + `rerender` -> `low_static`
- **RenderStates**: Literal type with values `'low_move'`, `'low_static'`, `'high'`.
- **RenderActions**: Literal type with values `'rerender'`, `'move'`, `'static'`, `'step'`.

**Parameters**
- `target_fps`: Target frames per second for low-resolution rendering. Default: 24 [S1, unofficial].
- `vis_rays_per_sec`: Visibility rays per second budget. Default: 100,000 if not initialized or not in global buffer [S1, unofficial].
- `max_res`: Maximum resolution height from control panel.

**Steps**
1. **Initialize Transitions**: Create an empty dictionary for each state. Set all actions to self-transition. Override specific transitions as defined in the Transitions Table [S1, unofficial].
2. **Interrupt Logic**: If the current state is `'high'` and the next action is `'move'` or `'rerender'`, set `interrupt_render_flag` to `True` and trigger `render_trigger` [S1, unofficial].
3. **Interrupt Check**: In a Python trace function, if the event is `'line'` and `interrupt_render_flag` is `True`, reset the flag and raise `IOChangeException` [S1, unofficial].
4. **Image Resolution Calculation**:
   - If `aspect_ratio` is 0, set it to 0.001 [S1, unofficial].
   - Get `max_res` from the control panel [S1, unofficial].
   - If state is `'high'`:
     - Set `height` to `max_res`.
     - Set `width` to `height * aspect_ratio`, capped at `max_res` [S1, unofficial].
   - If state is `'low_move'` or `'low_static'`:
     - Calculate `num_vis_rays` as `vis_rays_per_sec / target_fps` [S1, unofficial].
     - Calculate `height` as `sqrt(num_vis_rays / aspect_ratio)`, rounded to the nearest 10, clamped between 30 and `max_res` [S1, unofficial].
     - Set `width` to `height * aspect_ratio`, capped at `max_res` [S1, unofficial].
5. **Rendering**:
   - Update render AABB [S1, unofficial].
   - Calculate image resolution using the logic above [S1, unofficial].
   - Get camera [S1, unofficial].
   - Generate rays [S1, unofficial].
   - If model is `GaussianSplattingModel` or `DepthGSModel`, use `get_outputs_for_camera`; else use `get_outputs_for_camera_ray_bundle` [S1, unofficial].
   - If `crop_viewport` is true, set background color and use `background_color_override_context`; otherwise, no background override [S1, unofficial].
   - Evaluate model, render with `torch.no_grad()`, then set model to train mode [S1, unofficial].
6. **Output Sending**:
   - Get output keys. If changed, send output options message and update control panel [S1, unofficial].
   - Get selected output render [S1, unofficial].
   - Apply colormap [S1, unofficial].
   - If split mode, get split output, apply colormap, calculate split index, concatenate selected and split outputs with a divider line at `split_index` [S1, unofficial].
   - Convert to uint8 [S1, unofficial].
   - Send as background image [S1, unofficial].

**Pitfalls**
- If `IOChangeException` is raised during render, the output is discarded and not sent to the viewer [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np
import torch

# Types
RenderStates = ['low_move', 'low_static', 'high']
RenderActions = ['rerender', 'move', 'static', 'step']

class RenderStateMachine:
    def __init__(self, target_fps=24, vis_rays_per_sec=100000, max_res=1080):
        self.target_fps = target_fps
        self.vis_rays_per_sec = vis_rays_per_sec
        self.max_res = max_res
        self.current_state = 'low_move'
        self.interrupt_render_flag = False
        
        # Initialize transitions: all self-transitions
        self.transitions = {
            state: {action: state for action in RenderActions}
            for state in RenderStates
        }
        
        # Override specific transitions
        self.transitions['low_move']['static'] = 'low_static'
        self.transitions['low_static']['static'] = 'high'
        self.transitions['low_static']['step'] = 'high'
        self.transitions['low_static']['move'] = 'low_move'
        self.transitions['high']['move'] = 'low_move'
        self.transitions['high']['rerender'] = 'low_static'

    def set_action(self, action):
        """Triggered by user action or camera movement."""
        next_state = self.transitions[self.current_state][action]
        
        # Interrupt logic
        if self.current_state == 'high' and action in ['move', 'rerender']:
            self.interrupt_render_flag = True
            # Trigger render_trigger (implementation detail)
        
        self.current_state = next_state

    def calculate_resolution(self, aspect_ratio):
        if aspect_ratio == 0:
            aspect_ratio = 0.001
        
        if self.current_state == 'high':
            height = self.max_res
            width = min(height * aspect_ratio, self.max_res)
        else: # low_move or low_static
            num_vis_rays = self.vis_rays_per_sec / self.target_fps
            height = math.sqrt(num_vis_rays / aspect_ratio)
            height = round(height / 10) * 10
            height = max(30, min(height, self.max_res))
            width = min(height * aspect_ratio, self.max_res)
        
        return int(width), int(height)

    def render(self, model, camera, crop_viewport=False):
        # Interrupt check in trace function would raise IOChangeException here if flag is set
        if self.interrupt_render_flag:
            # This check is conceptual; actual check is in Python trace function
            pass 
        
        width, height = self.calculate_resolution(camera.aspect_ratio)
        
        # Update render AABB
        # ...
        
        # Generate rays
        # ...
        
        # Get outputs
        if isinstance(model, GaussianSplattingModel) or isinstance(model, DepthGSModel):
            outputs = model.get_outputs_for_camera(camera)
        else:
            ray_bundle = generate_rays(camera, width, height)
            outputs = model.get_outputs_for_camera_ray_bundle(ray_bundle)
        
        # Background override
        if crop_viewport:
            with background_color_override_context(outputs.background_color):
                rendered = evaluate_model(model, outputs)
        else:
            rendered = evaluate_model(model, outputs)
        
        # Post-processing
        with torch.no_grad():
            final_image = apply_colormap(rendered)
            if split_mode:
                split_output = get_split_output()
                split_colormap = apply_colormap(split_output)
                split_index = calculate_split_index()
                final_image = concatenate_with_divider(final_image, split_colormap, split_index)
            
            final_image_uint8 = convert_to_uint8(final_image)
        
        # Set model to train mode
        model.train()
        
        return final_image_uint8

    def send_output(self, image):
        # Get output keys
        # If changed, send output options message and update control panel
        # Get selected output render
        # Apply colormap
        # Convert to uint8
        # Send as background image
        pass
```

## Operating characteristics
- **Time Complexity**: Resolution calculation is $O(1)$. Rendering complexity depends on the model (e.g., Gaussian Splatting is generally $O(N)$ where $N$ is the number of splats).
- **Memory Complexity**: $O(1)$ for state machine storage. Rendering memory depends on the model and resolution.
- **Measured Performance**: Maintains target FPS (default 24) during low-resolution states by limiting visibility rays [S1, unofficial].
- **Strengths**: Responsive to user interaction; prevents stale high-res renders during movement; ensures consistent viewer updates.
- **Limitations**: Requires Python trace function for interruption, which may have performance overhead; discards incomplete renders, potentially causing flicker if not handled gracefully.
- **When to use**: Interactive volume rendering or Gaussian Splatting where camera movement is frequent and high resolution is not always necessary.
- **When not to use**: Batch rendering or scenarios where uninterrupted high-quality frames are critical.

## Background
The Render State Machine is part of the Gaussian-Splatting-Toolkit, an unofficial toolkit for Gaussian Splatting [S1, unofficial]. It was developed to address the need for responsive interactive rendering in 3D Gaussian Splatting, where high-resolution rendering can be computationally expensive and slow down interaction [S1, unofficial].

## Sources
[S1] Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, https://github.com/Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, fetched 2026-10-27 (unofficial)
