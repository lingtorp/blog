---
layout: post
title: "MeineKraft Rendering Pipeline"
last_modified_at: 2019-01-18
tags:
    - MeineKraft
---

I realized that I have never actually described how the rendering pipeline of MeineKraft works or even looks like. Here I will describe how it currently works and what some of the short terms goals for the project are and also some of the long term ones as well. Los geht's!

# Current Pipeline

### View frustum culling pass
Performs view frustum culling on all objects in a compute shader. This generates the draw commands for the next pass.

```glsl
// Sphere
struct BoundingVolume {
    vec3 center; 
    float radius;
};

uniform readonly buffer {
    BoundingVolume objects[];
};
```

Each object passed to the culling pass is in the form of a BoundingVolume object a.k.a a sphere. It would probably more efficient with AABBs but this works fine for now.

```cpp
/// Struct from OpenGL (glMultiDrawElementsIndirect)
struct DrawElementsIndirectCommand {
  uint32_t count = 0;         // # elements (i.e indices)
  uint32_t instanceCount = 0; // # instances (kind of drawcalls)
  uint32_t firstIndex = 0;    // index of the first element in the EBO
  uint32_t baseVertex = 0;    // indices[i] + baseVertex
  uint32_t baseInstance = 0;  // [gl_InstanceID / divisor] + baseInstance
  uint32_t padding0 = 0;      // Padding due to GLSL layout std140 16B alignment rule
  uint32_t padding1 = 0;
  uint32_t padding2 = 0;
};
```

1 draw command for each GraphicsBatch is created. A GraphicsBatch is a unique geometry + shader configuration bundle which is rendered via instanced drawcalls. The reasoning behind the decision to have this 1:1 mapping between draw commands and GraphicBatches is that the memory overhead of having 1 draw command per object was simply too large. It would probably not scale well and it is a bit cleaner to have the 1:1 mapping with the GraphicBatch as it is the one who manages all the related buffers for a set of geometry.

If the object passes the visibility test with the frustum the current drawcalls instance variable is incremented atomically.

```glsl
// Command buffer backed by Shader Storage Object Buffer (SSBO)
layout(std140, binding = 0) buffer DrawCommandsBlock {
    DrawCommand draw_commands[];
};
```

Each GraphicsBatch has one of these as draw commands buffers.

```glsl
const uint INSTANCE_IDX = atomicAdd(draw_commands[DRAW_CMD_IDX].instanceCount, 1);
index_buffer[INSTANCE_IDX] = idx; // Shader data index (idx) for objects
```

Here the *DRAW_CMD_IDX* is the a triple buffering index of the GraphicsBatch. This is in order to minimize the synchronization issues by having a single buffer buffer but three times as large and round robin between them. The idea is that one part is updated by the client (CPU), one is for the driver to keep track of and the last part is for the GPU to use. 

The *INSTANCE_IDX* is the index of the current object in the GraphicsBatch. This is the index that will be appended to a index buffer that is passed further down the pipeline. 

If the object is visible the index (*INSTANCE_INDEX*) for the object is appended to a buffer containing indices for all of the visible objects. These indices are later used to fetch the relevant information for each individual object in later stages.


### Shadowmapping pass 
There is only one directional light in the scene and this is setup as the viewpoint of this pass. 
Using the drawcommands generated in the culling pass the scene is rendered to a depth texture called shadowmap. This is used later in the lighting pass to apply shadows.

### Geometry pass
Renders the geometry, again with the generated drawcommands, and fills the global buffers with lots of information. 
* Depth, world position, normals, albedo, shading model id, etc

 Bild med alla texturer from typ RenderDoc eller ngt

Some of these depend on the exact shader configuration. Using the ubershader approach with ifdef there is some slight flexibility to change the information flow down the pipeline from this point.

// cubemaps , pbr , pbr scalar

### Lighting pass
Lightning calculations on the whole screen via a rendered quad. 

