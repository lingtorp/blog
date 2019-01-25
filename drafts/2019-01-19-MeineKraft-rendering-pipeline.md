---
layout: post
title: "MeineKraft Rendering Pipeline"
last_modified_at: 2019-01-18
tags:
    - MeineKraft
---

I realized today that I have never actually described how the rendering pipeline of MeineKraft works or even looks like. Today I will describe how it currently works and what some of my short terms goals are and also some of the long term ones as well. Los geht's!

# Current Pipeline
1. View frustum culling: Performs view frustum culling with a compute shader. This generates all of the draw commands for the next pass.
2. Geometry pass: Deals with the actual geometry which fills all of the global buffers: depth, world position, normals, albedo, shading model id, etc.
3. Lightning pass: Performs lightning calculations on the whole screen. The type of lightning applied depends on the shading model id that is passed along the pipeline.
4. Mandatory blit pass that copies the previous pass into the default framebuffer (due to restriction on rendering to the default framebuffer??).


# References
