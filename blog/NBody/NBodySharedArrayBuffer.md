---
title:  "N-Body Simulation (SharedArrayBuffer)"
permalink: "blog/NBodySharedArrayBuffer"
layout: post
---
*Updated: 12-01-2021*

After implementation of Web Workers, it is time for some optimization! Since the size of array that stores particle data is `4 * number of particles`, I noticed that I've been transferring large amount of data between main thread and Web Workers. (especially if I have multiple threads, I will be transferring this to every thread) Therefore, I have decided to implement SharedArrayBuffer in order to allow threads to directly access to the array that holds particle data instead of continuously receiving and sending back large data to main thread. 

## How to use SharedArrayBuffer

Similiar to WebGPU, SharedArrayBuffer requires some extra steps to be used. Due to security reason ([Spectre](https://meltdownattack.com/)), SharedArrayBuffer has been disabled across all browsers. For Chrome, it is now possible to enable SharedArrayBuffer by following any of these conditions:

- Make website "cross-origin isolated" using HTTP headers
- Add a command line flag "--enable-features=SharedArrayBuffer" (Windows) 
- Register for a token in [Chrome Origin Trials](https://developer.chrome.com/origintrials/#/trials/active)

In my case, I was actually unable to make my local website "cross-origin isolated" since local servers do not support sending headers. Therefore, I have decided to use other conditions to enable SharedArrayBuffer. For Origin Trials, I inserted my registered token in `index.html` like following:
#### index.html
```
<meta http-equiv="origin-trial" content="INSERT_TOKEN">
```
In order to see if the SharedArrayBuffer has been enabled properly, you can open Chrome's Inspect window -> `Application` -> `Frames` -> `top`, then scroll down to find `Origin Trials`. Here, you will find `UnrestrictedSharedArrayBuffer` with "Enabled" sign if the inserted token is working properly. 

## Implementation

Implementation of SharedArrayBuffer is actually very simple. When first creating an array for particle data, I created a buffer to be used to make the particle data a shared-memory:
#### mainCPU.ts
```
var particlesBuffer = new SharedArrayBuffer(Float32Array.BYTES_PER_ELEMENT * (numParticles * 4));
var particlesData = new Float32Array(particlesBuffer);
```
After initializing a variable called `transferData` (data that will be used by Web Workers), I simply have to convert the buffer back to `Float32Array` to be used to read and write over data:
#### cpuWorker.js
```
var particlesData = new Float32Array(event.data.particlesBuffer);
```

## Performance

Unfortunately, there was actually no performance difference between using SharedArrayBuffer and previous version. (without SharedArrayBuffer) Since I can only test the simulation with less than about 1000 particles with Web Workers, I incorrectly considered an array with size 4000 is large and its transfer over to Web Workers will some cost performance value. Overall, it was a great experience for me to learn about shared-memory buffers. However, I have learned that the next focus should be on optimizing the computation process (`O(n^2)` Time Complexity). Hopefully after optimization of computation, I will be able to test with higher number of particles to see any performance difference with SharedArrayBuffer.

#### My Personal Device Info:
- CPU: Intel(R) Core(TM) i7-9700k CPU @ 3.60GHz 3.60GHz
- RAM: 16.0 GB
- GPU: NVIDIA GeForce RTX 2080
- OS: Windows 10
- System Type: 64-bit, x64-based processor

#### Useful Links:
- [Enabling Cross-Origin Isolation](https://web.dev/cross-origin-isolation-guide/#enable-cross-origin-isolation)
- [Cross-Origin Isolation Headers](https://web.dev/coop-coep/)
- [SharedArrayBuffer Doc](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#browser_compatibility)