```cpp
  pass_started("Lightning pass");
  {
    const auto program = lightning_shader->gl_program;
    glBindFramebuffer(GL_FRAMEBUFFER, gl_lightning_fbo);

    glBindVertexArray(gl_lightning_vao);
    glUseProgram(program);

    glUniform1i(glGetUniformLocation(program, "environment_map_sampler"), gl_environment_map_texture_unit);
    glUniform1i(glGetUniformLocation(program, "shading_model_id_sampler"), gl_shading_model_texture_unit);
    glUniform1i(glGetUniformLocation(program, "emissive_sampler"), gl_emissive_texture_unit);
    glUniform1i(glGetUniformLocation(program, "ambient_occlusion_sampler"), gl_ambient_occlusion_texture_unit);
    glUniform1i(glGetUniformLocation(program, "pbr_parameters_sampler"), gl_pbr_parameters_texture_unit);
    glUniform1i(glGetUniformLocation(program, "diffuse_sampler"), gl_diffuse_texture_unit);
    glUniform1i(glGetUniformLocation(program, "normal_sampler"), gl_normal_texture_unit);
    glUniform1i(glGetUniformLocation(program, "position_sampler"), gl_position_texture_unit);
    glUniform1f(glGetUniformLocation(program, "screen_width"), screen_width);
    glUniform1f(glGetUniformLocation(program, "screen_height"), screen_height);

    glUniform3fv(glGetUniformLocation(program, "camera"), 1, &camera->position.x);

    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
  }
pass_ended();
```

```glsl
vec3 color = vec3(0.0);
switch (shading_model_id) {
    case 1: // Unlit
    color = unlit_render(diffuse);
    break;     
    case 2: // Physically based rendering with parameters sourced from textures
    case 3: // Physically based rendering with parameters sourced from scalars
    color = schlick_render(frag_coord, position, normal, diffuse, ambient_occlusion);
    // Emissive
    color += emissive; // TODO: Emissive factor missing
    break;
}
```

The type of lightning applied depends on the shading model id that is passed along the pipeline this is similiar to how Unreal Engine 4 works [reference]. 




```cpp
/// Copy final pass into default FBO
pass_started("Final blit pass");
{
    glBindFramebuffer(GL_READ_FRAMEBUFFER, gl_lightning_fbo);
    glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0);
    const auto mask = GL_COLOR_BUFFER_BIT;
    const auto filter = GL_NEAREST;
    glBlitFramebuffer(0, 0, screen_width, screen_height, 0, 0, screen_width, screen_height, mask, filter);
}
pass_ended();
```

Then lastly the mandatory blit pass that copies the previous pass into the default framebuffer (due to restriction on rendering to the default framebuffer??).

# Future Pipeline
Currently the pipeline is fixed but it would be interesting to have something similiar to a render/frame graph. This would represented as a DAG in which each node is a single pass with input and output buffers. This would require a lot of code just to setup and I get the feeling that this will be too much of maintenacne to be worth it. I suspect that a lot of performance could be gained when the redudant render passes are culled by some kind of compiler preprocessing of the DAG. Given that the number of passes in MeineKraft is hardcoded this is of no use at this point in time.

The shadowmapping pass needs to be improved due to the low quality and weird issues with the backside of objects not being in shadow. The state of it is currently very much a work in progress.

There are a lot of features I would like to add. Especially image-based ones such as depth-of-field, bloom, lensflares. 

I got this idea of creating a ray tracing pass where something like reflections can be computed. The rough idea is to have the material pass down a id on the reflective pixels and then run a ray tracing reflection through those pixels only. This would limit the number to only the visible reflective surfaces. It is also a great effect to have when looking strictly from a visual perspective. Technically it is seems feasiable and really cool.

The problem with trying to plan ahead is that I really do not know what constraints future features will place on the rendering system. Therefore I do not want to implement a whole render graph system that will just get in the way later down the road. 

# References
