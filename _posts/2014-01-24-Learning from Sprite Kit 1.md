---
layout: post
title: Learning from Sprite Kit 1 - Empty Scene
---

**TL;DR: Sprite Kit is pretty light-weighted inside.**

Starting with iOS 7 and OS X Mavericks (10.9) Apple has intergrated their own 2D game engine called [Sprite Kit](https://developer.apple.com/library/IOs/documentation/GraphicsAnimation/Conceptual/SpriteKit_PG/Introduction/Introduction.html). 

This post will not be about learning Sprite Kit itself but to see how it will work and render internally.
I will not discuss the benefits of Sprite Kit vs other 2D middle wares (Cocos2D, Kobold Kit, …). For this article series I will take a look at the iOS internals of Sprite Kit.

### Getting started

There is an iOS template called 'Sprite Kit Game'. It will offer a very simple entry point. Starting from this sample I removed the node and fps counter, the 'Hello World' label, the status bar and also I scale down the size of the space ship to 20%. The source code is publicly available on [GitHub](https://github.com/McZonk/SpriteKitInside/tree/1). To follow my investigation start this project on a physical device, not the simulator and capture an OpenGL ES frame. Supress the warning 'Shader profiler does not support multi-contexts capture'.

### OpenGL ES Call stack

![Empty scene OpenGL ES call stack](/content/images/2014/Jan/SpriteKit_EmptyCallstack.png)

The first thing we will take a look at it the OpenGL ES call stack. 16 calls are needed to draw an empty screen.

The first nine calls (0-8) are just basic setup. A few of them are `glGet...` which are fast because the graphics pipeline is still empty, so the CPU doesn't have to wait. It readys back information about the framebuffer and the renderbuffer.

The first interessting call (9) is `glBufferData`. In an empty scene no data will be transferred. The `GL_STREAM_DRAW` is a hint that this data will be refreshed every frame.

The call (10) to `glUseProgram` is unnecessary as long as no objects are drawn. But it will give us a hint which program is the default.

The next to calls (11-12) will be all the current drawing. Clearning the screen with the background color from `MyScene.m:17`.

The rest are just some clean up calls (13-15) which are from no interest right now.

### OpenGL ES Resources

![All OpenGL ES resources in an empty scene](/content/images/2014/Jan/SpriteKit_EmptySceneResources.png)

The next thing we should take a look at are the resources loaded into the GPU at this point.

#### Textures

![Sprite Kit bitmap font](/content/images/2014/Jan/SpriteKit_BitmapFont.png)

There are 4 textures loaded. Texture 1 is an 1×1 sized white image. The next two are empty textures and the fourth is the bitmap font used to draw fps and node count.

#### Buffers

There are two buffers used. There are bound in the vertex array object 0 which is the implicit object. The type of the first buffer is `GL_ARRAY_BUFFER`, it will hold vertex data and it is the buffer that was filled in `glBufferData` so is currently empty. The second buffer‘s type is `GL_ELEMENT_ARRAY_BUFFER` so it will hold indicies. This buffer's size is 126 000 bytes and the access is `GL_STATIC_DRAW`. It is likely that this data was prefilled in the setup phase. But since we are not drawing we skip this buffer for now.

#### Renderbuffers

There are three of the four framebuffers attached to the current renderbuffer. Interesstingly there is a depth and a stencil buffer. None of them is enabled right now. But it is surprising that a 2D engine has those.

#### Programs

Those are the first objects that are worth a more detailed look in an empty scene. There are four programs compiled and linked. It is very likely that those are everything that use by Sprite Kit. Also those programs are named with [debug labels](https://developer.apple.com/library/ios/documentation/3ddrawing/conceptual/opengles_programmingguide/Performance/Performance.html#//apple_ref/doc/uid/TP40008793-CH105-SW6). 

##### [FastSingle](https://gist.github.com/McZonk/8228086)

// vertex shader
attribute vec4 a_color;
attribute vec4 a_position;
attribute vec2 a_tex_coord;
varying highp vec2 v_tex_coord;
varying lowp vec4 v_color_mix;
void main() {
v_color_mix = a_color;
v_tex_coord = a_tex_coord;
gl_Position = a_position;
}

// fragment shader
uniform lowp sampler2D u_texture;
varying highp vec2 v_tex_coord;
varying lowp vec4 v_color_mix;
void main() {
gl_FragColor = v_color_mix * texture2D(u_texture, v_tex_coord);
}

This is proberly the most interessting shader and this is the also one that is already in use. All the other shaders are special purpose ones.
The vertex shader does not perform a single calculation it just passthrough the vertex data. Even the `gl_Position` is not calculated. The position data must already be normalized when passed into the shader.
The fragment shader is also straight forward. It multiplies the color of the vertex with the color of the texture.

##### [Video](https://gist.github.com/McZonk/8228152)

// vertex shader
attribute vec4 a_color;
attribute vec4 a_position;
attribute vec2 a_tex_coord;
varying mediump vec2 v_tex_coord;
varying lowp vec4 v_color_mix;
void main() {
v_color_mix = a_color;
v_tex_coord = a_tex_coord;
gl_Position = a_position;
}

// fragment shader
uniform lowp sampler2D u_texture_luma;
uniform lowp sampler2D u_texture_chroma;
uniform lowp vec3 u_ycbcr_matrix[4];
varying mediump vec2 v_tex_coord;
varying lowp vec4 v_color_mix;
void main() {
lowp vec3 color;
lowp vec3 ycbcr = vec3(
texture2D(u_texture_luma, v_tex_coord).r,
texture2D(u_texture_chroma, v_tex_coord).ba
);
color.r = dot(ycbcr, u_ycbcr_matrix[0]);
color.g = dot(ycbcr, u_ycbcr_matrix[1]);
color.b = dot(ycbcr, u_ycbcr_matrix[2]);
color += u_ycbcr_matrix[3];
gl_FragColor = vec4(color, 1.0) * v_color_mix;
}

The vertex shader is the exact same as before: no calculations, just a passthrough.
The fragment shader is also pretty similar but instead of only one RGB texture it uses a lumincance (Y) and a chroma (UV) texture and converts the data to RGB. We will talk about this shader in detail in a future post when we take a closer look at video textures.

##### [Discard](https://gist.github.com/McZonk/8228033)

// vertex shader
attribute vec4 a_position;
attribute vec2 a_tex_coord;
varying mediump vec2 v_tex_coord;
void main() {
v_tex_coord = a_tex_coord;
gl_Position = a_position;
}

// fragment shader
uniform lowp sampler2D u_texture;
varying mediump vec2 v_tex_coord;
void main() {
if (texture2D(u_texture, v_tex_coord).a <= 0.05) {
discard;
}
}

Nearly same vertex shader again, execpt that there is no color this time.
The fragment shader is something special. It doesn't write to `gl_FragColor` instead it just discards the output when the alpha value in the texture is very small. This is an uncommon shader. But there is a stencil and a depth buffer and this shader can be used to prefill one of them. I'm really interessted to see this shader be used.

##### [Selected](https://gist.github.com/McZonk/8228124)

// vertex shader
uniform float u_selected;
attribute vec4 a_position;
attribute vec4 a_color;
varying lowp vec4 v_color;
void main() {
if (u_selected > 0.0) {
v_color = vec4(0.5, 1.0, 1.0, 1.0);
} else {
v_color = a_color;
} 
gl_Position = vec4(a_position.xy, 0, 1.0);
}

// fragment shader
varying lowp vec4 v_color;
void main() {
gl_FragColor = v_color;
}

This is the first time there is a something calculated in the vertex shader. The uniform `u_selected` is used to override the color and set it to a bright cyan. Also the position's z-value is replaced with zero.
The fragment shader here is very minimal and just puts out the color. This program might be used for drawing objects without a texture, but then the selected wouldn't make sense. So I guess this program is used for debug drawing.

### Summary

Sprite Kit is drilled for performance. Even if we haven't draw a single sprite yet I'm really surprised about the easy internal structure. There is only a single shader that can be used for general drawing. Stay tuned, next week we will dig into drawing sprites.