---
layout: post
title: "Screen Space Ambient Occlusion (SSAO)"
last_modified_at: 2019-01-18
tags:
  - OpenGL
---

Ambient occlusion (AO) is basically the amount of local shadowing happening from the ambient light on a given surface. This shadowing is static and is caused by the ambient light which in most cases is faked by simply setting it to a some value. This method is cheap and produces results that are good enough in most cases.

The screen space part of screen space ambient occlusion (SSAO) comes from the fact that the algorithm developed by Crytek works in screen space. It does this by sampling nearby points on a hemisphere and calculate their screen space positions. Using these positions the samples are used to lookup the geometry depth values at these points. Then we calculate the actual depth values from the sample positions. If the calculated depth values from the sample positions are lower/higher than the ones stored by the geometry we know that they are 'inside' the geometry. This gives us a measurement of how occluded the currently processed pixel is by the nearby geometry. 

In order for SSAO via the provided shader to work there are some information that needs to be filled in the following buffers.
* Normal buffer - to orient the hemisphere
* Position buffer - to get
* Noise buffer

All of these are calculated in a 'depth-pass' prior to the SSAO pass as would be typically done in a deferred rendering pipeline. 

```glsl
#version 410 core 

uniform mat4 projection;

uniform sampler2D normal_sampler;
uniform sampler2D noise_sampler;
uniform sampler2D position_sampler;

uniform vec3 ssao_samples[64];

// SSAO Parameters
uniform float ssao_kernel_radius;
uniform float ssao_power;
uniform float ssao_bias;

uniform float scr_w; // Screen width
uniform float scr_h; // Screen height

const vec2 noise_scale = vec2(1280.0 / 8.0, 720.0 / 8.0);

out float ambient_occlusion;

void main() {
    // Screen position of the pixel
    vec2 frag_coord = vec2(gl_FragCoord.x / scr_w, gl_FragCoord.y / scr_h);
    vec3 normal = texture(normal_sampler, frag_coord.xy).xyz;
    vec3 position = texture(position_sampler, frag_coord.xy).xyz;

    // Orientate the kernel sample hemisphere randomly
    vec3 rvec = texture(noise_sampler, gl_FragCoord.xy * noise_scale).xyz; // Picks random vector to orient the hemisphere
    vec3 tangent = normalize(rvec - normal * dot(rvec, normal));
    vec3 bitangent = cross(normal, tangent);
    mat3 tbn = mat3(tangent, bitangent, normal); // f: Tangent -> View space

    ambient_occlusion = 0.0;
    const uint num_ssao_samples = 64;
    for (int i = 0; i < num_ssao_samples; i++) {
        // 1. Get sample point (view space)
        vec3 sampled = position + tbn * ssao_samples[i] * ssao_kernel_radius;

        // 2. Generate sample depth from sample point
        vec4 point = vec4(sampled, 1.0);
        point = projection * point;
        point.xy /= point.w;
        point.xy = point.xy * 0.5 + 0.5;

        // 3. Lookup depth at sample's position (screen space)
        float point_depth = texture(position_sampler, point.xy).z;

        // 4. Compare with geometry depth value
        if (point_depth >= sampled.z + ssao_bias) { ambient_occlusion += 1.0; }
    }
    // Calculate the average and invert to get OCCLUSION
    ambient_occlusion = 1.0 - (ambient_occlusion / float(num_ssao_samples));
    // Enhance the effect
    ambient_occlusion = pow(ambient_occlusion, ssao_power);
}
```
Usually they SSAO is a bit uneven and looks a bit off. Therefore the common solution to this is to simply blur it all using a simple blurring shader pass.

```glsl
#version 410 core 

uniform sampler2D input_sampler;

uniform int blur_size;     // Size of noise texture

uniform float scr_w;       // Screen width
uniform float scr_h;       // Screen height

out float fResult;

// Square average blur kernel
void main() {
    vec2 frag_coord = vec2(gl_FragCoord.x / scr_w, gl_FragCoord.y / scr_h);
    vec2 texel_size = 1.0 / vec2(textureSize(input_sampler, 0));
    float result = 0.0;
    for (int x = -blur_size; x < blur_size; x++) {
        for (int y = -blur_size; y < blur_size; y++) {
            vec2 offset = vec2(float(x), float(y)) * texel_size;
            result += texture(input_sampler, frag_coord + offset).r;
        }
    }
    fResult = result / (4 * blur_size * blur_size);
}
```
One way to improve this blurring pass is to use a Gaussian blurring kernel but in general a simple square blurring kernel will be a bit cheaper and not look too bad.

# More SSAO
* http://ogldev.atspace.co.uk/www/tutorial45/tutorial45.html
* http://developer.download.nvidia.com/SDK/10.5/direct3d/Source/ScreenSpaceAO/doc/ScreenSpaceAO.pdf
* http://john-chapman-graphics.blogspot.com/2013/01/ssao-tutorial.html
* https://learnopengl.com/Advanced-Lighting/SSAO
* http://artis.inrialpes.fr/Membres/Olivier.Hoel/ssao/p97-mittring.pdf