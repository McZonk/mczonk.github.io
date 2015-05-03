---
layout: post
title: Learning from Sprite Kit 2 - Sprites
---

**TL;DR: Sprite Kit batches sprite draw calls when possible.**

In the frist article about the Sprite Kit internals we have taken a look into the resources of an empty scene. Now let's start to draw something. I have taken the red spaceship and created an additionally green and blue version. Also texture altases are not used in this sample.

Again all the code to try this on your device can be found on [GitHub](https://github.com/McZonk/SpriteKitInside).

### Same texture, multiple objects

[Source Code](https://github.com/McZonk/SpriteKitInside/tree/2-SameObject)

![Drawing 12 red spaceships](/images/SpriteKit_12RedSpaceships.png)

This is the first and easiest example. Drawing 12 sprites with the red spaceship. When we are taking a look at the OpenGL ES call stack we find that there is only a single call needed to draw all the spaceships.

```
9 glBufferData(GL_ARRAY_BUFFER, 1536, <data>, GL_STREAM_DRAW)
10 glUseProgram(2)
11 glClearColor(0.1500000, 0.1500000, 0.3000000, 1.0000000)
12 glClear(GL_COLOR_BUFFER_BIT | GL_STENCIL_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
13 glBindTexture(GL_TEXTURE_2D, 5)
14 glDrawElements(GL_TRIANGLES, 72, GL_UNSIGNED_SHORT, nullptr)
```

Again we should take a look at the loaded buffers. The first interesting difference to the empty scene is that in call 9 `glBufferData`. This time it is not empty. Instead 1,536 bytes of data are uploaded into the GPU. You can examine this buffer but it is way easier to take a look when the buffer is drawn later.

The call 14 `glDrawElements` is the next new command send to the GPU here. When we take a look into the buffer 2 `GL_ELEMENT_ARRAY_BUFFER`. If we group it into 6 columns and display the values as `UInt16` we discover a pretty easy pattern.

This buffer contains groups of triangle indicies. `{0,1,2}, {2,3,0}, {4,5,6}, {6,7,4}, …` that draw into quads, because every two triangles two indicies are shared. Also this buffer is already prefilled with 126,000 bytes. Two bytes per index lead to 63,000 indicies. With three vertices per triangle we can draw up to 21,000 triangle or 10,500 quads per draw call.
Later in the article we will see what happens when we draw more than 10,500 quads in a scene.

If we take a look into the implicit Vertex Array Object 0 we can nicely examine the content of the data in the Array Buffer 1.

The `gl_VertexID` coresponds to the values in the index buffer. `a_color`, `a_position` and `a_tex_coord` are the values that are used in the default FastSingle shader. We can see that the color components transferred as bytes, while the rest are floats. The x- and y-position is already normalized and also the third and fourth component are pushed into the GPU.
If you look at the sizes, types and strides you will see that each vertex is 32 bytes long. But 4 × `GL_UNSIGNED_BYTE` color + 4 × `GL_FLOAT` position + 2 × `GL_FLOAT` texture coordinate is only 28 bytes. Still there is a 4 byte padding at the end. There is no shader that could access this value. The documentation suggests a [four byte alignment](https://developer.apple.com/library/ios/documentation/3ddrawing/conceptual/opengles_programmingguide/TechniquesforWorkingwithVertexData/TechniquesforWorkingwithVertexData.html#//apple_ref/doc/uid/TP40008793-CH107-SW7) and even if the alignment  should be 8 bytes on an A7 (64-bit) device the texture coordinates are on a 20 byte offset.
Maybe it is better for internal caching to use 32 bytes. But the bus is usually the bottleneck on mobile devices, so I'm sure there is a good reason.

### Multiple textures, multiple objects

![Drawing 12 random colored spaceships](/images/SpriteKit_MulticoloredSpaceships.png)

[Source Code](https://github.com/McZonk/SpriteKitInside/tree/2-DifferentObjects)

Same scene again, but this time a random texture is selected. Therefore the texture needs to be switched multiple times and multiple draw calls are needed.

```
9 glBufferData(GL_ARRAY_BUFFER, 1536, <data>, GL_STREAM_DRAW)
10 glUseProgram(2)
11 glClearColor(0.1500000, 0.1500000, 0.3000000, 1.0000000)
12 glClear(GL_COLOR_BUFFER_BIT | GL_STENCIL_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
13 glBindTexture(GL_TEXTURE_2D, 5)
14 glDrawElements(GL_TRIANGLES, 12, GL_UNSIGNED_SHORT, nullptr)
15 glBindTexture(GL_TEXTURE_2D, 6)
16 glDrawElements(GL_TRIANGLES, 12, GL_UNSIGNED_SHORT, 0x00000018)
17 glBindTexture(GL_TEXTURE_2D, 5)
18 glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, 0x00000030)
19 glBindTexture(GL_TEXTURE_2D, 6)
20 glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, 0x0000003c)
21 glBindTexture(GL_TEXTURE_2D, 7)
22 glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, 0x00000048)
23 glBindTexture(GL_TEXTURE_2D, 6)
24 glDrawElements(GL_TRIANGLES, 12, GL_UNSIGNED_SHORT, 0x00000054)
25 glBindTexture(GL_TEXTURE_2D, 5)
26 glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, 0x0000006c)
27 glBindTexture(GL_TEXTURE_2D, 7)
28 glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, 0x00000078)
29 glBindTexture(GL_TEXTURE_2D, 6)
30 glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, 0x00000084)
```

You can see that the texture is switched whenever the color of the starship changes. But you have still the exact same vertex and index buffers only more texture switches and draw calls.

When you enable `SKView.ignoresSiblingOrder` the number of draw calls is reduced to three, one per texture. But obviously you will change the output when objects overlap.

If you bump up the number of objects to 1,200 instead of twelve you will see the frame rate drop down massively to something between 20 and 25 on an iPhone 5s.

- Only red spaceships: 25fps
- Multicolor spaceships: 22fps
- Multicolor spaceships with `SKView.ignoresSiblingOrder` on: 24fps

I assumed that `SKView.ignoresSiblingOrder` will make a huge difference because it reduced the number of draw calls from around 800 to 3 when turned on. But it only changes the frames per second from 22 to 24, which an increase less than 10%. We can learn here is that a lot of draw calls doesn't affect the performance that much.
This doesn't tell you to ignore the number of draw calls, but maybe there are better ways to optimize the performance.

### Invisible objects

Sprite Kit early calculates all the object's transforms before the data is passed into the GPU. So the translation, scale and rotations is applied on the CPUs. This allows the engine to use the very simple passthrough vertex shaders. Also it enables Sprite Kit to sort out the objects with a bounding box that doesn't intersect the screen's rectangle. The reason is not that the rasterizer has to do less work, but it reduces the amount of data that needs to be transferred to the GPU.

### 10,500 (+1) objects

The index buffer only allows us to draw up to 10,500 quads in a single draw call. So what happens when we draw 10,501 objects.

```
9 glBufferData(GL_ARRAY_BUFFER, 1343872, <data>, GL_STREAM_DRAW)
10 glUseProgram(2)
11 glClearColor(0.1500000, 0.1500000, 0.3000000, 1.0000000)
12 glClear(GL_COLOR_BUFFER_BIT | GL_STENCIL_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
13 glBindTexture(GL_TEXTURE_2D, 5)
14 glDrawElements(GL_TRIANGLES, 62994, GL_UNSIGNED_SHORT, nullptr)
15 glViewport(0, 0, 1136, 640)
16 glGetIntegerv(GL_ARRAY_BUFFER_BINDING, {1})
17 glBufferData(GL_ARRAY_BUFFER, 256, <data>, GL_STREAM_DRAW)
18 glUseProgram(2)
19 glBindTexture(GL_TEXTURE_2D, 5)
20 glDrawElements(GL_TRIANGLES, 12, GL_UNSIGNED_SHORT, nullptr)
```

The first 10,499 quads are rendered in a single draw call (14). The next two objects are rendered in a second draw call (20). That is kind of surprising. I thought that only the last object needs an extra draw call. If you change the amount to 10,500 objects there will be still two draw calls. If you remove another object (10,499) only a single draw call is needed. This will bump up the frame rate from 2 to 5.
I guess there is an comparsion error in the Sprite Kit internals, somewhere a `<` instead `<=` is used.

### Blend modes

![Blend Modes](/images/SpriteKit_12NoBlendingSpaceships.png)

Blending is expensive and you cannot submit objects with different blend modes in a single draw call, but as demonstrated before an additonal draw call is not that expensive.
The default blend mode is `SKBlendModeScreen`, but when you change the blend mode of all the spaceships to `SKBlendModeReplace` no data from the framebuffer needs to be read back. If you draw 1,200 spaceships you can still keep 60 frame per second. Even if you draw 10,499 spaceships you can keep a frame rate of 45.

### Summary

This time we discovered a lot about how to use Sprite Kit itself. Even if I didn't touch the topic here, use texture atlasses. Maybe I will cover how to efficently group your textures in a future post. But having less textures reduces the amount of state changes and draw calls.
When you are drawing solid objects, don't blend. If you have translucent objects keep the image as small as possible. Just 1 pixel wide border of transparency around the image.

If you build your own engine you can still transfer a lot of the gathered knowledge. Multiple draw calls are not that expensive on iOS.
Sort out objects as early as possible. If you cannot do it on the CPU you might want to try occlusion queries.
The trick with a single buffer that is read sequentially is really nifty and seems to be one of the keys to high Sprite Key performance.
Reading back from a framebuffer is insanely expensive on iOS so try to avoid blending whenever it is possible.
