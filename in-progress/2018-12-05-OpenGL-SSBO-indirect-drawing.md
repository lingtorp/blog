# SSBO + indirect rendering = GPU generated draw commands

## Introduction
So, I got stuck on a project that I am currently working on but thought I might as well write about the things I have learnt this week. My plan is to integrate some kind of view frustum culling into my rendering engine, MeineKraft, however before I am able to do that I wanted to explore this thing called indirect rendering.

Our story begins with [glMultiDrawElementsIndirect](http://docs.gl/gl4/glMultiDrawElementsIndirect) and the singular version [glDrawElementsIndirect](http://docs.gl/gl4/glDrawElementsIndirect) which as the name suggests are _indirect_ versions of [glMultiDrawElements](http://docs.gl/gl4/glMultiDrawElements) and [glDrawElements](http://docs.gl/gl4/glDrawElements).


## Indirect drawcalls
Now that we got the confusion going we take a couple of steps back and see how the OpenGL drawcalls build on each other with each and everyone of them adding some sort of functionality. 

[glDrawElements](http://docs.gl/gl4/glDrawElements) - draws vertices using indices from _GL_ELEMENT_ARRAY_

[glMultiDrawElements](http://docs.gl/gl4/glMultiDrawElements) - 

[glDrawElementsIndirect](http://docs.gl/gl4/glDrawElementsIndirect) - 

[glMultiDrawElementsIndirect](http://docs.gl/gl4/glMultiDrawElementsIndirect) - 



## [Shader Storage Buffer Objects (SSBOs)](https://www.khronos.org/opengl/wiki/Shader_Storage_Buffer_Object)
The main thing that makes SSBOs so powerful is that they are large chunks of memory that can be both written *and* read by the shader. This really opens up a lot of interesting applications. One of these is the use of compute shaders to perform  view frustum culling. 

There are a couple of things to know about SSBOs before one can start to use them. 
```glsl
buffer ShaderStorageBufferObjectBlockName {
    uint count;  // Number of written matrices (read by the CPU)
    mat4 MVPs[]; // Model-View-Projection matrices
};
```
The last member of the _interface block_ can be a variable length array of whatever type you like. There is also runtime support for quering the length of the array with the built-in function _length_. The variable length member **must** be the last member in the _interface block_. The whole block can be marked as _readonly_ or _writeonly_.

One thing to note with SSBOs are all the access flags and when to use them. Given that we are basically allocating memory we need somewhere to put this memory and some rules about who gets to touch it and when. Another thing to watch out for is that certain combinations of flags are not valid such as marking a buffer as read only in creation but then trying to map a pointer with write access to it. 

### Creation
```cpp
GLuint ssbo = 0;
glGenBuffers(1, &ssbo);
glBindBuffer(GL_SHADER_STORAGE_BUFFER, ssbo);
glBufferStorage(GL_SHADER_STORAGE_BUFFER, NUM_OBJECTS * sizeof(glm::mat4), nullptr, GL_MAP_WRITE_BIT);
```

### Update
Updating the SSBO with glBufferData;
```cpp
glBufferData(GL_SHADER_STORAGE_BUFFER, pointlights.size() * sizeof(PointLight), pointlights.data(), GL_DYNAMIC_COPY);
```
... or updating the SSBO via a mapped pointer.
```cpp
GLvoid* ssbo_mvps = glMapBufferRange(GL_SHADER_STORAGE_BUFFER, 0, NUM_OBJECTS * sizeof(glm::mat4), GL_MAP_WRITE_BIT);
std::memcpy(ssbo_mvps, mvps.data(), NUM_OBJECTS * sizeof(glm::mat4));
glUnmapBuffer(GL_SHADER_STORAGE_BUFFER);
```

## Advanced Example 
### Computer Shader
```glsl
// Same as the OpenGL defined struct: DrawElementsIndirectCommand
struct DrawCommand {
    uint count;         // Num elements (vertices)
    uint instanceCount; // Number of instances to draw (a.k.a primcount)
    uint firstIndex;    // Specifies a byte offset (cast to a pointer type) into the buffer bound to GL_ELEMENT_ARRAY_BUFFER to start reading indices from.
    uint baseVertex;    // Specifies a constant that should be added to each element of indices​ when chosing elements from the enabled vertex arrays.
    uint baseInstance;  // Specifies the base instance for use in fetching instanced vertex attributes.
};

// Command buffer backed by a Shader Storage Object Buffer (SSBO)
layout(std140, binding = 0) writeonly buffer DrawCommandsBlock {
    DrawCommand draw_commands[];
};
```
Here we declare a simple struct that will represent one drawcall together with the SSBO that will store the drawcalls generated in the compute shader. The shaders job here is to generate all the drawcommands by culling the not visible objects. I will not touch upon how in the post. 

```glsl
void main() {
    /// PREMISE: Compute visible via a view frustum culling method 
    ...
    const uint idx = gl_LocalInvocationID.x; // Compute space is 1D where x in [0, N)
    draw_commands[idx].count = 25350;        // sphere.indices.size(); # of indices in the mesh (GL_ELEMENTS_ARRAY)
    draw_commands[idx].instanceCount = visible ? 1 : 0;
    draw_commands[idx].baseInstance = 0;     // See above
    draw_commands[idx].baseVertex = 0;       // See above
}
```
In this approach I have choosen to generate one drawcommand per visible object but another way to format this is to only generate one drawcommand per mesh. The problem with this approach is that we generate a lot more drawcommands and waste some memory space. The main benefit is that this is the simpler of the two.

When batching the drawcommands .. ?

### Rendering shader 
???

### Rendering code (CPU)
#### Setup
First of all we create the buffer that will hold all the drawcommands. 
```cpp
// Setup GL_DRAW_INDIRECT_BUFFER for indirect drawing (basically a command buffer)
glGenBuffers(1, &gl_state.ibo);
glBindBuffer(GL_DRAW_INDIRECT_BUFFER, gl_state.ibo);
glBufferStorage(GL_DRAW_INDIRECT_BUFFER, NUM_OBJECTS * sizeof(DrawElementsIndirectCommand), nullptr, GL_MAP_READ_BIT);
```
Next we hook the indirect command buffer up with the SSBO that is in the compute shader. We want the drawcommands to end up in the buffer bound at the target _GL_INDIRECT_BUFFER_.
```cpp
// Setup compute culling shader
const uint32_t program = gl_state.cull_shader.gl_program;
glUseProgram(program);
const uint32_t gl_draw_cmd_binding_point = 0; // Set in the computer shader via layout binding
const uint32_t gl_ssbo_block_idx = glGetProgramResourceIndex(program, GL_SHADER_STORAGE_BLOCK, "DrawCommandsBlock");
glShaderStorageBlockBinding(program, gl_ssbo_block_idx, gl_draw_cmd_binding_point);

glBindBuffer(GL_SHADER_STORAGE_BUFFER, gl_state.ibo);
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, gl_draw_cmd_binding_point, gl_state.ibo);
```
The last two lines are where the binding to the buffer with the drawcommands is made. Prior to that we simply assign the block index to a shader binding point (which is limited). 

#### Rendering
```cpp
glUseProgram(gl_state.cull_shader.gl_program);
glDispatchCompute(NUM_OBJECTS, 1, 1);
glMemoryBarrier(GL_COMMAND_BARRIER_BIT | GL_SHADER_STORAGE_BARRIER_BIT); 
/// glMultiDrawElementsIndirect(mode, element_type, *indirect, num_cmds, cmd_stride)
glMultiDrawElementsIndirect(GL_TRIANGLES, GL_UNSIGNED_INT, nullptr, 1, 0);
```
Using _glDispatchCompute_ we launch the compute shader in 1D compute space. In order to wait for the shader to finish its computation we need to place a memory barrier ... 

In this drawcall the number of commands are hardcoded to 1 due to the fact that there is only one type of object to render. 







Feedback, comments, thoughts can all be dropped via Twitter [@ALingtorp](https://twitter.com/ALingtorp).

## References & further reading
