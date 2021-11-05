---
title:  "N-Body Simulation (WebGPU)"
permalink: "blog/NBodyWebGPU"
layout: post
---
*Updated: 11-04-2021*

The first tool that I used to create N-Body Simulation was [WebGPU](https://www.w3.org/TR/webgpu/), which is a JavaScript API for accelerated graphics and compute in web broswers. (Still in development) WebGPU is an easily accessible tool for GPU programming, since it doesn't have a hardware requirement like Ndivida's CUDA. This was my first time getting into GPU programming, so I was very excited!

## Pipeline

In my project, I used `.wgsl` files to create functions that can be used by render and compute pipelines. Following is my code to render pipeline:
```
const spriteShaderModule = device.createShaderModule({ code: spriteWGSL });
const renderPipeline = device.createRenderPipeline({
    vertex: {
        module: spriteShaderModule,
        entryPoint: 'vert_main',
        buffers: [
          {
            // vertex buffer
            arrayStride: 4 * 4,
            stepMode: 'vertex',
            attributes: [
              {
                // vertex positions
                shaderLocation: 0,
                offset: 0,
                format: 'float32x2',
              },
            ],
          },
        ],
      },
      fragment: {
        module: spriteShaderModule,
        entryPoint: 'frag_main',
        targets: [
          {
            format: format as GPUTextureFormat
          },
        ],
      },
      primitive: {
        topology: 'point-list'
      },
})
```
#### spriteWGSL
```
[[stage(vertex)]]
fn vert_main([[location(0)]] particlePos : vec2<f32>) -> [[builtin(position)]] vec4<f32> {                            
    return vec4<f32>(particlePos, 0.0, 1.0);
}

[[stage(fragment)]] 
fn frag_main() -> [[location(0)]] vec4<f32> {
        return vec4<f32>(1.0, 1.0, 1.0, 1.0);
}
```

The use of render pipeline is to determine if the shape and color of object that I want to create. Since I will be creating particles, which is the most simple shape, my code ended up being pretty straightforward. The variable `renderPipeline` creates a render pipeline using the information from `spriteWGSL`. It will use  the information of `vertex` (size), and the information of `fragment` (color) to render objects to canvas. 

The WebGPU provides 5 primitive topologies that can determine the general shape of rendered objects. (`point-list`, `line-list`, `line-strip`, `triangle-list`, `triangle-strip`) This is actually pretty similiar to WebGL, so it will be easier to understand if you are familiar with WebGL. 

- `point-list`: Used for drawing a series of points 
- `line-list` Used for drawing line segments, but not connected to each other. 
- `line-strip` Used for drawing line segmenest and connected to each other. 
- `triangle-list` Used for drawing triangles.
- `triangle-strip` Used for drawing triangles that are connected to each other. 

### Rotating Objects

Since I will be using only particles in my project, I didn't have to worry about rotating my objects to make sure that they are facing toward the direction that they are moving towards. If you are wondering how that can be done, here is an example that I found: 
```
vertex: {
  module: spriteShaderModule,
  entryPoint: 'vert_main',
  buffers: [
    {
      // instanced particles buffer
      arrayStride: 4 * 4,
      stepMode: 'instance',
      attributes: [
        {
          // instance position
          shaderLocation: 0,
          offset: 0,
          format: 'float32x2',
        },
        {
          // instance velocity
          shaderLocation: 1,
          offset: 2 * 4,
          format: 'float32x2',
        },
      ],
    },
    {
      // vertex buffer
      arrayStride: 2 * 4,
      stepMode: 'vertex',
      attributes: [
        {
          // vertex positions
          shaderLocation: 2,
          offset: 0,
          format: 'float32x2',
        },
      ],
    },
  ],
}
```
#### spriteWGSL
```
[[stage(vertex)]]
fn vert_main([[location(0)]] a_particlePos : vec2<f32>,
             [[location(1)]] a_particleVel : vec2<f32>,
             [[location(2)]] a_pos : vec2<f32>) -> [[builtin(position)]] vec4<f32> {
  let angle = -atan2(a_particleVel.x, a_particleVel.y);
  let pos = vec2<f32>(
      (a_pos.x * cos(angle)) - (a_pos.y * sin(angle)),
      (a_pos.x * sin(angle)) + (a_pos.y * cos(angle)));
  return vec4<f32>(pos + a_particlePos, 0.0, 1.0);
}
```
[Source](https://austin-eng.com/webgpu-samples/samples/computeBoids#main.ts)

At first, you can definitely see how the code is more complicated. Under the `attributes:`, the `shaderLocation` and `offset` are defined to let the function `vert-main` to know where the position, velocity of particles can be found in the array that will store all the particles' properties. In this example, the particles' properties get stored in Float32Array. Each particle will take up 4 indices in the array: positionX, positionY, velocityX, and velocityY. Since Float32Array uses 4 bytes per element, the offset of velocity value needs to be `2 * 4` to make sure that values are being read in correct places. 

### Size of Objects

This was also something that I didn't have to worry about, since particles have fixed size. If you are wondering how changing size of objects can be done here is an example of determining size of `triangle-list` topology: 
```
const vertexBufferData = new Float32Array([
  -0.01, -0.02, 0.01,
  -0.02, 0.0, 0.02,
]);

const spriteVertexBuffer = device.createBuffer({
  size: vertexBufferData.byteLength,
  usage: GPUBufferUsage.VERTEX,
  mappedAtCreation: true,
});
new Float32Array(spriteVertexBuffer.getMappedRange()).set(vertexBufferData);
spriteVertexBuffer.unmap();
```
While it looks like variable `vertexBufferData` is holding 6 values, those 6 values are actually used for 3 Float32Array that will represent vertices:
1. First Array with value of -0.01 (x) and -0.02(y)
2. Second Array with value of 0.01 (x) and -0.02(y)
3. Third Array with value of 0.0 (x) and 0.02(y)

Hopefully you can visualize a triangle by plugging those 3 vertices! 

Lastly when you are rendering objects, make sure to call this command to set your vertex buffer to the encoder:
```
const passEncoder = commandEncoder.beginRenderPass(renderPassDescriptor);
passEncoder.setVertexBuffer(1, spriteVertexBuffer);
```

## BindGroups

## Computation

#### Useful Links:
- [WebGPU Samples by Austin Eng](https://austin-eng.com/webgpu-samples)
- [WebGPU Tutorial by Dr.Xu](https://www.youtube.com/channel/UCg14XfqXim0vpgabU3T7tRg/videos)
