# csRaymarching (Terminal Raymarcher)

A real-time CPU raymarcher that renders signed-distance-field scenes directly inside a terminal using ANSI / VT escape sequences and Unicode block characters.

It is intentionally "overkill ASCII art": multi-core raymarching, shading, and a smooth free-fly camera, all presented as chunky terminal pixels.

![Screen1](/screenshots/screenshot%20(3).png)
![Screen1](/screenshots/screenshot%20(4).png)

## Features

- CPU raymarching using signed distance fields (SDF)
- Parallel rendering using Parallel.For (scales well with core count)
- Half-block rendering mode for effectively double vertical resolution
- Basic shading using surface normals and optional fog
- Dirty-row terminal flushing for fast and stable redraws
- Smooth camera controls with adjustable movement speed

## Controls

- W / A / S / D - Move (x)
- E / Q - Move up / down (z)
- Arrow keys - Look (yaw / pitch)
- Left Shift - Sprint
- 1 / 2 / 3 / 4 - Set movement speed
- Space - Pause
- H - Toggle half-block rendering
- F - Toggle fog
- G - Toggle gamma colorization
- C - Cycle color theme
- R - Reset scene
- I - Toggle debug / info overlay
- F1 / F2 / F3 / F4 - Target FPS (30 / 60 [Default] / 120 / 144)
- Esc - Quit
- Num 0 - 5 - Scenes

## Requirements

- .NET 10
- A terminal that supports ANSI / VT escape sequences and Unicode block glyphs

Recommended terminal:
- Windows Terminal on Windows 11 (very stable under heavy redraw)

This project is intentionally CPU-heavy. Performance depends strongly on core count,
cache size, and sustained clock speed. Larger window will be harder to run smoothly. 
@ 1920x1080 terminal size, I get about 30-40 FPS with a R9 9900X3D.

![Screen1](/screenshots/screenshot%20(2).png)

## Project Layout (High Level)

- Program.cs  
  Main loop, frame pacing, input polling, resize handling, UI overlay

- RaymarchRenderer.cs  
  Core raymarch and shading pipeline (parallel rendering, half / full block modes)

- Scene.cs / DistanceFields.cs  
  Signed distance field scene description and distance functions

- Camera3D.cs / Ray.cs  
  Free-fly camera model and ray generation

- Shading.cs  
  Color, lighting, fog, and background shading

- ConsoleCanvas.cs / WindowsConsole.cs / ConsoleInteraction.cs  
  Fast terminal output, VT handling, and buffered rendering

- Settings.cs / Theme.cs  
  Runtime settings and visual themes

## Tweaking

Most tuning parameters live in Settings.cs (initialized in Program.cs), including:

- MaxRaymarchSteps
- MaxRenderDistance
- FieldOfView
- MoveSpeed
- EnableFog / FogDensity
- UseHalfBlocks
- TargetFps
- MapColorsWithGamma

## License

MIT License — see LICENSE file.

(c) 2026 NiTROZ
