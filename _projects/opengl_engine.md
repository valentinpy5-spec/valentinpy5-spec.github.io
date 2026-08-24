---
layout: page
title: "Reusable OpenGL Rendering Engine"
description: A small C++/OpenGL rendering engine built to learn GPU programming and graphics architecture from the ground up, now reused across several fluid-simulation projects.
img: assets/img/projects/opengl_engine/demo_sphere.png
importance: 5
category: Personnal
github: https://github.com/valentinpy5-spec/OpenGLEngine
---

<div class="mb-3">
  <a href="https://github.com/valentinpy5-spec/OpenGLEngine" target="_blank" rel="noopener noreferrer" class="btn btn-outline-dark">
    <i class="fa-brands fa-github fa-lg"></i>&nbsp; View on GitHub
  </a>
</div>

## What this project is

This is where I actually learned OpenGL and GPU programming, not by following a single tutorial end to end, but by building a small rendering engine from scratch and reusing it across several fluid-simulation projects until the pieces that belonged together became obvious. Earlier projects on this page ([SPH dam-break]({{ '/projects/sph_dam_break/' | relative_url }}) and its PCISPH sibling) each needed the same things: a window, a camera, shaders, mesh buffers, an ImGui overlay, none of which has anything to do with the physics itself. Pulling that shared layer out into its own project, and keeping it strictly independent of any specific simulation, was what forced me to actually understand what a GPU rendering pipeline is doing at each stage rather than copy-pasting boilerplate.

The code is on GitHub at [OpenGLEngine](https://github.com/valentinpy5-spec/OpenGLEngine). The screenshots below are from the engine's own standalone demo, no physics involved, just the rendering pipeline itself.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/opengl_engine/demo_sphere.png" title="Engine demo, sphere primitive" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/opengl_engine/demo_cube.png" title="Engine demo, cube primitive" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
  The standalone demo: an orbit camera, a simply lit primitive on a floor plane, and an ImGui panel to switch between shapes live, sphere on the left, cube on the right.
</div>

## What the engine can do

At its core, the engine handles everything needed to get a 3D scene on screen and let it be explored interactively: an orbit or first-person camera, mesh creation (basic primitives like spheres, planes and cubes, or loading a shape from an OBJ file), shader loading and compilation, and simple diffuse lighting. On top of that, it supports drawing many objects efficiently, including true GPU instancing for cases like thousands of particles rendered in a single draw call, which is exactly what its SPH counterpart needs. It also comes with a lightweight Dear ImGui integration, so any part of a scene can expose a live control panel without extra setup.

## Why building this mattered

Writing this engine is what made GPU architecture concrete for me: understanding why data needs to live in GPU buffers before it can be drawn, what a vertex and fragment shader are actually each responsible for, why instancing exists and when it matters versus issuing one draw call per object, and how a camera's view and projection matrices actually map 3D points onto a 2D screen. None of that is obvious from reading about OpenGL; it became clear by hitting the actual constraints of the API while trying to keep the code clean enough to reuse. Keeping the engine deliberately free of any simulation-specific code was also its own lesson in software architecture, forcing a real separation between "what renders a scene" and "what the scene represents physically".

## What's next

The most direct next step is extending the OBJ loader to support textures and materials, since it currently only reads geometry. Beyond that, screen-to-world ray casting is already available on the camera but not yet wired into the demo, which would be a natural step toward interactive object picking.

## References

- de Vries, J., *Learn OpenGL: Graphics Programming*, learnopengl.com: primary reference for shader compilation, mesh setup, and camera handling used throughout this engine.
- Cherno (Yan Chernikov), *The Cherno* YouTube series on OpenGL and game engine architecture: reference for structuring the engine as independent, composable classes rather than one monolithic renderer.
- Dear ImGui (Cornut, O. et al.), vendored directly in the project: the immediate-mode GUI library used for the live mesh-selection panel and the engine's general panel system.
