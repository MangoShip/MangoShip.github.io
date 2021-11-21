---
title:  "N-Body Simulation (Web Workers)"
permalink: "blog/NBodyWebWorkers"
layout: post
---
*Updated: 11-20-2021*

JavaScript Web Workers are threads that run scripts in the background. (Similiar to C++'s `thread`) I will be using Web Workers to divide up the work on particles' computations. The process is actually pretty simple:
### mainCPU.ts
```
var workerList = [];

for (let j = 0; j < numThreads; ++j) {
    var worker = new Worker('../src/cpuWorker.js');
    workerList[j] = worker;
}
```
I first create an array of workerList and fill each element with worker. I will be initalizing the workers only **once**, since initializing the workers for every frame will cost a significant value of performance. That means that these workers will get reused for every frame. These workers will get terminated when the simulation is getting reset. 

For every frame, I will get a nearly equal chunk size of work based on the number of workers. These workers will get the `startIndex` and `endIndex` to know which index in data array to start and end on. These information will get packed together and be sent over to each worker to indicate to begin working:
### mainCPU.ts
```
workerList[i].postMessage(transferData);

workerList[i].onmessage = function(event) {
  ...
}
```
### cpuWorker.js
```
self.onmessage = function(event) {
  ...
  postMessage(particlesData);
}
```
The `cpuWorker` is a worker file where it will perform some scripts in the background. It will receive a data from main thread and send back a new data to main thread once it has completed with its task. The function `.postMessage()` will be used to send data, and the function `.onmessage()` will act as a listener to receive any data. Once the main thread has received data from all the workers, it will render the particles' new locations into the canvas. 

## Measuring Performance

Similiar to WebGPU, I collected data (Average FPS for 10000 frames) with ___ particles and different number of threads:

![screenshot]()

First of all, you can see the significant difference of FPS compared to WebGPU. While WebGPU was completely fine running the simulation with more than 10000 particles, this version seems to be already struggling with ___ particles. **Insert more notes** 


## Synchronous

While I was working with Web Workers, I have received a suggestion to set up Web Workers with **synchronous** calls. Instead of calling the function `.postMessage()` to make the workers to start working, set up a `while` loop inside the workers to spin until some variable (`bool`) indicates to start working. 

After some experiments, I found out that it was **not** possible to set up the workers in a way that I wanted. However, I learned something interesting of Web Workers. When the Web Workers get initialized, they start spinning until the argument `event` from `self.onmessage = function(event) {}` get defined by the main thread. I learned when I call main thread's function `.postMessage()`, it simply defines the `event` which act as an indication for the workers to start working. Even though the functions `.postMessage()` and `.onmessage()` look like asynchronous call, there were actually some synchronous calls happening in the background.

#### My Personal Device Info:
- CPU: Intel(R) Core(TM) i7-9700k CPU @ 3.60GHz 3.60GHz
- RAM: 16.0 GB
- GPU: NVIDIA GeForce RTX 2080
- OS: Windows 10
- System Type: 64-bit, x64-based processor

#### Useful Links:
- [Web Workers Doc](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
