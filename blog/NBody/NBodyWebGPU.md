---
title:  "N-Body Simulation (WebGPU)"
permalink: "blog/NBodyWebGPU"
layout: post
---
*Updated: 11-04-2021*

The first tool that I used to create N-Body Simulation was [WebGPU](https://www.w3.org/TR/webgpu/), which is a JavaScript API for accelerated graphics and compute in web broswers. (Still in development) WebGPU is an easily accessible tool for GPU programming, since it doesn't have a hardware requirement like Ndivida's CUDA. This was my first time getting into GPU programming, so I was very excited!

I will be referring to specific file names from my code, so take a look at them if you are interested! [Github Link](https://github.com/MangoShip/NBodyWebGPU)

## 0. General Structure

Here is a general structure of my implementation of WebGPU to simulate the N-Body particle simulation
1. Get all variables needed for WebGPU API calls.
2. Create render and compute pipelines. They will hook up instructions from `.wgsl` files
3. Create and initialize a Float32Array that stores properties of all particles. (position and velocity)
4. Create Buffer and BindGroups for the compute pipeline. They will hook up a binding to particle data that can be used during computation.
5. Start computation (N-Body) on particles.
6. Start rendering of particles.
7. Measure performance (FPS) to see the duration of Step 5 and 6.
8. Repeat 5-7.

## 1. Getting Variables

There are four variables that I will to initialize. I went through this [link](https://www.w3.org/TR/webgpu/) to learn about them.
1. Adapter (GPUAdapter): An implementation of WebGPU on the system. It will be used to get GPUDevice. 
2. Device (GPUDevice): The logical instantiation of an adapter, top-level interface through which WebGPU interfaces are created. It will be used to call WebGPU API calls.
3. Context (webgpu): Context of canvas that can be configured for rendering the particles. 
4. Format: Specifies order of components, bits per components, and data type for the component. It will be used for context configuration. `bgra8unorm` means `blue, green, red, and alpha` `8 bits per components` `unsigned normalized data type`

## 2. Pipeline

In my project, I will be using `.wgsl` files to create functions that can be used by render and compute pipelines. Following is my code to render pipeline:
#### mainWebGPU.ts
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
#### sprite.wgsl
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

### 2.1. Rotating Objects

Since I will be using only particles in my project, I didn't have to worry about rotating my objects to make sure that they are facing toward the direction that they are moving to. If you are wondering how that can be done, here is an example that I found: [Source](https://austin-eng.com/webgpu-samples/samples/computeBoids#main.ts)
#### main.ts
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
#### sprite.wgsl
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

At first, you can definitely see how the code is more complicated. Under the `attributes:`, the `shaderLocation` and `offset` are defined to let the function `vert-main` to know where the position, velocity of particles can be found in the array that will store all the particles' properties. In this example, the particles' properties get stored in Float32Array. Each particle will take up 4 indices in the array: positionX, positionY, velocityX, and velocityY. Since Float32Array uses 4 bytes per element, the offset of velocity value needs to be `2 * 4` to make sure that values are being read in correct places. 

### 2.2. Size of Objects

This was also something that I didn't have to worry about, since particles have fixed size. If you are wondering how changing size of objects can be done here is an example of determining size of `triangle-list` topology: [Source](https://austin-eng.com/webgpu-samples/samples/computeBoids#main.ts)
#### main.ts
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
While it looks like variable `vertexBufferData` is holding 6 values, those 6 values are actually representing 3 vertices:
1. First vertex with value of -0.01 (x) and -0.02(y)
2. Second vertex with value of 0.01 (x) and -0.02(y)
3. Third vertex with value of 0.0 (x) and 0.02(y)

Hopefully you can visualize a triangle by plugging those 3 vertices! 
Let's say you want to make the triangle little wider, then you would want to replace x values of first (e.g. -0.01 -> -0.02) and second vertex. (e.g. 0.01 -> 0.02)

Lastly when you are rendering objects, make sure to call this command to set your vertex buffer to the encoder:
#### main.ts
```
const passEncoder = commandEncoder.beginRenderPass(renderPassDescriptor);
...
passEncoder.setVertexBuffer(1, spriteVertexBuffer);
```
[Source](https://austin-eng.com/webgpu-samples/samples/computeBoids#main.ts)

## 3. Storing Particles Data

Here, I will be creating a `Float32Array` with size of `Number of Particles * 4`. Each particle will take up 4 indices from the array. (PosX, PosY, VelX, VelY) Then, I will go through the array and initialize the positions of each particle with random values. 
```    
const initialParticleData = new Float32Array(numParticles * 4);
for (let i = 0; i < numParticles; ++i) {
    initialParticleData[4 * i + 0] = 2 * (Math.random() - 0.5); // posX
    initialParticleData[4 * i + 1] = 2 * (Math.random() - 0.5); // posY
    initialParticleData[4 * i + 2] = 0; // velX
    initialParticleData[4 * i + 3] = 0; // velY
}
```
The reason why I use `Float32Array` is due to how WebGPU contain WebGL application, which requires vectors and matrices to be passed as a `TypedArray` (e.g. `Float32Array`). 

## 4. Buffer and BindGroups



## 5. Particle Computation

## 6. Rendering Particles

## 7. Measuring Performance

#### Useful Links:
- [WebGPU Samples by Austin Eng](https://austin-eng.com/webgpu-samples)
- [WebGPU Tutorial by Dr.Xu](https://www.youtube.com/channel/UCg14XfqXim0vpgabU3T7tRg/videos)
