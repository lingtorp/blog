---
layout: post
title: "Render depth buffer to a texture."
last_modified_at: 2017-03-09T14:25:52-05:00
tags:
  - content
  - css
  - edge case
  - lists
  - markup
---

Today I solved a problem that was bugging me for almost two weeks. While rendering the depth values to a texture
bound to the depth buffer of a framebuffer object (FBO) I forgot to clear the depth buffer bit. This cause the depth values in the texture
to only by updated whenever the new values were smaller than the ones in the texture. The visual effect of this bug 
manifests itself by turning everything darker whenever the camera comes closer than previous closest position.

# Receipe
1. Create a new vertex and fragment shader.
- The vertex shader should do the transformations and pass the data along.
- The fragment shader does not need to do anything since we only care about the depth values.

	**Vertex shader**
	```glsl
	/* This might look really different depending on how you structure your shaders. */
	#version 410 core

	in vec3 position;

	uniform mat4 projection;
	uniform mat4 camera_view;

	void main() {
	    gl_Position = projection * camera_view * vec4(position, 1.0);
	}
	```
	**Fragment shader**
	```glsl
	#version 410 core
	void main() {}
	```

2. Create a new FBO.
	```cpp
	uint32_t gl_depth_fbo;
	glGenFramebuffers(1, &gl_depth_fbo);
	```

3. Create a new 2D texture.
	```cpp
	uint32_t gl_depth_texture_unit; /* This depends on what texture units you are using. */
	glActiveTexture(GL_TEXTURE0 + gl_depth_texture_unit);

	uint32_t gl_depth_texture;
	glGenTextures(1, &gl_depth_texture);
	glBindTexture(gl_depth_texture);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
	int screen_width = 1280; /* Depends on your setup */
	int screen_height = 720;
	glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, screen_width, screen_height, 0, GL_DEPTH_COMPONENT, GL_UNSIGNED_INT, nullptr);
	```

4. Attach the texture to the GL_DEPTH_ATTACHMENT of the new FBO.
	```cpp
	glFramebufferTexture(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, gl_depth_texture, 0);
	/* No need for glDrawBuffers since we are only interested in the sole depth buffer */
	```
Now the only thing left to do is render the scene, which depends on your setup. 
Remember to *clear* the depth buffer bit and also to bind the default framebuffer.
```cpp
/* Inside the rendering loop */
glBindFrameBuffer(GL_FRAMEBUFFER, gl_depth_fbo);
glClear(GL_DEPTH_BUFFER_BIT);
/* Render the scene; glDraw* */
```
You can always check if there is something wrong with the configuration of the FBO using the following snippet.
```cpp
GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
if (status != GL_FRAMEBUFFER_COMPLETE) {
  std::cerr << "Framebuffer configuration failed" << std::endl;
}
```
