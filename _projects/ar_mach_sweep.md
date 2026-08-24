---
layout: page
title: AR/VR Compressible Flow Visualization
description: Real-time augmented-reality visualization of a transonic Mach sweep around a wing, built in Unreal Engine 5 from CFD field data (pressure coefficient, Mach isosurfaces).
img: assets/img/projects/ar_mach_sweep/poster.png
importance: 1
category: Internships
---

## What this project is

An augmented-reality demo built during my current R&D internship at ISAE-SUPAERO (DAEP), where the goal is to make CFD results explorable and pedagogical rather than static. The video below shows the live result: a wing rendered in AR (camera passthrough, real classroom in the background) with a pressure-coefficient contour on its surface, a Mach slider driving the flow regime shown, and toggle buttons to bring up isoMach1 surfaces and Cp overlays interactively.

<div class="row justify-content-sm-center">
  <div class="col-sm-8 mt-3 mt-md-0">
    {% assign poster_url = "assets/img/projects/ar_mach_sweep/poster.png" | relative_url %}
    {% include video.liquid path="assets/video/projects/ar_mach_sweep_demo.mp4" class="img-fluid rounded z-depth-1" controls=true poster=poster_url %}
  </div>
</div>
<div class="caption">
  AR demo: pressure-coefficient field on the wing surface, live Mach slider (Ma = 0.711 shown), and toggle buttons for isoMach1 / Mach-slice / Cp overlays, composited on a real environment through the device camera.
</div>

## Why AR/VR for CFD post-processing

A Mach sweep is inherently a sequence of CFD solutions, one per freestream Mach number, each showing a different flow regime: attached subsonic flow at low Mach, a growing supersonic pocket closed by a shock as the critical Mach number is exceeded, and eventually a lambda-shock structure as the supersonic zone strengthens. Comparing these regimes on a 2D screen means flipping between static contour plots. Placing the same data in AR lets the wing be walked around and the flow regime swept continuously with a slider, which is closer to how the underlying physics, a family of solutions parameterized by Mach number, is actually structured.

## Data pipeline: from the CFD solver to Unreal Engine 5

Getting a CFD solution into an interactive AR scene means crossing four different tools, each with its own data format, and the pipeline is the actual engineering work behind this project:

1. **STAR-CCM+**: the flow is solved (SST k-ω turbulence closure, matched against a validated reference operating point) and the solution is exported in the `.case` format, the standard EnSight case format for time-series/parametric CFD results.
2. **ParaView**: the `.case` files are loaded and the fields of interest (surface pressure coefficient, local Mach number, the sonic isosurface) are extracted and re-exported as `.vtp` (VTK PolyData), a lightweight surface-mesh-plus-scalar-fields format, one per Mach case.
3. **Blender**: the `.vtp` surfaces are brought in and reworked with geometry nodes into clean, UE5-ready assets, exported as `.fbx`, the geometry nodes step is what turns raw simulation output into a mesh with the right topology and vertex attributes for real-time rendering.
4. **Unreal Engine 5**: the `.fbx` assets are imported and driven by Blueprints, which handle switching between Mach cases, animating the transition, and wiring the Mach slider and overlay toggles to the right pre-baked asset. The real-time interactivity in the demo is this Blueprint layer swapping in pre-computed data, not a live flow solver running inside the engine.

## Automating the pipeline

My actual internship work was making this four-tool pipeline usable by someone who isn't a pipeline developer: automating the STAR-CCM+ → ParaView → Blender → UE5 chain end to end and keeping each step as simple as possible, since the intended users are teaching staff producing new visualizations for their own courses, not necessarily comfortable scripting each tool by hand. Along the way this pipeline has already produced several videos used internally, including the one on this page.

## What the flow physics shows across the sweep

The Mach slider sweeps through the regimes that motivate the whole exercise: below the critical Mach number the flow stays subsonic everywhere and Cp varies smoothly over the wing; past the critical Mach number a local pocket of supersonic flow appears on the upper surface (visible as the isoMach1 surface breaking away from the wing skin) and is closed by a shock, which shows up as a sharp discontinuity in the surface Cp coloring; at higher sweep Mach numbers this can develop into a double, lambda-shaped shock structure rather than a single normal shock. Being able to scrub the Mach slider and watch the isoMach1 surface grow and the shock strengthen makes that progression, usually described only in words or a handful of static Cp plots, directly visible.

## Status and what's next

The AR interaction shown here (Mach slider, surface Cp/Mach coloring, isoMach1 and shock overlays) is working and is what the video demonstrates. The next piece of work is extending the automated pipeline to a full range of Mach cases rather than a hand-picked validated case, plus adding the interactive Cp(x/c) graph and clickable pedagogical annotations planned alongside the visual overlays.
